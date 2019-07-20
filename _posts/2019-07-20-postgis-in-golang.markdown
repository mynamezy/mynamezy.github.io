---
layout: post
title: "Golang database/sql中的实现自定义数据类型：geometry"
subtitle: "Define a custom type in golang database/sql"
author: "zy"
header-img: "img/post-bg-golang.jpg"
catalog: true
tags:
  - Golang
---

## 引言

在以前的博文中，我们粗略地介绍了postgis的使用。但是有一个问题，我们怎么将postgis和go orm结合起来。目前项目中使用的是gorm，翻遍其文档，对于如何定义geometry数据类型，仍是找不到任何蛛丝马迹。

好在github有开源的项目，提供了模版，就仿照着写了几个model，凑活着也能用吧。但是不知道其原理，用起来实在难受。所以，这篇博文要给自己一个机会，尝试理解golang database/sql 中的Scanner和Value接口，并实现自定义数据类型。

## database/sql 和数据库驱动

database/sql 是操作数据库常用的包。无论使用了任何orm, 都会引用这个package。这个包定义了一些sql操作的接口，比如说Open，Scanner，Value，具体的实现还需要使用不同数据库实现，这就是我们说的驱动。

mysql有一个很优秀的一个驱动是：github.com/go-sql-driver/mysq。在接口、驱动的设计上，”database/sql”的实现非常优秀。在gorm中, 其实是对不同的驱动做了wrappe，让我们更好地记住这些需要import的路径。

```
// From gorm source code

// Open initialize a new db connection, need to import driver first, e.g:
//
//     import _ "github.com/go-sql-driver/mysql"
//     func main() {
//       db, err := gorm.Open("mysql", "user:password@/dbname?charset=utf8&parseTime=True&loc=Local")
//     }
// GORM has wrapped some drivers, for easier to remember driver's import path, so you could import the mysql driver with
//    import _ "github.com/jinzhu/gorm/dialects/mysql"
//    // import _ "github.com/jinzhu/gorm/dialects/postgres"
//    // import _ "github.com/jinzhu/gorm/dialects/sqlite"
//    // import _ "github.com/jinzhu/gorm/dialects/mssql"
```

## 




