---
layout: post
title: "gorm中使用自定义类型"
subtitle: "custom type in gorm"
author: "zy"
header-img: "img/post-bg-golang.jpg"
catalog: true
tags:
  - Golang
---

> 武汉加油，中国加油！

## 引言
在我们使用golang进行开发的日常中，会发现使用很难将自定义类型映射到数据库的字段上。废话不多说，我们先来看看例子吧。

## Gorm模型定义

```
type User struct {
  gorm.Model
  Name         string
  Age          sql.NullInt64
  Birthday     *time.Time
  Email        string  `gorm:"type:varchar(100);unique_index"`
  Role         string  `gorm:"size:255"` // 设置字段大小为255
  MemberNumber *string `gorm:"unique;not null"` // 设置会员号（member number）唯一并且不为空
  Num          int     `gorm:"AUTO_INCREMENT"` // 设置 num 为自增类型
  Address      string  `gorm:"index:addr"` // 给address字段创建名为addr的索引
  IgnoreMe     int     `gorm:"-"` // 忽略本字段
}

```
上面是Gorm官方文档中的定义的一个模型，gorm一般可以识别一些常见的数据类型：
```
int64
float64
bool
[]byte
string
time.Time
nil - for NULL values
```
让我们来定义一个数据类型YesNoEnum，它的基础类型实际是一个布尔值: 

```
type YesNoEnum bool

const (
	Yes YesNoEnum = true
	No            = false
)

type Customer struct {
	CustomerID int64
	Active     YesNoEnum
}
```

在这种情况下，gorm是无法识别这种映射关系的(其实不单单是gorm，所有的 golang orm都无法识别)。gorm的文档中留下了这么一段话，可以作为指导思想，帮助我们实现自定义数据类型：
**模型（Models）通常只是正常的 golang structs、基本的 go 类型或它们的指针。 同时也支持sql.Scanner及driver.Valuer 接口（interfaces）**。

## 通过接口实现数据绑定

##### 接口文档

```
// sql.Scanner
type Scanner interface {
    // Scan assigns a value from a database driver.
    //
    // The src value will be of one of the following types:
    //
    //    int64
    //    float64
    //    bool
    //    []byte
    //    string
    //    time.Time
    //    nil - for NULL values
    //
    // An error should be returned if the value cannot be stored
    // without loss of information.
    //
    // Reference types such as []byte are only valid until the next call to Scan
    // and should not be retained. Their underlying memory is owned by the driver.
    // If retention is necessary, copy their values before the next call to Scan.
    Scan(src interface{}) error
}

// driver.Valuer
type Valuer interface {
    // Value returns a driver Value.
    // Value must not panic.
    Value() (Value, error)
}
```
简单一点来说，Scanner读取从数据库传来的数据，并转换成符合我们自定义的格式；相对的，Valuer则是将自己定义的数据结构，转换成sql看得懂的形式。

##### 接口实现

```
type YesNoEnum bool

// Value - Implementation of valuer for database/sql
func (yne YesNoEnum) Value() (driver.Value, error) {
    // value needs to be a base driver.Value type
    // such as bool.
	return bool(yne), nil
}

func (yne *YesNoEnum) Scan(value interface{}) error {
	// if value is nil, false
	if value == nil {
		// set the value of the pointer yne to YesNoEnum(false)
		*yne = YesNoEnum(false)
		return nil
	}
	if v, ok := value.(bool); ok {
    	// set the value of the pointer yne to YesNoEnum(v)
    	*yne = YesNoEnum(v)
    	return nil
    }
	// otherwise, return an error
	return errors.New("failed to scan YesNoEnum")
}   
```

## 总结
1. golang中sql包很难去识别自定义的数据类型，需要我们利用go sql包中的Scanner和Valuer接口来实现数据的绑定
2. 我们文中的例子简单地实现了bool类型的定义，我们很多时候还会使用json，map, postgis等自定义数据类型

## 引用
* [Go Web 小技巧（二）GORM 使用自定义类型, 赖林老师精品之作，值得借鉴](https://lailin.xyz/post/17394.html)
* [Golang：讓自訂型別支援 SQL](https://medium.com/@hunsin/golang-%E8%AE%93%E8%87%AA%E8%A8%82%E5%9E%8B%E5%88%A5%E6%94%AF%E6%8F%B4sql-97e14c31fd8a)
* [Scanners and Valuers with Go](https://husobee.github.io/golang/database/2015/06/12/scanner-valuer.html)
* [Gorm官方文档](https://gorm.io/docs/)

