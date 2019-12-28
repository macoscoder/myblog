---
title: "互斥锁"
date: 2011-12-23T13:05:31Z
draft: true
---

# 互斥锁

## 安装文档

debian 9上默认没有这个手册，需要自行安装

```sh
sudo apt install glibc-doc
```

```sh
$ dpkg-query -L glibc-doc | grep pthread
/usr/share/man/man3/pthread_atfork.3.gz
/usr/share/man/man3/pthread_cond_init.3.gz
/usr/share/man/man3/pthread_condattr_init.3.gz
/usr/share/man/man3/pthread_key_create.3.gz
/usr/share/man/man3/pthread_mutex_init.3.gz
/usr/share/man/man3/pthread_mutexattr_init.3.gz
/usr/share/man/man3/pthread_mutexattr_setkind_np.3.gz
/usr/share/man/man3/pthread_once.3.gz
/usr/share/man/man3/pthread_cond_broadcast.3.gz
/usr/share/man/man3/pthread_cond_destroy.3.gz
/usr/share/man/man3/pthread_cond_signal.3.gz
/usr/share/man/man3/pthread_cond_timedwait.3.gz
/usr/share/man/man3/pthread_cond_wait.3.gz
/usr/share/man/man3/pthread_condattr_destroy.3.gz
/usr/share/man/man3/pthread_getspecific.3.gz
/usr/share/man/man3/pthread_key_delete.3.gz
/usr/share/man/man3/pthread_mutex_destroy.3.gz
/usr/share/man/man3/pthread_mutex_lock.3.gz
/usr/share/man/man3/pthread_mutex_trylock.3.gz
/usr/share/man/man3/pthread_mutex_unlock.3.gz
/usr/share/man/man3/pthread_mutexattr_destroy.3.gz
/usr/share/man/man3/pthread_mutexattr_getkind_np.3.gz
/usr/share/man/man3/pthread_mutexattr_gettype.3.gz
/usr/share/man/man3/pthread_mutexattr_settype.3.gz
/usr/share/man/man3/pthread_setspecific.3.gz
```

这个包包含了mutex, mutexattr, cond, condattr等手册

## 互斥锁相关函数

```c
#include <pthread.h>

pthread_mutex_t fastmutex = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_t recmutex = PTHREAD_RECURSIVE_MUTEX_INITIALIZER_NP;
pthread_mutex_t errchkmutex = PTHREAD_ERRORCHECK_MUTEX_INITIALIZER_NP;

int pthread_mutex_init(pthread_mutex_t *mutex, const pthread_mutexattr_t *mutexattr);
int pthread_mutex_destroy(pthread_mutex_t *mutex);
int pthread_mutex_lock(pthread_mutex_t *mutex);
int pthread_mutex_unlock(pthread_mutex_t *mutex);
int pthread_mutex_trylock(pthread_mutex_t *mutex);
```

## 示例

```c
// mutex.c
#include <pthread.h>
#include <stdio.h>
#include <unistd.h>

static int global;
static pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

static void *thread_func(void *arg) {
    for (int i = 0; i < 1000000; i++) {
        pthread_mutex_lock(&mutex);
        global++;
        pthread_mutex_unlock(&mutex);
    }
}

int main() {
    pthread_t tid1, tid2;
    pthread_create(&tid1, NULL, thread_func, NULL);
    pthread_create(&tid2, NULL, thread_func, NULL);

    pthread_join(tid1, NULL);
    pthread_join(tid2, NULL);

    printf("global = %d\n", global);
}
```

```sh
$ cc -pthread mutex.c
$ ./a.out
global = 2000000
```

## 初始化一次

```c
#include <pthread.h>
pthread_once_t once_control = PTHREAD_ONCE_INIT;
int pthread_once(pthread_once_t *once_control, void (*init_routine)());
```
