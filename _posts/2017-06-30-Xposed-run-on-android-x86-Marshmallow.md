---
layout:     post
title:      "Xposed在android-x86 Marshmallow上运行"
date:       2017-06-30 20:40:45
author:     "faxiang1230"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
  - andrioid-x86
  - art
  - xposed
---
# 在Android-x86 M上運行xposed程序
## 下載android-x86代碼
```
mkdir marshmallow-x86;cd marshmallow-x86
repo init -u git://git.osdn.net/gitroot/android-x86/manifest -b marshmallow-x86
repo sync --no-tags -f -j8
--no-tags:减少不需要的tag下载，可以缩减下载的代码量
-f:当某个库因为网络原因货其他原因下载失败的时候可以继续进行，避免已经下载的代码不能写入到硬盘上
-j8:开启8个线程来下载，这个根据CPU和硬盘的性能来决定吧
```
## 下載xposed代碼
```
git clone https://github.com/rovo89/Xposed.git xposed
git clone https://github.com/rovo89/XposedInstaller.git
git clone https://github.com/rovo89/XposedTools.git
git clone https://github.com/rovo89/XposedBridge.git
git clone https://github.com/rovo89/android_art.git art
```
其中相關目錄結構：
```
marshmallow/frameworks/base/cmds/xposed
marshmallow/art 使用xposed提供的art替换掉原声的art
marshmallow/XposedTools
```
## 编译xposed
1. 在下载android-x86之后不要做任何修改就开始编译`source build/envsetup.sh;lunch android_x86_64-eng;make -j16 iso_img`

2. 开始编译xposed的art和app_process

将xposed相关库放在对应的目录,然后使art生效`make -j16 iso_img`

生成app_process和libxposed_art.so
```
cd frameworks/base/cmds/xposed
mm
```
看到生成的三个可执行文件:
```
/system/bin/app_process32_xposed
/system/bin/app_process64_xposed
/system/lib/libxposed_art.so
```
将生成的xposed文件替换系统原声的
```
mv /system/bin/app_process32_xposed /system/bin/app_process32
mv /system/bin/app_process64_xposed /system/bin/app_process64
```
3. 生成XposedBridge.jar文件

进入XposedBridge目录下,执行如下命令：
```
export ANDROID_HOME=/home/linux/sdc/linux/Android/Sdk
./gradlew app:assembleRelease lint
```
生成`./app/build/outputs/apk/app-release-unsigned.apk`文件，直接copy到marshmallow/out/xposde/java/XposedBridge.jar

关于jar和apk的关系:
```
一个 Android 工程最后变成 apk 包大概要做这么几件事儿：
1、生成 R.java 文件
2、将 .java 文件编译成 .class 文件
3、将 .class 文件打包成 .jar 文件
4、将所有 .jar 文件（包括依赖库）编译成 classes.dex 文件
5、将 assets 和 res 文件夹中所有的资源文件打包成一个 apk 包
6、将 classes.dex 文件添加进 apk 包
7、如果有使用 NDK 技术的话，将生成的 .so 文件添加进 apk 包
8、对 apk 包进行签名
说白了一个 apk 包就是由代码和资源组成的。代码的处理基本上就是编译，这个没什么可说的。
```
4. 生成xposed.prop文件
配置build.pl需要的perl module

```
perl -MCPAN -e shell

install Config::IniFiles
install File::ReadBackwards
install File::Tail
install Archive::Zip
install Tie::IxHash
```
进入XposedTools目录下，配置build.conf文件(作者给了一个build.conf.sample,我也是基于这个修改的；这里面的参数含义可以在https://github.com/rovo89/XposedTools/blob/master/README.md#configuration-buildconf找到)
```
[General]
outdir = /root/test/marshmallow/out/xposed
javadir = /root/test/marshmallow/xposedBrigded

[Build]
# Please keep the base version number and add your custom suffix
version = 65 (custom build by xyz / %s)
makeflags = -j8

[GPG]
sign = release
user = android-x86_64

# Root directories of the AOSP source tree per SDK version
[AospDir]
23 = /root/test/marshmallow/

# SDKs to be used for compiling BusyBox
# Needs https://github.com/rovo89/android_external_busybox
[BusyBox]
x86 = 23
```
然后使用作者提供的脚本工具来生成xposed.prop
```
./build.pl -t x86:23 -s prop
```
目前还不支持x86_64，暂时使用x86将就用吧;

将生成出来的xposed.prop放到out/target/product/x86_64/system下

5. 重新打包镜像

`make -j16 iso_img`

6. 使用XposedInstaller应用

进入XposedInstaller,build这个应用
```
export ANDROID_HOME=/home/linux/sdc/linux/Android/Sdk
./gradlew build

find -iname *.apk
```
安装apk时可以看到XposedInstaller应用提示Xposed framework已经安装完毕，并且可以使用xposed提供的一些应用

7. 遗留问题

脚本里写着需要如下module，我并没有完全加入，这个问题还需要解决  
``` xposed libxposed_art libart libart-compiler libart-disassembler libsigchain dex2oat oatdump patchoat```
