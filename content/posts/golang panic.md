---
title: "go异常处理"
date: 2011-12-23T13:05:31Z
draft: true
---

# go异常处理

## 代码封装

```go
// Do do try-catch-finally
type Do struct {
    Try     func()
    Catch   func(e interface{})
    Finally func()
}

// Call call try
func (d Do) Call() {
    defer func() {
        if e := recover(); e != nil {
            d.Catch(e)
        }
        if d.Finally != nil {
            d.Finally()
        }
    }()
    d.Try()
}

// Go go try
func (d Do) Go() {
    go d.Call()
}
```

## 测试

```go
func main() {
    Do{
        Try: func() {
            a := [...]int{1, 2, 3, 4, 5}
            for i := 0; ; i++ {
                fmt.Println(a[i]) // 这里会越界，导致panic
            }
        },
        Catch: func(e interface{}) {
            fmt.Println(e)
        },
        Finally: func() {
            fmt.Println("finally")
        },
    }.Call()
}
```

输出:

```sh
1
2
3
4
5
runtime error: index out of range
finally
```

## 说明

按照golang的说法，**panic被认为是程序bug**，也就是可以避免的错误，相当于java中unchecked exception\
这种错误最好就是让程序及早crash，以便修复bug，而不是去catch错误

而正常的程序错误是用error来表示，通过函数返回值来报告错误，这样的错误也叫做expected error，相当于java中的checked exception\
这种错误是程序员控制不了的，比如用户输入不合法，io错误等，要求必须检查

## 总结

对于server类的长期运行程序，有必要捕获由于请求的handler造成的panic，并记录日志，避免程序crash
