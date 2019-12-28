---
title: "linux中的睡觉函数"
date: 2011-12-23T13:05:31Z
draft: true
---

# linux中的睡觉函数

## 1. sleep库函数

```c
#include <unistd.h>
unsigned int sleep(unsigned int seconds);
```

`sleep`只能精确到秒，**不推荐**使用

## 2. nanosleep系统调用

```c
#include <time.h>
int nanosleep(const struct timespec *req, struct timespec *rem);
```

`nanosleep`精确到纳秒

## 3. clock_nanosleep系统调用

```c
#include <time.h>
int clock_nanosleep(clockid_t clock_id, int flags, const struct timespec *request, struct timespec *remain);
```

`clock_id`参数可以指定为

* CLOCK_REALTIME    真实时间
* CLOCK_MONOTONIC   单调时间

`flags`参数可以为

* 0                 指定相对时间
* TIMER_ABSTIME     指定绝对时间

`clock_nanosleep`支持指定相对时间和绝对时间

## 示例 sleep.c

```c
#include <errno.h>
#include <signal.h>
#include <stdio.h>
#include <time.h>

#define NANO    (1LL)
#define MICRO   (1000 * NANO)
#define MILLI   (1000 * MICRO)
#define SECOND  (1000 * MILLI)
#define MINUTE  (60 * SECOND)
#define HOUR    (60 * Minute)

long long nano_time() {
    struct timespec ts;
    clock_gettime(CLOCK_REALTIME, &ts);
    return ts.tv_sec * SECOND + ts.tv_nsec;
}

void sleep_nano(long long rel) {
    struct timespec ts = {
        .tv_sec = rel / SECOND,
        .tv_nsec = rel % SECOND,
    };
    while (nanosleep(&ts, &ts) == -1 && errno == EINTR)
        ;
}

void sleep_rel(long long rel) {
    struct timespec ts = {
        .tv_sec = rel / SECOND,
        .tv_nsec = rel % SECOND,
    };
    while (clock_nanosleep(CLOCK_MONOTONIC, 0, &ts, &ts) == EINTR)
        ;
}

void sleep_abs(long long abs) {
    struct timespec ts = {
        .tv_sec = abs / SECOND,
        .tv_nsec = abs % SECOND,
    };
    while (clock_nanosleep(CLOCK_REALTIME, TIMER_ABSTIME, &ts, NULL) == EINTR)
        ;
}

static void handler(int sig) {}

int main() {
    struct sigaction sa;
    sa.sa_handler = handler;
    sa.sa_flags = 0;
    sigemptyset(&sa.sa_mask);
    sigaction(SIGINT, &sa, NULL);

    for (int i = 1;; i++) {
        sleep_nano(SECOND * 1);
        // sleep_rel(SECOND * 1);
        // sleep_abs(nano_time() + SECOND * 1);
        printf("%d\n", i);
    }
}
```

## 演示

```sh
$ cc sleep.c
$ ./a.out
1
^C^C^C2
^C^C^C^C^C^C^C^C^C^C^C^C3
^C^C^C^C^C^C^C^C^C^C^C4
^C^C^C^C^C^C^C^C^C^C^C^C5
^C^C^C^C^C^C^C^C^C^C^C^C6
^C^C^C^C^C^C^C^C^C^C^C^C7
^C^C^C^C^C^C^C^C^C^C^C^C8
^C^C^C^C^C9
10
^\[1]    15269 quit       ./a.out
```
