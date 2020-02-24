---
layout: post
title: "Golang中关于反射的理解与运用"
subtitle: "reflect in Golang"
author: "zy"
header-img: "img/post-bg-golang.jpg"
catalog: true
tags:
  - Golang
---

> 武汉加油，中国加油！

## 引言
趁着这两天的时间，写了一个简单的gorm扩展包(其实就是一些简单的回调)。里面运用了不少反射的知识，我觉得有必要再梳理一下反射的概念，遂写下这篇博文。

## 关于反射的概念
在以前的学习中，我基本没有接触过计算机领域中反射。反射的概念就是：这类应用通过采用某种机制来实现对自己行为的描述（self-representation）和监测（examination），并能根据自身行为的状态和结果，调整或修改应用所描述行为的状态和相关的语义。

## golang类型设计的原则
1. 一个变量包括两部分，分别是type和value
2. type又可以分为两种类型，包括static type和concrete type，简单来说static type是你在编码时候看得见的类型（比如int,string）,concrete type是runtime看到的类型
3. 类型断言能否成功，取决于变量的concrete type, 而不是static type

接下来要说一下反射，就是建立在类型之上的。Golang编码时指定的变量类型是静态的（也就是指定int, string这些变量，它的type 是static type）, 在创建变量的时候就已经确定。反射主要与Golang的interface类型相关（它的type是concrete type），只有interface才有反射一说。

所以在Golang的实现中，每个interface变量都有一个对应pair, pair中记录了实际变量的值和类型：（value, type）。value是实际变量值，type是实际变量的类型。一个interface{}类型的变量实际包含了2个指针，一个指针指向值的类型（对应concrete type）, 另一个指针指向实际的值（对应的value）。

interface及其pair的存在，是Golang中实现反射的前提，理解了pair, 就更容易理解反射。反射就是用来检检测存储在接口变量内部（值value,类型concrete type）pair的一种机制。

## reflect的基本功能TypeOf和ValueOf

既然反射是用来检检测存储在接口变量内部（值value,类型concrete type）pair的一种机制，那我们的反射包中有什么方法可以让我们可以直接获取到变量内部的信息呢？它提供了两个方法可以让我们访问接口变量的内容，分别是reflect.ValueOf() 和 reflect.TypeOf()。
```
// ValueOf returns a new Value initialized to the concrete value
// stored in the interface i.  ValueOf(nil) returns the zero 
func ValueOf(i interface{}) Value {...}

翻译一下：ValueOf用来获取输入参数接口中的数据的值，如果接口为空则返回0


// TypeOf returns the reflection Type that represents the dynamic type of i.
// If i is a nil interface value, TypeOf returns nil.
func TypeOf(i interface{}) Type {...}

翻译一下：TypeOf用来动态获取输入参数接口中的值的类型，如果接口为空则返回nil
```

简单一点来说，reflect.TypeOf()是获取pair中的type，reflect.ValueOf()获取pair中的value，示例如下：
```
package main

import (
	"fmt"
	"reflect"
)

func main() {
	var num float64 = 1.2345

	fmt.Println("type: ", reflect.TypeOf(num))
	fmt.Println("value: ", reflect.ValueOf(num))
}

运行结果:
type:  float64
value:  1.2345
```
1. reflect.TypeOf: 直接给到了我们想要type类型，比如float64, int, 各种pointer, struct等真实类型
2. reflect.ValueOf: 直接给到了我们想要的具体的值，如1.2345这个具体数值，或者类似于&{1 "hmz" 25}这样的struct值
3. 也就是说明反射可以将"接口类型变量"转换为"反射类型对象"，反射类型指的是reflect.Type和reflect.Value这两种。其次，要记住"反射类型对象"这个名词，有助于我们更好地理解后面的内容。

除此之外，这个反射类型对象也有一些方法，可以帮助我们得到想要的信息，比如说Name()和Kind()方法：
```
package main

import (
	"fmt"
	"reflect"
)

type INT int

func main(){
	var a INT
	fmt.Println(reflect.TypeOf(a).Name())//INT
	fmt.Println(reflect.TypeOf(a).Kind())//int
}
```
可以看到，Kind()方法返回的是基础类型，Name()方法返回的是自定义类型。

## 从relect.Value中获取接口interface的信息

当执行reflect.ValueOf(interface)之后，就可以得到一个类型为"reflect.Value"变量，可以通过它本身的interface()方法获得接口变量的真是内容，然后可以通过类型判断进行转换。但这也是分情况的，一是已知原有类型，二是原有类型未知，让我们分情况进行说明。

#### 已知原有类型（进行强制转换）
当我们已知道原有类型时，我们可以直接通过interface方法然后进行强制转换，获得接口变量的真实内容。

```
package main

import (
	"fmt"
	"reflect"
)

func main() {
	var num float64 = 1.2345

	pointer := reflect.ValueOf(&num)
	value := reflect.ValueOf(num)

	// 可以理解为“强制转换”，但是需要注意的时候，转换的时候，如果转换的类型不完全符合，则直接panic
	// Golang 对类型要求非常严格，类型一定要完全符合
	// 如下两个，一个是*float64，一个是float64，如果弄混，则会panic
	convertPointer := pointer.Interface().(*float64)
	convertValue := value.Interface().(float64)

	fmt.Println(convertPointer)
	fmt.Println(convertValue)
}

运行结果：
0xc42000e238
1.2345
```
1. 转换的时候，如果转换的类型不完全符合，则直接panic, 类型要求非常严格
2. 转换的时候，要区分是指针还是值
3. 也就是说反射可以将"反射类型对象"再重新转换为"接口类型变量"

#### 未知原有类型（遍历探测其field）
很多情况下, 我们可能并不知道其具体类型，那就需要遍历探测其field来得知道，示例如下：
```
package main

import (
	"fmt"
	"reflect"
)

type User struct {
	Id   int
	Name string
	Age  int
}

func main() {

	user := User{1, "hmz", 25}

	DoFiledAndMethod(user)

}

// 通过接口来获取任意参数，然后一一揭晓
func DoFiledAndMethod(input interface{}) {

	getType := reflect.TypeOf(input)
	fmt.Println("get Type is :", getType.Name())

	getValue := reflect.ValueOf(input)
	fmt.Println("get all Fields is:", getValue)

	// 获取方法字段
	// 1. 先获取interface的reflect.Type，然后通过NumField进行遍历
	// 2. 再通过reflect.Type的Field获取其Field
	// 3. 最后通过Field的Interface()得到对应的value
	for i := 0; i < getType.NumField(); i++ {
		field := getType.Field(i)
		value := getValue.Field(i).Interface()
		fmt.Printf("%s: %v = %v\n", field.Name, field.Type, value)
	}
}

运行结果：
get Type is : User
get all Fields is: {1 hmz 25}
Id: int = 1
Name: string = hmz
Age: int = 25
```
获取未知类型的接口变量的真是内容，需要以下几个步骤：
1. 先获取interface的reflect.Type, 然后通过NumField进行遍历
2. 再通过reflect.Type的Field方法获取其Field
3. 最后通过Field的interface()方法得到对应接口类型变量的值

## 通过reflect.Value设置实际变量的值

reflect.Value是通过reflect.ValueOf(X)获得的，只有当X是指针的时候，才可以通过reflec.Value修改实际变量X的值，即：要修改反射类型的对象就一定要保证其值是“addressable”的。
```
package main

import (
	"fmt"
	"reflect"
)

func main() {

	var num float64 = 1.2345
	fmt.Println("old value of pointer:", num)

	// 通过reflect.ValueOf获取num中的reflect.Value，注意，参数必须是指针才能修改其值
	pointer := reflect.ValueOf(&num)
	newValue := pointer.Elem()

	fmt.Println("type of pointer:", newValue.Type())
	fmt.Println("settability of pointer:", newValue.CanSet())

	// 重新赋值
	newValue.SetFloat(77)
	fmt.Println("new value of pointer:", num)
	
	// 如果reflect.ValueOf的参数不是指针，会如何？
	pointer = reflect.ValueOf(num)
	//newValue = pointer.Elem() // 如果非指针，这里直接panic，“panic: reflect: call of reflect.Value.Elem on float64 Value”
}

运行结果：
old value of pointer: 1.2345
type of pointer: float64
settability of pointer: true
new value of pointer: 77
```
1. 需要传入的参数是*float64这个指针，然后通过Elem()去获取所指向的Value, 注意，这一定要是指针。
2. 如果传入的参数不是指针，而是变量，那么通过Elem获取原始值对应的对象则直接则直接panic, 通过CanSet方法可以查询是否可以设置

## 关于reflect的性能

Golang的反射很慢，这个和它的API设计有关系。在java中，我们一般的使用反射都是这样来操作的：
```
Field field = clazz.getField("hello");
field.get(obj1);
field.get(obj2);
```
这个取得的反射对象类型是java.lang.reflect.Field，它是可以复用的，只要传入不同的obj, 就可以取得这个obj上对应的Field。

但是Golang的反射不是这样设计的：
```
type_ := reflect.TypeOf(obj)
field, _ := type_.FieldByName("hello")
```
这里取出来的field对象是reflect.StructField 类型，但是没有办法用来取得对应对象上的值。如果要取值，得用另外一个对象，不能是type的反射：
```
type_ := reflect.ValueOf(obj)
fieldValue := type_.FieldByName("hello")
```
这里取出来的field对象是reflect.Value类型的，它是一个具体的值，而不是一个可复用的反射对象。除此之外，每次反射都需要malloc这个reflect.value对象，并且涉及到GC, 所以比较慢，性能较差。

## 总结

反射可以大大提高程序的灵活性，使得interface{}有更大的发挥余地，但也有性能上的消耗。也就是说：
1. 反射必须结合interface才能玩得转
2. 变量的type要是concrete type的(也就是说interface变量)才有反射一说

反射可以将"接口类型变量"转换为"反射类型对象"
1. 反射使用TypeOf和ValueOf函数从接口中获取反射类型对象信息

反射可以将"反射类型对象"转换为"接口类型对象"
1. reflect.value.Interface().(已知的类型)
2. 遍历reflect.Type的Field获取其Field

反射可以修改反射类型对象，但是其值必须是"addressable"。 

洋洋洒洒写了这么多，但其实还不是不太能区分接口类型变量和接口类型对象。但是不妨碍我们先把反射使用起来，文中有表述不准确的部分，我们后面再来补充。

## 引用
* [Golang的反射reflect深入理解和示例](https://juejin.im/post/5a75a4fb5188257a82110544)
* [Golang中反射类型对象，kind和name方法的需求](https://blog.csdn.net/qq_34673519/article/details/101391543)
* [反射三大定律, 官方文档](https://blog.golang.org/laws-of-reflection)







