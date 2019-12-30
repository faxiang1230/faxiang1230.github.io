---
layout:     post
title:      "Intel vs. AT&T 汇编语法"
date:       2019-12-30 11:50:45
author:     "faxiang1230"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
  - x86
  - assembler
  - 翻译
---
# Intel vs. AT&T 汇编语法
>翻译自:http://staffwww.fullcoll.edu/aclifton/courses/cs241/syntax.html

x86(32位和64位)有两种语法:AT&T和intel,有些汇编器只能支持一种，有些汇编器可以两种都支持。
在标准linux和其他的在线资源上主要使用`AT&T`语法，但是我们(斯坦福大学课堂)主要使用Intel语法。
下面的表中展示了两种语法的区别:

|    -|Intel|AT&T|
|-----|-----|----|
|注释|`;`|`//`|
|指令|不带后缀`add`|带有长度后缀的`addq`|
|寄存器使用|eax, ebx|%eax,%ebx|
|立即数使用|	0x100	|$0x100|
|寄存器值引用|	`[eax]`	|`(%eax)|
|间接寻址|`[base + reg + reg * scale + displacement]`|	`displacement(reg, reg, scale)``|

在通常间接引用时，Intel语法将会计算`基址+偏移`,而且操作码允许一个单独的立即数偏移，而在AT&T语法中，你必须自己做，把相加的结果当做括号中第三个值。

主要的区别是Intel不需要明确指出使用了指令操作数的长度，而AT&T需要显式地指明他们并且通过在指令操作码后面跟上相应的长度后缀，下面是一个两个值相加的示例:
`add eax, ebx;`  
这是Intel语法，将两个32bit的寄存器相加(因为eax和ebx都是32位寄存器)，并且把结果放在了`eax`，其中Intel语法中目的寄存器放在前面，源寄存器放在后面。因为源和目标都是32bit，汇编器就知道他只需要转换成32位的`加指令`.而在AT&T语法中，相同的指令需要写成这样:
`addq %ebx, %eax ;`  
在AT&T语法中，寄存器使用都需要带有`%`,并且源寄存器放在前面，目的寄存器放在后面，和Intel是相反的，而且最大的区别是指令的`q`后缀，`q`是`quadword`的缩写，代表32bit值。这个语法明确告诉了汇编器操作码的长度，不需要通过寄存器的大小来推测出来正确的指令长度。

可能在一个指令中有两个不同长度的寄存器，例如，将`ebx`的低16位加到`eax`中，Intel语法只需要写成这样:
`add eax, bx ;`  
但是在AT&T语法中,你需要这样写:
`addzqd %bx, %eax ;`  
其中指令后缀`d`代表`doubleword`,`z`代码它是一个无符号相加(用零填充高位)，如果我们是有符号相加，我们需要一个符号扩展`s`.

后缀'b'、'w'、'l'分别表示操作数为字节（byte，8 bit）、字（word，16bit）和长字（long，32bit）

这样看来intel语法更加简洁，对编程人员更加友好，很可惜在linux上AT&T才是主流。

## 扩展阅读
http://www.cs.cmu.edu/afs/cs/academic/class/15213-f01/docs/gas-notes.txt  
https://www.cnblogs.com/hdk1993/p/4820353.html  
