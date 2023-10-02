---
title: Adb命令
categories: 命令行
---

#### 1. 常用命令：
```
adb devices #查看连接设备

adb -s cf123456 shell # 指定连接设备使用命令

adb install test.apk # 安装应用

adb install -r demo.apk #安装apk 到sd 卡：

adb uninstall cn.com.test.mobile #卸载应用，需要指定包

adb uninstall -k cn.com.test.mobile #卸载app 但保留数据和缓存文件

adb shell pm list packages #列出手机装的所有app 的包名

adb shell pm list packages -3 #列出除了系统应用的第三方应用包名

adb shell pm clear cn.com.test.mobile #清除应用数据与缓存

adb shell am start -ncn.com.test.mobile/.ui.SplashActivity #启动应用

adb shell dumpsys package #包信息Package Information

adb shell dumpsys meminfo #内存使用情况Memory Usage

adb shell am force-stop cn.com.test.mobile #强制停止应用

adb logcat #查看日志

adb logcat -c #清除log 缓存

adb logcat *:W #过滤warn日志

adb reboot #重启

adb shell top -s 10 #查看占用内存前10 的app

adb push <local> <remote> #从本地复制文件到设备

adb pull <remote> <local> #从设备复制文件到本地

adb bugreport #查看bug 报告

adb help #查看ADB 帮助
```

#### 2.查看版本信息&应用信息
```
adb shell getprop ro.build.version.release  查看Android 系统版本

adb shell getprop ro.product.model  查看设备型号

adb shell dumpsys battery   电池状况

adb get-serialno    获取设备序列号

adb shell wm size   查看屏幕分辨率

adb shell wm density    查看屏幕密度

adb shell settings get secure android_id  查看android_id

adb shell -> su -> service call iphonesubinfo 1    查看IMEI

adb shell cat /sys/class/net/wlan0/address  Mac地址,设备不同可能地址不同

adb shell cat /proc/cpuinfo     CPU 信息

adb shell cat /proc/meminfo     内存信息

adb shell cat /system/build.prop    更多硬件与系统属性

adb shell ps    查看进程

adb shell pm list packages  列出手机装的所有app 的包名

adb shell pm list packages -s   列出系统应用的所有包名

adb shell pm list packages -3   列出除了系统应用的第三方应用包名

adb shell pm list packages | find "test" win    列出手机装带有的test的包

adb shell pm list packages | grep ‘test’ linux  列出手机装带有的test的包

adb shell cat /sys/class/net/wlan0/address 获取MAC 地址, 根据系统版本参数可能不同

adb shell dumpsys activity services [<packagename>] 查看正在运行的Services

<packagename> 参数不是必须的，指定<packagename> 表示查看与某个包名相关的Services，不指定表示查看所有Services。

<packagename> 不一定要给出完整的包名，比如运行adb shell dumpsys activity services com.tencent，那么包名com.tencent.weixin、com.tencent.qq 等相关的Services 都会列出来。
```

#### 3.安装和卸载

```
adb install <apkfile> 参数apkfile 为.apk 文件名称

adb install -r test.apk 保留数据和缓存文件，重新安装apk

adb install -s test.apk 安装apk到sd卡

adb uninstall -k com.test.mobile    卸载app 但保留数据和缓存文件

adb shell pm uninstall -k --user 0 <packagename> 卸载系统应用

```

#### 4.启动停止服务
```
adb start-server

adb kill-server

adb -P <port> start-server  指定adb server 的网络端口port （默认为5037）启动服务
```

#### 5.与应用交互
```
adb shell pm clear <packagename>  清除应用数据与缓存

adb shell am force-stop <packagename>   强制停止应用

adb shell am start -n com.tencent.mm/.ui.LauncherUI     启动activity

adb shell am startservice -n com.tencent.mm/.plugin.accountsync.model.AccountAuthenticatorService  启动 Service

adb shell am broadcast -a android.intent.action.BOOT_COMPLETED   向所有组件广播 BOOT_COMPLETED (开机广播)

adb shell am broadcast -a android.intent.action.BOOT_COMPLETED -n org.mazhuang.boottimemeasure/.BootCompletedReceiver 只向某个应用发送广播BOOT_COMPLETED：

adb shell am broadcast -a com.android.action.PLAY -p com.android.car -f 0x00000020 --es BACKGROUND 'FALSE' 指定包名，携带参数。
```
#### 6.模拟按键/输入
```
adb shell input keyevent 26  #电源键
adb shell input keyevent 82  #菜单键
adb shell input keyevent 3  #HOME 键
adb shell input keyevent 4 #返回键
adb shell input keyevent 24 #增加音量
adb shell input keyevent 25 #降低音量
adb shell input keyevent 164 #静音
adb shell input keyevent 85  #播放/暂停
adb shell input keyevent 86 #停止播放
adb shell input keyevent 87 #播放下一首
adb shell input keyevent 88 #播放上一首
adb shell input keyevent 126 #恢复播放
adb shell input keyevent 127 #暂停播放
adb shell input keyevent 224 #点亮屏幕
adb shell input keyevent 223 #熄灭屏幕
adb shell input swipe 300 1000 300 500  #滑动解锁，向上滑动手势解锁
#参数 300 1000 300 500 分别表示起始点x坐标 起始点y坐标 结束点x坐标 结束点y坐标 
adb shell input text hello #焦点处于某文本框时输入文本
```

#### 7.实用功能
##### 截图
```
adb exec-out screencap -p > img.png     #老版本无exec-out命令，只适合于新版的截图

adb shell screencap -p /sdcard/img.png      #老版本截图先保存在设备端

-p 指定保存文件为 png 格式

```
##### 录制屏幕
```
adb shell screenrecord /sdcard/filename.mp4
```
##### 查看连接过的WiFi密码
```
/data/misc/wifi/WifiConfigStore.xml
```
##### 刷机相关命令
```
adb reboot recovery     重启到Recovery模式
```

##### 日志相关
```
adb logcat -c && adb logcat >log.txt

adb logcat *:E >log.txt

ANR文件导出：
1.adb pull data/anr/traces.txt
2.adb bugreport C:\Users\summer\Desktop
```

##### 统计App启动耗时

```
adb shell start -S -W com.example.camera/.MainActivity
```