---
title: "文件监控"
date: 2011-12-23T13:05:31Z
draft: true
---

# 文件监控

## 用例

```sh
$ go install demo/fwatch
$ $GOPATH/bin/fwatch /go/src/demo | xargs -L1 goreturns -l | xargs -L1 goreturns -w &
[1] 29568 29569 29570
```

监控/go/src/demo目录及其子目录及未来创建的子目录下的文件修改动作
如果文件发生修改则对其执行格式化

## 代码

```go
package main

import (
    "fmt"
    "log"
    "os"
    "path/filepath"

    "github.com/fsnotify/fsnotify"
)

func main() {
    if len(os.Args) != 2 {
        fmt.Fprintf(os.Stderr, "Usage: %s pathname\n", os.Args[0])
        os.Exit(1)
    }
    // 创建fsnotify.Watcher
    watcher, err := fsnotify.NewWatcher()
    if err != nil {
        log.Fatal(err)
    }
    defer watcher.Close()
    // 遍历目录
    err = walk(os.Args[1], func(path string) error {
        return watcher.Add(path)
    })
    if err != nil {
        log.Fatal(err)
    }
    // 监听事件
    err = listen(watcher)
    if err != nil {
        log.Fatal(err)
    }
}

func walk(path string, visit func(path string) error) error {
    // 转换成绝对路径
    abspath, err := filepath.Abs(path)
    if err != nil {
        return err
    }
    // 遍历目录
    err = filepath.Walk(abspath, func(path string, info os.FileInfo, err error) error {
        if err != nil {
            return nil
        }
        if info.IsDir() || path == abspath {
            return visit(path)
        }
        return nil
    })
    if err != nil {
        return err
    }
    return nil
}

func listen(watcher *fsnotify.Watcher) error {
    for {
        select {
        case event, ok := <-watcher.Events:
            if !ok {
                return nil
            }
            switch event.Op {
            case fsnotify.Create:
                err := addWatch(watcher, event.Name)
                if err != nil {
                    return err
                }
            case fsnotify.Write:
                fmt.Println(event.Name)
            }
        case err, ok := <-watcher.Errors:
            if !ok {
                return nil
            }
            return err
        }
    }
}

func addWatch(watcher *fsnotify.Watcher, pathname string) error {
    fi, err := os.Stat(pathname)
    if err != nil {
        return err
    }
    if fi.IsDir() {
        return watcher.Add(pathname)
    }
    return nil
}
```
