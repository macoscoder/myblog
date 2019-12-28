---
title: "The clone() System Call"
date: 2011-12-23T13:05:31Z
draft: true
---

# The clone() System Call

clone()系统调用允许精细控制进程的创建

## 函数原型

```c
#define _GNU_SOURCE
#include <sched.h>
int clone(int (*func)(void *), void *child_stack, int flags, void *arg, /* pid_t *ptid, struct user_desc *tls, pid_t *ctid */ );
```

### 参数说明

* 参数`func`指定子进程执行的函数
* 参数`child_stack`指定`func`的执行栈
* 参数`flags`指定父子进程共享的资源
* 参数`arg`指定调用`func`的参数
* 参数`ptid`提供一个变量(在父进程内存中)，用于保存子进程id
* 参数`tls`指定子线程的线程局部存储的描述符
* 参数`ctid`提供一个变量(在子进程内存中)，用于保存子进程id

### flags说明

* `CLONE_FILES` 父子进程共享打开的文件描述符表(POSIX threads要求此flag)
* `CLONE_FS` 父子进程共享文件系统相关属性，包括`umask`, `root directory`, `current working directory`(POSIX threads要求此flag)
* `CLONE_SIGHAND` 父子进程共享信号处理器表(POSIX threads要求此flag)
* `CLONE_VM` 父子进程共享页表，也即共享虚拟内存页(POSIX threads要求此flag)
* `CLONE_THREAD` 将子进程置于父进程所在的线程组中(POSIX threads要求此flag)
* `CLONE_PARENT_SETTID` 将子进程id写入参数`ptid`指向的变量中(在父进程内存空间)(NPTL线程实现使用这个flag)
* `CLONE_CHILD_SETTID` 将子进程id写入参数`ctid`指向的变量中(在子进程内存空间)
* `CLONE_CHILD_CLEARTID` 当子进程终止时清掉(zero)参数`ctid`指向的变量(NPTL线程实现使用这个flag)
* `CLONE_SETTLS` 参数`tls`描述的buffer作为子线程的线程局部存储(NPTL线程实现使用这个flag)
* `CLONE_SYSVSEM` 父子进程共享System V semaphore undo values
* `CLONE_NEWNS` 父子进程不共享挂载点名称空间
* `CLONE_PTRACE` 子进程可以被追踪
* `CLONE_UNTRACED` 子进程不能被追踪
* `CLONE_VFORK` 挂起父进程，直到子进程调用`exev`或`_exit`
