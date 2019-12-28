---
title: "条件变量"
date: 2011-12-23T13:05:31Z
draft: true
---

# 条件变量

## 函数原型

```c
#include <pthread.h>

pthread_cond_t cond = PTHREAD_COND_INITIALIZER;

int pthread_cond_init(pthread_cond_t *cond, pthread_condattr_t *cond_attr);
int pthread_cond_destroy(pthread_cond_t *cond);
int pthread_cond_signal(pthread_cond_t *cond);
int pthread_cond_broadcast(pthread_cond_t *cond);
int pthread_cond_wait(pthread_cond_t *cond, pthread_mutex_t *mutex);
int pthread_cond_timedwait(pthread_cond_t *cond, pthread_mutex_t *mutex, const struct timespec *abstime);
```

## 用条件变量实现golang的channel

golang的`channel`是goroutine之间通信的通道

要实现`channel`，需要先实现一个队列

```c
// queue.c
#include "queue.h"
#include <stdlib.h>

struct node {
    void *value;        // 用户数据
    struct node *next;  // 指向下一个结点
};

struct queue {
    struct node *front; // 指向队头
    struct node *rear;  // 指向队尾
    int len;            // 队列长度
};

// 新建空队列
struct queue *queue_new() {
    struct queue *q = calloc(1, sizeof(struct queue));
    q->front = NULL;
    q->rear = NULL;
    q->len = 0;
    return q;
}

// 什么也不做的删除器
static inline void _del(void *val) {
}

// 释放队列
void queue_free(struct queue *q, void (*del)(void *)) {
    del = del ?: _del;
    while (q->len > 0)
        del(queue_dequeue(q));
    free(q);
}

// 向队尾添加结点
void queue_enqueue(struct queue *q, void *val) {
    struct node *node = calloc(1, sizeof(struct node));
    node->value = val;
    node->next = NULL;
    if (q->len == 0) {
        q->front = node;
        q->rear = node;
    } else {
        q->rear->next = node;
        q->rear = node;
    }
    q->len++;
}

// 从队头删除结点
void *queue_dequeue(struct queue *q) {
    void *val = q->front->value;
    struct node *front = q->front->next;
    free(q->front);
    q->front = front;
    q->len--;
    if (q->len == 0)
        q->rear = NULL;
    return val;
}

int queue_len(struct queue *q) {
    return q->len;
}
```

```c
// queue.h
#ifndef QUEUE_H
#define QUEUE_H

typedef struct queue queue;

extern queue *queue_new();
extern void queue_free(queue *q, void (*del)(void *));
extern void queue_enqueue(queue *q, void *val);
extern void *queue_dequeue(queue *q);
extern int queue_len(queue *q);

#endif // QUEUE_H
```

有了`queue`之后可以实现`channel`

```c
// channel.c
#include "channel.h"
#include "queue.h"
#include <pthread.h>
#include <stdlib.h>
#include <time.h>

// 超时时间
// 接收线程定时醒来查看channel是否有数据
// 发送线程定时醒来查看channel是否有容量
#ifndef TIMEOUT
#define TIMEOUT 100000000 // 100ms
#endif

struct channel {
    pthread_cond_t cond_send; // 条件变量，发送线程等待条件
    pthread_cond_t cond_recv; // 条件变量，接收线程等待条件
    pthread_mutex_t mutex;    // 互斥锁，负责保护q
    queue *q;                 // 保存数据的队列
    int cap;                  // 最大容量
};

// 创建channel
struct channel *channel_make(int cap) {
    struct channel *c = calloc(1, sizeof(struct channel));
    pthread_cond_init(&c->cond_send, NULL);
    pthread_cond_init(&c->cond_recv, NULL);
    pthread_mutex_init(&c->mutex, NULL);
    c->q = queue_new();
    c->cap = cap;
    return c;
}

// 释放channel
void channel_release(struct channel *c, void (*del)(void *)) {
    pthread_cond_destroy(&c->cond_send);
    pthread_cond_destroy(&c->cond_recv);
    pthread_mutex_destroy(&c->mutex);
    queue_free(c->q, del);
    free(c);
}

static struct timespec make_abstime() {
    struct timespec ts;
    clock_gettime(CLOCK_REALTIME, &ts);
    ts.tv_sec += TIMEOUT / 1000000000;
    ts.tv_nsec += TIMEOUT % 1000000000;
    return ts;
}

// 发送数据到channel里
void channel_send(struct channel *c, void *val) {
    pthread_mutex_lock(&c->mutex);
    queue_enqueue(c->q, val);
    while (queue_len(c->q) > c->cap) { // 当队列满时通知接收线程
        pthread_cond_signal(&c->cond_recv);
        struct timespec ts = make_abstime();
        pthread_cond_timedwait(&c->cond_send, &c->mutex, &ts);
    }
    pthread_mutex_unlock(&c->mutex);
}

// 从channel接收数据
void *channel_receive(struct channel *c) {
    pthread_mutex_lock(&c->mutex);
    while (queue_len(c->q) == 0) { // 当队列空时通知发送线程
        pthread_cond_signal(&c->cond_send);
        struct timespec ts = make_abstime();
        pthread_cond_timedwait(&c->cond_recv, &c->mutex, &ts);
    }
    void *val = queue_dequeue(c->q);
    pthread_mutex_unlock(&c->mutex);
    return val;
}

// channel当前数据量
int channel_len(struct channel *c) {
    pthread_mutex_lock(&c->mutex);
    int len = queue_len(c->q);
    pthread_mutex_unlock(&c->mutex);
    return len;
}

// channel最大容量
int channel_cap(struct channel *c) {
    return c->cap;
}
```

```c
// channel.h
#ifndef CHANNEL_H
#define CHANNEL_H

typedef struct channel channel;

extern channel *channel_make(int cap);
extern void channel_release(channel *c, void (*del)(void *));
extern void channel_send(channel *c, void *val);
extern void *channel_receive(channel *c);
extern int channel_len(channel *c);
extern int channel_cap(channel *c);

#endif // CHANNEL_H
```

用线程代替goroutine

```c
// go.c
#include "go.h"
#include <pthread.h>
#include <stdlib.h>

struct start_arg {
    void (*func)(void *);
    void *arg;
};

static void *start(void *arg) {
    pthread_setcancelstate(PTHREAD_CANCEL_DISABLE, NULL);
    pthread_detach(pthread_self());
    struct start_arg *sa = arg;
    sa->func(sa->arg);
    free(sa);
    return NULL;
}

void go(void (*func)(void *arg), void *arg) {
    struct start_arg *sa = malloc(sizeof(struct start_arg));
    sa->func = func;
    sa->arg = arg;
    pthread_t tid;
    pthread_create(&tid, NULL, start, sa);
}
```

```c
// go.h
#ifndef GO_H
#define GO_H

extern void go(void (*func)(void *arg), void *arg);

#endif // GO_H
```

用例

```c
// main.c
#include "channel.h"
#include "go.h"
#include <stdio.h>
#include <stdlib.h>
#include <time.h>

// 准备放在channel中的元素
struct element {
    int val;
};

static struct element *element_new(int val) {
    struct element *e = malloc(sizeof(struct element));
    e->val = val;
    return e;
}

static void element_free(struct element *e) {
    free(e);
}

// 生产者线程
static void run(void *arg) {
    channel *c = arg;
    for (int i = 0;; i++) {
        struct element *e = element_new(i); // 由生产者线程创建
        channel_send(c, e);
        printf("send %d\n", e->val);
        // 睡眠一会儿，降低生产速度
        // 如果生产速度太快，就会填满channel才会通知消费者线程
        // 如果生产速度太慢，消费者线程会自己醒来主动查看channel是否有数据
        // 同理，对于消费者
        // 如果消费速度太快，就会清空channel才会通知生产者线程
        // 如果消费速度太慢，生产者线程会自己醒来主动查看channel是否有容量
        struct timespec ts = {.tv_sec = 0, .tv_nsec = 50000000}; // 50ms
        nanosleep(&ts, NULL);
    }
}

int main() {
    // 创建一个容量为10的channel
    channel *c = channel_make(10);

    // 启动生产者线程
    go(run, c);

    // 主线程作为消费者
    for (;;) {
        struct element *e = channel_receive(c);
        printf("receive %d\n", e->val);
        element_free(e); // 由消费者线程释放
    }
}
```

运行log

```sh
$ cc -pthread queue.c channel.c go.c main.c
$ ./a.out
send 0
send 1
receive 0
receive 1
send 2
send 3
receive 2
receive 3
send 4
^C
$
```

```sh
$ cc -pthread -DTIMEOUT=1000000000 queue.c channel.c go.c main.c
$ ./a.out
send 0
send 1
send 2
send 3
send 4
send 5
send 6
send 7
send 8
send 9
receive 0
receive 1
receive 2
receive 3
receive 4
receive 5
receive 6
receive 7
receive 8
receive 9
receive 10
send 10
send 11
send 12
send 13
send 14
send 15
send 16
send 17
^C
```
