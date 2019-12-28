---
title: "database/sql 包的使用"
date: 2011-12-23T13:05:31Z
draft: true
---

# database/sql 包的使用

## 安装 mysql driver

```sh
go get -v github.com/go-sql-driver/mysql
```

## 创建连接池: `sql.Open`

```go
func newPool() *sql.DB {
    cfg := mysql.NewConfig()
    cfg.User = "root"
    cfg.Passwd = "xxxxxx"
    cfg.Net = "tcp"
    cfg.Addr = "127.0.0.1:3306"
    cfg.DBName = "mydb"
    dsn := cfg.FormatDSN()
    db, err := sql.Open("mysql", dsn)
    if err != nil {
        log.Fatal(err)
    }
    if err := db.Ping(); err != nil {
        log.Fatal(err)
    }
    return db
}

var pool = newPool()
```

`sql.DB`表示一个连接池\
`sql.Open`的第一个参数是驱动名称，这里是`"mysql"`\
这个名称是在`mysql`包初始化时注册的，代码见:
> github.com/go-sql-driver/mysql/driver.go

```go
func init() {
    sql.Register("mysql", &MySQLDriver{})
}
```

`sql.Open`的第二个参数是数据源名称，这里通过`mysql.Config`结构来配置，然后调用`FormatDSN`方法得出数据源名称为:\
`"root:xxxxxx@tcp(127.0.0.1:3306)/mydb"`

`db.Ping()`用来检查网络连通性以及用户密码是否正确

注意:\
`pool`是一个包级变量，生命周期持续整个应用，所以不需要关闭，`sql.DB`设计上就是作为长期存活的对象来使用的

## 查询数据: `(*sql.DB).Query`

```go
// User 用户
type User struct {
    ID   int64
    Name string
}

// QueryUser 根据id查询用户
// 注意：如果结果集为空集，这个代码是有问题的
func QueryUser(id int64) (*User, error) {
    rows, err := pool.Query("select `id`, `name` from `user` where `id` = ?", id)
    if err != nil {
        return nil, err
    }
    defer rows.Close() // 注意这里，一定要关闭
    user := User{}
    for rows.Next() {
        if err := rows.Scan(&user.ID, &user.Name); err != nil {
            return nil, err
        }
        break
    }
    if err := rows.Err(); err != nil {
        return nil, err
    }
    return &user, nil
}
```

连接池对程序员是透明的，这里并不需要显式的从连接池里获取连接，而是通过连接池来执行查询语句\
`Query`方法返回一个`*Rows`指针，代表结果集

要注意的是`defer rows.Close()`如果忘了关闭，可能会造成连接泄露

`rows.Scan`方法有个方便的特性，如果id在数据库里是`varchar(50)`类型，我们传的参数&user.ID指向`int64`，\
这依然可以工作，`Scan`方法会执行自动转换

## 单行查询: (*sql.DB).QueryRow

上面那个查询的例子，最多只有一个结果，使用单行查询更简单

```go
// QueryUser 单行查询
func QueryUser(id int64) (*User, error) {
    row := pool.QueryRow("select `id`, `name` from `user` where `id` = ?", id)
    user := User{}
    if err := row.Scan(&user.ID, &user.Name); err != nil {
        if err == sql.ErrNoRows {
            return nil, nil  // 返回 (*User)(nil) 表示查询结果不错在
        }
        return nil, err
    }
    return &user, nil
}
```

## 插入、更新、删除: `(*sql.DB).Exec`

```go
// InsertUser 插入用户
func InsertUser(name string) (int64, error) {
    res, err := pool.Exec("insert into `user` (`name`) values (?)", name)
    if err != nil {
        return 0, err
    }
    return res.LastInsertId()
}
```

```go
// UpdateUser 更新用户
func UpdateUser(id int64, name string) error {
    _, err := pool.Exec("update `user` set `name` = ? where `id` = ?", name, id)
    if err != nil {
        return err
    }
    return nil
}
```

```go
// DeleteUser 删除用户
func DeleteUser(id int64) error {
    _, err := pool.Exec("delete from `user` where `id` = ?", id)
    if err != nil {
        return err
    }
    return nil
}
```

## 事务：`(*sql.DB).Begin, (*sql.DB).Commit, (*sql.DB).Rollback`

```go
// UpdateFooBar 更新
func UpdateFooBar(id int64, x, y string) (err error) {
    tx, err := pool.Begin()
    if err != nil {
        return
    }
    defer func() {
        switch {
        case err != nil:
            _ = tx.Rollback() // ignore error
        default:
            err = tx.Commit() // modify named return value
        }
    }()
    _, err = tx.Exec("update `foo` set `x` = ? where `id` = ?", x, id)
    if err != nil {
        return
    }
    _, err = tx.Exec("update `bar` set `y` = ? where `id` = ?", y, id)
    if err != nil {
        return
    }
    return
}
```

事务保证所有操作都在同一个连接上执行

## 错误处理

### 迭代结果集错误: `(*Rows).Err`

```go
for rows.Next() {
    // ...
}
if err = rows.Err(); err != nil {
    // 处理错误
}
```

### 单行查询错误: `sql.ErrNoRows`

```go
var name string
err := db.QueryRow("select `name` from `user` where `id` = ?", 1).Scan(&name)
if err != nil {
    if err == sql.ErrNoRows { // 没有查询到结果
        return "", nil
    }
    return "", err
}
return name, nil
```

**注意: 只有`QueryRow`方法才会返回`sql.ErrNoRows`错误，`Query`方法不会返回这个错误**

### 通过错误码识别错误

```go
if err, ok := err.(*mysql.MySQLError); ok {
    switch err.Number {
    case mysqlerr.ER_ACCESS_DENIED_ERROR:
    // 权限错误
    default:
    // 其他错误
    }
}
```

mysql的错误码列表在下面的包中维护:

> github.com/VividCortex/mysqlerr

## null字段: `ifnull`

在设计数据库表时尽量避免`null`字段，因为go的一些值类型无法表达`null`，比如`string`, `int` 等

如果无法避免`null`字段，则使用`ifnull`函数设置默认值，例如：

```sql
select `id`, ifnull(`name`, '') as `name` from `user` where `id` = 22
```
