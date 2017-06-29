---
layout:     post
title:      "Android BootAnimation"
subtitle:   " \"How to create bootanimation.zip\""
date:       2017-06-29 17:40:45
author:     "faxiang1230"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
  - bootanimation
  - android
---
# Android开机动画修改
## 开机动画的启动
开机动画是位于kernel起来之后到laucher/setupwizard应用之间的一段动画，如果android的启动非常快的话，假如只有1s，完全没有做开机动画的必要.
正是因为android需要启动非常多的服务而且是单线程启动，没有并行化工作，所以额外的耗费时间；
再加上OEM在这加点自己的私货，时间是蹭蹭往上涨。
android有时候做得也不太好，以Lollipop上为例,在启动的时候packagemanager会挨个检查已经安装应用是否破损，这会挨个访问apk文件并且解压遍历文件，安装的应用越多，size越大，花费时间越久；我在OPENTHOS上省略了这一步才算抑制了开机时间
我手里的华为手机开机30s以上，如果应用装得比较多时间还会往上涨.
总之:开机动画就是android遮丑用的，就像360应用安装时总是给你显个动画的意思一样
言归正传,开机动画的启动
```
service bootanim /system/bin/bootanimation
    class core
    oneshot
```
开机之后也可以手动使用`start bootanim`来启动一下动画，不过基本上只是闪一下.
bootanimation的代码位于`frameworks/base/cmds/bootanimation/`下，可以看成使用NDK开发的应用，是依赖Native服务的，所以它的启动也是非常晚的;
所以这个修改的余地也非常大，可以添加开机声音或者干脆更改为开机视频得了;

普通的开机动画加载过程:
`frameworks/base/cmds/bootanimation/BootAnimation.cpp`中有三个位置,有限遍历顺序SYSTEM_ENCRYPTED_BOOTANIMATION_FILE->OEM_BOOTANIMATION_FILE->SYSTEM_BOOTANIMATION_FILE:
```
static const char OEM_BOOTANIMATION_FILE[] = "/oem/media/bootanimation.zip";
static const char SYSTEM_BOOTANIMATION_FILE[] = "/system/media/bootanimation.zip";
static const char SYSTEM_ENCRYPTED_BOOTANIMATION_FILE[] = "/system/media/bootanimation-encrypted.zip";
```
## 开机动画内容
我们这里不关心顺序，直接看bootanimation.zip这个压缩包文件:
```
├── audio_conf.txt
├── desc.txt
├── part0
│   ├── audio.wav
│   ├── LOGO完整居中_00000.png
│   ├── LOGO完整居中_00001.png
└── part1
    ├── Loading完整居中_00000.png
    ├── Loading完整居中_00001.png
    ├── Loading完整居中_00002.png
    ├── Loading完整居中_00003.png
```
包含 part0 part1 文件夹和 desc.txt 文件，part0，part1文件夹里面放的是动画拆分的图片，格式为 png 或 jpg。

压缩包文件的格式:zip格式，压缩等级为0(压缩级别为0-9，数字越大压缩率越高)

如何生成压缩包:
```
zip -0 -r ../bootanimation.zip ./*
```
desc.txt 文件内容如下：
```
800 480 15
p 1 0 part0
p 0 0 part1
```
part0放有音乐因为只放一次音乐所以循环1次，part1没有音乐所以无限循环。

说明：

第一行：800 为宽度，480 为高度，15 为帧数。第二行开始 p 为标志符，接下来第二列为循环

次数（０为无限循环），第三项为两次循环之间间隔的帧数，第四项为对应的目录名。播放动画时会

按照图片文件名顺序自动播放。

### 开机音乐

如需开机音乐，将开机音乐放入 part0 目录中，命名为 audio.wav。

在根目录中加入 audio_conf.txt，内容如下：
```
card=0
device=0
period_size=2048
period_count=2
mixer "Mic Playback Switch"=1 1
```
## 开机播放视频
完善的一个播放视频的patch:[查看][1]

这个patch还不完整，因为在Launcher或者setupwizard启动之后会经过windowmanagerservice->surfaceflinger设置`service.bootanim.exit`为真，bootanimation中的`CheckExit`一直循环读取这个属性;

所以为了完整播放开机动画可以将退出条件更改:视频播放完或者延迟固定时间等
[1]: https://github.com/faxiang1230/work/blob/master/android-x86/patch/0001-Support-Mediaplayer-for-bootanimation.patch        "Patch"
