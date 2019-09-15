---
layout: post
title: "cgo中链接c的动态库"
subtitle: "Using C Dynamic Libraries In Go Programs"
author: "zy"
header-img: "img/post-bg-golang.jpg"
catalog: true
tags:
  - Golang
  - C语言
---

## 引言

cgo在使用c/c++资源的时候一般有三种形式：1.直接使用源码 2.动态链接库 3. 静态链接库。直接使用的源码就是在import "C"之前的注释部分包含c代码，或者在当前的包中包含c/c++源文件。链接静态库和动态库的方式比较类似，都是通过在LDFLAGS中指定要链接的库。在项目中的动态链接库用得比较多，所以我们就着重来介绍一下如何去链接动态库，以及其中会遇到的坑。

## LDFLAGS和CFLAGS

##### 举个栗子
```
package controller

/*
#cgo LDFLAGS: -L ${SRCDIR}/bupt/msg  -Wl,-rpath ${SRCDIR}/bupt/msg -lactive
#cgo CFLAGS: -I ${SRCDIR}/bupt/msg
#include <stdint.h>
#include <stdlib.h>
#include "active_server.h"
*/
import "C"

func main() {
  outSize := (*C.uint)(unsafe.Pointer(&(make([]byte, 4)[0])))
  if C.active_msg_handler(outSize) != 0 {
     panic("error")
  }
}
```

先看一下LDFLAGS的参数使用。${SRCDIR}指的是当前文件所在的路径，${SRCDIR}/bupt/msg指的是存在动态链接库的文件夹所在的路径。项目中的动态库的文件名为active.so, 所以很明显-lactive的意义就是去链接active.so这个动态链接库。

-L和-Wl,-rpath后面都跟了同样的参数，这又有什么区别呢？-L这段指令是用于指定动态库的目录。编译时指定-L目录，只是在程序链接成可执行文件时使用，但是在运行程序的时候，并不会去指定的目录寻找动态库，这时候就需要使用-Wl,-rpath指令了。这块牵涉到链接器和动态链接器的区别，其中奥义，还需要细细品味。

CFLAGS的使用方法很简单，就是加载头文件所有的路径，然后将头文件include进来。接下来我们就可以直接使用c库中的接口方法，传入参数后，处理返回结果就大功告成了。

##### 相对路径和绝对路径

在上文的例子中，我们的参数都使用了绝对路径，那么我们可以使用相对路径吗？在CFLAGS中，我们是可以使用相对路径的。因为历史遗留问题，LDFLAGS不支持相对路径，我们必须为链接库指定绝对路径。

##### 一个项目中同时链接多个动态库

在这种场景下，我们应该注意的一点是，每个动态库的命名都应该是不同的。在做项目重构的时候，旧服务的动态链接库的命名都是一致的，导致运行的时候出现了问题。

## 访问c中的struct

cgo中传入参数的时候，我们经常要访问struct，将结构体指针作为参数传输。使用的方式为：`C.struct_<struct的名字>`，记得释放内存。

```
cData := (*C.struct__log_export_header_)(C.CBytes(data)) //data为入参
defer C.free(unsafe.Pointer(cData))
```

## 总结

本文对cgo中链接动态库做了一个简单的介绍，针对其中遇到过的坑也做了总结。不足的时候对于编译链接的理解并不是很深刻，也仅仅是停留在可以使用的程度。后面会读一读《程序员的自我修养-链接、装载与库》这本书，针对其中的盲区会再做总结。


## 引用

* [CGO简明教程](https://juejin.im/entry/5b5089e3e51d45198565aaa2)
* [CGO官方指南](https://golang.org/cmd/cgo/#hdr-Go_references_to_C)
* [cgo使用需要注意的一些问题](https://www.bandari.net/blog/24)
* [还是chai大佬的go语言圣经](https://chai2010.cn/advanced-go-programming-book/ch2-cgo/ch2-10-link.html)
