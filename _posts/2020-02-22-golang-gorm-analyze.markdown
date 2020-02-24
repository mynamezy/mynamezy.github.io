---
layout: post
title: "gorm查询流程分析"
subtitle: "gorm query analyze"
author: "zy"
header-img: "img/post-bg-golang.jpg"
catalog: true
tags:
  - Golang
---

## 引言
在前面的博文中，我们学习了接口的使用，学习了利用Scanner和Valuer接口去构造自定义类型，还有database/sql如何去构造数据库的连接池。那我们现在来看看最常用的database orm是如何运作的。仿照志哥的思路，我们也来分析一下，并做一些补充介绍。

## gorm包中的核心数据结构

下面是gorm的核心数据结构，我们常用的操作都是建立在几个数据结构之上。

##### DB
这个数据结构，会存储db的相关信息。每次都会绑定不同的value, 比如说操作对象Account{}

```
type DB struct {
    sync.RWMutex                       // 读写锁，适用于读多写少的场景
    Value        interface{}           // 一般传入实际操作的表所对应的结构体
    Error        error                 // DB操作失败的error
    RowsAffected int64                 // 操作影响的行数

    // single db
    db                SQLCommon        // SQL接口，包括（Exec、Prepare、Query、QueryRow）
    blockGlobalUpdate bool             // 为true时，可以在update在没有where条件是报错，避免全局更新
    logMode           logModeValue     // 日志模式，gorm提供了三种   
    logger            logger           // 内部日志实例
    search            *search          // 查询相关的条件，保存搜索的条件where, limit, group
    values            sync.Map         // value Map

    // global db
    parent        *DB                  // 父db，为了保存一个空的初始化后的db，也为了保存curd注册的的callback方法
    callbacks     *Callback            // callback方法
    dialect       Dialect              // 不同类型数据库对应的不同实现的相同接口 
    singularTable bool                 // 表名是否为复数形式，true时为user，false时为users
}
```

##### Scope
scope保存sql执行的相关信息，可以理解为只针对本地数据库操作有效的一个环境：

```
type Scope struct {
    Search          *search // 检索条件, 保存搜索的条件where, limit, group
    Value           interface{}
    SQL             string //sql
    SQLVars         []interface{}
    db              *DB //sql.db
    instanceID      string
    primaryKeyField *Field
    skipLeft        bool
    fields          *[]*Field //字段
    selectAttrs     *[]string
}
```
##### Callbacks
保存各种操作需要执行的调用链，例如Find函数，需要调用queries数组中所有的函数：
```
type Callback struct {
    creates    []*func(scope *Scope)
    updates    []*func(scope *Scope)
    deletes    []*func(scope *Scope)
    queries    []*func(scope *Scope)
    rowQueries []*func(scope *Scope)
    processors []*CallbackProcessor
}
```

## 查询示例
在这个查询是示例中，Table, Where, Order都是查询条件，Find方法才真正实现了查询，并将查询得到的数据绑定给相应的结构体。
```
DBEngine.Table(entry.TableName).
    Where(entry.sql, entry.values).
    Order(entry.order).
    Find(entry.result)

```

## step1: gorm的初始化操作

毫无疑问，在所有的数据库操作（Exec, Query）之前，都需要先初始化数据库。
```
db, err := gorm.Open("mysql", "user:password@/dbname?charset=utf8&parseTime=True&loc=Local")
if err ！= nil {
    log.Errorf("init error!")
}
```
gorm的open的方法：

```
func Open(dialect string, args ...interface{}) (db *DB, err error) {
    if len(args) == 0 {
        err = errors.New("invalid database source")
        return nil, err
    }
    var source string
    var dbSQL SQLCommon
    var ownDbSQL bool

    switch value := args[0].(type) {
    case string:
        var driver = dialect
        if len(args) == 1 {
            source = value
        } else if len(args) >= 2 {
            driver = value
            source = args[1].(string)
        }
        // 调用go基础库的Open方法获得db的connention赋给dbSQL，
        // 此时还没有真正连接数据库, 只有真正执行操作的时候才会连接数据库
        dbSQL, err = sql.Open(driver, source)
        ownDbSQL = true
    case SQLCommon:
        dbSQL = value
        ownDbSQL = false
    default:
        return nil, fmt.Errorf("invalid database source: %v is not a valid type", value)
    }
    // 初始化DB
    db = &DB{
        db:        dbSQL,
        logger:    defaultLogger,
        callbacks: DefaultCallback,
        dialect:   newDialect(dialect, dbSQL),
    }
    // 将初始化的DB保存到db.parent中
    db.parent = db
    if err != nil {
        return
    }
    // 调用go基础库的Ping方法检测数据库connention是否可以连通
    if d, ok := dbSQL.(*sql.DB); ok {
        if err = d.Ping(); err != nil && ownDbSQL {
            d.Close()
        }
    }
    return
}
```

## step2: 执行Table方法，添加table name条件

```
func (s *DB) Table(name string) *DB {
    clone := s.clone()     
    clone.search.Table(name)  // 赋值table name
    clone.Value = nil         // 附空
    return clone
}
```
在学习gorm源码的过程中，我们经常会用到clone这个方法。那么它的作用到底是什么呢？大多数教程都会告诉你：避免交叉影响。但是这个交叉影响到底是什么？只能是靠悟了，下面来说说我的看法吧。

```
func (s *DB) clone() *DB {
	db := &DB{
		db:                s.db,
		parent:            s.parent,
		logger:            s.logger,
		logMode:           s.logMode,
		Value:             s.Value,
		Error:             s.Error,
		blockGlobalUpdate: s.blockGlobalUpdate,
		dialect:           newDialect(s.dialect.GetName(), s.db),
		nowFuncOverride:   s.nowFuncOverride,
	}

	s.values.Range(func(k, v interface{}) bool {
		db.values.Store(k, v)
		return true
	})

	if s.search == nil {
		db.search = &search{limit: -1, offset: -1}
	} else {
		db.search = s.search.clone()
	}

	db.search.db = db
	return db
}
```
从clone()的源码中，我们可以看到，这是一种深拷贝。深拷贝的含义就是：假设B复制了A，当修改A时，看B是否会发生变化，如果B也跟着变了，说明这是浅拷贝，拿人手短，如果B没变，那就是深拷贝，自食其力。clone方法将深拷贝之后所得到的对象返回给调用方，可以帮助我们实现骚气的链式调用。

![](/img/in-post/post-golang-gorm/db.png)

DB0是我们在数据库初始化时得到的对象，它是所有DB object的parent.db。可以看出，所有db操作都会拥有同一个parent.db。在我们这次查询操作中，会出现三个DB实例, 分别由三个查询条件产生。

## step3: 执行Where方法，添加where条件：
```
func (s *DB) Where(query interface{}, args ...interface{}) *DB {
    return s.clone().search.Where(query, args...).db
}

func (s *search) Where(query interface{}, values ...interface{}) *search {
	s.whereConditions = append(s.whereConditions, map[string]interface{}{"query": query, "args": values})
	return s
}
```
## step4: 执行Order方法，添加order条件


```
// 类似Where，reorder为true会强制刷掉gorm默认的order by
func (s *DB) Order(value interface{}, reorder ...bool) *DB {
    return s.clone().search.Order(value, reorder...).db
}

func (s *search) Order(value interface{}, reorder ...bool) *search {
    // 如果reorder为true，先清除s.orders
    if len(reorder) > 0 && reorder[0] {
        s.orders = []interface{}{}
    }
    // 将value拼接，存入s.orders
    if value != nil && value != "" {
        s.orders = append(s.orders, value)
    }
    return s
}
```
## step5：执行Find方法，实现真正的查询：

在这个step里面，NewScope的作用是返回一个scope实例，为这次数据库操作生成有效的操作环境。然后再调用inlineCondition，最后执行callcallbacks一系列方法实现真正的查询操作，并将db返回。这时候，才是真正完成一次查询的流程。

```
// NewScope方法就是初始化一个scope
func (s *DB) NewScope(value interface{}) *Scope {
    dbClone := s.clone()
    // 此时赋值value
    dbClone.Value = value
    scope := &Scope{db: dbClone, Value: value}
    if s.search != nil {
        scope.Search = s.search.clone()
    } else {
        scope.Search = &search{}
    }
    return scope
}

// scope.Search.Where实际上也是执行条件拼接，由于我们在调用的时候没有在Find中传入条件，所以这个方法不会被执行
func (s *search) Where(query interface{}, values ...interface{}) *search {
    s.whereConditions = append(s.whereConditions, map[string]interface{}{"query": query, "args": values})
    return s
}

// 最重要的就是callcallbacks方法，是真正执行的地方
func (scope *Scope) callCallbacks(funcs []*func(s *Scope)) *Scope {
    defer func() {
        if err := recover(); err != nil {
            if db, ok := scope.db.db.(sqlTx); ok {
                db.Rollback()
            }
            panic(err)
        }
    }()
    // 循环里面所有的注册的funcs
    for _, f := range funcs {
        (*f)(scope)
        if scope.skipLeft {
            break
        }
    }
    return scope
}

// 这里的funcs实在程序启动时init方法注册的
func init() {
    DefaultCallback.Query().Register("gorm:query", queryCallback)
    DefaultCallback.Query().Register("gorm:preload", preloadCallback)
    DefaultCallback.Query().Register("gorm:after_query", afterQueryCallback)
}

```
第一个回调是根据对应的条件来查询数据，第二个回调是用来预加载关联关系（目前的例子中暂时没有用到），第三个回调afterQueryCallback方法提供了反射调用结构体的AfterFind方法，如果在查询前结构体实现了AfterFind方法就会被调用。

## 总结
1. 我们首先介绍了gorm中的核心结构体：DB，Scope, Callbacks, 这三个结构体基本贯穿了所有的数据库操作。
2. 其次，我们选择了一个简单的数据库query查询，并对它的调用流程做了一个大概的介绍。理解的关键是db.clone()，每个查询条件的赋值，以及最后的callbacks。gorm已经迭代很久了，贡献的开发者也不在少数，所以有时候我们很难全部理解代码中的每一个细节。对于我目前的水平来，理解大部分已经不错啦。
3. 虽然说我们不应该一叶障目，但是有些地方确实理解得还是很粗糙, 比如说数据库驱动注册，预加载关联关系模块等，这些问题留到后面去思考吧。

* [gorm查询流程源码分析](hhttps://studygolang.com/articles/20361)
* [GORM自定义Gorm.Model实现自动添加时间戳](https://www.cnblogs.com/sgyBlog/p/10154424.html)
* [gorm 简单调用源码分析](https://blog.csdn.net/qq_17612199/article/details/79437795)
* [gorm中文教程](https://jasperxu.github.io/gorm-zh/associations.html)












