---
layout: post
title:  "Windows下使用ndk-gdb进行调试"
date:   2014-09-08 12:36:25
categories: Android
---

**转载请注明出处:http://xujim.github.io**

在ndkr9上进行验证，可参考ndkpath/docs/NDK-GDB.html文件，其必须满足一些条件。

除此之外由于其本身的bug:

1. 在windows下你必须在cygwin下调试
2. 在cygwin下，你必须重新设置NDK_MODULE_PATH到环境变量中.否则报类似以下错误：

AndroidNDK installation path: /Library/AndroidSDK/ndk/ 
Usingspecific adb command: /Library/AndroidSDK/platform-tools/adb 
ADBversion found: Android Debug Bridge version 1.0.31 
UsingADB flags: 
Usingauto-detected project path: . 
Foundpackage name: com.dev.project 
jni/Android.mk:18: * Android NDK:Aborting. . Stop. 
ABIstargetted by application: Android NDK: 
DeviceAPI Level: 17 
DeviceCPU ABIs: armeabi-v7a armeabi 
ERROR:The device does not support the application's targetted CPU ABIs! 
Devicesupports: armeabi-v7a armeabi 
Packagesupports: Android NDK: 

怎么解决呢？可参考如下:

I fixed doingthis:
export NDK_MODULE_PATH=path_to_look_for_modules
Where path_to_look_for_modules should be the parent directory of your moduledeclared in the Android.mk. That is, if you have /myproject/mylibs/otherlib export the path /myproject/mylibs
If you have several paths, asusual:
export NDK_MODULE_PATH=path1:path2:path3

所以调试cocos2d-x的话，你就需要设置

export NDK_MODULE_PATH=/cygdrive/f/cocos2d-x:/cygdrive/f/cocos2d-x/external:/cygdrive/f/cocos2d-x/cocos
Pastedfrom <http://stackoverflow.com/questions/15067215/ndk-gdb-error-device-does-not-support-the-applications-targetted-cpu-abis>

3. 你还必须装gnu-make,因为ndk-build用的是:

The tricky part here is that to run ndk-build, you do not need cygwin; worse - you should use ndk\prebuilt\windows\bin\make and never cygwin\bin\make! But to run ndk-gdb, you need cygwin andits make

Pastedfrom <http://stackoverflow.com/questions/12020245/ndk-gdb-on-windows>

4. 可用ndk-gdb --verbose看调试信息

5. ndk-gdb必须在你的project目录里面运行，因为它会自动查找libs里面的符号，并自动运行对应的gdb

