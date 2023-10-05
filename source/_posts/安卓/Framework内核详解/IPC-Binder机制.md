---
title: IPC Binder机制
date: 2023-10-04
categories:
- [Android,Framework内核解析]
---

# binder驱动
学习binder的时候，一定要牢记这张图的层级结构，这样在讲解的时候，你才能知道讲解的是哪块内容。
![image](images/binder_driver_files/binder1.png)

这块我们主要掌握四个方法：

1.  binder\_init：初始化binder驱动
2.  binder\_open：打开binder驱动
3.  binder\_mmap：设置内存映射
4.  binder\_ioctl：数据读写操作

其中 binder\_init 方法是在idle初始化的时候就会调用：

```c
// kernel/common/drivers/android/binder.c
// 设备驱动入口函数
device_initcall(binder_init);
```

其他三个方法与应用层的调用关系如下：

```c
// kernel/common/drivers/android/binder.c
const struct file_operations binder_fops = {
    ... ...
    .unlocked_ioctl = binder_ioctl,
    .mmap = binder_mmap,
    .open = binder_open,
    ... ...
};
```

其中 unlocked\_ioctl 其实就是 ioctl。所以我们应用层实际调用的是下面三个方法：
![58201170.png](images/binder_driver_files/58201170.png)
接下来我们一个个分析binder驱动的这几个方法，知道这几个方法的作用后，我们在应用层调用对应方法时就知道在干什么了。

## binder\_init--驱动设备的初始化

binder\_init 主要是binder驱动的初始化，代码如下：

```c
// kernel/common/drivers/android/binder.c
static int __init binder_init(void)
{
    ... ... 
    // 初始化 binder 设备
    ret = init_binder_device(device_name);
    ... ...
}
```

接着我们来看下 init\_binder\_device 方法：

```c
static int __init init_binder_device(const char *name)
{
    ... ....
    // 为binder设备分配内存
    binder_device = kzalloc(sizeof(*binder_device), GFP_KERNEL);
    if (!binder_device)
        return -ENOMEM;
    ... ...
    // 注册 Binder 驱动
    ret = misc_register(&binder_device->miscdev);
    ... ...
    // 返回注册的驱动    
    return ret;
}
```

总结：

1.  为binder驱动分配内存
2.  注册binder驱动

## binder\_open--打开binder驱动

binder\_open就是打开binder驱动，这个在进程创建的时候就会执行，进程都是由zygote创建的，这个过程我们之前已经介绍了，如果对于zygote启动流程不清楚的，可以再去看下那块的流程，这里不再重复介绍了，下来我们只分析相关的代码：
进程创建时会调用 onZygoteInit 方法去创建 ProcessState对象：

### open的执行

```c
// frameworks/base/cmds/app_process/app_main.cpp
virtual void onZygoteInit()
{
    // 初始化 ProcessState
    sp<ProcessState> proc = ProcessState::self();
    ... ...
}



// frameworks/native/libs/binder/ProcessState.cpp
sp<ProcessState> ProcessState::self()
{
    ... ...
    // 单例模式，如果有则直接返回
    if (gProcess != nullptr) {
        return gProcess;
    }
    // 没有才创建
    gProcess = new ProcessState(kDefaultDriver);
    // 返回
    return gProcess;
}
```

ProcessState::self 这个方法主要做了两件事：

1.  通过单例模式，让每个进程只有一个 ProcessState 对象。
2.  传入了一个参数，这个参数就是 ProcessState 准备启动的驱动路径。

    const char\* kDefaultDriver = "/dev/binder";

然后我们接着看下 ProcessState 的构造方法

```c
ProcessState::ProcessState(const char *driver)
    : mDriverName(String8(driver))
    , mDriverFD(open_driver(driver))
    , ... ...
{
    ... ...
    if (mDriverFD >= 0) {
        // 调用mmap，实现内存映射，最终调用到内核的 binder_mmap 方法
        mVMStart = mmap(nullptr, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0);
        ... ...
    }
    ... ...
}



static int open_driver(const char *driver)
{
    // 因为 Linux 一切皆文件，所以打开驱动，就跟打开一个文件一样
    // 最终调用到内核的 binder_open 方法
    int fd = open(driver, O_RDWR | O_CLOEXEC);
    ... ...
    return fd;
}
```

### binder\_open方法

```c
// kernel/common/drivers/android/binder.c
static int binder_open(struct inode *nodp, struct file *filp)
{
    struct binder_proc *proc;
    ... ...
    // 为 binder_proc结构体 在 kernel分配内存空间
    proc = kzalloc(sizeof(*proc), GFP_KERNEL);
    ... ...
    // 初始化 pro 结构体
    INIT_LIST_HEAD(&proc->todo);
    init_waitqueue_head(&proc->freeze_wait);
    ... ...
    // 将 binder_proc与 filp关联起来，这样下次通过 filp 就能找到这个 proc了
    filp->private_data = proc;
    ... ...
    // 将 proc_node 节点添加到 binder_procs 的队列头部
    hlist_add_head(&proc->proc_node, &binder_procs);
    ... ...
}
```

binder\_open方法主要做了以下几件事：

1.  创建binder\_proc对象。
2.  把当前进程等信息保存到binder\_proc对象。（该对象管理 IPC 所需的各种信息并拥有其他结构体的根结构体）
3.  把binder\_proc对象保存到文件指针filp，这个文件指针指向的就是 /dev/binder。
4.  把binder\_proc加入到全局链表binder\_procs。

其实说到底，binder\_open就做了一件事，初始化 binder\_proc。
那么binder\_proc是什么呢？可能很多同学不清楚，binder是不是要和很多进程通信？那它怎么区分不同的进程呢？每个进程是不是有自己的不同需求，那它怎么保存呢？其实这些就是通过binder\_proc 这个结构体来保存的，所以一个 binder\_proc 结构体对应一个进程，保存这个进程的信息，例如，pid、要处理的事务。然后所有进程的 binder\_proc 又是以一个双向链表保存的。如下图：
![72865820.png](https://lingzhiwen.github.io/images/binder_driver_files/72865820.png)

## binder\_mmap--映射

```c
// kernel/common/drivers/android/binder.c
static int binder_mmap(struct file *filp, struct vm_area_struct *vma)
{
    // 通过文件指针拿到对应进程的 binder_proc 结构体
    struct binder_proc *proc = filp->private_data;
    ... ... 
    return binder_alloc_mmap_handler(&proc->alloc, vma);
}



int binder_alloc_mmap_handler(struct binder_alloc *alloc,
                  struct vm_area_struct *vma)
{
    ... ...
    // 如果传入的大小大于4M，保证映射内存大小不超过 4M，此处为（1M-8K）
    alloc->buffer_size = min_t(unsigned long, vma->vm_end - vma->vm_start,
                   SZ_4M);
   ... ...
    // 将 proc中的 buffer指针指向这块虚拟内存
    alloc->buffer = (void __user *)vma->vm_start;
    // 分配物理内存
    alloc->pages = kcalloc(alloc->buffer_size / PAGE_SIZE,
                   sizeof(alloc->pages[0]),
                   GFP_KERNEL);
    ... ...
    // 如果是异步，大小为同步的一半
    alloc->free_async_space = alloc->buffer_size / 2;
    // 把用户空间的虚拟地址保存在alloc里
    // alloc 就是 binder_proc 结构体的一个成员，这样我们就可以随时拿到 vma
    binder_alloc_set_vma(alloc, vma);
    ... ...
}
```

![74579249.png](images/binder_driver_files/74579249.png)
总结：binder\_mmap 做的事情就是，拿到指向用户空间和内核空间的共享内存的指针，并将他们同时指向 同一个物理内存。
我们使用这个共享内存的时候，因为这两个指针都在 binder\_proc 结构体中，所以随时可以拿到。
又因为 指向同一个物理内存，所以内核向指针buffer中存入数据，用户从指针 vma 就可以取出相同数据。

## binder\_ioctl--数据读写操作

客户端向服务端发送数据时，在用户层会执行 ioctl，对应的会执行到内核层的 binder\_ioctl。
![56256675.png](images/binder_driver_files/56256675.png)

    // kernel/common/drivers/android/binder.c
    static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
    {
        // 有很多命令，主要需要关注下面两个
        switch (cmd) {
        // 读写操作--使用频率最高
        case BINDER_WRITE_READ:
            ret = binder_ioctl_write_read(filp, cmd, arg, thread);
            ... ...
            break;
        // 设置 servicemanager 为全局上下文，这个我们介绍servicemanager时，再具体介绍
        case BINDER_SET_CONTEXT_MGR_EXT: {
            ... ...
            break;
        }
        ... ...
    }

这里我们主要分析下 binder\_ioctl\_write\_read 方法。

### binder\_ioctl\_write\_read

    static int binder_ioctl_write_read(struct file *filp,
                    unsigned int cmd, unsigned long arg,
                    struct binder_thread *thread)
    {
        // 将用户空间数据 ubuf 拷贝到 bwr
        if (copy_from_user(&bwr, ubuf, sizeof(bwr))) {
            ... ...
        }
       ... ...
        // 如果写入数据大于0，执行写入方法
        if (bwr.write_size > 0) {
            // 这个方法就会根据不同命令，去执行具体代码
            ret = binder_thread_write(proc, thread,
                          bwr.write_buffer,
                          bwr.write_size,
                          &bwr.write_consumed);
            ... ...
        }
        // 如果读取数据大于0，执行读取方法
        if (bwr.read_size > 0) {
            // 这个方法就会根据不同命令，去执行具体代码
            ret = binder_thread_read(proc, thread, bwr.read_buffer,
                         bwr.read_size,
                         &bwr.read_consumed,
                         filp->f_flags & O_NONBLOCK);
            ... ...
        }
        ... ...
    }

可以看出这里面主要就是调用了 binder\_thread\_write 和 binder\_thread\_read 方法。


# 启动ServiceManager

## 启动ServiceManager进程

ServiceManager是由init进程通过解析init.rc文件而创建的，其所对应的可执行程序servicemanager，所对应的源文件是service\_manager.c，进程名为servicemanager。

```c
// system/core/rootdir/init.rc
// 114
on init
    // 395
    start servicemanager
```

启动ServiceManager的入口函数是`service_manager.c`中的main()方法。
解析文件 `frameworks/native/cmds/servicemanager/servicemanager.rc` 执行 servicemanager

```c
service servicemanager /system/bin/servicemanager
    class core animation
```

通过 Android.bp 文件可知，servicemanager 的源码是 `main.cpp`

```c
// frameworks/native/cmds/servicemanager/Android.bp
// 32
cc_binary {
    name: "servicemanager",
    defaults: ["servicemanager_defaults"],
    init_rc: ["servicemanager.rc"],
    srcs: ["main.cpp"],
}
```

![64706269.png](https://lingzhiwen.github.io/images/servicemanager_files/64706269.png)

ServiceManager 启动的时候主要做了三件事：

1.  打开binder驱动
2.  成为上下文的管理者，这样所有的进程都可以获取到它
3.  循环等待服务的注册

## <main@main.cpp>

```c
// frameworks/native/cmds/servicemanager/main.cpp
int main(int argc, char** argv) {
    const char* driver = argc == 2 ? argv[1] : "/dev/binder";
    // 1.打开binder驱动
    sp<ProcessState> ps = ProcessState::initWithDriver(driver);
    ... ...
    // 2.成为上下文的管理者，这样所有的进程都可以获取到它
    ps->becomeContextManager(nullptr, nullptr);
    ... ...
    // 3.将binder回调设置到 looper中
    BinderCallback::setupTo(looper);
    // 4.循环处理服务的注册，例如AMS、WMS等的注册
    while(true) {
        looper->pollAll(-1);
    }
    ... ...
}
```

### ProcessState::initWithDriver--打开binder驱动

```c
// frameworks/native/libs/binder/ProcessState.cpp
sp<ProcessState> ProcessState::initWithDriver(const char* driver)
{
    // 初始化 ProcessState 对象，这个里面就会调用到binder_open方法，打开驱动
    gProcess = new ProcessState(driver);
}



ProcessState::ProcessState(const char *driver)
    : mDriverName(String8(driver))
    , mDriverFD(open_driver(driver))
    , ... ...
{
    ... ...
    if (mDriverFD >= 0) {
        // 调用mmap，实现内存映射，最终调用到内核的 binder_mmap 方法
        mVMStart = mmap(nullptr, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0);
        ... ...
    }
    ... ...
}
```

### ps->becomeContextManager

成为上下文的管理者，这样所有的进程都可以获取到它。

```c
bool ProcessState::becomeContextManager(context_check_func checkFunc, void* userData)
{
    ... ...
    // 通过ioctl，传递 BINDER_SET_CONTEXT_MGR_EXT 指令
    int result = ioctl(mDriverFD, BINDER_SET_CONTEXT_MGR_EXT, &obj);
    ... ...
}
```

上面方法通过调用ioctl，实际就会进入binder\_ioctl，然后执行命令 BINDER\_SET\_CONTEXT\_MGR\_EXT。

```c
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
    ... ...
    switch (cmd) {
    ... ...
    case BINDER_SET_CONTEXT_MGR_EXT: {
        ... ...
        ret = binder_ioctl_set_ctx_mgr(filp, &fbo);
        ... ...
    }
    ... ...
}
```

binder\_ioctl\_set\_ctx\_mgr 方法主要做了两件事：

1.  为servicemanager 绑定 uid，也就是我们在binder直播课中说的，binder会为服务绑定uid，所以比传统的IPC机制更安全。
2.  为 servicemanager 创建 binder\_node 结构体。

#### 什么是binder\_node结构体呢？

binder\_node 结构体就是 servicemanager 在binder 驱动层的binder 实体对象。
我们可以根据下图理解：
![8a7025a8-cb1a-4227-93c4-82d5faf9e74f.jpeg](images/servicemanager_files/8a7025a8-cb1a-4227-93c4-82d5faf9e74f.jpeg)
所有服务端在用户进程都有个binder实体，在驱动层也会对应有一个binder实体，这样binder驱动和服务端就会一一对应。
然后binder驱动根据binder实体会对应生成一个binder的代理对象，这样binder驱动中，binder的实体和代理对象也一一对应了。
最后客户端就可以根据这个binder代理对象，最终找到对应的binder实体对象。

### BinderCallback::setupTo(looper)--循环等待服务注册

```c
// frameworks/native/cmds/servicemanager/main.cpp
class BinderCallback : public LooperCallback {
public:
    static sp<BinderCallback> setupTo(const sp<Looper>& looper) {
        ... ...
        // 写入准备执行的命令：BC_ENTER_LOOPER
        IPCThreadState::self()->setupPolling(&binder_fd);
        ... ...
        // 执行 BC_ENTER_LOOPER，并进入等待
        IPCThreadState::self()->flushCommands();
        ... ...
    }
};
```

#### IPCThreadState::self()->setupPolling

```c
// frameworks/native/libs/binder/IPCThreadState.cpp
int IPCThreadState::setupPolling(int* fd)
{
    ... ...
    // 写入准备执行的命令：BC_ENTER_LOOPER
    mOut.writeInt32(BC_ENTER_LOOPER);
... ...
}
```

#### IPCThreadState::self()->flushCommands

```c
void IPCThreadState::flushCommands()
{
    ... ...
    talkWithDriver(false);
    ... ...
}
```

talkWithDriver 方法就会执行 ioctl。

```c
status_t IPCThreadState::talkWithDriver(bool doReceive)
{
    ... ...
    // 执行 ioctl
    if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0)
    ... ...
}
```

进入binder驱动后，接着就会执行 binder\_thread\_write 和 binder\_thread\_read 方法。
binder\_thread\_write 主要是设置 状态。

```c
static int binder_thread_write(struct binder_proc *proc,
            struct binder_thread *thread,
            binder_uintptr_t binder_buffer, size_t size,
            binder_size_t *consumed)
{
    switch (cmd) {
        ... ...
        case BC_ENTER_LOOPER:
            // 设置转态为 BINDER_LOOPER_STATE_ENTERED，准备进入循环
            thread->looper |= BINDER_LOOPER_STATE_ENTERED;
            break;
    ... ...
}
```

binder\_thread\_read 则让servicemanager 进入等待。

```c
static int binder_thread_read(struct binder_proc *proc,
                  struct binder_thread *thread,
                  binder_uintptr_t binder_buffer, size_t size,
                  binder_size_t *consumed, int non_block)
{
    ... ...
    // 阻塞，等待其他服务注册
    ret = binder_wait_for_work(thread, wait_for_proc_work);
    ... ...
}
```

接下来就是等待客户端来注册，当然这儿所说的客户端就是 AMS、WMS等服务，相对于servicemanager来说，一切皆是客户端。


# 获取ServiceManager对象

系统服务向servicemanager注册或者我们应用向servicemanager去获取系统服务时，都需要先获取 servicemanager的代理对象，那么我们这个视频就来介绍，如何获取 servicemanager的代理对象。
下面我们以AMS注册为例，在 systemserver 启动时，就会调用 AMS 的 setSystemProcess 方法，AMS就是通过该方法向servicemanager注册的。如果对 systemserver启动来不了解的同学，可以先去学习 systemserver 的启动流程。

```java
// frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
public void setSystemProcess() {
    ... ... 
    // 将 AMS 注册到 servicemanager 中
    ServiceManager.addService(Context.ACTIVITY_SERVICE, this, /* allowIsolated= */ true,
    ... ...
}
```

```java
// frameworks/base/core/java/android/os/ServiceManager.java
public static void addService(String name, IBinder service, boolean allowIsolated,
            int dumpPriority) {
    getIServiceManager().addService(name, service, allowIsolated, dumpPriority);
    ... ...
}
```

其中 getIServiceManager 方法就是获取 servicemanager 的代理对象，也是我们这节课要分析的，addService 方法我们后续再分析。

## getIServiceManager

```java
// frameworks/base/core/java/android/os/ServiceManager.java
private static IServiceManager getIServiceManager() {
    ... ...
    sServiceManager = ServiceManagerNative
                .asInterface(Binder.allowBlocking(BinderInternal.getContextObject()));
    ... ...
}
```

ServiceManagerNative.asInterface(Binder.allowBlocking(BinderInternal.getContextObject()));这个方法我们又需要分为两块分析，一个是方法体，一个是参数，如下：

1.  ServiceManagerNative.asInterface 是如何返回 servicemanager 的代理对象的
2.  Binder.allowBlocking(BinderInternal.getContextObject()) 这个方法参数到底是什么呢？

我们首先分析方法的参数到底是什么？再来分析方法体。

## Binder.allowBlocking(BinderInternal.getContextObject())

Binder.allowBlocking 这个方法很简单，就是返回它的参数，所以Binder.allowBlocking(BinderInternal.getContextObject())具体返回什么，就只需要看 BinderInternal.getContextObject() 就行了。
BinderInternal.getContextObject() 实际是一个 native 方法，所以我们直接找到实际方法，如下:

```java
// frameworks/base/core/jni/android_util_Binder.cpp
static jobject android_os_BinderInternal_getContextObject(JNIEnv* env, jobject clazz)
{
    // 通过 ProcessState 的方法，创建 BpBinder 对象，并返回
    sp<IBinder> b = ProcessState::self()->getContextObject(NULL);
    // 创建 BinderProxy对象并返回，并将 BinderProxy 和 BpBinder 绑定
    return javaObjectForIBinder(env, b);
}
```

### ProcessState::self()->getContextObject

我们先来分析 ProcessState::self()->getContextObject 方法。

```java
// frameworks/native/libs/binder/ProcessState.cpp
sp<IBinder> ProcessState::getContextObject(const sp<IBinder>& /*caller*/)
{
    // 获取 handle 为 0 的代理对象
    sp<IBinder> context = getStrongProxyForHandle(0);
    ... ...
    return context;
}
```

    sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle)
    {
        ... ...
        b = BpBinder::create(handle);
        ... ...
    }

这个地方我们需要注意，创建 BpBinder对象时，传入的handle为0，就像门牌号一样，这个是规定的，为0 就表示找的是 servicemanager 的代理对象。

### javaObjectForIBinder

```java
// frameworks/base/core/jni/android_util_Binder.cpp
jobject javaObjectForIBinder(JNIEnv* env, const sp<IBinder>& val)
{
    ... ...
    // 实际就是执行 BinderProxy的 getInstance方法
    jobject object = env->CallStaticObjectMethod(gBinderProxyOffsets.mClass,
            gBinderProxyOffsets.mGetInstance, (jlong) nativeData, (jlong) val.get());
    ... ...
    return object;
}
```

```java
// frameworks/base/core/java/android/os/BinderProxy.java
private static BinderProxy getInstance(long nativeData, long iBinder) {
    ... ...
    // 创建 BinderProxy 对象
    result = new BinderProxy(nativeData);
    ... ...
    // 将 BinderProxy 和 iBinder 绑定，iBinder实际就是 BpBinder
    sProxyMap.set(iBinder, result);
    ... ...
    // 返回 BinderProxy
    return result;
}
```

### 总结

通过分析可知，Binder.allowBlocking(BinderInternal.getContextObject()) 实际就是创建了 BpBinder 和 BinderProxy，并将二者关联，然后返回 BinderProxy 对象。

## ServiceManagerNative.asInterface

我们再来分析 ServiceManagerNative.asInterface 方法：

```java
// frameworks/base/core/java/android/os/ServiceManagerNative.java
public static IServiceManager asInterface(IBinder obj) {
    ... ...
    // 返回 ServiceManagerProxy 对象，obj 为 BinderProxy 对象
    return new ServiceManagerProxy(obj);
}
```

到这儿我们就拿到了每一个层级的 servicemanager 的代理对象，如下图：
![77485381.png](https://lingzhiwen.github.io/images/%E8%8E%B7%E5%8F%96servicemanager%E5%AF%B9%E8%B1%A1_files/77485381.png)
应用层为：ServiceManagerProxy
Framework层为：BinderProxy
native层为：BpBinder
内核层代理为：binder\_ref 结构体
内核层实体为：binder\_node 结构体


# 注册服务

我们以AMS注册为例，在 systemserver 启动时，就会调用 AMS 的 setSystemProcess 方法，AMS就是通过该方法向servicemanager注册的。如果对 systemserver启动还不了解的同学，可以先去学习 systemserver 的启动流程。

```c
// frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
public void setSystemProcess() {
    ... ... 
    // 将 AMS 注册到 servicemanager 中
    ServiceManager.addService(Context.ACTIVITY_SERVICE, this, /* allowIsolated= */ true,
    ... ...
}
```

```c
// frameworks/base/core/java/android/os/ServiceManager.java
public static void addService(String name, IBinder service, boolean allowIsolated,
            int dumpPriority) {
    getIServiceManager().addService(name, service, allowIsolated, dumpPriority);
    ... ...
}
```

getIServiceManager 我们前一节课已经分析了，可以理解为执行了 new ServiceManagerProxy(参数为BinderProxy对象);
我们在看 addService 方法前，我们先来看下 ServiceManagerProxy 的构造方法做了什么。

```java
// frameworks/base/core/java/android/os/ServiceManagerNative.java
public ServiceManagerProxy(IBinder remote) {
    mRemote = remote;
    mServiceManager = IServiceManager.Stub.asInterface(remote);
}
```

可以看到这个地方使用了 AIDL，根据AIDL的理解还是可以知道，实际就是返回了一个Proxy对象，然后 remote = BinderProxy。
接着我们再来看 addService 方法：

```java
// frameworks/base/core/java/android/os/ServiceManagerNative.java
public void addService(String name, IBinder service, boolean allowIsolated, int dumpPriority)
            throws RemoteException {
    mServiceManager.addService(name, service, allowIsolated, dumpPriority);
}
```

这个方法就会执行到 AIDL对应的 Proxy.addService 方法，根据我们前面介绍的AIDL，我们可以知道生成的AIDL代码，如下：
这个方法里面就会执行 mRemote.transact，也就是执行 BinderProxy.transact。所以接着我们来分析下 BinderProxy.transact 方法。

```java
public void addService(String name, IBinder service, boolean allowIsolated, int dumpPriority) throws RemoteException {
    Parcel data = Parcel.obtain();
    Parcel reply = Parcel.obtain();
    ... ...
    // 此处 service == AMS
    data.writeStrongBinder(service);
    ... ...
    // 实际就是执行 BinderProxy.transact
    mRemote.transact(ADD_SERVICE_TRANSACTION, data, reply, 0);
    ... ...
}
```

接下来我们分别分析 `data.writeStrongBinder(service);` 和 `mRemote.transact` 这两个方法。

## data.writeStrongBinder

这个方法实际就是调用一个native方法，最终进入jni层，代码如下：

```java
// frameworks/base/core/jni/android_os_Parcel.cpp
static void android_os_Parcel_writeStrongBinder(JNIEnv* env, jclass clazz, jlong nativePtr, jobject object)
{
    ... ...
    // 创建  JavaBBinder 对象，并保存在本地
    // 下面分别介绍这个方法和参数
    const status_t err = parcel->writeStrongBinder(ibinderForJavaObject(env, object));
    ... ...
}
```

### 参数：ibinderForJavaObject

这个方法的作用是创建 JavaBBinder 对象

```java
// frameworks/base/core/jni/android_util_Binder.cpp
sp<IBinder> ibinderForJavaObject(JNIEnv* env, jobject obj)
{
    ... ...
    // 获取 JavaBBinderHolder 对象
    JavaBBinderHolder* jbh = (JavaBBinderHolder*)
            env->GetLongField(obj, gBinderOffsets.mObject);
    // 获取 JavaBBinder 对象
    return jbh->get(env, obj);
    ... ...
}
```

```c
// frameworks/base/core/jni/android_util_Binder.cpp
sp<JavaBBinder> get(JNIEnv* env, jobject obj)
{
    ... ...
    // 创建 JavaBBinder对象
    b = new JavaBBinder(env, obj);
    ... ...
    // 返回 JavaBBinder 对象
    return b;
}
```

### 方法：parcel->writeStrongBinder

```c
// 参数为 JavaBBinder
status_t Parcel::writeStrongBinder(const sp<IBinder>& val)
{
    return flattenBinder(val);
}
```

```c
// 参数为 JavaBBinder
status_t Parcel::flattenBinder(const sp<IBinder>& binder)
{
    ... ...
    // 通过 JavaBBinder 获取 BBinder
    BBinder *local = binder->localBinder();
    ... ...
}
```

## BinderProxy.transact

```c
public boolean transact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
    ... ...
    // 这是一个native方法
    return transactNative(code, data, reply, flags);
}
```

transactNative 是一个 native 方法，会执行 android\_util\_Binder.cpp 中的 android\_os\_BinderProxy\_transact 方法：

```c
// frameworks/base/core/jni/android_util_Binder.cpp
static jboolean android_os_BinderProxy_transact(JNIEnv* env, jobject obj,
        jint code, jobject dataObj, jobject replyObj, jint flags) // throws RemoteException
{
    ... ...
    // 获取 BpBinder 对象
    IBinder* target = getBPNativeData(env, obj)->mObject.get();
    ... ...
    // 执行 BpBinder.transact
    status_t err = target->transact(code, *data, reply, flags);
    ... ...
}
```

### BpBinder.transact

```c
// frameworks/native/libs/binder/BpBinder.cpp
status_t BpBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    status_t status = IPCThreadState::self()->transact(
            mHandle, code, data, reply, flags);
}
```

```c
// frameworks/native/libs/binder/IPCThreadState.cpp
status_t IPCThreadState::transact(int32_t handle,
                                  uint32_t code, const Parcel& data,
                                  Parcel* reply, uint32_t flags)
{
    // 写入命令 BC_TRANSACTION
    err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, nullptr);
    // 类似于网络请求中的 request，最终会执行 ioctl 方法
    err = waitForResponse(reply);
}
```

根据我们前面课程的内容，可以知道，ioctl最终会调用到两个方法，一个是 binder\_thread\_write，一个是 binder\_thread\_read，如下图：
![f0bb73ba-2d8f-4739-8093-cfe61f6c29f8.png](https://lingzhiwen.github.io/images/%E6%9C%8D%E5%8A%A1%E7%9A%84%E6%B3%A8%E5%86%8C%E5%92%8C%E8%8E%B7%E5%8F%96_files/f0bb73ba-2d8f-4739-8093-cfe61f6c29f8.png)
这两个方法就是根据不同的命令，也叫协议，来进行不同的处理的，一次完整的通信协议如下图：

![4d20efa9-d0b9-4d0c-a371-0b5e35b96b6b.jpeg](https://lingzhiwen.github.io/images/%E6%9C%8D%E5%8A%A1%E7%9A%84%E6%B3%A8%E5%86%8C%E5%92%8C%E8%8E%B7%E5%8F%96_files/4d20efa9-d0b9-4d0c-a371-0b5e35b96b6b.jpeg)
Binder协议包含在IPC数据中，分为两类:

1.  BINDER\_COMMAND\_PROTOCOL：binder请求码，以”BC\_“开头，简称BC码，用于从IPC层传递到Binder Driver层；
2.  BINDER\_RETURN\_PROTOCOL ：binder响应码，以”BR\_“开头，简称BR码，用于从Binder Driver层传递到IPC层；

上面我们已经写入了 BC\_TRANSACTION 命令，然后通过 binder\_thread\_write 方法，就会将这个命令发给 binder驱动处理，binder驱动 会先返回一个 BR\_TRANSACTION\_COMPLETE 命令，让 AMS客户端 挂起，然后将客户端发过来的数据放到内核和服务端的内存共享区域，这样服务端就也能获取到数据了，然后通过 binder\_thread\_read 方法，向 servicemanager服务端 发送 BR\_TRANSACTION 命令，唤醒 servicemanager服务端，servicemanager服务端就会处理该命令，执行如下代码：

```c
// frameworks/native/libs/binder/IPCThreadState.cpp
status_t IPCThreadState::executeCommand(int32_t cmd)
{
    ... ...
    switch ((uint32_t)cmd) {
        ... ...
        case BR_TRANSACTION:
        {
            ... ...
            // the_context_object 就是 ServiceManager
            error = the_context_object->transact(tr.code, buffer, &reply, tr.flags);
            ... ...
        }
        break;
    ... ...
}
```

ServiceManager.transact 方法最终会调用到 ServiceManager 的 addService 方法，如下：

```c
frameworks/native/cmds/servicemanager/ServiceManager.cpp
Status ServiceManager::addService(const std::string& name, const sp<IBinder>& binder, bool allowIsolated, int32_t dumpPriority) {
    ... ...
    // 将服务按名称 保存在 mNameToService 中
    mNameToService[name] = Service {
        .binder = binder,
        .allowIsolated = allowIsolated,
        .dumpPriority = dumpPriority,
        .debugPid = ctx.debugPid,
    };
    ... ...
}
```

# 获取服务

获取服务的流程与添加服务的流程类似，这里就不再赘述了，最终会调用 ServiceManager 的 getService 方法，如下：

```c
// frameworks/native/cmds/servicemanager/ServiceManager.cpp
Status ServiceManager::getService(const std::string& name, sp<IBinder>* outBinder) {
    *outBinder = tryGetService(name, true);
    ... ...
}
```

```c
sp<IBinder> ServiceManager::tryGetService(const std::string& name, bool startIfNotFound) {
    ... ...
    sp<IBinder> out;
    ... ...
    // 根据名称，找到对应的服务，然后返回
    if (auto it = mNameToService.find(name); it != mNameToService.end()) {
        service = &(it->second);
        ... ...
        out = service->binder;
    }
    ... ...
    return out;
}
```

# 总结

![f9e0a58d-3762-4424-b372-17edb50b688e.png](https://lingzhiwen.github.io/images/%E6%9C%8D%E5%8A%A1%E7%9A%84%E6%B3%A8%E5%86%8C%E5%92%8C%E8%8E%B7%E5%8F%96_files/f9e0a58d-3762-4424-b372-17edb50b688e.png)
接下来我们总结下binder的整个流程：

1.  servicemanager 启动时，会将自己设置为大管家，这样所有的客户端都可以通过handle为0，找到servicemanager服务。
2.  AMS注册服务，首先AMS会获取servicemanager的binder代理对象，然后将自己的binder代理传给servicemanager。
3.  用户APP获取AMS服务，首先会获取servicemanager的binder代理对象，然后通过servicemanager拿到AMS的binder代理对象。然后用户APP就可以通过AMS这个代理对象与AMS进行通信了。


