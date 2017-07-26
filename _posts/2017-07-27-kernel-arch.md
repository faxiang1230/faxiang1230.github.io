---
layout:     post
title:      "kernel相关的内容概览"
date:       2017-07-27 01:40:45
author:     "faxiang1230"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
  - kernel
  - linux
---

# kernel的重要点总结
## 进程
### 进程创建
### 进程调度
#### 调度策略
#### runqueue
#### vruntime
#### 抢占
##### 内核抢占
##### 用户抢占
#### 进程切换
### 负载均衡
#### 负载均衡时机
#### 负载均衡策略
## 中断
## 内存管理
### 物理内存管理
#### 伙伴系统
#### 每cpu高速缓存
### 内存系统初始化
### 虚拟内存管理
### swap
### 缓存
### 页面回收
### 页表
### 缺页异常
## 信号
## 中断
### 硬件中断过程
### 软中断过程
### 硬中断和软中断的区别
## 临界区
### 原子锁
### 信号量
### 原子变量的实现原理
### 死锁
## 文件系统
### VFS
### 具体的文件系统
## IO子系统
### IO架构
### 调优
## module
## 内核安全审计
## 设备驱动模型
### 初始化和地址空间
## kernel实践记录
### debug方法
### qemu+kernel
### kgdb
### panic
### user mode linux
### 重要的全局变量
### ramdump和crash分析
### how to submit a patch to kernel
### kernel名人录
### kprobe
### perf工具
### ftrace
### systemtap
