---
layout: post
title: "Golang中的字符编码与字符数据类型"
subtitle: "string, rune, byte in Golang"
author: "zy"
header-img: "img/post-bg-golang.jpg"
catalog: true
tags:
  - OS
  - Golang 
---

> Go!

## 引言
在这个时间点，其实我已经读过写过"不少"Golang代码（我吹的）。但相信许多人和我一样，对于string, rune , byte这三个字符数据类型还没有那么了解，持着一种"又不是不能用的态度"。在写作的过程中，需要将一些很基本的技术点用通俗易懂的语言表达出来，上文提到的态度会严重影响文章的质量，甚至令读者产生误解，这不是写作的初衷。故写下这篇基础的文章，抛砖引玉。

## 操作系统中的字符编码

##### ASCII码
上个世纪60年代，美国制定了一套字符编码，对英语字符与二进制位之间的关系，做了统一规定。这被称为ASCII码，一直沿用至今。 
ASCII码一共规定了128个字符的编码，比如空格”SPACE”是32（二进制00100000），大写的字母A是65（二进制01000001）。这128个符号（包括32个不能打印出来的控制符号），只占用了一个字节的后面7位，最前面的1位统一规定为0。
简单一点来说，就是使用**单字节的整数来表示英文字符，我们称之为字符的ASCII码表示**。以后有人问你，"A"的ASCII码是什么，直接回答65就好了。

##### Unicode字符集
在英文中，仅仅用128个符号就足够了，但是在其他文字中是远远不够的。这时候，Unicode出现了。Unicode将全世界的字符都纳入其中，每一个字符都有独一无二的编码。比如，U+0639表示阿拉伯字母Ain，U+0041表示英语的大写字母A，U+4E25表示汉字”严”。

这时候，我们来思考一个问题？表示中文的"严"至少需要两个字节，但是英文字符字符只需要一个字节。如果英文字符的前面字节全部变为0，就会造成极大的浪费。所以造成了Unicode字符集在世界上有多种存储方式，而且迟迟无法推广，直到互联网的兴起才解决了这个问题。
                                               

##### UTF-8
utf-8编码的出现，几乎统一整个互联网，形成了事实上标准。简单来说，UTF-8就是在互联网上使用最广的一种Unicode的实现方式。


## String

Go语言中，string就是：读取使用utf-8编码的字节切片(slice)。因此用len函数获取到的长度并不是字符个数，而是字节个数。for循环遍历输出的也是各个字节:

```
a := "Randal";
for i := 0; i < len(a); i++ {
 fmt.Printf("%x ", a[i])
 fmt.Printf("%c ", a[i])
}
// 输出结果
52 61 6e 64 61 6c
Randal

```
```
a := "中国";
fmt.Println(len(a))
for i := 0; i < len(a); i++ {
 fmt.Printf("%x ", a[i])
}
for i := 0; i < len(a); i++ {
 fmt.Printf("%c ", a[i])
}

// 输出结果
6
E4 B8 AD E5 9B BD
ä¸­å½
```

但我们在对中文使用格式化输出的时候，出现了乱码。原因是当字符的utf-8编码超过1个字节的时候，格式化输出单个字符就会出现乱码的情况，rune可以帮助我们解决乱码的问题。

## rune

rune是int32的别名，代表字符的Unicode码，采用4个字节存储。将string转成rune就意味着任何一个字符都用4个字节来存储其unicode码，这样每次遍历的时候返回的就是unicode码，而不再是字节了，这样就可以解决乱码问题。

```
var s string
   s = "中国"
   r := []rune(s)
   for i := 0; i < len(r); i++ {
     fmt.Printf("%x", r[i])
   }
   for i := 0; i < len(r); i++ {
     fmt.Printf("%c", r[i])
   }
  // 输出结果
  4e2d 56fd
  中国
```
通过for range对字符串进行遍历时，每次获取到的对象都是rune类型的，因此下面的方式也可以解决乱码问题。
```
   var s string
   s = "中国"
   for _, item := range s {
     fmt.Printf("%c", item)
   }
   // 输出结果
   中国
```

## bytes

byte操作的对象也是字节切片，与string的不可变不同，byte是可变的。Go的字符串是utf-8编码的，每个字符长度是不确定的，一些字符可能是1、2、3或者4个字节结尾。

```
   s1 := "abcd"
   b1 := []byte(s1)
   fmt.Println(b1) // [97 98 99 100]

   s2 := "中文"
   b2 := []byte(s2)
   fmt.Println(b2)  // [228 184 173 230 150 135], unicode，每个中文字符会由三个byte组成
```

## 总结
这个博文是站在巨人的肩膀上完成的，因为严重参考了前辈们的分享，感谢伟大的互联网。

## 参考

* [Go系列 string、bytes、rune的区别](https://juejin.im/post/5c1a2db5f265da61682b52f5)
* [Go String与Byte切片之间的转换](https://github.com/jemygraw/TechDoc/blob/master/Go%E7%A4%BA%E4%BE%8B%E5%AD%A6/Go%20String%E4%B8%8EByte%E5%88%87%E7%89%87%E4%B9%8B%E9%97%B4%E7%9A%84%E8%BD%AC%E6%8D%A2.markdown)
* [深入理解计算机系统之字符编码](https://blog.csdn.net/zyhmz/article/details/55657773)










