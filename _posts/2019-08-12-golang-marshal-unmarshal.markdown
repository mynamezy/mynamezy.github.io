---
layout: post
title: "Golang的序列化反序列化之路--Binary，Protobuf"
subtitle: "Marshal and Unmarshal"
author: "zy"
header-img: "img/post-bg-golang.jpg"
catalog: true
tags:
  - Golang
---

> 纸上得来终觉浅，绝知此事要躬行

## 引言

在前面的《JSON到底是什么?》一文中，我们了解到了关于json的序列化和反序列化方法。最近在重构一个和c混编的项目，觉得自己对golang的序列化反序列化有了一些粗浅的了解，遂写下此文。

## 序列化和反序列化是啥？

掌握某个技术的使用技巧之前，我们最好先知道这块技术到底是干什么的？或者说这块技术是用来解决什么问题的？

我想每个开发小伙伴应该都知道，如果我们想要对数据做持久化存储，就必须将存在IO设备中。这些IO可以是U盘，机械硬盘，移动硬盘等等。但是这些数据是以哪种方式进行存取的呢？我们必须要制定数据的传输格式，才能实现数据的存取。

我们伟大的前辈们，在很早之前就设计了很多精致优雅的数据结构，来帮助我们提高运行和存储的效率。但是这些数据结构要在网络中传输或者保存到文件中，就必须对其进行编码和解码，这就是我们所说的序列化和反序列化。

这些编码格式可以是字符形式的文本格式，或二进制数据形式的压缩格式。字符形式的文本格式占用的存贮空间多但透明度高，二进制数形式的压缩格式占用的存贮空间少但缺少透明度。Json，Base64的编码格式是很常见的，本来就重点介绍一下Binary 和 Protobuf。


## Binary

对binary库有一种相见恨晚的感觉，它可以实现结构体实例和byte数组的转换。binary库基本对应了rails中BinData gem包，但是golang的binary库更清晰直观，让人可以一看就理解本质。
下面是一个简单的例子，我们首先对一个结构体进行了赋值，然后根据LittleEndian对数据进行编解码：

```
type Message struct {
	Id   uint8
	Size uint8
}

func BinaryRW() {
    m1 := Message{1, 1024}
    buf := new(bytes.Buffer)

    if err := binary.Write(buf, binary.LittleEndian, m1); err != nil {
        log.Fatal("binary write error:", err)
    }

    var m2 Message
    if err := binary.Read(buf, binary.LittleEndian, &m2); err != nil {
        log.Fatal("binary read error:", err)
    }
}
```

在我的项目中，我们更多的使用binary来读数据（反序列化）：

```
func BinaryR() {
    res := Message{}
    data = []byte{1,16}
    
    if err := binary.Read(bytes.NewBuffer(data), binary.LittleEndian, &res); err != nil {
        log.Fatal("binary read error:", err)
    }
}
```
## Protobuf
终于来到了大名鼎鼎的protobuf。这是由google发明的用于跨平台，跨语言的结构化数据存储格式，使用范围非常广。安装的过程我们就不再赘述了，google一把就都有了。

总的来说，protobuf可以分为两部分：一是编译器protoc，二是分包组包所用到的库。

编译器就是用来编译proto文件的，然后生成对应目标语言的文件。分包组包的库主要帮助我们获取对应的结构体实例。多说无益，我们先来看一个例子吧，如果不是特别声明，golang一般使用的proto2的语法。

```
// product.pd.go
package protobuf;

message Product {
  optional int32 version = 1;
  enum EncryptionType {
        NONE = 0;
        AES = 1;
        CUSTOM = 2;
  }
  optional EncryptionType encryption = 2;
  enum PlatformType {
        SERVER = 0;
        ANDROID = 1;
        IOS = 2;
        PC = 3;
  }
  optional PlatformType platform = 3;
}  
```
在对应的路径下执行protoc --go_out=. *.proto，就可以获得product.pd.go文件。在这里，我们可以看到enum的使用方式。

我们来看一个序列化和反序列化的栗子：

```
func protoSeDe() {
    server := protobuf.Product_SERVER
    seData := &protobuf.Product{
       Version:  proto.Int32(1),
       Platform: &server,
    }
	
    data, err := proto.Marshal(seData)
    if err != nil {
    	log.Fatal("unmarshaling error: ", err)
    }
    
    deData := &protobuf.Product{}
    err := proto.Unmarshal(data, deData)
    if err != nil {
    	log.Fatal("unmarshaling error: ", err)
    } 
}
```
这个栗子中有一个需要注意的点，那就是在Product{}实例初始化时对枚举值的使用。Product_SERVER在protobuf包中是一个常量，基于proto2的语法，我们在结构体初始化的时候，需要先赋值再引用。而在proto3的语法中，我们可以直接使用。

## 总结
纸上得来终觉浅，绝知此事要躬行。两个例子看似很简单，但还是花了我不少时间和精力去查阅资料，不过感觉还是有所收获的。

其次，这里并没有具体介绍proto的语法，protobuf应该还有很多更高级的功能可以使用。

## 引用

* [使用Google Protocol Bufffers进行通信(Ruby & C)](https://blog.csdn.net/zyhmz/article/details/80633339) 这是我以前的博文，对proto的语法的使用做了简单的总结，建议找个官方的文档
* [Golang中使用protobuf高效编码](https://yryz.net/post/go-protobuf-usage.html) 
* [proto2中使用enum的stack overflow答疑](https://stackoverflow.com/questions/50464205/how-to-set-a-protobuf2-enum-using-golang) 
* [golang数据传输格式-序列化与反序列化](https://www.cnblogs.com/yinzhengjie/p/7807051.html)
* [Golang 序列化方式及对比](https://blog.csdn.net/fengfengdiandia/article/details/79986237)

























