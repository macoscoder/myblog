---
title: "进程终止"
date: 2011-12-23T13:05:31Z
draft: true
---

# 进程终止

## 终止方式

进程终止的两种方式

* 正常终止
* 非正常终止

### 正常终止

系统调用

```c
#include <linux/unistd.h>
void exit_group(int status);
```

```c
#include <unistd.h>
void _exit(int status);
```

库函数

```c
#include <stdlib.h>
void exit(int status);
```

`exit`和`_exit`的区别是`exit`会额外执行以下动作:

* Call exit handlers, in reverse order of their registration
* Flush stdio stream

### 非正常终止

进程收到信号导致的终止称为非正常终止

硬件产生的信号

`SIGILL`, `SIGBUS`, `SIGFPE`, `SIGSEGV`

终端驱动产生的信号

`SIGHUP`, `SIGINT`, `SIGQUIT`

系统调用

```c
#include <signal.h>
int kill(pid_t pid, int sig);
```

```c
int tkill(int tid, int sig);
int tgkill(int tgid, int tid, int sig);
```

库函数

```c
#include <signal.h>
#include <pthread.h>
int pthread_kill(pthread_t thread, int sig);
int pthread_sigqueue(pthread_t thread, int sig, const union sigval value);
```

```c
#include <signal.h>
int raise(int sig);
int killpg(int pgrp, int sig);
int sigqueue(pid_t pid, int sig, const union sigval value);
```

```c
#include <stdlib.h>
void abort(); // SIGABRT
```

### 终止状态(termination status)

进程终止时会产生一个终止状态值[0-255]，这个值会被父进程收集到\
在`shell`中可以通过`shell`变量`$?`来获取`shell`的子进程的终止状态

正常终止的终止状态称为退出状态(由`_exit`等函数的参数指定，0表示成功)\
非正常终止的终止状态为终止该进程的信号值

#### 退出状态(exit status)

尽管`_exit`的`status`参数类型是`int`，但是只有低8位有效(status & oxff)，范围为[0-255]

```c
// exit.c
#include <stdlib.h>
int main() {
    exit(256); // 256 = 0x100
}
```

```sh
$ cc exit.c
$ ./a.out
$ echo $?
0
```

#### 非正常终止的终止状态

前面提到: 非正常终止的终止状态为终止该进程的信号值\
由于linux的标准信号的取值范围是[1-31]，实时信号的取值范围是[SIGRTMIN-SIGRTMAX]

```sh
$ bash -c 'kill -l'
 1) SIGHUP       2) SIGINT       3) SIGQUIT      4) SIGILL       5) SIGTRAP
 6) SIGABRT      7) SIGBUS       8) SIGFPE       9) SIGKILL     10) SIGUSR1
11) SIGSEGV     12) SIGUSR2     13) SIGPIPE     14) SIGALRM     15) SIGTERM
16) SIGSTKFLT   17) SIGCHLD     18) SIGCONT     19) SIGSTOP     20) SIGTSTP
21) SIGTTIN     22) SIGTTOU     23) SIGURG      24) SIGXCPU     25) SIGXFSZ
26) SIGVTALRM   27) SIGPROF     28) SIGWINCH    29) SIGIO       30) SIGPWR
31) SIGSYS      34) SIGRTMIN    35) SIGRTMIN+1  36) SIGRTMIN+2  37) SIGRTMIN+3
38) SIGRTMIN+4  39) SIGRTMIN+5  40) SIGRTMIN+6  41) SIGRTMIN+7  42) SIGRTMIN+8
43) SIGRTMIN+9  44) SIGRTMIN+10 45) SIGRTMIN+11 46) SIGRTMIN+12 47) SIGRTMIN+13
48) SIGRTMIN+14 49) SIGRTMIN+15 50) SIGRTMAX-14 51) SIGRTMAX-13 52) SIGRTMAX-12
53) SIGRTMAX-11 54) SIGRTMAX-10 55) SIGRTMAX-9  56) SIGRTMAX-8  57) SIGRTMAX-7
58) SIGRTMAX-6  59) SIGRTMAX-5  60) SIGRTMAX-4  61) SIGRTMAX-3  62) SIGRTMAX-2
63) SIGRTMAX-1  64) SIGRTMAX
```

我们假设`shell`变量`$?`的值为2，那么我们无法区分是正常终止(`exit(2)`)，还是非正常终止(^C, send `SIGINT`)\
`shell`的办法是，如果进程是被信号终止的，那么设置`$?`的值为**128+signo**

另外

`shell`使用`126`表示命令不能执行，使用`127`表示命令找不到\
`exit`使用`128`表示参数无效

基于这个原因，进程正常终止时我们可以使用的`exit status`范围缩小为**[0-125]**

示例:

```c
// pause.c
#include <unistd.h>
int main() {
    pause();
}
```

```sh
$ cc pause.c
$ ./a.out
^C           # send SIGINT (2)
$ echo $?
130          # 128 + 2
```

`shell`是如何区分子进程是正常终止还是非正常终止的？答案是通过系统调用`wait`的结果参数**wait status**，具体参考`wait(2)`手册
