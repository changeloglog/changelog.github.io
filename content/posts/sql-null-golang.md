---
title: "Go 如何处理 SQL 中的 NULL"
date: 2022-06-28T20:08:11+08:00
draft: false
---

如果你用 Go 操作过 MySQL，那你就知道 `NULL` 有多烦人了，当数据库中某个字段的值是 `NULL` 时，你可能会遇到一些问题。

`database/sql` 包会将 `NULL` 转换为 `nil`，但是它不能赋值给 `int`、`float64`、`string`、`bool` 等类型。

这篇文章介绍几个处理 MySQL 中的 `NULL` 值的方法。

### 0）为什么需要特别关注 NULL

在 Go 中，每个类型都有一个 `零` 值，声明一个变量不指定值，则变量会被初始化为零值。

```go
var i int
fmt.Printf("%#v\n", i) // => 0

var s string
fmt.Printf("%#v\n", s) // => ""
```

特别的是指针，它的零值是 `nil`。

```go
var p *int
fmt.Printf("%#v\n", p) // => nil
```

你不能用 `nil` 指针做任何事，例如下面的代码会 panic。

```go
fmt.Printf("%#v\n", *p) // => 💥
```

类似的，结构体中的每个字段都有一个零值

```go
type R struct {
	i int
	s string
	p *int
}

var r R
fmt.Printf("%#v\n", r) // => {i:0, s:"", p:(*int)(nil)}
```

我们需要关注这个问题的原因在于，数据库中 `NULL` 会被转换为 `nil`，而不是对应类型零值。

将 `nil` 赋值给 `int`、`float64`、`string`、`bool` 等基本类型，是不合法的。

以下有几种方法来应对这个问题。

### 1) 使用指针

在 Go 中，指针通常用来传递变量，这样可以避免值复制，在这个场景中，指针也可以用来处理 NULL 值。

```go
type User struct {
	Id   int64
	Name *string
	Age  *int32
}

err := db.QueryRow("SELECT NULL, 2").Scan(&u.Name, &u.Age)
```

数据库中的 `NULL` 将被转换为 `nil`，然后可以安全的赋值给 `u.Name`。

User 结构体在进行 `json.Marshall()` 时，`nil` 值将会被转换为 JSON 中的 `NULL`，这符合预期。

但是在你的代码中使用 `u.Name` 前需要先检查是否为 `nil`，否则会引发 `panic`。

### 2) 使用 sql.Null*

使用 `sql` 包中 `sql.NullString`, `sql.NullInt32` 等类型来替代原始类型，每个 Null* 类型都有一个 `Valid` 方法以验证值有效性。Null* 的问题是它不能很好的支持如 JSON 编码，你必须用其它办法来支持 JSON 编码。

```go
type User struct {
	Id   int64
	Name sql.NullString
	Age  sql.NullInt32
}

err := db.QueryRow("SELECT 'John', NULL").Scan(&u.Name, &u.Age)
```
上面的例子中，可以使用 `u.Name.Valid` 来检查值是否有效，有效则为 `true`。另外使用 `u.Name.String` 来获取值，如 `u.Name.Valid` 为 `true` 则值为 `John`，否则为 `""`。

在你的代码中，也可以不使用 `Valid` 来验证值有效性，直接使用 `u.Name.String` 就是安全的。

### 3) 使用 COALESCE 函数

大多数数据库中(SQLite, PostgreSQL, MySQL, ...) 都有 `COALESCE` 函数，它返回第一个不为 `NULL` 的值。

`COALESCE(name, '')`，如 name 字段为 `NULL`，则返回 `''` 空字符串。

```go
type User struct {
	Id   int64
	Name string
	Age  int32
}

err := db.QueryRow("SELECT COALESCE(NULL, '')").Scan(&u.Name)
```

### 4） 不使用 NULL 值

比如将数据库字段的默认值从 `NULL` 改为 `''`，如果你的字段值不会出现 `NULL`，则就不用处理这个问题了。

本文提到了几种方式，具体使用哪种，还需要根据实际情况来决定。