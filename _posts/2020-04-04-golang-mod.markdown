---
layout: post
title: "golang的包管理工具(1)：vendor"
subtitle: "go vendor"
author: "zy"
header-img: "img/post-bg-golang.jpg"
catalog: true
tags:
  - Golang
---

> 武汉加油，中国加油！

## 引言
go的包管理工具经常受到人们的诟病，并且网上的博客经常一顿乱抄，让人使用起来一脸懵逼。前段时间遇到了一些坑，也看了不少相关的文档。现在，把这些知识，汇总起来，希望以后再也不要跳进这个坑里。

当然，自从go 1.12出来之后，go mod大行其道，已经成功事实上的官方标准。但是，我们以前的项目中还有不少是使用go vendor来管理的。在了解go mod的今生之前，我们有必要先来回顾一下go mod的前世：go vendor, 也方便对这些旧的项目进行改造。

## GOPATH和GOROOT

golang的初学者肯定会接触到这两个环境变量, 像笔者在最开始的时候就被他们绕晕了。

#### GOROOT并不是必须要设置的
安装go的时候，我们默认会安装在/usr/local/go这个路径下面，当然也允许自定义安装位置。GOROOT的目的就是告知go当前的安装位置，编译的时候从GOROOT去找SDK的system libariry。

如果需要更新golang的版本，比如说从go 1.13升级到1.14，我们可以参考[golang安装升级指南](https://golang.org/doc/install#uninstall)。

#### GOPATH必须要设置

GOPATH的目的就是为了告知go, 需要代码的时候，去哪里查找。注意这里的代码，包括本项目和引用外部项目的代码。GOPATH可以随着项目的不同而重新设置。

GOPATH下面会有3个目录结构：src, bin, pkg。src目录go编译时查找代码的地方,  bin文件是二进制文件下载的目的地, pkg是编译生成文件的lib文件存储的地方。

## 本项目的内部依赖

以kubernetes为例子，kubernetes/cmd/kubectl/kubectl.go中引用了app包中的kubectl.go，代码是这样写的：

```
import (
	"os"
	"k8s.io/kubernetes/cmd/kubectl/app"
)
```
那么go在编译的时候怎么查找这个包呢？

这就是GOPATH发挥作用的时候了。go编译时会去$GOPATH/src/目录去查找需要的代码，因此只要上面app/kubectl.go在$GOPATH/src/k8s.io/kubernetes/cmd/kubectl/里面，go编译的时候就能找到，那么自然的，kubernetes/cmd/kubectl/kubectl.go也需要放到$GOPATH/src/k8s.io/里去。最终$GOPATH里的代码结构是这样的：

```
├── src
│   ├── k8s.io
│   │   └── kubernetes
│   │       ├── cmd
│   │       │   ├── kubectl
│   │       │   │   ├── app
│   │       │   │   │   ├── BUILD
│   │       │   │   │   └── kubectl.go
│   │       │   │   ├── BUILD
│   │       │   │   ├── kubectl.go
│   │       │   │   └── OWNERS
```
当然，这种设计方式，就需要我们将我们正在开发的代码，放在$GOPATH/src的目录结构下。

## 管理外部的依赖包

不可避免的我们会使用外部的依赖包，go没有像java使用maven那样来管理依赖包，而是直接使用GOPATH来管理外部依赖。

golang允许import不同的代码库的代码，例如：github.com, k8s.io, golang.org等等。对于需要import的代码，可以使用go get 命令取下来放到$GOPATH/src的目录中去。例如go get github.com/silenceshell/hcache, 会下载到$GOPATH/src/github.com/silenceshell/hcache中去，当其他项目在import github.com/silenceshell/hcache的时候也就能找到对应的代码了。

看到这里也就应该明白了，对于go来说，其实并不care你的代码是内部还是外部的，总之都在GOPATH里面，任何import包的路径从GOPATH开始的，唯一的区别是，就是内部依赖的包都是由开发者自己开发的，但是外部依赖的包都是go get下来的。

## vendor

依赖GOPATH来解决go import有个很严重的问题：如果项目依赖的包做了修改，或者干脆删掉了，会影响我的项目。因此在1.5版本以前，为了规避这个问题，通常会将当前使用的依赖包拷贝出来。为了能让项目继续使用这些依赖包，我们可以有这么几个方法：

1. 将依赖包拷贝到项目的源码树中，然后修改import
2. 将依赖包拷贝到项目源码中，然后修改GOPATH
3. 在某个文件中记录依赖包的版本，然后将GOPATH中的依赖包更新至对应的版本（因为依赖包实际上是个git库，可以切换版本）

go作为一个现代化的语言，居然要用这么复杂并且不直观的又不标准的方法来管理依赖，难怪有好多人在前期非常不看好go的前景。

为了解决这个问题，go在1.5版本引入了vendor属性。简单来说，vendor属性就是让go在编译时，优先从项目的源码树根目录下的vendor目录查找代码。如果vendor中没有，再去GOPATH中查找。

以[kube-keepalived-vip](https://github.com/kubernetes-retired/contrib/tree/master/keepalived-vip)为例, 该项目会调用k8s.io/kubernetes的库(Client)，但如果你用1.5版本的kubernetes代码来编译keepalived，会编译不过：

```
./controller.go:107: undefined: "k8s.io/kubernetes/pkg/client/unversioned".Client
```
查下代码会发现1.5版本中代码有变化，已经没有这个Client了。这就是前面说的依赖GOPATH来解决go import所带来的问题，代码对不上了。

kube-keepalived-vip项目用vendor目录解决了这个问题：该项目把所有依赖的包都拷贝到了vendor目录下，对于需要编译该项目的人来说，只要把代码从github上clone到$GOPATH/src以后，就可以进去go build了(注意，必须将kube-keepalived-vip项目拷贝到$GOPATH/src目录中，否则go会无视vendor目录，仍然去$GOPATH/src中去找依赖包)。

但是vendor目录又带来了一些新的问题：

1. vendor目录中依赖包没有版本信息。这样依赖包脱离了版本管理，对于升级、问题追溯，会有点困难
2. 如何方便的得到本项目依赖了哪些包，并方便的将其拷贝到vendor目录下？这也太困难了吧？

## 总结

1. golang最开始的时候是使用GOPATH来使用管理包，后面加入了vendor属性
2. 针对上面提出的vendor的问题，我们后面可以看go mod是怎么解决的

## 引用
* [go依赖包管理工具对比](https://ieevee.com/tech/2017/07/10/go-import.html#goroot%E5%B9%B6%E4%B8%8D%E6%98%AF%E5%BF%85%E9%A1%BB%E8%A6%81%E8%AE%BE%E7%BD%AE%E7%9A%84)
* [golang安装升级指南](https://golang.org/doc/install#uninstall)





