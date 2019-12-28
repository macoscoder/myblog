---
title: "多线程中信号处理方法"
date: 2011-12-23T13:05:31Z
draft: true
---

# 多线程中信号处理方法

## 方法说明

* 在创建子线程之前，主线程阻塞所有信号，随后创建的子线程继承主线程的`signal mask`
* 创建一个专用线程，用于处理信号(使用`sigwait`, `sigwaitinfo`, `sigtimedwait`)

## 示例

```c
#include "go.h"
#include "log.h"
#include <signal.h>
#include <unistd.h>

void sig_handler(void *arg) {
    int sig;
    sigset_t mask;
    sigfillset(&mask);
    for (;;) {
        sigwait(&mask, &sig);    // 同步等待信号，将异步信号处理转换成多线程处理
        debugf("sig = %d", sig); // debugf内部加锁了
        if (sig == SIGINT)
            _exit(1);
    }
}

void foo(void *arg) {
    for (;;) {
        debugf("foo");
        sleep(1);
    }
}

int main() {
    sigset_t mask;
    sigfillset(&mask);
    sigprocmask(SIG_SETMASK, &mask, NULL); // 屏蔽所有信号
    go(sig_handler, NULL);                 // 专用线程用于处理信号
    go(foo, NULL);
    pause();
}
```

* `go.h` 参考[cond](http://www.flyblog.tech/cond)
* `log.h` 参考[thread specific data](http://www.flyblog.tech/thread%20specific%20data)

## 演示

```sh
$ ./a.out
DEBUG: foo
DEBUG: foo
^\DEBUG: sig = 3
DEBUG: foo
^\DEBUG: sig = 3
^\DEBUG: sig = 3
DEBUG: foo
^CDEBUG: sig = 2
```
