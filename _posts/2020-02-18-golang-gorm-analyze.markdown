---
layout: post
title: "golang database/sql连接池分析"
subtitle: "connecting pool of database/sql"
author: "zy"
header-img: "img/post-bg-golang.jpg"
catalog: true
tags:
  - Golang
---

> 武汉加油，中国加油！

## 引言
这段时间想理解一下gorm的源码，但是对于数据库的连接池一直不是很了解，严重阻碍了我理解的进度。所以我们先来分析一下database/sql包，gorm就是基于这个包来实现功能的。

## 连接池作用：为了提高服务性能
连接池是将已经创建好的连接保存在池子中。当有请求过来的时候，直接使用已经创建好的连接来访问数据库。
基本原理是这样：
1. 建立数据库连接池子对象（服务器启动）
2. 按照配置中指定的参数创建初始数量的数据库连接（空闲连接数）
3. 对于一个数据库的访问请求，直接从连接池中获得一个连接。如果数据库连接池对象没有空闲的连接，而且连接数没有达到最大(最大活跃连接数)，则创建一个新的数据库连接
4. 读写数据库
5. 关闭数据库连接（并非真正的关闭，而是将其放入空闲队列中。当实际空闲连接数大于初始空闲连接数，则释放掉掉多余的连接）
6. 释放数据库资源池对象（服务器停止，维护，释放数据库连接池对象，并释放所有连接）

连接池放了N个Connection对象，对象存放在内存中。应用程序每次都从池子里面获得Connection对象。这样省略了创建和销毁连接的过程，是一种典型的以空间换时间的策略。

## database/sql包

"database/sql"是golang原生的包，这个包定义了一些sql操作的接口，具体的实现还需要不同的数据库驱动去做。在接口，驱动的设计上，"database/sql"的实现也是非常优秀的，其设计的思想上有很多地方是可以借鉴的。"database/sql"除了定义接口外，还实现另外一个重要的功能：连接池，我们在实现其他网络通信时候也可以借鉴其实现。

```
package main

import(
    "fmt"
    "database/sql"
    _ "github.com/go-sql-driver/mysql"
)

func main(){

    db, err := sql.Open("mysql", "username:password@tcp(host)/db_name?charset=utf8&allowOldPasswords=1")
    if err != nil {
        fmt.Println(err)
        return
    }
    defer db.Close()

    rows,err := db.Query("select * from test")

    for rows.Next(){
        //row.Scan(...)
    }
    rows.Close()
}
```
"database/sql"的用法很简单，首先Open打开一个数据库，然后调用Query, Exec执行数据库操作。github.com/go-sql-driver/mysql具体实现了database/sql/driver的接口，所以最终具体的数据库操作都是调用github.com/go-sql-driver/mysql的方法。同一个数据库只需要调用一次Open即可，所以在一个项目中，我们一般将db声明为全局变量，只在项目启动的时候生成对象。

### 驱动注册

import _ "github.com/go-sql-driver/mysql"前面的”_”作用是：不需要把该包都导进来，只执行包的init()方法。mysql驱动正是通过这种方式注册到”database/sql”中的。init()通过Register()方法将mysql驱动添加到sql.drivers(类型：make(map[string]driver.Driver))中，MySQLDriver实现了driver.Driver接口：

```
//github.com/go-sql-driver/mysql/driver.go
func init() {
    sql.Register("mysql", &MySQLDriver{})
}

type MySQLDriver struct{}

func (d MySQLDriver) Open(dsn string) (driver.Conn, error) {
    ...
}

//database/sql/driver/driver.go

var (
	driversMu sync.RWMutex
	drivers   = make(map[string]driver.Driver)
)

type Driver interface {
    // Open returns a new connection to the database.
    // The name is a string in a driver-specific format.
    //
    // Open may return a cached connection (one previously
    // closed), but doing so is unnecessary; the sql package
    // maintains a pool of idle connections for efficient re-use.
    //
    // The returned connection is only used by one goroutine at a
    // time.
    Open(name string) (Conn, error)
}

//database/sql/sql.go
func Register(name string, driver driver.Driver) {
    driversMu.Lock()
    defer driversMu.Unlock()
    if driver == nil {
        panic("sql: Register driver is nil")
    }
    if _, dup := drivers[name]; dup {
        panic("sql: Register called twice for driver " + name)
    }
    drivers[name] = driver
}
```
### 连接池实现
连接池整体处理流程图，大家可以先简单梳理一下流程，然后我们会结合代码来了解连接池的实现：
![](/img/in-post/post-golang-connection-pool/go_sql.jpg)

##### 初始化DB

```
db, err := sql.Open("mysql", "username:password@tcp(host)/db_name?charset=utf8&allowOldPasswords=1")

```
sql.Open()是取出对应的db，这时候mysql还没有建立连接，只是初始化了一个sql.DB结构。DB是database/sql包中的核心结构，所有相关数据都保存在此结构中；Open同时启动了一个connectionOpener协程，其作用后面再分析。

```
type DB struct {
    driver driver.Driver  //数据库实现驱动
    dsn    string  //数据库连接、配置参数信息，比如username、host、password等
    numClosed uint64

    mu           sync.Mutex          //锁，操作DB各成员时用到
    freeConn     []*driverConn       //空闲连接
    connRequests map[uint64]chan connRequest //连接请求
    nextRequest  uint64              // 下一个等待请求的key
    numOpen      int                 //已建立连接或等待建立连接数
    openerCh    chan struct{}        //用于connectionOpener
    closed      bool
    dep         map[finalCloser]depSet
    lastPut     map[*driverConn]string // stacktrace of last conn's put; debug only
    maxIdle     int                    //最大空闲连接数
    maxOpen     int                    //数据库最大连接数
    maxLifetime time.Duration          //连接最长存活期，超过这个时间连接将不再被复用
    cleanerCh   chan struct{}
}
```   
maxIdle(默认值2)、maxOpen(默认值0，无限制)、maxLifetime(默认值0，永不过期)可以分别通过SetMaxIdleConns、SetMaxOpenConns、SetConnMaxLifetime设定。

##### 获取连接
上面说了，Open时是没有建立数据库连接的，只有等到用的时候才会实际建立连接。获取可用连接的操作有两种策略：
1. cachedOrNewConn, 用空闲连接则优先使用，没有则创建
2. alwaysNewConn, 不管有没有空闲连接都重新创建

实现连接池子的代码很长，所以代码我们就不全部贴出来了，结合一些核心代码来总结流程。首先，我们以一个query请求为例子：
```
rows, err := db.Query("select * from test")

// database/sql/sql.go

func (db *DB) Query(query string, args ...interface{}) (*Rows, error) {
    var rows *Rows
    var err error
    //maxBadConnRetries = 2
    for i := 0; i < maxBadConnRetries; i++ {
        rows, err = db.query(query, args, cachedOrNewConn)
        if err != driver.ErrBadConn {
            break
        }
    }
    if err == driver.ErrBadConn {
        return db.query(query, args, alwaysNewConn)
    }
    return rows, err
}

func (db *DB) query(query string, args []interface{}, strategy connReuseStrategy) (*Rows, error) {
    ci, err := db.conn(strategy)
    if err != nil {
        return nil, err
    }

    //到这已经获取到了可用连接，下面进行具体的数据库操作
    return db.queryConn(ci, ci.releaseConn, query, args)
}
```

step1:

首先检查freeConn里面是否有空闲连接，如果有且未超时则直接复用，返回连接。如果没有或者连接已经过期则进行下一步。

step2:

如果没有空闲连接，而且当前建立的连接数已经达到最大限制则将请求加入connRequests的map中，并阻塞在这里，直到其他协程将占用的连接释放或connectionOpenner创建。 
```
if db.maxOpen > 0 && db.numOpen >= db.maxOpen {
		req := make(chan connRequest, 1)
		reqKey := db.nextRequestKeyLocked()
		db.connRequests[reqKey] = req
		db.waitCount++
		db.mu.Unlock()

		waitStart := time.Now()

		// Timeout the connection request with the context.
		select {
		case <-ctx.Done():
			// Remove the connection request and ensure no value has been sent
			// on it after removing.
			db.mu.Lock()
			delete(db.connRequests, reqKey)
			db.mu.Unlock()

			atomic.AddInt64(&db.waitDuration, int64(time.Since(waitStart)))

			select {
			default:
			case ret, ok := <-req:
				if ok && ret.conn != nil {
					db.putConn(ret.conn, ret.err, false)
				}
			}
			return nil, ctx.Err()
		case ret, ok := <-req:
			atomic.AddInt64(&db.waitDuration, int64(time.Since(waitStart)))

			if !ok {
				return nil, errDBClosed
			}
			if ret.err == nil && ret.conn.expired(lifetime) {
				ret.conn.Close()
				return nil, driver.ErrBadConn
			}
			if ret.conn == nil {
				return nil, ret.err
			}
			// Lock around reading lastErr to ensure the session resetter finished.
			ret.conn.Lock()
			err := ret.conn.lastErr
			ret.conn.Unlock()
			if err == driver.ErrBadConn {
				ret.conn.Close()
				return nil, driver.ErrBadConn
			}
			return ret.conn, ret.err
		}
	}
```

step3：

创建一个连接时候，需要先将numOpen加1。 如果等创建完连接再把numOpen加1，会导致多个协程同时创建连接，导致一部分创建失败，浪费资源。先提前锁定numOpen，创建失败再将其减掉。

step4:

创建失败的时候，我们需要一些善后的操作。毫无疑问，首先是将db.numOpen减1。其次是根据db.connRequests的等待长度创建连接，这样操作的依据是：numOpen在连接成功创建之前就已经加1，这时候如果numOpen已经达到最大值，再有conn请求将阻塞在step2，这个请求会等着先前进来的请求释放资源。假如先前进来的资源创建连接全部失败并直接返回了，那些阻塞的请求就会继续阻塞，因为不可能有连接释放（极限设想），直到有新请求进来重新成功创建连接。这种情况显然是有问题的，所以maybeOpenNewConnections将通知connectionOpener，根据db.connRequests的等待长度以及可创建的最大连接数重新创建连接，然后将新创建的连接发给阻塞的请求。

##### 释放连接
数据库连接在被使用之后需要归还给连接池以供其他请求使用，释放连接的操作是putConn()：

```
func (db *DB) putConn(dc *driverConn, err error) {
    ...

    //如果连接已经无效，则不再放入连接池
    if err == driver.ErrBadConn {
        db.maybeOpenNewConnections()
        dc.Close() //这里最终将numOpen数减掉
        return
    }
    ...

    //正常归还, 有等待连接的请求则将连接发给它们，否则放入freeConn
    added := db.putConnDBLocked(dc, nil)
    ...
}
```

step1:

首先检查当前归还的连接在使用的过程中是否已经失效，如果无效则不再放入连接池

step2:

检查当前是否有等待连接的阻塞请求，有的话将当前连接发送给等待时间最长的请求。没有的话再判断空闲请求是否到达上限，没有则放入freeConn空闲连接池，达到上限则释放连接。

## 总结

1. 由于创建和释放连接都有很大的开销（尤其是访问远程数据库，每次建立连接的时候需要进行TCP三次握手，释放连接的时候需要进行TCP的四次握手，造成的开销是不可以避免的。为了提升系统访问数据库的性能，可以事先创建若干连接置于连接池中，需要时直接从连接池获取，使用结束时归还连接池而不必关闭连接，从而避免频繁创建和释放连接所造成的开销，这是典型的用空间换取时间的策略
2. "database/sql"定义了很多数据库操作的接口，但是具体的实现都是由数据库驱动来做的
3. 除此之外，"database/sql"还实现了connection pool，只要理清楚获取连接和释放连接的逻辑，连接池的模块就差不多吃透了。

## 引用

* [golang sql连接池](https://blog.51cto.com/8999585/2170821)
* [在进行数据库编程时，连接池有什么作用？](https://cloud.tencent.com/developer/article/1198915)
* [golang sql 包连接池分析](https://www.jianshu.com/p/45ea39e79de3)
* [Golang Mysql笔记（一）--- 连接与连接池](https://www.jianshu.com/p/340eb943be2e)
* [连接池的作用及讲解](https://www.cnblogs.com/sharpest/p/6240475.html)
* [连接池的作用就是为了提高性能](https://www.jianshu.com/p/28e1c27f38d7)


 
   



