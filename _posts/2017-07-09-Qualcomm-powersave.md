---
layout:     post
title:      "高通平台的power简述"
date:       2017-07-09 18:40:45
author:     "faxiang1230"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
  - android
  - qualcomm
  - power
---
# 高通平台Power简述

下面的东西总结性居多，没有涉及到具体的方法，操作性不强，只是作为一个全局概览。

## Power状态
standby：手机静置，驻网，无数据活动，无小区handover  
idle:用户没有明确告诉手机休息的时候，phone本身也没有后台或周期任务，很可能就进入cpu idle例程；除了cpu还有外设此时无任务可做也会尝试主动挂起自己;所有条件都满足之后就是cpu idle;  
suspend:当手机明确不使用时，特别是按了power键灭屏告诉手机我要休息的时候，此时phone会主动发起suspend过程，将所有的设备主动尝试挂起  

suspend和idle看起来很想，主要的区别是两个睡眠程度不一样，相对而言suspend睡得深度更深一些，那对应的也是有代价的，唤醒时间长，换句话就是延迟高;有人在鼓捣一个S2I(suspend to idle)的东东，结合suspend和idle的优点:平均电流更低，唤醒延迟小   
http://events.linuxfoundation.org/sites/events/files/slides/what-is-suspend-to-idle.pdf

## Power测试
历史(下面的测试怎么来的):  
**power体验**:  
都知道，手机提示你快没电的时候你应该啥都不敢干了吧，或者赶紧找来充电器，总之这时候手机功能再好玩你也不能好好体验了;power体验就是超长时间使用，不过这个不现实，在phone和电池限定后，使用时再怎么厉害也不能突破物理定律;  
**power优化**:  
就是能不耗电的时候就不耗电，该用的电一点也不会少;  
**power测试怎么来的**:  
1.用户看到的是各种场景下的功耗，从续航能力和手机是否发热来判断这个手机的功耗品质如何;    
2.benchmark工具做了一些简单的例子来对手机进行一系列测试，得出来很多的数据展示手机的功耗如何;  
3.开发拿到1,2数据后就开始思考，这个为什么功耗高了？  
开始肢解这个问题，测试各个部分的电流状况，找出电流异常的模块，然后问软件开发为啥，软件开发告诉他是另外一个module使用这个引起的，吧啦吧啦，最后找到root原因;  
4.重复几次就总结出经验了，每次这么麻烦的找模块，不如逆向证明如何?  
分割出来很多的测试，分别来测试每个module自举是否有问题;  
如果各个module都没有问题，那么组合起来试试如何?  
如果组合起来也没有问题，那么可以让用户体验试试.  

power测试可能就是这么扣出来的(纯属瞎掰),然后手机厂商也拿这个测试结果向消费者证明自身的功耗;  
非常佩服肢解power系统的人，非arch级别的人很难整出来这种测试方案;

如何分类:  
- 手机使用时典型的场景和典型设备功耗  
1.典型场景：使用浏览器上网，播放音乐（有线耳机/蓝牙耳机），录音，拍照，录像，下载，安装/卸载软件，蓝牙连接，wifi连接时/闲置时  
2.典型设备：蓝牙，wifi，FM，GPS，Modem在不同网络制式下的功耗，Audio，Media,Camera  

- 一般会先要求测试各个独立的场景，然后可能是复合场景的功耗测试。  
复合场景:基本上是挑选一些实际中使用的场景  
听着音乐上网，发短信，开着wifi打电话，蓝牙播放音乐  
## Power管理

高通的原理图还是画的比较清楚的  
![image](/images/8974arch.jpg).

RPM:  
rpm(resource power manager,右上角很小的一个chip，好像是一个低级的arm处理器)管理整个系统的功耗，莫看rpm就占一点点空间，站在power角度上看，rpm是master，其他的所有设备，子系统都是slave，都得听他的;  
管理范畴：clock，power rail，bus width等  
管理的框架：NPA(Node Power Arch)，关于如何投票可以参考AP侧的clock的driver和Modem，LPASS中的投票部分；  
rpm的框架：rpm和各个子系统的关系：通过NPA来管理所有的请求，通过SMEM来通信  
rpm中有PMIC的driver，可以操作PMIC;另外还有BUS,CLK,DDR的driver，可以控制他们的clock和power rail(power rail很蛋疼，不知道怎么翻译过来)。  

MPM:  
上面哪个图干脆就没画MPM，不过不影响它的重要性  
MPM会在rpm睡眠后接管整个power系统，最主要的功能是能够接收外部事件唤醒整个系统，所以很重要的一部分是中断的管理。

子系统:  
APSS(application processor subsystem),  
LPASS(lower power audio subsystem),  
MMSS(multimedia subsystem),  
MDSS(Modem subsystem),  
WCNSS(wireless connect network subsystem)
## Power debug的手段：

软件方法:rpm log,kernel log,dynamic_debug,ftrace,wakelock,debug_mask,suspend/idle过程的理解;  
rpm log:高通平台上power最直接的日志，能够显示出各个子系统的一些clock和power状态  
F3log:一些关于npa()的日志,子系统和rpm通信的框架，能够dump出一些最后的消息  
APSS上主要就是kernel的运行了,方法多种多样:dmesg,dynamic_debug,ftrace,wakelock,top,powertop,systrace,  

其他方法：  
dump，可以dump出睡眠时各个GPIO的配置(针对低电流过高的问题)；  
dump使用过程中的clock状态(针对使用场景中功耗过高)  

## Power常见问题思路
- 不能进入vdd_min状态
```
通过rpm_stats来确认是否一直未进入vdd_min;
  未进入:通过rpm log来确认哪个子系统没有vote
    APSS:打开clock debug，查找哪个使用cxo
    Modem:MCPM log
    WCNSS:wcnss log和在dmesg和logcat中Wifi/Gps/BT/NFC相关的日志.
```
- 底电流过高
```
check睡眠时的GPIO状态的配置，特别是修改过高通参考设计部分的GPIO
QCN是否烧写，RF是否校准
其他的外设是否都关闭了
```
- 底电流正常，但是波峰比较多
```
debug中断:
APSS侧可以打开中断的debug_mask，比较/proc/interrupts;
```
- 最困难的是和各种组合场景的功耗，几个操作一关联复杂度就直线上升

```
各个设备的功耗参考:
CPU和Modem，其他的DSP，他们都是有工作频率的，不同的工作频率功耗不一样嘛。想办法搞到对比的工作频率和电压值，对比一下就知道这些chip是否正常;其中CPU被大家研究烂了，最简单的top直接就能看出来cpu的繁忙程度，不过它往往也是消耗cpu大户，往往排在前几的位置;注意使用工具的overhead

其他的设备应该只有开和关两种状态吧，一般都可以在dmesg中找到设备工作的一些状态；没有现成的就新造嘛.
```

power是比较依赖实践的一个事，可以简单的进行对比测试来缩小问题的范围，然后找到针对性的方法来进一步的debug;  
可以尝试从零坐起:先将设备(WIFI，BT，,sensor:light,autorotate,压力，握力)都关闭，集中调试子系统的状态，然后挨个打开设备调试设备的driver（一般比较简单）.  

## Android侧的debug手段
- 1.wakeup唤醒源:  
唤醒进入休眠状态下的phone:  
外部中断事件:例如触摸了手机,手机的sensor工作，都可能导致中断唤醒phone  
内部中断事件:一般分为ap侧和modem侧，ap侧的一般是定时器，modem侧的基本和协议相关:小区切换，信号强度变化，周期性尝试切换回home网；  

```
mount –t debugfs none /sys/kernel/debug
echo 1 > /sys/kernel/debug/clk/debug_suspend
echo 1 > /sys/module/msm_show_resume_irq/parameters/debug_mask
echo 4 > /sys/module/wakelock/parameters/debug_mask
echo 1 > /sys/module/lpm_levels/parameters/debug_mask
echo 0x16 > /sys/module/smd/parameters/debug_mask

adb dumpsys alarm >alarms.txt

echo 0 > /proc/timer_stats && sleep 10 && echo 1 > /proc/timer_stats &&
sleep 30 && cat /proc/timer_stats > /data/timer_stats &
```

- 2.suspend fail:  
灭屏之后总是suspend过程失败，一般都是有wakeup source问题；其余的很容易通过dmesg来判断问题点.
```
直接在suspend过程中埋点，打印一些log，特别是wakeup source的打印

/sys/kernel/debug/wakeup_sources
active_since：自上次激活以来的时间

利用ftrace收集:
echo "power:wakeup_source_activate power:wakeup_source_deactivate" >
/sys/kernel/debug/tracing/set_event
```
