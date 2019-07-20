---
layout: post
title: "利用Golang和Ruby写二进制文件"
subtitle: "Writing Binary file in Golang and Ruby"
author: "zy"
header-img: "img/post-bg-golang.jpg"
catalog: true
tags:
  - Ruby
  - Golang 
---

> stay foolish.

## 引言
这两天和嵌入式的老哥打交道，免不了要读写二进制文件。本来想着是一些很简单的操作，谁知道花了不少时间。原因还是自己对一些方法和概念了解得没有那么到位，遂写下这篇博文，以便于后面review。目前的项目是用ruby写的，所以我们先来说说ruby的一些概念。


## ruby中的 Array#pack 和 String#unpack
ruby中的pack和unpack方法和python是很类似的，国内的很多博客，一上来就是对pack和unpack的使用方法做一通介绍。但是**最本质**的点却都没有提到，这两个方法到底是用来干嘛的？

C语言中通常允许开发者去操作内存（虽然我早就忘了怎么去操作），但是ruby和golang却是不允许的。在日常的工作中，我们可能经常需要去操作底层的bits和bytes，ruby提供了这两个方法方便我们去操作。

##### MSB vs LSB

MSB: 大端模式，数据的高字节存储在低地址中，而数据的低字节则存放在高地址中。

LSB: 小端模式，数据的低位保存在内存的低地址中，而数据的高位保存在内存的高地址中。

##### unpack的简单使用
在下面的这个case中，我们想获取字符串'A'在操作系统中的二进制表示。通过查看ASCII表，发现十进制的65的代表字符串'A'，所以其二进制表示为01000001。
```
'A'.unpack('b*')
=> ["10000010"]
```
b*和B*代表了操作系统中不同的字节存储顺序（MSB，LSB），所以unpack的不同参数获得的二进制表示是不一样的。
```
'A'.unpack('B*')
=> ["01000001"]
```

##### ['hex value'].pack('H*')
pack和unpack的教程很多，在这里就不用一一赘述了，具体的教程可以看文末的参考链接。我们在这里提一个比较常用的使用方法，但是绝大多数教程都没有涉及。

['hex value'].pack('H*') 这种方法可以将十六进制字符串真正地转化为二进制表示，写入文件中就可以成为二进制文件。在下面的case中，'AB'并不是ASCII表示，而是代表了hex AB。参数'H\*'就是告诉pack方法将输入的值作为hex来看待。

```
['AB34'].pack('H*')
=> "\xAB4"
```
'\x'符号表示这个值是一个十六进制的表示，所以'\xAB'还是很容易理解的，但是紧跟的字符'4'呢？通过查看ASCII码表，我们可以发现，hex 34（十进制52）转换为可显示字符'4'，现在就不难理解pack方法后的输出的字符串了。
同理，我们可以对'0a'做一下pack操作, hex 0a转换为ASCII控制字符'\n'：
```
['0a'].pack('H*')
 => "\n"
```

## ruby读写二进制文件

理解了上面的['hex value'].pack('H*')使用方法，利用ruby读写二进制文件就很简单了：

```
## write
test_key = "80747a332714f4d8f03a0f6d27180a13"
File.open("/user/root_key.bin", 'wb') do |output|
   output.write [test_key].pack("H*")
end

## read
File.open("/user/root_key.bin","rb") do|fp|
   int=fp.sysread(2) # 读取两个字节
   p int  # "\x80t"
   p int.unpack('H*') # ["8074"]
end
```


## golang写二进制文件

golang的操作就直接很多，利用DecodeString将string变成bytes就可以：
```
func main() {
  wireteString := "80747a332714f4d8f03a0f6d27180a13"
  key, _ := hex.DecodeString(wireteString) // string ---> []bytes
  fmt.Printf("[key test]: %x\n", key) //80747a332714f4d8f03a0f6d27180a13
  err := ioutil.WriteFile("/user/root_key.bin", key, 0666) //写入文件(字节数组)
  fmt.Println(err)
}
```

## vim打开文件乱码

可以看到ruby和golang生成的二进制文件是一样的的（通过计算md5），但是用vim打开后出现了乱码。面对这种情况，我们可以使用vim -b filename打开文件，然后输入:%!xxd命令转换为十六进制显示，输入:%!xxd -r返回原显示文件。
```
:%!xxd
=> 00000000: 8074 7a33 2714 f4d8 f03a 0f6d 2718 0a13  .tz3'....:.m'...
```
## 总结

操作系统的字符编码的知识很基础，但有时候又是非常困难的。比如说我现在还是搞不明白为啥用vim打开文件会出现乱码？hex后面一堆乱七八糟的格式是什么意思？感觉还是有很多东西要去归纳总结。但无论如何，总算是对ruby和golang如何去从操作底层的比特流有了大概的梳理，算是消灭了一部分知识盲区吧。

这篇博文大篇幅是在讲ruby，golang只是占了很小一部分。这么做原因有两个：一是目前的项目是用ruby写的，后面要用golang去重构，需要在平时思考总结的时候做一些知识储备；二是利用两种不同的方式去实现同一种功能是一件很有趣的事情，它们之间可以相互佐证，丰富你看问题的角度。

## 参考

* [Ruby pack unpack](https://blog.bigbinary.com/2011/07/20/ruby-pack-unpack.html)
* [pack模板字符串](http://www.kuqin.com/rubycndocument/man/pack_template_string.html)
* [Golang简单写文件操作的四种方法](https://www.cnblogs.com/shiluoliming/p/8312928.html)
* [Linux查看二进制文件](https://blog.csdn.net/weixin_34174132/article/details/86039002)
