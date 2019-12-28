---
title: "Linux Timer"
date: 2011-12-23T13:05:31Z
draft: true
---

# Linux Timer

## Classical UNIX API

### 1. setitimer

```c
#include <sys/time.h>
int setitimer(int which, const struct itimerval *new_value, struct itimerval *old_value);
int getitimer(int which, struct itimerval *curr_value);
```

#### which参数说明

* `ITIMER_REAL`: real time, At each expiration, a `SIGALRM` signal is generated.
* `ITIMER_VIRTUAL`: user-mode CPU time, At each expiration, a `SIGVTALRM` signal is generated.
* `ITIMER_PROF`: total (i.e., both user and system) CPU time, At each expiration, a `SIGPROF` signal is generated.

### 2. alarm

```c
#include <unistd.h>
unsigned int alarm(unsigned int seconds);
```

`alarm`相当于`setitimer`(which参数指定为`ITIMER_REAL`)

**注意**: 对于Classical UNIX API，每个进程**只有**三个定时器

## POSIX Timer API

```c
#include <signal.h>
#include <time.h>
int timer_create(clockid_t clockid, struct sigevent *sevp, timer_t *timerid);
int timer_settime(timer_t timerid, int flags, const struct itimerspec *new_value, struct itimerspec *old_value);
int timer_gettime(timer_t timerid, struct itimerspec *curr_value);
int timer_delete(timer_t timerid);
int timer_getoverrun(timer_t timerid);

Link with -lrt.
```

### 示例1: 使用信号

```c
#include <signal.h>
#include <stdio.h>
#include <sys/signalfd.h>
#include <time.h>
#include <unistd.h>

#define SIG_TIMER SIGRTMAX

int main() {
    timer_t tid;
    struct sigevent se;
    se.sigev_notify = SIGEV_SIGNAL;
    se.sigev_signo = SIG_TIMER;
    timer_create(CLOCK_REALTIME, &se, &tid);

    struct itimerspec its;
    its.it_value.tv_sec = 1;
    its.it_value.tv_nsec = 0;
    its.it_interval = its.it_value;
    timer_settime(tid, 0, &its, NULL);

    sigset_t mask;
    sigemptyset(&mask);
    sigaddset(&mask, SIG_TIMER);
    sigprocmask(SIG_BLOCK, &mask, NULL);

    int fd = signalfd(-1, &mask, 0);

    int expire_count = 0;
    for (;;) {
        struct signalfd_siginfo siginfo;
        read(fd, &siginfo, sizeof(struct signalfd_siginfo));
        expire_count += (1 + siginfo.ssi_overrun);
        printf("expire_count = %d\n", expire_count);
    }

    timer_delete(tid);
    close(fd);
}
```

### 示例2: 使用线程

```c
#include <pthread.h>
#include <signal.h>
#include <stdio.h>
#include <time.h>

static pthread_mutex_t mtx = PTHREAD_MUTEX_INITIALIZER;
static pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
static int expire_count = 0;

static void thread_func(union sigval sv) {
    timer_t *tidp = sv.sival_ptr;

    pthread_mutex_lock(&mtx);
    expire_count += (1 + timer_getoverrun(*tidp));
    pthread_mutex_unlock(&mtx);

    pthread_cond_signal(&cond);
}

int main() {
    timer_t tid;
    struct sigevent se;
    se.sigev_notify = SIGEV_THREAD;
    se.sigev_notify_attributes = NULL;
    se.sigev_notify_function = thread_func;
    se.sigev_value.sival_ptr = &tid;
    timer_create(CLOCK_REALTIME, &se, &tid);

    struct itimerspec its;
    its.it_value.tv_sec = 1;
    its.it_value.tv_nsec = 0;
    its.it_interval = its.it_value;
    timer_settime(tid, 0, &its, NULL);

    pthread_mutex_lock(&mtx);

    for (;;) {
        pthread_cond_wait(&cond, &mtx);
        printf("expire_count = %d\n", expire_count);
    }

    timer_delete(tid);
}
```

## Linux-specific

```c
#include <sys/timerfd.h>
int timerfd_create(int clockid, int flags);
int timerfd_settime(int fd, int flags, const struct itimerspec *new_value, struct itimerspec *old_value);
int timerfd_gettime(int fd, struct itimerspec *curr_value);
```

### 示例

```c
#include <stdio.h>
#include <sys/timerfd.h>
#include <time.h>
#include <unistd.h>

int main() {
    int fd = timerfd_create(CLOCK_REALTIME, TFD_CLOEXEC);
    struct itimerspec its;
    its.it_value.tv_sec = 1;
    its.it_value.tv_nsec = 0;
    its.it_interval = its.it_value;
    timerfd_settime(fd, 0, &its, NULL);

    long long expire_count = 0;
    for (;;) {
        long long count;
        read(fd, &count, sizeof(long long));
        expire_count += count;
        printf("expire_count = %lld\n", expire_count);
    }

    close(fd);
}
```
