---
title: Handler消息机制
date: 2023-10-03
categories:
- [Android,Framework内核解析]
---
# Handler详解
在android开发中，经常会在子线程中进行一些操作，当操作完毕后会通过handler发送一些数据给主线程，通知主线程做相应的操作。探素其背后的原理：子线程 handler 主线程 其实构成了线程模型中的经典问题 生产者-消费者模型。生产者-消费者模型：生产者和消费者在同一时间段内共用同一个存储空间，生产者往存储空间中添加数据，消费者从存储空间中取走数据。

![handler_01.png](images/handler/handler_01.png)

好处： - 保证数据生产消费的顺序（通过MessageQueue,先进先出） - 不管是生产者(子线程)还是消费者（主线程）
都只依赖缓冲区（handler），生产者消费者之间不会相互持有，使他们之间没有任何耦合
源码分析：
Handler
    Handler机制的相关类
    创建Looper
        创建MessageQueue以及Looper与当前线程的绑定
    Looper.loop()
    创建Handler
    创建Message
    Message和Handler的绑定
    Handler发送消息
    Handler处理消息 

## Handler机制的相关类
Hanlder：发送和接收消息 
Looper：用于轮询消息队列，一个线程只能有一个Looper 
Message： 消息实体
MessageQueue： 消息队列用于存储消息和管理消息

## 创建Looper
```
创建Looper的方法是调用Looper.prepare() 方法
```

在ActivityThread中的main方法中为我们prepare了
```
public static void main(String[] args) {
        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ActivityThreadMain");
        //其他代码省略...
        Looper.prepareMainLooper(); //初始化Looper以及MessageQueue
        ActivityThread thread = new ActivityThread();
        thread.attach(false);
        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }
        if (false) {
            Looper.myLooper().setMessageLogging(new
            LogPrinter(Log.DEBUG, "ActivityThread"));
        }
        // End of event ActivityThreadMain.
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        Looper.loop(); //开始轮循操作

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```
Looper.prepareMainLooper();
```
    public static void prepareMainLooper() {
        prepare(false);//消息队列不可以quit
        synchronized (Looper.class) {
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            sMainLooper = myLooper();
        }   
    }
```
prepare有两个重载的方法，主要看 prepare(boolean quitAllowed) quitAllowed的作用是在创建MessageQueue时
标识消息队列是否可以销毁， **主线程不可被销毁** 下面有介绍
```
public static void prepare() {
            prepare(true);//消息队列可以quit    
    }
    //quitAllowed 主要
    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {//不为空表示当前线程已经创建了Looper
            throw new RuntimeException("Only one Looper may be created per thread");
            //每个线程只能创建一个Looper
        }
        sThreadLocal.set(new Looper(quitAllowed));//创建Looper并设置给sThreadLocal，这样get的时候就不会为null了
    }
```

### 创建MessageQueue以及Looper与当前线程的绑定
```
private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed);//创建了MessageQueue
    mThread = Thread.currentThread(); //当前线程的绑定
}
```
MessageQueue的构造方法
```
MessageQueue(boolean quitAllowed) {
//mQuitAllowed决定队列是否可以销毁 主线程的队列不可以被销毁需要传入false, 在MessageQueue的quit()方法就不贴源码了
    mQuitAllowed = quitAllowed;
    mPtr = nativeInit();
}
```
## Looper.loop()
同时是在main方法中 Looper.prepareMainLooper() 后Looper.loop(); 开始轮询
```
public static void loop() {
    final Looper me = myLooper();//里面调用了sThreadLocal.get()获得刚才创建的Looper对象
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }//如果Looper为空则会抛出异常
    final MessageQueue queue = me.mQueue;

    // Make sure the identity of this thread is that of the local process,
    // and keep track of what that identity token actually is.
    Binder.clearCallingIdentity();
    final long ident = Binder.clearCallingIdentity();

    for (;;) {
        //这是一个死循环，从消息队列不断的取消息
        Message msg = queue.next(); // might block
        if (msg == null) {享学课堂享学课堂享学课堂
            //由于刚创建MessageQueue就开始轮询，队列里是没有消息的,等到Handler sendMessage  enqueueMessage后队列里才有消息
            // No message indicates that the message queue is quitting.
            return;
        }
    // This must be in a local variable, in case a UI event sets the logger
    Printer logging = me.mLogging;
    if (logging != null) {
        logging.println(">>>>> Dispatching to " + msg.target + " " + msg.callback + ": " + msg.what);
    }
    msg.target.dispatchMessage(msg);//msg.target就是绑定的Handler，详见后面Message的部分，Handler开始
    //后面代码省略.....
    msg.recycleUnchecked();
    }
}
```

## 创建Handler
最常见的创建handler
```
Handler handler=new Handler(){
    @Override
    public void handleMessage(Message msg) {
        super.handleMessage(msg);
    }
};
```
在内部调用 this(null, false);
```
public Handler(Callback callback, boolean async) {
    //前面省略
    mLooper = Looper.myLooper();//获取Looper，**注意不是创建Looper**！
    if (mLooper == null) {
        throw new RuntimeException("Can't create handler inside thread that has not called Looper.prepare()");
    }
    mQueue = mLooper.mQueue;//创建消息队列MessageQueue
    mCallback = callback; //初始化了回调接口
    mAsynchronous = async;
}
```
Looper.myLooper()；
```
//这是Handler中定义的ThreadLocal ThreadLocal主要解多线程并发的问题
// sThreadLocal.get() will return null unless you've called prepare().
static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();

public static @Nullable Looper myLooper() {
    return sThreadLocal.get();
}
```
sThreadLocal.get() will return null unless you’ve called prepare(). 这句话告诉我们get可能返回null 除非先调用
prepare()方法创建Looper。在前面已经介绍了   
## 创建Message
可以直接new Message 但是有更好的方式 Message.obtain。因为可以检查是否有可以复用的Message,用过复用避
免过多的创建、销毁Message对象达到优化内存和性能的目地
```
public static Message obtain(Handler h) {
        Message m = obtain();//调用重载的obtain方法
        m.target = h;//并绑定的创建Message对象的handler

        return m;
    }
    public static Message obtain() {
        synchronized (sPoolSync) {//sPoolSync是一个Object对象，用来同步保证线程安全
            if (sPool != null) {//sPool是就是handler dispatchMessage 后 通过recycleUnchecked 回收用以复用的Message
                Message m = sPool;
                sPool = m.next;
                m.next = null;
                m.flags = 0; // clear in-use flag
                sPoolSize--;
                return m;
            }
        }
    return new Message();
}
```
## Message和Handler的绑定
创建Message的时候可以通过 Message.obtain(Handler h) 这个构造方法绑定。当然可以在 在Handler 中的
enqueueMessage（）也绑定了，所有发送Message的方法都会调用此方法入队，所以在创建Message的时候是可
以不绑定的
```
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this; //绑定
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
}
```
## Handler发送消息
Handler发送消息的重载方法很多，但是主要只有2种。 sendMessage(Message) sendMessage方法通过一系列重
载方法的调用，sendMessage调用sendMessageDelayed，继续调用sendMessageAtTime，继续调用
enqueueMessage，继续调用messageQueue的enqueueMessage方法，将消息保存在了消息队列中，而最终由
Looper取出，交给Handler的dispatchMessage进行处理
我们可以看到在dispatchMessage方法中，message中callback是一个Runnable对象，如果callback不为空，则直接
调用callback的run方法，否则判断mCallback是否为空，mCallback在Handler构造方法中初始化，在主线程通直接
通过无参的构造方法new出来的为null,所以会直接执行后面的handleMessage()方法
```
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {//callback在message的构造方法中初始化或者使用shandler.post(Runnable)时候才不为空
            handleCallback(msg);
        } else {
            if (mCallback != null) {//mCallback是一个Callback对象，通过无参的构造方法创建出来的handler，该属性为null，此段不执行
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
        handleMessage(msg);//最终执行handleMessage方法
    }
}
private static void handleCallback(Message message) {
    message.callback.run();
}
```
## Handler处理消息
在handleMessage(Message)方法中，我们可以拿到message对象，根据不同的需求进行处理，整个Handler机制的
流程就结束了。
## 小结
handler.sendMessage 发送消息到消息队列MessageQueue，然后looper调用自己的loop()函数带动
MessageQueue从而轮询messageQueue里面的每个Message，当Message达到了可以执行的时间的时候开始执
行，执行后就会调用message绑定的Handler来处理消息。大致的过程如下图所示

handler机制就是一个传送带的运转机制。
1）MessageQueue就像履带。
2）Thread就像背后的动力，就是我们通信都是基于线程而来的。
3）传送带的滚动需要一个开关给电机通电，那么就相当于我们的loop函数，而这个loop里面的for循环就会带着不断
的滚动，去轮询messageQueue
4）Message就是 我们的货物了。
## 难点问题
### 线程同步问题
Handler是用于线程间通信的，但是它产生的根本并不只是用于UI处理，而更多的是handler是整个app通信的框架，
大家可以在ActivityThread里面感受到，整个App都是用它来进行线程间的协调。Handler既然这么重要，那么它的
线程安全就至关重要了，那么它是如何保证自己的线程安全呢？
Handler机制里面最主要的类MessageQueue，这个类就是所有消息的存储仓库，在这个仓库中，我们如何的管理好
消息，这个就是一个关键点了。消息管理就2点：1）消息入库（enqueueMessage），2）消息出库（next），所以
这两个接口是确保线程安全的主要档口。
enqueueMessage 源码如下：
```
boolean enqueueMessage(Message msg, long when) {
    if (msg.target == null) {
        throw new IllegalArgumentException("Message must have a target.");
    }
    if (msg.isInUse()) {
        throw new IllegalStateException(msg + " This message is already in use.");
    }
    // 锁开始的地方
    synchronized (this) {
        if (mQuitting) {
            IllegalStateException e = new IllegalStateException(msg.target + " sending message to a Handler on a dead thread");
            Log.w(TAG, e.getMessage(), e);
            msg.recycle();
            return false;
        }
        msg.markInUse();
        msg.when = when;
        Message p = mMessages;
        boolean needWake;
        if (p == null || when == 0 || when < p.when) {
            // New head, wake up the event queue if blocked.
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;
        } else {
            // Inserted within the middle of the queue. Usually we don't have to wake
            // up the event queue unless there is a barrier at the head of the queue
            // and the message is the earliest asynchronous message in the queue.
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            msg.next = p; // invariant: p == prev.next
            prev.next = msg;
        }
        // We can assume mPtr != 0 because mQuitting is false.
        if (needWake) {
            nativeWake(mPtr);
        }
    }  
}
//锁结束的地方
```
synchronized锁是一个内置锁，也就是由系统控制锁的lock unlock时机的。在多线程的课程中我们有详细分析过，
有问题的同学可以去研究一下。
```
synchronized (this) 
```
这个锁，说明的是对所有调用同一个MessageQueue对象的线程来说，他们都是互斥的，然而，在我们的Handler里
面，一个线程是对应着一个唯一的Looper对象，而Looper中又只有一个唯一的MessageQueue（这个在上文中也有
介绍）。所以，我们主线程就只有一个MessageQueue对象，也就是说，所有的子线程向主线程发送消息的时候，
主线程一次都只会处理一个消息，其他的都需要等待，那么这个时候消息队列就不会出现混乱。

......未完待续......