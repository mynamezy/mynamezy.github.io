---
layout: post
title: "C/C++编译过程简述"
subtitle: "C and C++ compiling process"
author: "zy"
header-img: "img/post-bg-os-metro.jpg"
catalog: true
tags:
  - C/C++
---

> 武汉加油，中国加油！

## 引言
最近在使用golang的swig组件调用c++的动态连接库，不调不知道，一调吓一跳，各种问题。而且在和算法同时调试的过程中，由于自己对基本的编译知识了解太少了，闹出不少笑话。趁着这个机会，先来回忆一下最基本编译知识吧。

## gcc 和 g++ 的区别

在linux系统上，从源文件到目标文件的转化都是由编译器完成的。先以hello.c程序的编译为例子：
```
gcc -o hello hello.c
```
gcc编译器读取源文件hello.c, 并把它翻译成为一个可执行文件hello。

这个翻译过程可以划分为成为四个阶段逐步执行：预处理，编译，汇编，链接，如下图所示：

![](/img/in-post/post-cgo-params/compiling.jpg)

gcc 最开始的时候是 GNU C Compiler, 就是一个c编译器。但是后来因为这个项目里边集成了更多其他不同语言的编译器，GCC就代表 the GNU Compiler Collection，所以表示一堆编译器的合集，g++则是GCC的c++编译器。

现在我们在编译代码时调用的gcc，已经不是当初那个c语言编译器了，更确切的说它是一个驱动程序，根据代码的后缀名来判断调用c编译器还是c++编译器 (g++)。比如我们的代码后缀是*.c，它会调用c编译器还有linker去链接c的library。如果我们的代码后缀是cpp, 它会调用g++编译器，当然library call也是c++版本的。

说了这么多，真是太混乱了。没关系，我们现在把gcc当成c语言的编译器，g++当成c++语言编译器用就好了。

其次，还有一些关编译工具的小知识(非权威，也是结合实际中遇到的问题以及大佬们的博客得出的)：
1. 如果编译器的版本大于gcc-4.8, 最好加上compiler-flag -std=c++11; 
2. 加上-Wall这个compiler flag, 让编译器报出所有可能的报警
3. 如果需要调试程序的话，请加上-g, 保留符号信息

# gcc生成可以执行文件的例子(C++版)

hello_world.cpp的源代码: 
```
#include <iostream>
int main(int argc,char *argv[])
{
    std::cout << "hello, world" << std::endl;
    return(0);
}
```
执行编译指令并执行编译文件, linux系统下的可执行文件都是elf格式的。 然后我们就可以得到熟悉的"hello, world"：

```
g++ hello_world.cpp -o hello_world_elf_file
./hello_world_elf_file
hello, world
```

##### Gcc-预处理（预处理器）

```
g++ -E hello_world.cpp -o hello_world.ii
```
以.cpp后缀结尾的是C++的源代码，这类代码一般需要进行编译前的预处理。 以.ii后缀结尾的也是C++的源文件，但这类型的源文件已经进行了预处理，其复制了include头文件中内容，并展开了宏定义的内容。

编译参数-E的作用是：将源代码用编译***预处理器***处理后不再执行其他动作。这里的helloworld.cpp 的源代码，仅仅有六行，而且该程序除了显示一行文字外什么都不做。但是，经过预处理后的版本却超过了1200行。这里主要是因为头文件iostream被包含进来，而且它又包含了其他的头文件。除此之外，还有若干个处理输入和输出类的定义。


##### Gcc-编译（编译器）

```
g++ -S hello_world.cpp -o hello_world.s
```

选项-S的作用就是指示编译器将程序编译成汇编语言，输出汇编语言的代码。

##### Gcc-汇编（汇编器）

```
g++ -C hello_world.cpp -o hello_world.o
```
-C的作用就是将hello_world.s翻译成机器语言保存在hello.o中，这是个二进制文件。

##### Gcc-链接（链接器）
```
g++ hello_world.o -o hello_world_elf_file
```
终于到了最后一个步骤，那就是连接目标代码，生成可执行文件。

重新来看一下这个小程序，在这个程序中并没有定义"cout"的函数。准确来说，cout这个函数有些特殊，用的是运算符重载，确切的说是重载了"<<"运算符。这里如果使用printf()函数会更好，但这里先把"cout"当作函数处理吧。

在编译预处理的过程中，iostream有该函数的声明，而没有函数的实现。那么，在哪里实现了"cout"函数呢？系统把这些函数的实现都放到名为stdc++的库文件中了。在没有特别的指定时，g++会默认到系统"／usr/lib"路径下进行查找，也就是链接到stdc++库函数中去。就可以实现"cout", 这就是链接的作用。

## 总结
1. 编译器的编译过程可以归纳为：
```
源文件－－>预处理－－>编译/优化－－>汇编－－>链接-->可执行文件
```
2. 这篇博客中对于如何链接动态库静态库没有做过多的阐述，下篇文章中，我们会就着golang使用swig插件来讲述动态库的使用。

## 引用
* [知识回顾 - C/C++ 编译常识](https://www.jianshu.com/p/fca4d54dff0c)
* [gcc编译过程简述](https://www.cnblogs.com/dfcao/p/csapp_intr1_1-2.html)
* [gcc g++支持C++11 标准编译及其区别](https://www.cnblogs.com/kex1n/p/7072092.html)
* [linux下编写c++，include的那些头文件在什么地方](https://v.godloveworld.com/shi-101567827.html)
* [linux下 C++的标准库头文件所在目录](https://blog.csdn.net/qq_27576655/article/details/81095747)
* [gcc和g++是什么关系？(知乎中李锋大佬的回答)](https://www.zhihu.com/question/20940822)











