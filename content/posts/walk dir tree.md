---
title: "目录树遍历"
date: 2011-12-23T13:05:31Z
draft: true
---

# 目录树遍历

## 示例

```sh
$ go install demo/walk
$ walk -f /usr/include | grep stdio.h
/usr/include/_stdio.h
/usr/include/c++/4.2.1/tr1/stdio.h
/usr/include/secure/_stdio.h
/usr/include/stdio.h
/usr/include/sys/stdio.h
/usr/include/xlocale/_stdio.h
```

## 代码 $GOPATH/src/demo/walk/main.go

```go
package main

import (
    "flag"
    "fmt"
    "log"
    "os"
    "path/filepath"
)

var (
    f bool
    d bool
)

func init() {
    flag.BoolVar(&f, "f", false, "只显示文件")
    flag.BoolVar(&d, "d", false, "只显示目录")
    flag.Usage = func() {
        fmt.Fprintf(os.Stderr, "Usage: %s [-d]|[-f] dirpath\n", os.Args[0])
        flag.PrintDefaults()
    }
}

func main() {
    flag.Parse()
    if flag.NArg() != 1 {
        flag.Usage()
        os.Exit(1)
    }
    err := filepath.Walk(flag.Arg(0), visit)
    if err != nil {
        log.Fatal(err)
    }
}

func visit(path string, info os.FileInfo, err error) error {
    if err != nil {
        log.Print(err)
        return nil
    }
    switch {
    case f:
        if !info.IsDir() {
            fmt.Println(path)
        }
    case d:
        if info.IsDir() {
            fmt.Println(path)
        }
    default:
        fmt.Println(path)
    }
    return nil
}
```
