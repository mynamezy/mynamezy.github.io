---
layout: post
title: "七层负载均衡后的Remote Address 和 X-Forwarded-For"
subtitle: "Remote Address and X-Forwarded-For"
author: "zy"
header-img: "img/post-bg-os-metro.jpg"
catalog: true
tags:
  - 协议
---

## 引言

在最近的项目中，由于前面七层负载均衡器的改动，出现了无法获取用户真实ip的现象，最终定位到是X-Forwarded-For的取值出现了问题，借此机会来分析一下七层负载均衡后的用户真实ip是怎么获取的。

## X-Forwarded-For

通过名字就知道，X-Forwarded-For 是一个 HTTP 扩展头部。HTTP/1.1（RFC 2616）协议并没有对它的定义，它最开始是由 Squid 这个缓存代理软件引入，用来表示 HTTP 请求端真实 IP。如今它已经成为事实上的标准，被各大 HTTP 代理、负载均衡等转发服务广泛使用，并被写入 RFC 7239（Forwarded HTTP Extension）标准之中。

X-Forwarded-For 请求头格式非常简单，就这样：

```
X-Forwarded-For: client, proxy1, proxy2
```
可以看到，XFF 的内容由「英文逗号 + 空格」隔开的多个部分组成，最开始的是离服务端最远的设备ip，然后是每一级代理设备的ip(不断往数组中追加信息)。

## request.remote_ip

一些开发者为了获取客户IP，我们经常会使用request.remote_ip来获得用户IP。但是很多用户都是通过代理来访问服务器的，如果使用remote_ip这个全局变量来获取IP，开发者往往获得的是代理服务器的IP，并不是用户真正的IP。我们就从著名的web开发框架rails，来看看remote_ip这个方法是怎么获取用户真实ip的，以小见大。

```
def remote_ip
    remote_addrs = split_ip_addresses(get_header('REMOTE_ADDR'))
    remote_addrs = reject_trusted_ip_addresses(remote_addrs)
    return remote_addrs.first if remote_addrs.any?
    forwarded_ips = split_ip_addresses(get_header('HTTP_X_FORWARDED_FOR'))
    return reject_trusted_ip_addresses(forwarded_ips).last || get_header("REMOTE_ADDR")
end

def reject_trusted_ip_addresses(ip_addresses)
    ip_addresses.reject { |ip| trusted_proxy?(ip) }
end

def trusted_proxy?(ip)
    ip =~ /\A127\.0\.0\.1\Z|\A(10|172\.(1[6-9]|2[0-9]|30|31)|192\.168)\.|\A::1\Z|\Afd[0-9a-f]{2}:.+|\Alocalhost\Z|\Aunix\Z|\Aunix:/i
end
```
来解释一下这块ruby的代码：
1. 表面上取的是header里面的REMOTE_ADDR，但实际上取的是TCP链接(socket)的ip。即使七层负载均衡器中往header里面添加了REMOTE_ADDR，服务中一样取不到用户真实的ip
2. rails设置了可信任的网关ip列表，任何在列表里面的ip都不会被当作用户真实ip
3. 拿不到REMOTE_ADDR，会去取X-Forwarded-For列表中的最后一个ip
4. 从算法上看，rails的remote_ip方法确实有不合理之处: 一是REMOTE_ADDR为什么不直接去取header中的信息？ 二是X-Forwarded-For为什么不取数组中的第一个ip？

通过模拟用户请求链路，分析一下remote_ip的取值：

```
         客户端=>正向代理=>透明代理=>服务器反向代理=>Web服务器    //模拟用户请求链路
```
1. 客户端直接连接 Web 服务器（假设 Web 服务器有公网地址），则 request.remote_ip获取到的是客户端的真实ip(REMOTE_ADDR) 
2. 假设 Web 服务器前部署了反向代理（比如 Nginx），则request.remote_ip获取到的是反向代理设备的ip（HTTP_X_FORWARDED_FOR最后一个ip）
3. 客户端通过正向代理直接连接 Web 服务器（假设 Web 服务器有公网地址），则request.remote_ip获取到的正向代理设备的ip(HTTP_X_FORWARDED_FOR最后一个ip)
4. 其实我们只需要记住的是，REMOTE_ADDR永远是获取和web server建立TCP链接的ip。X-Forwarded-For会记录所有网关的代理信息。

## nginx，kong 配置

目前，大部分的nginx和kong都是是7层代理，而不是4层代理。7层代理的意思我们只能从修改第七层的包信息，因此不可能在tcp/ip这层（传输层）做手脚，只能在http/https这个应用层协议中做文章。nginx的策略是: 往http/https请求中, 添加额外的header信息, 以此来完成真实客户端ip的信息传递。下面是nginx中的一些内部变量的定义：

```
$remote_addr #来自对端socket的ip地址
$remote_port #来自对端socket的port信息
$proxy_add_x_forwarded_for #http/https请求流经的所有代理的ip
```
在nginx配置中, 需要在location标签下添加如下项:
```
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```
X-Real-IP我们很容易可以理解了，就是取得和我们服务器建立tcp链接的ip地址。这里就需要X-Forwarded-For来记录整个链路上ip信息了。

## 总结

这是一篇比较浅显的博文，主要是针对项目中出现的问题做了一个归纳。目的只是为了可以更好理解Remote Address 和 X-Forwarded-For的使用场景，以及结合nginx使用可以产生的效果。

后面如果还有更深入的思考，会继续写下去。比如说，X-Forwarded-For被伪造了，我们应该如何取拿用户真实ip？

## 引用
* [X-Forwarded-For的一些理解](https://blog.csdn.net/zyhmz/article/details/82505344)
* [HTTP 请求头中的 X-Forwarded-For](https://imququ.com/post/x-forwarded-for-header-in-http.html)
* [Nginx 之 X-Forwarded-For 中首个IP一定真实吗？](https://blog.csdn.net/myle69/article/details/83041486)




