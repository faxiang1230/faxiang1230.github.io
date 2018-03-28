---
layout:     post
title:      "杂活"
subtitle:   " \"没啥技术含量\""
date:       2017-06-29 20:57:45
author:     "faxiang1230"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
  - 杂活
---
# 我干过的杂活们
## 重启setupwizard
老板想要第一次开机启动后，给里面加点料，然后再恢复现场，用户就感觉不到我们做过的伟大工作了.

简单而言就是重启setupwizard应用；我们都知道这个应用属于一次性的应用，只要用户运行过一次，就再也不运行了;

其实很简单，程序运行完之后请求packagemanager将自己删除掉

### setupwizard应用启动

ActivityManager启动到最后是会主动启动`category android:name="android.intent.category.HOME"`的应用，而还有个属性`android:priority`(从1-10,数字越大,优先级越低)决定应用启动的顺序，Launcher的优先级为默认的10，所以setupwizard应用先于Launcher启动:
```
<activity android:enabled="true" android:excludeFromRecents="true" android:launchMode="singleTask" android:name="com.otosoft.setupwizard.SetupWizardActivity" android:theme="@android:style/Theme.Material.Light.NoActionBar">
    <intent-filter android:priority="9">
        <action android:name="android.intent.action.MAIN"/>
        <action android:name="com.otosoft.setupwizard.MAIN"/>
        <category android:name="android.intent.category.DEFAULT"/>
        <category android:name="android.intent.category.HOME"/>
    </intent-filter>
</activity>
```
### 如何避免以后启动呢
应用在结束的时候做了这么一件事:通知PackageManager将自己disable掉，以后就不要启动我了
```
PackageManager pm = getPackageManager();
ComponentName name = new ComponentName(this, DefaultActivity.class);
pm.setComponentEnabledSetting(name, PackageManager.COMPONENT_ENABLED_STATE_DISABLED,
    PackageManager.DONT_KILL_APP);
```
每次重启之后也不会启动这个应用，那么肯定是存在flash或者硬盘上了，顺着找就可以看到是存储在`/data/system/users/0/package-restrictions.xml`文件中；不管是删除整个文件也好还是只删除特定行也好，都可以重新使能setupwizard应用
### 方法
`rm /data/system/users/0/package-restrictions.xml`
## 锁屏应用
锁屏三大要素:  
- 1.监听屏幕亮灭的广播  
静态或者动态注册广播`android.intent.action.SCREEN_ON`,实现在亮屏的时候能够启动开始强制用户认证过程  
下面两个就是怎么防止用户绕过认证过程  
- 2.使导航栏和状态栏消失  
这个功能是在`SystemUI`中完成的，通过`intent`和`SystemUI`互通消息.  
状态栏中可以实现消息下拉，还能够直接进入`Setting`中;导航栏更厉害，有`HOME`直接到`Launcher`界面，`历史功能`中可以杀死应用
- 3.禁止HOME键,历史功能  
手机上有可能有硬件HOME键或者其他的软件按键，这可能就绕过了锁屏应用，所以肯定是需要禁止按键功能
- 4.防止后台被杀  
普通应用如果转为后台进程，那么它的重要性就相应降低了，当内存不足的时候优先杀这些进程，所以还需要防止后台被杀
