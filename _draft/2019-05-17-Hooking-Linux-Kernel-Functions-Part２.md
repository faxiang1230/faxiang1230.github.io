---
layout:     post
title:      "Hooking Linux Kernel Functions, Part 2"
subtitle: "How to Hook Functions with Ftrace"
date:       2019-04-09 20:40:45
author:     "faxiang1230"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
  - 翻译
  - hook
---
>>https://www.apriorit.com/dev-blog/546-hooking-linux-functions-2

Ftrace is a Linux kernel framework for tracing Linux kernel functions. But our team managed to find a new way to use ftrace when trying to enable system activity monitoring to be able to block suspicious processes. It turns out that ftrace allows you to install hooks from a loadable GPL module without rebuilding the kernel. This approach works for Linux kernel versions 3.19 and higher for the x86_64 architecture.

This is the second part of our three-part series on hooking Linux kernel function calls. In this article, we explain how you can use ftrace to hook critical function calls in the Linux kernel. We also describe and test two theories for protecting a Linux kernel module from ftrace hooks. Read Hooking Linux Kernel Functions, Part 1: Looking for the Perfect Solution to learn more about other approaches that can be used for accomplishing this task.

## A new approach: Using ftrace for Linux kernel hooking

What is an ftrace? Basically, ftrace is a framework used for tracing the kernel on the function level. This framework has been in development since 2008 and has quite an impressive feature set. What data can you usually get when you trace your kernel functions with ftrace? Linux ftrace displays call graphs, tracks the frequency and length of function calls, filters particular functions by templates, and so on. Further down this article you’ll find references to official documents and sources you can use to learn more about the capabilities of ftrace.

The implementation of ftrace is based on the compiler options -pg and -mfentry. These kernel options insert the call of a special tracing function — mcount() or __fentry__() — at the beginning of every function. In user programs, profilers use this compiler capability for tracking calls of all functions. In the kernel, however, these functions are used for implementing the ftrace framework.

Calling ftrace from every function is, of course, pretty costly. This is why there’s an optimization available for popular architectures — dynamic ftrace. If ftrace isn’t in use, it nearly doesn’t affect the system because the kernel knows where the calls mcount() or __fentry__() are located and replaces the machine code with nop (a specific instruction that does nothing) at an early stage. And when Linux kernel trace is on, ftrace calls are added back to the necessary functions.
### Description of necessary functions

The following structure can be used for describing each hooked function:
