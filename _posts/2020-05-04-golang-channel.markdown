---
layout: post
title: "golang channel(1): channel的无阻塞读写"
subtitle: "golang channel"
author: "zy"
header-img: "img/post-bg-golang.jpg"
catalog: true
tags:
  - Golang
---

## 引言

在channel的使用时候，我们经常会遇到一些阻塞的场景, 让人傻傻分不清。无论是无缓冲管道，还是缓冲管道，都会存在阻塞的情况。这篇博文，我们就来总结一下哪些情况下会存在阻塞，以及如何使用select解决阻塞。在我们了解这些阻塞场景的同时，也可以学会更好地使用管道。

## channel简介

不同于传统的多线程并发模型使用共享内存来实现线程间通信，golang的哲学是使用channel进行协程之间的通信，来实现数据共享。这种方式的优点是通过提供原子的通信原语，避免了竞争情形(race condition)下复杂的锁机制。channel可以看成是一个FIFO的队列，对FIFO队列的读写都是原子操作，不需要加锁。

从字面上看，channel的意思大概就是管道的意思。channel是一种go协程用以接收或发送消息的消息队列，channel就像两个go协程之间的导管，来实现各种资源的同步。可以用下图示意：

![](/img/in-post/post-golang-channel/channel-image.jpg)

## 阻塞场景

##### 无缓冲管道

无缓冲管道的特点是，发送的数据需要被读取后，发送才会完成，它阻塞场景为：

* 管道中无数据，但执行读管道
```
func ReadNoDataFromNoBufCh() {
   noBufCh := make(chan int) // 无缓冲信道的声明方式
   <-noBufCh
   fmt.Println("read from no buffer channel success")
   
   // Output
   // fatal error: all goroutines are asleep - deadlock!
}
```

* 管道中无数据，向管道写数据，但无协程读取
```
func WriteNoBufCh() {
   ch := make(chan int)
   ch <- 1
   fmt.Println("write success no block")
   
   // Output
   // fatal error: all goroutines are asleep - deadlock!
}
```
上述示例代码中output注释代表了函数的执行结果，这两个函数都由于阻塞在管道操作而无法继续向下执行，最后报了死锁的错误。下面我们来看看正确的使用方式，避免死锁：

```
func testSimple() {
   ch := make(chan int)
   go func() {
      time.sleep(20 * time.Second)
      ch <- 1
   }()
   value := <- ch
   fmt.Println("value : ", value)
}
```
上面这个简单的例子就是新开启的goroutine向一个无缓冲信道发送了一个1的值，那么主进程的ch在20秒之后就会收到这个值的信息，一次线程间通信就完成了。

##### 有缓冲管道

有缓存管道的特点是，可以向管道中写入数据后直接返回，缓存中有数据时可以从管道中读到数据直接返回，这时候缓存管道是不会阻塞的，它的阻塞场景是：

* 管道的缓存无数据，但执行读管道

```
func ReadNoDataFromBufCh() {
     bufCh := make(chan int, 1)
     <-bufCh
     fmt.Println("read from no buffer channel success")
     // Output
     // fatal error: all goroutines are asleep - deadlock!
}
```
* 管道中缓存已经占满，向管道写数据，但无协程读数据

```
func WriteBufChButFull() {
     ch := make(chan int, 1)
     // make ch full
     ch <- 100
     ch <- 1
     fmt.Println("write success no block")
     // Output
     // fatal error: all goroutines are asleep - deadlock!
}
```
## 使用select + timeout，实现无阻塞读写

早期的select函数是用来监控一系列的文件句柄，一旦其中一个文件句柄发生IO操作，该select调用就会被返回，这就是著名的多路复用。golang在语言级别直接支持select，用于处理异步IO问题。select语句属于条件分支流程控制语句，不过它只能用于通道。使用方法和switch非常类似。

下面的这组case是利用select实现选择操作，里面有一组case语句，它会执行其中无阻塞的那一个。如果都阻塞了，那就等待其中一个不阻塞，进而继续执行，它有一个default语句，该语句是永远不会阻塞的，我们可以借助它实现无阻塞的操作。

#####  读操作

```
// 无缓冲管道读
func ReadNoDataFromNoBufChWithSelect() {
     bufCh := make(chan int)
     if v, err := ReadWithSelect(bufCh); err != nil {
         fmt.Println(err)  
     } else {
         fmt.Printf("read: %d\n", v)
     }
     // output
     // channel has no data 
}

// 有缓冲管道读
func ReadNoDataFromBufChWithSelect() {
     bufCh := make(chan int, 1)
     if v, err := ReadWithSelect(bufCh); err != nil {
        fmt.Println(err)
     } else {
        fmt.Printf("read: %d\n", v)
     }
     // Output
     // channel has no data
}

// select结构实现管道读

func ReadWithSelect(ch chan int) (x int, err error) {
     select {
     case x = <-ch:
          return x, nil
     default:
          return 0, errors.New("channel has no data") 
     }
}
```
##### 写操作

```
// 无缓冲管道写
func WriteNoBufChWithSelect() {
     ch := make(chan int)
     if err := WriteChWithSelect(ch); err != nil {
         fmt.Println(err) 
     } else {
         fmt.Println("write success")
     }
     
     // output
     // channel blocked, can not write
}

// 有缓冲管道写

func WriteBufChButFullWithSelect() {
     ch := make(chan int, 1)
     // make ch full
     ch <- 100
     if err := WriteChWithSelect(ch); err != nil {
         fmt.Println(err)
     } else {
         fmt.Println("write success")
     }
     
     // Output
     // channel blocked, can not write
}

// select结构实现管道写

func WriteChWithSelect(ch chan int) error {
      select {
      case ch <- 1:
         return nil
      default:
         return errors.New("channel blocked, can not write")
      }        
}
```

##### 使用定时器实现超时

使用default实现的无阻塞管道阻塞有一个缺陷：当管道不可读或者不可写的时候，会立即返回。而实际的场景和需求是：我们先读或者写数据，如果在规定的时间内无法读写，程序返回。

定时器材可以帮助我们解决这个问题，比如说，我给管道读写数据的容忍时间是500ms, 如果无法读写，便立即返回：

```
func ReadWithSelect(ch chan int) (x int, err error) {
    timeout := time.NewTimer(time.Microsecond * 500)
     select {
     case x = <-ch:
         return x, nil
     case <-timeout.C:
         return 0, errors.New("read time out")
     }
}

func WriteChWithSelect(ch chan int) error {
    timeout := time.NewTimer(time.Microsecond * 500)
    select {
    case ch <- 1:
        return nil
    case <-timeout.C:
        return errors.New("write time out")
    }
}
```
## 总结

1. 上文中介绍了四种管道阻塞的场景，如果我们混淆使用，很容易造成死锁
2. select + timeout是常见的实现管道无阻塞读写的方法
3. 这里只是简单地介绍了管道的使用，后面会继续做更深入的理解

## 引用

* [golang channel 使用总结](https://www.cnblogs.com/tobycnblogs/p/9935465.html)
* [Go的channel常见使用方式](https://www.jianshu.com/p/554e210bdca4)
* [深入理解Golang之channel](https://www.lagou.com/lgeduarticle/79467.html)
* [一招教你无阻塞读写Golang channel](https://yq.aliyun.com/articles/675968)



















