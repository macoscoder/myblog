---
title: "shell job control"
date: 2011-12-23T13:05:31Z
draft: true
---


# shell job control

`session`和`process group`的设计是为了支持`shell job control`

## 概念说明

* `process group`: 一组进程的集合，又称为`job`; `pgid`等于`process group leader`的`pid`
* `process group leader`: 创建进程组的进程
* `session`: 一些进程组的集合，`sid`等于`session leader`的`pid`
* `session leader`: 创建会话的进程
* `controlling terminal`: 控制终端由`session leader`首次打开一个终端设备时建立，会话中的所有进程共享一个控制终端
* `controlling process`: 控制终端建立后，`session leader`成为该控制终端的`controlling process`
* `job control`: 指用户同时执行多个job的能力，一个job在前台，其它job在后台

补充说明

1. 一个会话最多有一个`foreground process group`，其他进程组为`background process group`
2. 当用户在终端上键入`Ctl-C`, `Ctl-\`, `Ctl-Z`时，相应信号发送给前台进程组的所有成员
3. 当控制终端断开连接时，内核向`session leader`(即: 控制进程)发送`SIGHUP`和`SIGCONT`信号，此时分两种情况
    1. 如果控制进程捕获或忽略了`SIGHUP`信号，则内核不向前台进程组发送`SIGHUP``SIGCONT`信号
    2. 如果控制进程没有捕获或忽略`SIGHUP`信号，则内核向前台进程组发送`SIGHUP`和`SIGCONT`信号

上述第3点的第1小点的一个示例是`shell`，在`shell`退出之前会发送`SIGHUP`和`SIGCONT`信号给`shell`创建的所有进程组的成员\
`nohup(1)`命令忽略`SIGHUP`信号，从而避免被`shell`发送的`SIGHUP`信号杀掉

上述第2点和第3点的第2小点的示例将在后面给出

## 示意图

![session](/image/session.png)

## 进程组

### 获取进程组id

```c
#include <unistd.h>
pid_t getpgrp(void); /* equivalent to getpgid(0); */
pid_t getpgid(pid_t pid); /* obsolete */
```

### 设置进程组id

```c
#include <unistd.h>
int setpgid(pid_t pid, pid_t pgid);
int setpgrp(void); /* equivalent to setpgid(0, 0); obsolete */
```

## 会话

### 获取会话id

```c
#include <unistd.h>
pid_t getsid(pid_t pid);
```

### 创建会话

```c
#include <unistd.h>
pid_t setsid(void);
```

1. 当调用进程非进程组leader时，`setsid`创建会话，其中包含一个进程组，其中包含调用进程，`sid`,`pgid`等于调用进程的`pid`
2. 当调用进程是进程组leader时，`setsid`返回-1，并设置`errno`为`EPERM`

## 控制终端

### 获取控制终端描述符

```c
#include <fcntl.h>
int fd = open("/dev/tty", O_RDWR);
```

如果调用进程没有控制终端，则`open`返回-1，并设置`errno`为`ENXIO`

这个方法可以确保获取控制终端的描述符，而描述符0,1,2可能被重定向

### 移除调用进程的控制终端

```c
#include <fcntl.h>
#include <sys/ioctl.h>
int fd = open("/dev/tty", O_RDWR);
ioctl(fd, TIOCNOTTY);
```

如果调用进程是控制进程，则内核向前台进程组发送`SIGHUP`和`SIGCONT`信号

## 前台进程组

前台进程组是由终端驱动维护(跟踪)的

### 获取控制终端的前台进程组

```c
#include <unistd.h>
pid_t tcgetpgrp(int fd);
int tcsetpgrp(int fd, pid_t pgrp);
```

### 设置控制终端的前台进程组

```c
#include <unistd.h>
int tcsetpgrp(int fd, pid_t pgrp);
```

## 示例

```c
#include "log.h"
#include <errno.h>
#include <fcntl.h>
#include <signal.h>
#include <string.h>
#include <unistd.h>

static pid_t create_child(void (*func)()) {
    pid_t pid = fork();
    switch (pid) {
    case -1:
        fatalf("fork");
    case 0:
        func();
        _exit(0);
    default:
        return pid;
    }
}

static void child() {
    setpgid(0, 0); // 确保无论父进程先运行还是子进程先运行，子进程的pgid都会被设置

    sigset_t mask;
    sigfillset(&mask);
    sigprocmask(SIG_SETMASK, &mask, NULL); // 屏蔽所有信号

    for (;;) {
        int sig;
        sigwait(&mask, &sig);
        infof("child pid = %d, sig = %d (%s)", getpid(), sig, strsignal(sig));
    }
}

int main() {
    pid_t pid = create_child(child);
    setpgid(pid, pid); // 确保无论父进程先运行还是子进程先运行，子进程的pgid都会被设置

    int fd = open("/dev/tty", O_RDWR); // 获取控制终端描述符
    if (fd == -1)
        fatalf("open: %s", strerror(errno));

    tcsetpgrp(fd, pid);         // 将子进程所在的进程组设为前台进程组
    pid_t pgid = tcgetpgrp(fd); // 查看前台进程组id
    infof("foreground pgid = %d", pgid);

    pause();
}
```

### `SIGHUP`信号的演示

```sh
$ tty
/dev/pts/1
$ cc -pthread log.c main.c
$ exec ./a.out 2> log.txt
```

再开一个终端窗口

```sh
$ ps -t pts/1 -o sid,pgid,pid,tty,comm
  SID  PGID   PID TTY      COMMAND
 7234  7234  7234 pts/1    a.out
 7234  7274  7274 pts/1    a.out
$ cat log.txt
info: foreground pgid = 7274
```

从上面的输出看到，pid为7234的进程为控制进程，前台进程组id为7274\
根据前面补充说明的第3点的第2小点可知，当关掉`pts/1`终端窗口时，内核将发送`SIGHUP`和`SIGCONT`给前台进程组

关掉`pts/1`终端窗口后，查看log.txt

```sh
$ cat log.txt
info: foreground pgid = 7274
info: child pid = 7274, sig = 1 (Hangup)
info: child pid = 7274, sig = 18 (Continued)
```

从上面的输出看到，前台进程组中的进程收到了`SIGHUP`和`SIGCONT`信号

### `SIGINT`, `SIGQUIT`, `SIGTSTP`演示

```sh
$ ./a.out
^Cinfo: child pid = 7481, sig = 2 (Interrupt)
^\info: child pid = 7481, sig = 3 (Quit)
^Zinfo: child pid = 7481, sig = 20 (Stopped)
```

从上面输出看到，前台进程组中的进程收到了`SIGINT`, `SIGQUIT`, `SIGTSTP`等信号

## job control

### 在后台执行命令(或管道命令)

```sh
$ sleep 1000 &
[1] 7636
$ sleep 1000 &
[2] 7637
$ sleep 1000 &
[3] 7638
$ jobs
[1]   Running                 sleep 1000 &
[2]-  Running                 sleep 1000 &
[3]+  Running                 sleep 1000 &
```

说明

* 方括号里的号码是任务号
* 方括号后面的是进程id(或管道中最后一个命令的进程id)
* `jobs`输出所有后台任务
* `+`表示当前任务
* `-`表示上一个当前任务

### fg, bg

`fg`命令将指定任务设为前台任务
`bg`命令恢复指定的已停止的后台任务

任务号的引用

* %+ 引用当前任务(默认情况)
* %- 引用上一个当前任务
* %n 引用指定的任务

### job control示意图

![job control](/image/job_control.png)

### job control信号(SIGTSTP, SIGTTIN, SIGTTOU)和终端生成信号（SIGINT, SIGQUIT, SIGHUP）的处理

处理job control信号和终端生成信号的一般规则是，如果这些信号的当前动作是忽略，则不处理

一个例子是：当命令是使用`nohup(1)`执行时，`nohup(1)`会忽略`SIGHUP`信号，这种情况下，调用者的意图就是忽略(不处理)`SIGHUP`信号\
也就是当终端断开连接时，进程继续执行

处理上述的一般规则外，对于`SIGTSTP`信号的处理还需特殊处理

大多数命令行程序都不需要处理`SIGTSTP`信号，但是执行屏幕处理的程序(e.g., vi)需要处理`SIGTSTP`信号\
因为这类程序需要修改终端属性，当进程被挂起时，需要恢复终端设置

`SIGTSTP`信号的设计方法是:

1. 保存当前终端设置
2. 恢复原来的终端设置
3. 重置`SIGTSTP`的动作为`SIG_DFL`
4. raise(SIGTSTP)
5. unblock `SIGTSTP`
6. reblock `SIGTSTP`
7. 重建`SIGTSTP`的信号处理器
8. 恢复步骤1保存的终端设置

说明

* 重置`SIGTSTP`的动作为`SIG_DFL`，以便再次发送SIGTSTP信号后，停止进程
* raise(SIGTSTP)后，并不会立即是进程停止，而是信号被pending
* 在执行信号处理器时，默认当前信号被屏蔽，unblock `SIGTSTP`后，pending的信号立即递送给进程，进程立即停止
* 在进程恢复执行后，reblock `SIGTSTP`，以便后续代码的执行不会再次被`SIGTSTP`中断

### 孤儿进程组

当一个进程组里的所有成员与当前会话中其它进程组都没有血缘关系时，该进程组为孤儿进程组\
这里的血缘关系仅指父关系，即没有父进程，所以称为孤儿进程组

进程组变为孤儿进程组的发生情况: 当该进程组的成员与当前会话中其他进程组最后的血缘关系断掉时发生，也即:

1. 该进程组中最后一个有父进程(在当前会话的其他进程组中)的成员终止时
2. 该进程组中最后一个有父进程(在当前会话的其他进程组中)的成员的父亲终止时

要注意的是，孤儿进程组里的成员并不一定都是孤儿进程，有可能父进程在别的会话中，例如:

```sh
$ ps -o stat,sid,pgid,pid,ppid,comm -s `echo $$`
STAT   SID  PGID   PID  PPID COMMAND
Ss   16173 16173 16173 16172 bash
R+   16173 16188 16188 16173 ps
```

这里的`bash`所在的进程组即为孤儿进程组，`bash`有父进程(16172)，但是不在当前会话(16173)中

当一个进程组变为孤儿进程组时，如果该进程组内有进程处于stop状态，那么内核向该孤儿进程组内所有成员发送一个`SIGHUP`和`SIGCONT`信号\
如果内核不这么做，那些停止的成员可能永远都不会恢复运行

孤儿进程组中的进程读写控制终端时返回`EIO`，而不是收到`SIGTTIN`或`SIGTTOU`信号，这两个信号会导致进程停止，一旦停止，并不能通过job control\
恢复运行

通过`kill(1)`命令显式向孤儿进程组内的进程发送`SIGTSTP`, `SIGTTIN`, `SIGTTOU`信号时：

1. 如果接受进程捕获或忽略了信号，则信号被递送
2. 否则信号不会被递送，而是被内核丢弃
