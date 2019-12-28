---
title: "Posix Thread"
date: 2011-12-23T13:05:31Z
draft: true
---

# Posix Thread

## 线程栈布局

![thread](/image/thread.png)

## 线程创建

```c
#include <pthread.h>
int pthread_create(pthread_t *thread, const pthread_attr_t *attr, void *(*start_routine)(void *), void *arg);
```

## 线程终止

```c
#include <pthread.h>
void pthread_exit(void *value_ptr);
```

## 等待线程终止

```c
#include <pthread.h>
int pthread_join(pthread_t thread, void **value_ptr);
```

默认情况下线程终止后会变成僵尸线程，需要由别的线程进行收尸(`pthread_join`)

## 设置线程为detached

```c
#include <pthread.h>
int pthread_detach(pthread_t thread);
```

将线程设置为detached，线程终止后不会变成僵尸线程

## 获取线程id

```c
#include <pthread.h>
pthread_t pthread_self(void);
```

## 比较线程id是否相等

```c
#include <pthread.h>
int pthread_equal(pthread_t t1, pthread_t t2);
```

## 线程属性

```c
#include <pthread.h>
int pthread_attr_init(pthread_attr_t *attr);
int pthread_attr_destroy(pthread_attr_t *attr);
int pthread_attr_setstack(pthread_attr_t *attr, void *stackaddr, size_t stacksize);
int pthread_attr_getstack(const pthread_attr_t *restrict attr, void **restrict stackaddr, size_t *restrict stacksize);
int pthread_attr_setstacksize(pthread_attr_t *attr, size_t stacksize);
int pthread_attr_getstacksize(const pthread_attr_t *attr, size_t *stacksize);
int pthread_attr_setguardsize(pthread_attr_t *attr, size_t guardsize);
int pthread_attr_getguardsize(const pthread_attr_t *attr, size_t *guardsize);
int pthread_attr_setstackaddr(pthread_attr_t *attr, void *stackaddr);
int pthread_attr_getstackaddr(const pthread_attr_t *attr, void **stackaddr);
int pthread_attr_setdetachstate(pthread_attr_t *attr, int detachstate);
int pthread_attr_getdetachstate(const pthread_attr_t *attr, int *detachstate);
int pthread_attr_setinheritsched(pthread_attr_t *attr, int inheritsched)
int pthread_attr_getinheritsched(const pthread_attr_t *attr, int *inheritsched)
int pthread_attr_setschedparam(pthread_attr_t *attr, const struct sched_param *param)
int pthread_attr_getschedparam(const pthread_attr_t *attr, struct sched_param *param)
int pthread_attr_setschedpolicy(pthread_attr_t *attr, int policy)
int pthread_attr_getschedpolicy(const pthread_attr_t *attr, int *policy)
int pthread_attr_setscope(pthread_attr_t *attr, int contentionscope)
int pthread_attr_getscope(const pthread_attr_t *attr, int *contentionscope);
```

## 示例

```c
// pthread.c
#include <pthread.h>
#include <stdio.h>
#include <unistd.h>

static void *thread_func(void *arg) {
    printf("thread id = %d\n", pthread_self());
    sleep(100);
}

int main() {
    pthread_attr_t attr;
    pthread_attr_init(&attr);
    pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_JOINABLE); // 需收尸

    pthread_t tid;
    pthread_create(&tid, &attr, thread_func, NULL);

    pthread_attr_destroy(&attr);

    pthread_join(tid, NULL); // 收尸
}
```

```sh
$ cc -pthread pthread.c
$ ./a.out &
[1] 15358
$ thread id = 1947428608
$ pstree -ps 15358
systemd(1)───sshd(430)───sshd(14633)───sshd(14642)───bash(14643)───a.out(15358)───{a.out}(15359)
```

花括号里的是线程
