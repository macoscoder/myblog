---
title: "读symbol link本身"
date: 2011-12-23T13:05:31Z
draft: true
---


# 读symbol link本身

## 示例(mac os)

```sh
$ go install demo/catsymlink
$ $GOPATH/catsymlink /etc
/private/etc
```

## 代码

```go
package main

import (
    "fmt"
    "log"
    "os"
    "path/filepath"
)

func main() {
    if len(os.Args) != 2 {
        fmt.Fprintf(os.Stderr, "Usage: %s link\n", os.Args[0])
        os.Exit(1)
    }
    pth, err := os.Readlink(os.Args[1])
    if err != nil {
        log.Fatal(err)
    }
    pth, err = filepath.Abs(pth)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println(pth)
}
```

`filepath.Abs`函数相当于C的`realpath`函数
