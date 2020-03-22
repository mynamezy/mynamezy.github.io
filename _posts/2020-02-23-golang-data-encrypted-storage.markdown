---
layout: post
title: "利用自定义类型和gorm回调对数据库数据进行加密（TO DO）"
subtitle: "custom type in gorm"
author: "zy"
header-img: "img/post-bg-golang.jpg"
catalog: true
tags:
  - Golang
---

> 武汉加油，中国加油！

## 引言

在很多情况下，我们的敏感数据都是不允许明文存储的，要进行加密存储。这个时候我们可以利用自定义类型和gorm回调来实现目标。

## 实现Scanner和Valuer接口
仿照上一篇博文的写法，我们实现了Scan()和Value()方法，存储的算法AES-128-CBC加密(PKCS7UnPadding)，然后使用base 64 encode，读取的时候则是逆过来。此外，aes的密匙和向量都是初始化的时候赋值。

```
type EncryptData string

var config Config

func (e *EncryptData) Scan(val interface{}) error {
	if len(val.(string)) == 0 {
		*e = ""
		return nil
	}
	res, _ := Base64Decode(val.(string))
	data, _ := AesDecrypt(res, config.Key, config.Iv)
	*e = EncryptData(data)
	return nil
}

func (e EncryptData) Value() (driver.Value, error) {
	return e.Encrypt(), nil
}

func (e *EncryptData) Encrypt() string {
	res := AesEncrypt([]byte(*e), config.Key, config.Iv)
	return Base64Encode(res)
}
```

## gorm回调

##### Models定义

首先，我们现在定义一个简单的数据结构，目标是对Email这字段进行加密存储： 
```
type Account struct {
	gorm.Model
	Email          string `gorm:"not null"`
}
```
然后为Email字段增加一个tag, 声明其为需要被加密的字段。其次，还需要新增一个结构体成员：EncryptedEmail，目前只支持对字符串进行加密。
```
type Account struct {
	gorm.Model
	Email          string `gorm:"not null" secure:"encrypted"`
	EncryptedEmail encrypted_storage.EncryptData
}
```
在我们的实际应用场景中，经常会有存量数据。这些存量数据会和新的加密数据并存一段时间，用户很多时候都会希望在创建，更新，查询的时候，也可以对生成加密数据，这样就是

##### before create 和 before update 回调
```
func InsertOrUpdate(scope *gorm.Scope) {
	if !scope.HasError() {
		fields := scope.Fields()
		for _, v := range fields {
			if tag := v.Tag.Get("secure"); tag == "" {
				continue
			} else {
				unencryptedValue := v.Field.Interface().(string)
				unencryptedName := v.Name
				err := scope.SetColumn("Encrypted"+unencryptedName, EncryptData(unencryptedValue))
				if err != nil {
					fmt.Println(errors.Wrapf(err, "EncryptedStorage fail"))
				}
			}
		}
	}
}
```
说明：
1. scope.Fields()返回的当前结构体变量的field数组(reflect.Value, 反射类型变量)
2. 遍历field数组，然后根据tag(secure)来找到我们声明的需要被加密的字段
3. 利用SetColumn方法对新增添的字段进行加密

##### before query 回调
```
func BeforeQuery(scope *gorm.Scope) {
	if !scope.HasError() {
		indirectValue := scope.IndirectValue()
		if indirectValue.Kind() == reflect.Slice {
			return
		} else {
			fields := scope.Fields()
			for _, v := range fields {
				if tag := v.Tag.Get("secure"); tag == "" {
					continue
				} else {
					unencryptedValue := v.Field.Interface().(string)
					unencryptedName := v.Name
					tableName := scope.TableName()
					idField, _ := scope.FieldByName("ID")
					id := idField.Field.Interface().(uint)
					encryptField, _ := scope.FieldByName("Encrypted" + unencryptedName)
					fmt.Println(encryptField.IsBlank)
					if encryptField.IsBlank {
						scope.NewDB().Where("id = ?", id).Table(tableName).UpdateColumn("encrypted_email", EncryptData(unencryptedValue))
					}
				}
			}
		}
	}
}
```
说明：
1. scope.IndirectValue()返回scope的反射类型变量，如果是该值是一个指针，则通过Elem()方法，获取其真实值
2. 其次，我们indirectValue.Kind()需要判断这个反射类型变量的类型。如果是一个切片(slice)，我们跳过这个回调，暂时没有对批处理操作进行开发
3. v.Field.Interface().(string)是将"反射类型对象"再转换为"接口类型变量"

## 总结

1. 利用自定义的数据类型，我们已经可以很好地对数据进行加解密，而且不影响gorm的使用
2. 但是在我们的实际的工作场景中，经常需要处理存量数据。而且这些存量数据还可能处于动态增长的阶段，很难对数据进行一次性操作，所以构造gorm的回调还是很有必要的

## 引用

* [Gorm 源码分析(二) 简单query分析](https://segmentfault.com/a/1190000019490869)
* [GORM自定义Gorm.Model实现自动添加时间戳](https://www.cnblogs.com/sgyBlog/p/10154424.html)
* [gorm 简单调用源码分析](http://www.voidcn.com/article/p-zhtyzjda-brw.html)
* [gorm源代码解读](https://blog.csdn.net/cexo425/article/details/78831055)






