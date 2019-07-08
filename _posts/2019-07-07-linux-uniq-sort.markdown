---
layout: post
title: "利用uniq和sort指令，实现对文件的排序去重"
subtitle: 'Using uniq and sort instructions to sort and de-duplicate files'
author: "zy"
header-img: "img/post-bg-unix-linux.jpg"
catalog: true
tags:
  - Linux
---

> “Yeah It's on. ”

## 引言
由于公司业务的迭代，需要对部分数据进行筛洗，涉及到对大文件的排序和去重。刚开始接触到这个这个任务，也是想尽各种方法：使用redis、bitmap 或高效的排序算法等。但最后都发现实现这些方法比较繁琐，而且极其吃内存，不是很可行。在百抓挠腮之际，看到了一些优秀回答者的建议，就是利用sort进行排序，然后再利用uniq进行去重。

刚开始看到这个回答，我是持有怀疑态度的。但是经过实践发现，利用 uniq 和 sort 指令，其中间数据不会全部存在内存中，而是大部分存在磁盘里，是非常安全的做法。处理了几个4G左右的文件，速度也是非常快的。下面将这些如何使用这两个指令，做一个总结，方便后面的回顾。

## uniq
先利用cat，看看原有的内容：
```
$ cat testfile #原有内容  
test 30  
test 30  
Hello 95  
Hello 95  
Linux 85  
Linux 85 
```
使用uniq 命令删除重复的行后，有如下输出结果：
```
$ uniq testfile     #删除重复行后的内容  
test 30  
Hello 95  
Linux 85 
```
## sort
但是我们现在又面临一个问题，就是如果重复的行是不相邻的，是没有办法去重的。不慌，可以利用另一个指令，sort + 管道 + uniq：
```
$ sort  testfile | uniq
Hello 95  
Linux 85 
test 30
```
其次，如果我们还想统计各行在文中出现的次数：

```
$ sort testfile | uniq -c
2 Hello 95  
2 Linux 85 
2 test 30
```
最后，我们还想根据出现的次数进行排序，sort的-n参数可以帮助我们实现这个功能，重定向到tmp.csv的文件中：

```
sort testfile | uniq -c | sort -n > tmp.csv
```

## 总结
这些简单的指令可以帮助我们快速地实现文件排序去重。目前来看，对中型文件，速度还是可以的。希望后面有机会的话，可以了解一下uniq和sort的实现原理，知其然，也知其所以然。

## 参考
* [Linux uniq 指令](https://www.runoob.com/linux/linux-comm-uniq.html)
* [Linux sort 指令](https://www.runoob.com/linux/linux-comm-sort.html)