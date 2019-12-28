---
title: "线程专用数据"
date: 2011-12-23T13:05:31Z
draft: true
---

# 线程专用数据

## 函数原型

```c
#include <pthread.h>
int pthread_key_create(pthread_key_t *key, void (*destructor)(void *));
int pthread_key_delete(pthread_key_t key);
int pthread_setspecific(pthread_key_t key, const void *pointer);
void *pthread_getspecific(pthread_key_t key);
```

## 示例

```c
// log.c
#include "log.h"
#include <pthread.h>
#include <stdarg.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

static pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
static pthread_key_t key;

__attribute__((constructor))
static void init() {
    pthread_key_create(&key, free);
}

static void output(const char *prefix, const char *format, va_list params) {
    pthread_mutex_lock(&mutex);
    const char *tag = pthread_getspecific(key);
    if (tag)
        fputs(tag, stderr);
    fputs(prefix, stderr);
    vfprintf(stderr, format, params);
    fputs("\n", stderr);
    pthread_mutex_unlock(&mutex);
}

void set_thread_tag(const char *tag) {
    char *buf = strdup(tag);
    pthread_setspecific(key, buf);
}

void debugf(const char *format, ...) {
    va_list params;
    va_start(params, format);
    output("DEBUG: ", format, params);
    va_end(params);
}

void infof(const char *format, ...) {
    va_list params;
    va_start(params, format);
    output("INFO: ", format, params);
    va_end(params);
}

void fatalf(const char *format, ...) {
    va_list params;
    va_start(params, format);
    output("FATAL: ", format, params);
    va_end(params);
    exit(1);
}
```

```c
// log.h
#ifndef LOG_H
#define LOG_H

extern void set_thread_tag(const char *tag);

extern void debugf(const char *format, ...);
extern void infof(const char *format, ...);
extern void fatalf(const char *format, ...);

#endif // LOG_H
```

```c
// main.c
#include "go.h"
#include "log.h"
#include <unistd.h>

void foo(void *arg) {
    set_thread_tag("<thread1> ");
    for (;;) {
        debugf("foo");
        sleep(1);
    }
}

void bar(void *arg) {
    set_thread_tag("<thread2> ");
    for (;;) {
        debugf("bar");
        sleep(1);
    }
}

int main() {
    go(foo, NULL);
    go(bar, NULL);
    pause();
}
```

这个例子是给不同线程的`log`加`tag`，在调试时可以使用`grep`过滤不同线程的`log`\
`go`函数参考[另一篇博客](http://www.flyblog.tech/cond)

演示

```sh
$ cc -pthread go.c log.c main.c
$ ./a.out
<thread2> DEBUG: bar
<thread1> DEBUG: foo
<thread1> DEBUG: foo
<thread2> DEBUG: bar
<thread1> DEBUG: foo
<thread2> DEBUG: bar
^C
$
```
