---
title: "等待子进程"
date: 2011-12-23T13:05:31Z
draft: true
---

# 等待子进程

## 系统调用

```c
#include <sys/types.h>
#include <sys/wait.h>
pid_t wait(int *wstatus);
pid_t waitpid(pid_t pid, int *wstatus, int options);
int waitid(idtype_t idtype, id_t id, siginfo_t *infop, int options);
```

```c
#include <sys/types.h>
#include <sys/time.h>
#include <sys/resource.h>
#include <sys/wait.h>
pid_t wait3(int *wstatus, int options, struct rusage *rusage);
pid_t wait4(pid_t pid, int *wstatus, int options, struct rusage *rusage);
```

## waitpid

### pid参数

* pid > 0 指定等待的子进程id
* pid = 0 等待父进程所在进程组中的任意子进程
* pid < -1 等待进程组id等于-pid中的任意子进程
* pid = -1 等待任意子进程

### options参数

options是位掩码参数

* `WUNTRACED` 当等待的子进程停止运行时`waitpid`返回
* `WCONTINUED` 当等待的子进程恢复运行时`waitpid`返回
* `WNOHANG` 当参数`pid`指定的子进程未终止时，`waitpid`立即返回，且返回值为0

有两种方法停止一个进程:

* 通过ptrace(2)系统调用
* 通过信号(SIGSTOP, SIGTSTP, SIGTTIN, SIGTTOU)

当一个进程没有被追踪时，如果指定`WUNTRACED`选项，当进程停止时 `waitpid` 会返回\
当一个进程被追踪时，无论是否指定`WUNTRACED`选项，当进程停止时 `waitpid` 会返回

`wait(&wstatus)`等价于`waitpid(-1, &wstatus, 0)`

### wstatus参数 (wait status)

结果参数`wstatus`用来获取当前进程所处的状态:

* 正常终止状态
* 非正常终止状态
* 停止运行状态
* 恢复运行状态

术语 `wait status` 包含上述四种情况\
术语 `termination status` 包含上述前两种情况\
术语 `exit status` 指的是上述第一种情况

`shell`变量`$?`获取的是最后执行的命令的`termination status`

尽管`wstatus`是指向`int`类型，但是只使用了低16位，如下图所示

![file system](/image/ws.png)

检查`wstatus`内容的标准宏，注意下面的`wstatus`参数是`int`类型，不是`int*`类型

```c
#include <sys/wait.h>
WIFEXITED(wstatus)
WIFSIGNALED(wstatus)
WIFSTOPPED(wstatus)
WIFCONTINUED(wstatus)
```

* `WIFEXITED` 返回是否是正常终止状态
* `WIFSIGNALED` 返回是否是非正常终止状态
* `WIFSTOPPED` 返回是否是停止状态
* `WIFCONTINUED` 返回是否是恢复状态

```c
#include <sys/wait.h>
WEXITSTATUS(wstatus)
WTERMSIG(wstatus)
WCOREDUMP(wstatus)
WSTOPSIG(wstatus)
```

* `WEXITSTATUS` 当进程正常终止时，获取进程的退出状态值
* `WTERMSIG` 当进程异常终止时，获取导致进程终止的信号值
* `WCOREDUMP` 当进程异常终止时，获取是否产生核心转储文件
* `WSTOPSIG` 当进程停止时，获取导致进程停止的信号值

### waitpid示例

```c
#include "log.h"
#include <string.h>
#include <sys/wait.h>
#include <unistd.h>

static void print_wait_status(int wstatus) {
    debugf("wstatus: 0x%x", wstatus);
    if (WIFEXITED(wstatus)) {
        debugf("child exited, status=%d", WEXITSTATUS(wstatus));
        return;
    }
    if (WIFSIGNALED(wstatus)) {
        int sig = WTERMSIG(wstatus);
        debugf("child killed by signal %d (%s)", sig, strsignal(sig));
        if (WCOREDUMP(wstatus))
            debugf("core dumped");
        return;
    }
    if (WIFSTOPPED(wstatus)) {
        int sig = WSTOPSIG(wstatus);
        debugf("child stopped by signal %d (%s)", sig, strsignal(sig));
        return;
    }
    if (WIFCONTINUED(wstatus)) {
        debugf("child continued");
        return;
    }
    fatalf("print_wait_status");
}

int main() {
    int pid = fork();
    switch (pid) {
    case -1: // error
        fatalf("fork");
    case 0: // child
        sleep(30);
        _exit(0);
    default: // parent
        break;
    }
    debugf("child pid %d", pid);
    int wstatus;
    for (;;) {
        int pid = waitpid(pid, &wstatus, WUNTRACED | WCONTINUED);
        if (pid == -1)
            break;
        print_wait_status(wstatus);
    }
}
```

```sh
$ ./a.out&
[2] 31707
debug: child pid 31710
$ kill -stop 31710
debug: wstatus: 0x137f
debug: child stopped by signal 19 (Stopped (signal))
$ kill -cont 31710
debug: wstatus: 0xffff
debug: child continued
$ kill 31710
debug: wstatus: 0xf
debug: child killed by signal 15 (Terminated)
[2]  + 31707 done       ./a.out
```

```sh
$ ./a.out
debug: child pid 31756
debug: wstatus: 0x0
debug: child exited, status=0
```

## waitid

### idtype参数与id参数

`idtype`参数:

* `P_ALL` 等待任意子进程，忽略`id`参数
* `P_PID` 等待`id`指定的子进程
* `P_PGID` 等待`id`指定的进程组中的任意子进程

`getpgrp(2)`可以获取调用进程的进程组id

### options 选项参数

* `WEXITED` 当等待的子进程正常终止，非正常终止时，`waitid` 返回
* `WSTOPPED` 当等待的子进程停止运行时，`waitid`返回
* `WCONTINUED` 当等待的子进程恢复运行时，`waitid`返回
* `WNOHANG` 当等待的子进程无状态变化时，`waitpid`立即返回，且返回值为0
* `WNOWAIT` 只收集子进程的状态信息，子进程继续保持可等待状态

### infop结果参数

`waitid`收集的子进程状态信息写入参数infop指向的结构中，写入的字段有:

* `si_code` 状态码
  * `CLD_EXITED` 指示子进程正常终止
  * `CLD_KILLED` 指示子进程非正常终止
  * `CLD_DUMPED` 指示子进程非正常终止，并产生核心转储
  * `CLD_STOPPED` 指示子进程停止运行
  * `CLD_CONTINUED` 指示子进程恢复运行
  * `CLD_TRAPPED` 指示被追踪的子进程停止运行
* `si_pid` 子进程id
* `si_signo` 设置为`SIGCHLD`
* `si_status` 退出状态或信号值
* `si_uid` 子进程的真实用户id

两种情况下`waitid`返回0

* 成功收集子进程的状态信息
* 当指定`WNOHANG`选项时，子进程无状态变化时

要区分这两种情况的一个办法是:\
在调用`waitid`函数前，将`infop`指向的结构清零，在函数返回后，根据结构是否被填充值来区分

### waitid示例

```c
#include "log.h"
#include <string.h>
#include <sys/wait.h>
#include <unistd.h>

void print_wait_siginfo(siginfo_t *infop) {
    switch (infop->si_code) {
    case CLD_EXITED:
        debugf("child exited, status=%d", infop->si_status);
        break;
    case CLD_KILLED:
        debugf("child killed by signal %d (%s)", infop->si_status,
               strsignal(infop->si_status));
        break;
    case CLD_DUMPED:
        debugf("child killed by signal %d (%s) core dumped", infop->si_status,
               strsignal(infop->si_status));
        break;
    case CLD_STOPPED:
        debugf("child stopped by signal %d (%s)", infop->si_status,
               strsignal(infop->si_status));
        break;
    case CLD_CONTINUED:
        debugf("child continued by signal %d (%s)", infop->si_status,
               strsignal(infop->si_status));
        break;
    case CLD_TRAPPED:
        debugf("child trapped by signal %d (%s)", infop->si_status,
               strsignal(infop->si_status));
        break;
    default:
        fatalf("print_wait_siginfo");
        break;
    }
}

int main() {
    int pid = fork();
    switch (pid) {
    case -1: // error
        fatalf("fork");
    case 0: // child
        sleep(30);
        _exit(0);
    default: // parent
        break;
    }
    debugf("child pid %d", pid);
    for (;;) {
        siginfo_t si;
        bzero(&si, sizeof(siginfo_t));
        int r = waitid(P_PID, pid, &si, WEXITED | WSTOPPED | WCONTINUED);
        if (r == -1)
            break;
        print_wait_siginfo(&si);
    }
}
```

```sh
$ ./a.out&
[1] 32205
debug: child pid 32208
$ kill -stop 32208
debug: child stopped by signal 19 (Stopped (signal))
$ kill -cont 32208
debug: child continued by signal 18 (Continued)
$ kill 32208
debug: child killed by signal 15 (Terminated)
[1]  + 32205 done       ./a.out
```

```sh
$ ./a.out
debug: child pid 32286
debug: child exited, status=0
```

## wait3和wait4

除了rusage参数外

```c
wait3(&wstatus, options, NULL);
```

等价于

```c
waitpid(-1, &wstatus, options);
```

除了rusage参数外

```c
wait4(pid, &wstatus, options, NULL);
```

等价于

```c
waitpid(pid, wstatus, options);
```

这两个系统调用是**非标准的**，**不推荐使用**

关于进程的资源使用信息可以通过`getrusage(2)`获取
