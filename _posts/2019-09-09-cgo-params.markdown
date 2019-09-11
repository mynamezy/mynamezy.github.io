---
layout: post
title: "cgo的参数传递"
subtitle: "summary of cgo parameters"
author: "zy"
header-img: "img/post-bg-golang.jpg"
catalog: true
tags:
  - Golang
  - C语言
---

## 参数
在涉及到跨语言的调用时，参数的定义和转换可以算是一门学问。我们需要对两种语言的参数都有所了解，注意内存资源的申请和释放，注意类型转换的规则等细节。下面我们就着参数传递来说一说，使用cgo的时候会遇到哪些坑。

## c语言的指针变量的声明

#### 定义指针变量
仿佛又回到了最初的起点，大一的时候指针真是学得一塌糊涂。趁着这个机会，一起来回顾一下基础的知识，亡羊补牢，为时不晚。

在c语言中，变量的地址往往是编译系统自动分配的。对于我们用户来说，并不知道某个变量的具体地址。所以我们定义一个指针变量p, 把普通变量a的地址直接赋予指针变量p。有如下的两种写法：


```
## 定义时直接进行初始化赋值
unsigned char a;
unsigned char *p = &a;
```
```
## 定义后再进行赋值
unsigned char a;
unsigned char *p;
p = &a;
```
大家仔细看会看出来这两种写法的区别，它们都是正确的。我们在定义的指针变量前边加了个`*`，这个 `*p` 就代表了这个 p 是个指针变量，不是个普通的变量，它是专门用来存放变量地址的。此外，我们定义`*p`的时候，用了unsigned char来定义，这里表示的是这个指针指向的变量类型是unsigned char型的。

#### 字符串指针（指向字符串的指针）

在c语言中，没有像golang, ruby等面向对象的语言一样，有特定的字符串类型，通常就是字符串放在一个数组里面：

```
#include<stdio.h>
#include<string.h>
 
void main(){
    char str[] = {80, 81, 82, 83}; // str[] = "PQRS"
	int len = strlen(str),i;
	for (i = 0; i < len; i++) {
		printf("%c", str[i]);
	}
    printf("\n");
	char *p = str;
	for (i = 0; i < len; i++) {
		printf("%c", *(p++));
	}
        printf("\n");
        for (i = 0; i < len; i++) {
		printf("%c", *(str+i));
	}
}

## 运行结果：
PQRS
PQRS
PQRS
```
在上面的示例代码中，字符串数组也可以被认为是一个指针。除了字符数组之外，c语言还支持使用一个指针指向字符串的方式来表示字符串（表述有点拗口，但理解应该不难）。为什么可以用字符数组表示字符串，非要弄个指针来表示字符串呢？

它们最根本的区别是在内存中的存储区域不一样，字符数组存储在全局数据区或栈区，而以指针形式表示的字符串却存储在常量区。全局数据区和栈区的字符串（也包括其他数据）有读取和写入的权限，而常量区的字符串（也包括其他数据）只有读取权限，没有写入权限。一句话概括：数组形字符串存放在全局数据区或栈区，可读可写。指针字符串存放在常量区，只读不能写。
                                                                                                                                                  
## 参数传递

#### 基本数值类型

golang中基本数值类型的内存模型和c语言是一样的，就是连续几个字节（1／2／4／8字节）。因此传递数值类型时可以直接将golang的基本数值类型转换成对应的cgo类型然后传递给c函数使用，反之亦然：

```
package main
/*
#include <stdint.h>
static int32_t add(int32_t a, int32_t b) {
	return a + b;
}
*/
import "C"
import "fmt"
func main() {
	var a, b int32 = 1, 2
	var c int32 = int32(C.add(C.int32_t(a), C.int32_t(b)))
	fmt.Println(c) // 3
}
```
golang和c的基本数值类型转换表对照表如下：

<table>
   <tr>
      <td>C语言类型</td>
      <td>CGO类型</td>
      <td>Go语言类型</td>
   </tr>
   <tr>
      <td>char</td>
      <td>C.char</td>
      <td>byte</td>
   </tr>
   <tr>
      <td>singed char</td>
      <td>C.schar</td>
      <td>int8</td>
   </tr>
   <tr>
      <td>unsigned char</td>
      <td>C.uchar</td>
      <td>uint8</td>
   </tr>
   <tr>
      <td>short</td>
      <td>C.short</td>
      <td>int16</td>
   </tr>
   <tr>
      <td>unsigned short</td>
      <td>C.ushort</td>
      <td>uint16</td>
   </tr>
   <tr>
      <td>int</td>
      <td>C.int</td>
      <td>int32</td>
   </tr>
   <tr>
      <td>unsigned int</td>
      <td>C.uint</td>
      <td>uint32</td>
   </tr>
   <tr>
      <td>long</td>
      <td>C.long</td>
      <td>int32</td>
   </tr>
   <tr>
      <td>unsigned long</td>
      <td>C.ulong</td>
      <td>uint32</td>
   </tr>
   <tr>
      <td>long long int</td>
      <td>C.longlong</td>
      <td>int64</td>
   </tr>
   <tr>
      <td>unsigned long long int</td>
      <td>C.ulonglong</td>
      <td>uint64</td>
   </tr>
   <tr>
      <td>float</td>
      <td>C.float</td>
      <td>float32</td>
   </tr>
   <tr>
      <td>double</td>
      <td>C.double</td>
      <td>float64</td>
   </tr>
   <tr>
      <td>size_t</td>
      <td>C.size_t</td>
      <td>uint</td>
   </tr>
</table>

但是需要注意的是，c中的int在标准中是没有定义具体字长的，我们一般认为是4字节。cgo中C.int类型则明确定义了其字长为4，golang中的字长为8。因此C.int对应的golang类型不是int而是int32。为了避免误用，C代码最后使用C99标准的数据类型，对应的转换关系如下：

<table>
   <tr>
      <td>C语言类型</td>
      <td>CGO类型</td>
      <td>Go语言类型</td>
   </tr>
   <tr>
      <td>int8_t</td>
      <td>C.int8_t</td>
      <td>int8</td>
   </tr>
   <tr>
      <td>uint8_t</td>
      <td>C.uint8_t</td>
      <td>uint8</td>
   </tr>
   <tr>
      <td>int16_t</td>
      <td>C.int16_t</td>
      <td>int16</td>
   </tr>
   <tr>
      <td>uint16_t</td>
      <td>C.uint16_t</td>
      <td>uint16</td>
   </tr>
   <tr>
      <td>int32_t</td>
      <td>C.int32_t</td>
      <td>int32</td>
   </tr>
   <tr>
      <td>uint32_t</td>
      <td>C.uint32_t</td>
      <td>uint32</td>
   </tr>
   <tr>
      <td>int64_t</td>
      <td>C.int64_t</td>
      <td>int64</td>
   </tr>
   <tr>
      <td>uint64_t</td>
      <td>C.uint64_t</td>
      <td>uint64</td>
   </tr>
</table>


#### unsafe.Pointer

我们在进行Golang的学习的时候，应该听过这么一句话：Go的指针不支持指针转换。但这是为什么呢？

Golang作为一门静态语言，所有的变量都必须为标量类型，也就是说不同的类型不能进行赋值，计算等跨类型操作。 那么指针也对应着相对的类型，也在Compile的静态类型检查的范围内。同时静态类型语言，也称为强类型，也就是一旦定义，就不能再改变它。下面我们看一个错误示例：

```
func main(){
    num := 5
    numPointer := &num

    flnum := (*float32)(numPointer)
    fmt.Println(flnum)
}

# command-line-arguments
...: cannot convert numPointer (type *int) to type *float32
```
在示例中，我们创建了一个 num 变量，值为 5，类型为 int。取了其对于的指针地址后，试图强制转换为 *float32，结果就是报错。

为了解决这个问题，我们就需要用到unsafe.Pointer，其实Golang中很多转换方法的底层库都用到了unsafe.Pointer。它表示任意类型且类型可寻址的指针值，可以在不同的指针类型之间进行转换。

但是这个库为什么叫unsafe呢？简单来说，就是不推荐使用，因为它是不安全的。只有特殊的场景下才可以使用它。



#### 切片
铺垫了这么久，终于来到了我们今天的重点，那就是怎么去传递数组指针？我们可以先粗略地看一下slice的内存模型。可以看到，slice并不是真正意义上的动态数组，而是一个引用类型。slice总是指向一个底层array，slice的声明也可以像array一样，只是不需要长度。当然，关于slice的变长方案，我们可以后面再仔细研究，这不是本文的重点。

![](/img/in-post/post-cgo-params/slice.png)

由于底层内存模型的差异，不能直接将golang切片的指针传给c函数调用，而是需要将存储切片数据的内部缓冲区的首地址及切片长度取出传传递：

```
package main
/*
#include <stdint.h>
static void fill_255(char* buf, int32_t len) {
	int32_t i;
	for (i = 0; i < len; i++) {
		buf[i] = 255;
	}
}
*/
import "C"
import (
	"fmt"
	"unsafe"
)
func main() {
	b := make([]byte, 5)
	fmt.Println(b) // [0 0 0 0 0]
	C.fill_255((*C.char)(unsafe.Pointer(&b[0])), C.int32_t(len(b)))
	fmt.Println(b) // [255 255 255 255 255]
}

```
很多时候，我更倾向于上边这样子的使用方法（[]byte -> char*）。这样子在c语言中就不必malloc和free了，而把内存管理交给了go语言这边来处理，用完就自动释放了。

#### 字符串
先来看看golang string的内存模型：
![](/img/in-post/post-cgo-params/golangstring.png)

golang字串符串并没有用 ‘\0’ 终止符标识字符串的结束，因此直接将golang字符串底层数据指针传递给c函数是不行的。一种方案类似切片的传递一样将字符串数据指针和长度传递给c函数后，c函数实现中自行申请一段内存拷贝字符串数据然后加上未层终止符后再使用。另一种更好的方案是使用标准库提供的C.CString()将golang的字符串转换成C字符串然后传递给C函数调用：

```
package main
/*
#include <stdint.h>
#include <stdlib.h>
#include <string.h>
static char* cat(char* str1, char* str2) {
	static char buf[256];
	strcpy(buf, str1);
	strcat(buf, str2);
	return buf;
}
*/
import "C"
import (
	"fmt"
	"unsafe"
)
func main() {
	str1, str2 := "hello", " world"
	// golang string -> c string
	cstr1, cstr2 := C.CString(str1), C.CString(str2)
	defer C.free(unsafe.Pointer(cstr1)) // must call
	defer C.free(unsafe.Pointer(cstr2))
	cstr3 := C.cat(cstr1, cstr2)
	// c string -> golang string
	str3 := C.GoString(cstr3)
	fmt.Println(str3) // "hello world"
}
```
需要注意的是C.CString()返回的C字符串是在堆上新创建的并且不受GC的管理，使用完后需要自行调用C.free()释放，否则会造成内存泄露。

## 总结

本文先是粗略回顾了c语言指针的相关内容，主要是针对字符串指针。其次，顺带提了一下unsafe.Pointer和golang slice，string的内存模型，浮光掠影，后面一定还要加深理解。

然后是针对slice和string在cgo是使用过程存在的坑做了一些总结，在我的项目中，用得最多的还是[]byte到char*的转换。

## 参考
* [chai大佬的go语言圣经](https://chai2010.cn/advanced-go-programming-book/ch2-cgo/ch2-01-hello-cgo.html)
* [golang cgo 使用总结](http://litang.me/post/golang-cgo/) 
* [Cgo中如何将 []byte 转换为 *char](https://segmentfault.com/q/1010000009515544)
* [CGO类型及常见类型转换](https://linkscue.com/2018/11/12/2018-11-12-cgo-types-covertion/)
* [slice内存模型](https://blog.csdn.net/tennysonsky/article/details/78789878)
* [有点不安全却又一亮的 Go unsafe.Pointer](https://www.jianshu.com/p/fb25dce48317)
* [多语言在线编译工具](https://tool.lu/coderunner)
* [C语言基础——字符串指针（指向字符串的指针）](https://blog.csdn.net/u013812502/article/details/81196367)
* [C语言指针变量的声明](http://c.biancheng.net/cpp/html/1925.html)





