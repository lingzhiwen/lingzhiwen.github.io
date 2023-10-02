---
title: Android源码编译
categories: 命令行
---

**一、java环境配置**
```
udo apt-get install openjdk-8-jdk
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
export PATH=$JAVA_HOME/bin:$PATH
export CLASSPATH=.:$JAVA_HOME/lib:$JAVA_HOME/lib/tools.jar
java -version
```

**二、依赖库安装**

```
sudo apt-get install git  gnupg flex bison gperf build-essential zip curl zlib1g-dev gcc-multilib g++-multilib libc6-dev-i386 lib32ncurses-dev x11proto-core-dev libx11-dev lib32z1-dev libgl1-mesa-dev libxml2-utils xsltproc unzip python libtinfo5 libncurses5-dev libncurses5 u-boot-tools
```




**三、问题集锦**

- Ensure Jack service is installed and started
```
1.从/etc/java-8-openjdk/security/java.security中找到jdk.tls.disabledAlgorithms取消TLSv1, TLSv1.1 禁用;

2.检查jack 服务是否启动
cd prebuilts/sdk/tools
./jack-admin start-server/stop-server/kill-server
```



- lex:aidl<=system/tools/aidl/aidl_language_I.II FAILED:/

```
export LC_ALL=C
```

**四、framework.jar的位置**

```
out/target/common/obj/JAVA_LIBRARIES/framework_intermediates/classes.jar
```
