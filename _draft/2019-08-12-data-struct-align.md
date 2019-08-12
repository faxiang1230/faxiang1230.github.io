---
layout:     post
title:      "数据结构对齐"
date:       2019-08-12 16:40:45
author:     "faxiang1230"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
  - linux
---
# 数据结构对齐
数据结构对齐主要和数据在计算机内存中怎么组织存放并且怎么被访问相关，但是它包括三个小方面:数据对齐，数据结构填充，和packing
现代计算机的CPU访问对齐的数据时效率是最高的，对齐意味着数据的地址是数据大小的整数倍。而数据结构对齐还意味着每个成员都是地址对齐的，为了保证地址对齐，可能需要在结构体元素间进行一些必要填充，或者是结构体对象后面追加一些填充从而下一个对象是地址对齐。
Data structure alignment refers to the way data is arranged and accessed in computer memory. It consists of three separate but related issues: data alignment, data structure padding, and packing.

The CPU in modern computer hardware performs reads and writes to memory most efficiently when the data is naturally aligned, which generally means that the data address is a multiple of the data size. Data alignment refers to aligning elements according to their natural alignment. To ensure natural alignment, it may be necessary to insert some padding between structure elements or after the last element of a structure.
尽管数据结构对齐在所以现代计算机上是一个基础的常识，很多计算机语言都已经默认实现数据是对齐存放的，比如`Ada,PL/I,Pascal，C/C++,Rust,C#`,汇编语言也允许在一些特殊的情境下，通过数据填充来进行对齐
Although data structure alignment is a fundamental issue for all modern computers, many computer languages and computer language implementations handle data alignment automatically. Ada,[1][2] PL/I,[3] Pascal,[4] certain C and C++ implementations, D,[5] Rust,[6] C#,[7] and assembly language allow at least partial control of data structure padding, which may be useful in certain special circumstances.
一个内存地址a是n字节对齐的时候，其中n是2的阶乘，在这里，一个字节是内存访问的最小单元，每一个内存地址都有一个字节。而n字节对齐的地址在二进制表示的时候最低的`log2(n)`位地址是0.另外一种说法是这个机器是b(bit)对齐的，也就是它是b/8(字节)对齐的，例如64bit对齐就是8字节对齐。
A memory address a is said to be n-byte aligned when a is a multiple of n bytes (where n is a power of 2). In this context a byte is the smallest unit of memory access, i.e. each memory address specifies a different byte. An n-byte aligned address would have a minimum of log2(n) least-significant zeros when expressed in binary.

The alternate wording b-bit aligned designates a b/8 byte aligned address (ex. 64-bit aligned is 8 bytes aligned).

A memory access is said to be aligned when the data being accessed is n bytes long and the datum address is n-byte aligned. When a memory access is not aligned, it is said to be misaligned. Note that by definition byte memory accesses are always aligned.

A memory pointer that refers to primitive data that is n bytes long is said to be aligned if it is only allowed to contain addresses that are n-byte aligned, otherwise it is said to be unaligned. A memory pointer that refers to a data aggregate (a data structure or array) is aligned if (and only if) each primitive datum in the aggregate is aligned.

Note that the definitions above assume that each primitive datum is a power of two bytes long. When this is not the case (as with 80-bit floating-point on x86) the context influences the conditions where the datum is considered aligned or not.

Data structures can be stored in memory on the stack with a static size known as bounded or on the heap with a dynamic size known as unbounded.
翻译自:
https://en.wikipedia.org/wiki/Data_structure_alignment
