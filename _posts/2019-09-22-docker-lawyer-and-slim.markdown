---
layout: post
title: "docker的镜像分层原理，以及镜像的瘦身和优化"
subtitle: "docker image lawyer and docker-slim"
author: "zy"
header-img: "img/post-bg-golang.jpg"
catalog: true
tags:
  - docker
---

## 引言
在公司的k8s课程上，Lyle老师讲到了如何去为docker镜像瘦身；在项目重构的过程中，自己也意识到了要对镜像进行优化，从而加快构建的速度。透过现象看本质，我们先来为熟悉一下镜像分层的原理，然后根据这部分知识去对我们的镜像进行瘦身和优化。

## 关于基础镜像

base镜像有两层含义：

1. 不依赖于其他镜像，比如我们可以从ubutun 16.04开始构建
2. 其他镜像可以之为基础进行扩展，比如我们可以安装curl, telnet, vim等调试工具 

所以，能被称作base镜像的通常都是各种Linux发行版的docker镜像，比如说Ubutun，Debian, CentOS等。逆向来说，base镜像提供了最小安装的Linux发行版。我们来看看我们常用的ubuntu镜像的大小：
```
$ docker images ubuntu                                                                                               
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
ubuntu              18.04               a2a15febcdf3        5 weeks ago         64.2MB
```
我们神奇地发现，tag为18.04的ubuntu镜像居然只有64.2MB，这也太小了吧？这是因为docker镜像在运行时直接使用了宿主机器的Kernel。

Linux的操作系统是由内核空间和用户空间组成的。内核空间是kernel, 用户空间是rootfs, 不同Linux发行版的区别主要是rootfs。比如ubuntu 使用upstart管理服务，apt管理软件包；而centOS7使用systemd和yum。这些都是用户空间上的区别，Linux Kernel差别不大。所以Docker可以同时支持多种镜像，从而模拟出多种操作系统环境。

正如上文所说的一样，base镜像只是用户空间和发行版一致。Kernel使用的是docker宿主机器的Kernel。例如 CentOS 7 使用 3.x.x 的 kernel，如果 Docker Host 是 Ubuntu 18.04，那么在 CentOS 容器中使用的实际是 Host 4.x.x 的 kernel。

## docker镜像的分层结构

通过镜像id, 我们可以知道镜像的历史：

```
$ docker history ec4fc3135ce9                                                                                                                    ‹ruby-2.4.1›
IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
ec4fc3135ce9        7 days ago          /bin/sh -c #(nop)  ENV CLUSTER=INTERNAL         0B
a495a434a35a        7 days ago          /bin/sh -c #(nop)  ENV ENV=FAT                  0B
b429f5a4bc68        7 days ago          /bin/sh -c #(nop)  ENV APOLLO=9442b3b…          0B
694f64c1c338        7 days ago          /bin/sh -c #(nop) COPY multi:67bab49383a7589…   25.2MB
5ba3201923e7        7 days ago          /bin/sh -c #(nop) COPY dir:6e2f661316dddc5ea…   28.2MB
c9c040be36a5        8 days ago          /bin/sh -c #(nop) WORKDIR /go/src/code.buptop…   0B
b478d37611c7        8 days ago          /bin/sh -c apt-get update && apt-get install…   180MB
a2a15febcdf3        5 weeks ago         /bin/sh -c #(nop)  CMD ["/bin/bash"]            0B
<missing>           5 weeks ago         /bin/sh -c mkdir -p /run/systemd && echo 'do…   7B
<missing>           5 weeks ago         /bin/sh -c set -xe   && echo '#!/bin/sh' > /…   745B
<missing>           5 weeks ago         /bin/sh -c [ -z "$(apt-get indextargets)" ]     987kB
<missing>           5 weeks ago         /bin/sh -c #(nop) ADD file:c477cb0e95c56b51e…   63.2MB
```
当我们执行docker run去启动镜像的时候，一个新的可写层会加载到镜像的顶部。这一层通常被称为容器层，往下就是镜像层。容器层可以读写，容器中所有文件的变更都发生在这一层。镜像层是read-only的，只允许读取。下图来自于官方文档，和我本地用到的镜像有些不一样，但是原理都是一致的。
![](/img/in-post/post-docker-image/container-layers.jpg)

第一列是image id, 最上面的是我们加载的环境变量CLUSTER=INTERNAL，下面几行都是我们dockerfile里面定义的步骤堆栈。由此可以看出，每个步骤都将创建一个img id，一直追溯到a2a15febcdf3，这就是我们的基础镜像。关于`<missing>`的部分，说明这些层不在我们的主机上。

最后一列是每一层的大小。我们在dockerfile里面的最后一个步骤是导入环境变量，不涉及文件的变更，这一层的大小是0。所以我们创建的镜像是在base镜像之上的，并不是完全复制一份，而是共享base的内容。这时候，如果我们新建一个镜像，同样是共享base镜像。

## docker的copy-on-write (CoW)策略

docker通过一个叫做copy-on-write (CoW)的策略来保证base镜像的安全性，以及更高的性能和空间利用率。这个策略，简单来说，就是在启动容器的时候，最上层容器层是可写层，之下都是镜像层。

##### 从容器中读取文件

从最上层镜像开始查找，往下找，找到文件后读取并放入内存。如果已经在内存（宿主机内存）中，可以直接使用。也就是说，在同一台机器上运行的docker容器共享运行时相同的文件。

##### 向容器中添加文件

上文已经说过，容器里面只有容器层是可写的，添加的文件不会影响镜像层。

##### 容器中修改文件

这个时候就会真正用到CoW的策略。修改文件时，会从上往下寻找文件，找到后复制到容器层。对容器的使用用户来说，看到的是镜像层里面的文件，也就是在容器层修改这个文件。

##### 容器中删除文件

从上往下寻找文件，找到后在容器中记录删除，其实只是软删除。软删除会导致镜像的体积增加，不会减少。但如果我们在同层删除文件，是可以有效地减少镜像体积的。下文中会给出示例。

综上，docker镜像的通过分层实现了资源共享， 通过copy-on-write实现了文件隔离。

## 为镜像瘦身的意义
上文铺垫了这么多，我们最终的目的还是要为镜像瘦身。但是瘦身的收益有哪些呢？我们也可以了解一下：
1. 大镜像往往意味着要安装更多的依赖，有时候构建的时间过长，严重影响了开发的效率。小镜像利于我们加快构建和部署的速度。
2. 越小的镜像意味着无用的程序越少，被攻击的目标越小，提高安全性，减少被攻击的风险。
3. 虽然现在硬盘的存储已经很廉价了，企业购买云硬盘，1T的成本也就几块钱。但蚊子再小也是肉呀，还是要珍惜存储开销。

## 瘦身大法
说了这么多，我们终于要进入正题，下面就通过四个点，来总结一下我们的瘦身大法：

##### 选择一个小的基础镜像

![](/img/in-post/post-docker-image/image1.png)
  
我们一般会选择Alpine这个发行版，体积适中，而且包含包管理器和shell环境。

##### 安装依赖和清除缓存的命令在同一层操作

我们先来看一个例子：
```
# dockerfile 1
FROM alpine
RUN wget https://a.com/1.0.0.zip

# dockerfile 2
FROM alpine
RUN wget https://a.com/1.0.0.zip
RUN rm 1.0.0.zip

# dockerfile 3
FROM alpine
RUN wget https://a.com/1.0.0.zip && rm 1.0.0.zip

test 3 351a80e99c22 5 seconds ago 5.53MB
test 2 ad27e625b8e5 49 seconds ago 6.1MB
test 1 165e2e0df1d3 About a minute ago 6.1MB
```
其实我们大多数人都有清除缓存的习惯，但需要知道的是，只有同一层执行命令去删除文件才能减少镜像体积。

同理，除了删除语句要放在同一层外，我们一些用于安装依赖的公共的语句也可以放在同一层，减少最终的层数。

##### 多阶段构建
分离编译和构建是golang里面减少镜像体积的大杀器：

```
# 使用golang镜像作为builder镜像   
FROM golang:1.12 as builder

WORKDIR /go/src/github.com/go/helloworld/

COPY app.go .

RUN go build -o app .

#编译完成之后使用alpine作为基础镜像            

FROM alpine:custom as prod

RUN apk --no-cache add ca-certificates WORKDIR /root/
# 从builder中复制编译好的二进制文件
            
COPY --from=builder 

/go/src/github.com/go/helloworld/app .

CMD ["./app"]
```  

##### 可以瘦，但不要太瘦了

有很多人喜欢极小极简洁的镜像，甚至连一些基本的调试工具都没有，比如说curl, ping, telnet等。在一些特殊情况下，这些工具可以帮助我们快速debug, 极大地提高效率。

## 总结

综上，docker镜像的通过分层实现了资源共享，通过copy-on-write实现了文件隔离。然后我们又对镜像的瘦身原则做了一些归纳总结，简单来说，就是减少层，并且减少层的大小，同时要兼顾我们调试时的效率。

## 引用

* [理解Docker镜像分层（这个老哥写得很好，就是排版有点乱）](https://www.cnblogs.com/woshimrf/p/docker-container-lawyer.html)
* 还有Lyle老师的博客，我实在是找不到了，下次补上吧















