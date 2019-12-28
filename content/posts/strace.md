---
title: "trace system calls and signals"
date: 2011-12-23T13:05:31Z
draft: true
---

# trace system calls and signals

## 用法

```sh
strace [options] command [args]
```

## 说明

`strace`运行命令，追踪命令调用的系统调用和收到的信号，默认输出到`stderr`

输出的每行包括

* system call name
* arguments
* return value

例如

```c
open("/dev/null", O_RDONLY) = 3
```

如果系统调用失败，还会输出errno符号和错误描述信息

```c
open("/foo/bar", O_RDONLY) = -1 ENOENT (No such file or directory)
```

## 选项

### 过滤选项

```sh
-e [qualifier=][!]value1[,value2]...
-v 输出execve的envp参数以及结构类型的非压缩版本
```

`qualifier`可以为

* `trace` 追踪指定的系统调用
* `abbrev` 压缩指定的结构类型，默认值为`all`，如果指定`-v`选项，则`abbrev=none`
* `verbose` 解引用指定系统调用的结构参数，默认值为`all`
* `raw` 以十六进制输出指定系统调用的参数
* `signal` 追踪指定的信号，默认值为`all`
* `read` dump指定的描述符的读数据
* `write` dump指定的描述符的写数据
* `fault` 使指定的系统调用失败

如果不指定`qualifier`，则默认为`trace`，例如:

`strace -e open date`等价于`strace -e trace=open date`

`!`的作用是取反，例如:

`strace -e trace=!open date`是追踪除`open`以外的系统调用

### 输出选项

```sh
-o filename 重定向stderr到filename
-s strsize 指定字符串参数的最大输出长度，默认为32
```

### 统计选项

```sh
-c Count time, calls, and errors for each system call and report a summary on program exit.
```

### 追踪选项

```sh
-f 追踪子进程和子线程
-ff 追踪子进程和子线程，输出各个进程的追踪信息到filename.pid，需同时使用-o filename选项
```

### 启动选项

```sh
-p pid 追踪pid指定的进程
```

## 举例

### 查看fork调用clone使用的参数

```c
// fork.c
#include <stdlib.h>
#include <sys/wait.h>
#include <unistd.h>

int main() {
    switch (fork()) {
    case -1:
        exit(1);
    case 0:
        _exit(0);
    default:
        wait(NULL);
    }
}
```

```sh
$ cc fork.c
$ strace -e trace=clone ./a.out
clone(child_stack=NULL, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD, child_tidptr=0x7f5aa2dc29d0) = 8778
--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=8778, si_uid=1000, si_status=0, si_utime=0, si_stime=0} ---
+++ exited with 0 +++
$
```

从上面的输出看到

参数`child_stack`为`NULL`，因为子进程有自己独立的虚拟内存空间，这是与线程的一个区别

`flags`参数为

* CLONE_CHILD_CLEARTID // 当子进程退出时清掉参数`child_tidptr`指向的变量(在子进程中)
* CLONE_CHILD_SETTID // 将子进程id写到参数`child_tidptr`指向的变量中
* SIGCHLD // 指定当子进程终止时，父进程收到的信号

返回值8778是子进程的id

以\-\-\-开头和结尾的行显示了当子进程终止时，父进程收到的信号`SIGCHLD`，后面花括号里显示了结构`siginfo_t`的一些信息

以+++开头和结尾的行显示了进程的终止状态，正常终止或者非正常终止(killed by signal)

### 查看pthread_create调用clone使用的参数

```c
// thread.c
#include <pthread.h>
#include <stdlib.h>
#include <sys/wait.h>
#include <unistd.h>

static void *thread_func(void *arg) {
    pthread_exit(NULL);
}

int main() {
    pthread_t t;
    pthread_create(&t, NULL, thread_func, NULL);
    pthread_join(t, NULL);
}

```

```sh
$ cc -pthread thread.c
$ strace -e trace=clone ./a.out
clone(child_stack=0x7f2912cdbff0, flags=CLONE_VM|CLONE_FS|CLONE_FILES|CLONE_SIGHAND|CLONE_THREAD|CLONE_SYSVSEM|CLONE_SETTLS|CLONE_PARENT_SETTID|CLONE_CHILD_CLEARTID, parent_tidptr=0x7f2912cdc9d0, tls=0x7f2912cdc700, child_tidptr=0x7f2912cdc9d0) = 9003
+++ exited with 0 +++
$
```

从上面的输出看到

设置了`child_stack`参数，因为子进程(线程)共享父进程的虚拟内存空间，需要为子线程分配独立的栈

`flags`参数为

* CLONE_VM // 共享虚拟内存
* CLONE_FS // 共享文件系统相关的属性，如umask, cwd等
* CLONE_FILES // 共享文件描述符表
* CLONE_SIGHAND // 共享信号处置表
* CLONE_THREAD // 子进程与父进程在同一个线程组中
* CLONE_SYSVSEM // share a single list of System V semaphore undo values
* CLONE_SETTLS // 使用参数`tls`描述的buffer作为线程局部存储
* CLONE_PARENT_SETTID // 将子进程id写到结果参数`parent_tidptr`指向的变量中
* CLONE_CHILD_CLEARTID // 当子进程退出时清掉参数`child_tidptr`指向的变量

共享进程属性是指父子进程(task_struct)引用同一个attribute object instance，浅拷贝(引用计数)\
不共享是指父子进程引用不同的attribute object instance，深拷贝

注意观察参数`parent_tidptr`和`child_tidptr`指向同一个地址`0x7f2912cdc9d0`，因为子线程与父进程共享虚拟内存

返回值9003是子进程的id
