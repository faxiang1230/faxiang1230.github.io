---
layout:     post
title:      "系统性能杂想"
date:       2017-07-26 19:50:45
author:     "faxiang1230"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
  - linux
  - performance
---
# 软件的性能
像brendan gregg说的那样,性能是一个比较主观的指标,也是一个非常广泛的指标;  

泛指用户使用的一切场景能够提供良好的使用体验,而开发没有那么多精力来尝试所有的场景,所以一般来说都是控制常见场景的性能指标;  

常见的PC性能指标:  

画面流畅,不卡顿  

按键响应速度快:  

网络速度快:  

使用场景:玩大型网游,浏览网页,在线看电影,远程会议,下载文件,办公  

用户使用的感觉最终是由应用来反馈的,所以性能指标不仅包括系统还有应用;  

系统给应用提供地基,所以地基一定要牢,性能一定要好(怎么算好这也是一项大工程);应用在上面搭台唱戏,
地下观众只看到最终应用的效果,所以应用也是需要满足性能指标的;所以我们看到android不仅自身做了大量
的性能优化工作,而且还给应用开发提供了很多优化参考,以最终能够达到理想的机器整体使用体验良好的效果;  

而在开发的过程中,引起性能的问题主要有两类:架构上的问题和小问题  
架构上设计时如果没有充分考虑性能或者没有预料到性能下降的场景,很明显架构上很可能存在许多的问题,在理论上就已经不满足性能指标了,所以无论再怎么优化也没用;  
我们可能也需要重新设计架构,这个从大型软件上基本都可以看到,架构被重构了N+1次,Linux kernel的进程调度系统;  
小问题:架构上没有问题时,那么另一种可能是我们的性能被很多很多的小问题给拉低了,最后病入膏肓,谁也救不了;我遇到了一个项目开发,里面整了大量的临时方案,从commit message上  
看到了非常多的临时fix方案,我真的不相对这种patch讲太多;
这种临时方案author可能觉着是奇思妙想,这样就可以解决问题,但是其他开发接手的时候满脑袋问号?为什么这样提?当时发生什么情况了?  
很可能后面的patch一个一个覆盖上去,谁也不知道最初的原因了,这样系统进入迅速死亡期;  

不光是小项目,大项目也是一个道理,由于持续的开发,代码越来越多,越来越臃肿,已经不可能有人完全看懂所有的code了,里面会加入各种patch(完全看不懂,谁看谁怀孕);  
杜绝临时方案,尽可能地拖延软件死亡的时间;针对性的进行性能测试并且立时解决.  