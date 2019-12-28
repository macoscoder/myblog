---
title: "进程时间"
date: 2011-12-23T13:05:31Z
draft: true
---

# 进程时间

## 概述

进程时间是指进程消耗的CPU时间，分为

* user cpu time
* system cpu time

## times系统调用

```c
#include <sys/times.h>
clock_t times(struct tms *buf);
```

## 示例

```c
#include <stdio.h>
#include <sys/times.h>
#include <unistd.h>

int main() {
    for (long long i = 0; i < 10000000; i++) {
        getpid();
    }
    int CLK_TCK = sysconf(_SC_CLK_TCK); // USER_HZ
    struct tms tms;
    times(&tms);
    double utime = (double)tms.tms_utime / CLK_TCK;
    double stime = (double)tms.tms_stime / CLK_TCK;
    printf(
        "user:\t%.3fs\n"
        "sys:\t%.3fs\n",
        utime, stime);
}
```

### 演示

```sh
$ cc time.c
$ time ./a.out
user:   1.160s
sys:    1.460s

real    0m2.642s
user    0m1.169s
sys     0m1.463s
```

## clock库函数

```c
#include <time.h>
clock_t clock(void);
```

`clock`库函数获取的是进程的**user cpu time**和**system cpu time**之和

虽然`clock`和`times`的返回值类型相同，但是由于历史原因`clock`返回的值的单位与`times`不同\
要转换成秒，`clock`返回的值需要除以`CLOCKS_PER_SEC`，这个值固定为**1000000**

由于通过`times`系统调用可以计算出总的cpu时间，所以**不推荐**使用`clock`函数
