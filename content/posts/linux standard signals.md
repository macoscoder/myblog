---
title: "linux standard signals"
date: 2011-12-23T13:05:31Z
draft: true
---

# linux standard signals

信号应该算是linux编程中比较复杂的东西了，这里是学习神书(TLPI)中关于linux标准信号的一些记录

## 信号值定义

```path
/usr/include/x86_64-linux-gnu/bits/signum-generic.h
/usr/include/x86_64-linux-gnu/bits/signum.h
```

```c
#define SIGHUP    1  /* Hangup. */
#define SIGINT    2  /* Interactive attention signal. */
#define SIGQUIT   3  /* Quit. */
#define SIGILL    4  /* Illegal instruction. */
#define SIGTRAP   5  /* Trace/breakpoint trap. */
#define SIGABRT   6  /* Abnormal termination. */
#define SIGBUS    7  /* Bus error. */
#define SIGFPE    8  /* Erroneous arithmetic operation. */
#define SIGKILL   9  /* Killed. */
#define SIGUSR1   10 /* User-defined signal 1. */
#define SIGSEGV   11 /* Invalid access to storage. */
#define SIGUSR2   12 /* User-defined signal 2. */
#define SIGPIPE   13 /* Broken pipe. */
#define SIGALRM   14 /* Alarm clock. */
#define SIGTERM   15 /* Termination request. */
#define SIGSTKFLT 16 /* Stack fault (obsolete). */
#define SIGCHLD   17 /* Child terminated or stopped. */
#define SIGCONT   18 /* Continue. */
#define SIGSTOP   19 /* Stop, unblockable. */
#define SIGTSTP   20 /* Keyboard stop. */
#define SIGTTIN   21 /* Background read from control terminal. */
#define SIGTTOU   22 /* Background write to control terminal. */
#define SIGURG    23 /* Urgent data is available at a socket. */
#define SIGXCPU   24 /* CPU time limit exceeded. */
#define SIGXFSZ   25 /* File size limit exceeded. */
#define SIGVTALRM 26 /* Virtual timer expired. */
#define SIGPROF   27 /* Profiling timer expired. */
#define SIGWINCH  28 /* Window size change (4.3 BSD, Sun). */
#define SIGPOLL   29 /* Pollable event occurred (System V). */
#define SIGPWR    30 /* Power failure imminent. */
#define SIGSYS    31 /* Bad system call. */

#define SIGIO  SIGPOLL /* I/O now possible (4.2 BSD). */
#define SIGIOT SIGABRT /* IOT instruction, abort() on a PDP-11. */
#define SIGCLD SIGCHLD /* Old System V name */
```

## 信号默认动作

| signal    | value | action | comment                                    |
|:----------|:-----:|:------:|:-------------------------------------------|
| SIGHUP    | 1     | term   | Terminal hangup                            |
| SIGINT    | 2     | term   | Ctl-C                                      |
| SIGQUIT   | 3     | core   | Ctl-\\                                     |
| SIGILL    | 4     | core   | Illegal Instruction                        |
| SIGTRAP   | 5     | core   | Trace/breakpoint trap                      |
| SIGABRT   | 6     | core   | Abort signal from abort(3)                 |
| SIGBUS    | 7     | core   | Memory access error                        |
| SIGFPE    | 8     | core   | Divide by zero                             |
| SIGKILL   | 9     | term   | **Can not be caught, blocked, or ignored** |
| SIGUSR1   | 10    | term   | User-defined signal 1                      |
| SIGSEGV   | 11    | core   | Invalid memory reference                   |
| SIGUSR2   | 12    | term   | User-defined signal 2                      |
| SIGPIPE   | 13    | term   | Broken pipe: write to pipe with no readers |
| SIGALRM   | 14    | term   | Real-time timer expired                    |
| SIGTERM   | 15    | term   | **Terminate process gracefully**           |
| SIGSTKFLT | 16    | term   | Stack fault on coprocessor (unused)        |
| SIGCHLD   | 17    | ignore | Child stopped or terminated                |
| SIGCONT   | 18    | cont   | Continue if stopped                        |
| SIGSTOP   | 19    | stop   | **Can not be caught, blocked, or ignored** |
| SIGTSTP   | 20    | stop   | Ctl-Z                                      |
| SIGTTIN   | 21    | stop   | Terminal read from background process      |
| SIGTTOU   | 22    | stop   | Terminal write from background process     |
| SIGURG    | 23    | ignore | Urgent(out-of-band) data on socket         |
| SIGXCPU   | 24    | core   | CPU time limit exceeded                    |
| SIGXFSZ   | 25    | core   | File size limit exceeded                   |
| SIGVTALRM | 26    | term   | Virtual timer expired                      |
| SIGPROF   | 27    | term   | Profiling timer expired                    |
| SIGWINCH  | 28    | ignore | Terminal window size change                |
| SIGPOLL   | 29    | term   | Signal-Driven IO                           |
| SIGPWR    | 30    | term   | Power failure                              |
| SIGSYS    | 31    | core   | Invalid system call                        |

## 信号处理器

### 信号处理器设计原则

* 确保只调用可重入函数
* 确保自己是可重入函数
* 保护和恢复现场(e.g.: errno)

### 安装信号处理器

```c
#include <errno.h>
#include <signal.h>
#include <stddef.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

static volatile sig_atomic_t sig_val;

static void handler(int sig) {
    sig_val = sig;
}

static void install_sig_handler(int sig) {
    struct sigaction sa;
    sa.sa_handler = handler;
    sa.sa_flags = 0;
    sigfillset(&sa.sa_mask); // 屏蔽所有信号
    sigaction(sig, &sa, NULL);
}

static void handle_sig() {
    switch (sig_val) {
    case SIGTERM:
    case SIGQUIT:
    case SIGINT:
        // 做一些清理工作
        // ...
        exit(sig_val);
    default:
        break;
    }
}

static void do_task() {
    sleep(1);
}

int main() {
    install_sig_handler(SIGTERM);
    install_sig_handler(SIGQUIT);
    install_sig_handler(SIGINT);

    while (1) {
        handle_sig();
        do_task();
    }
}
```

### 专用栈(alternate signal stack)

默认情况下，信号处理器是使用进程的栈空间，当进程因为某些原因(e.g:无穷递归)耗尽了进程的栈空间时，内核会产生一个`SIGSEGV`信号\
此时由于进程栈空间已经耗尽，无法再执行`SIGSEGV`的信号处理器，解决这个问题的办法是为信号处理器提供预分配的专用栈

### 建立专用栈

```c
#include <signal.h>
int sigaltstack(const stack_t *ss, stack_t *old_ss);

typedef struct {
    void  *ss_sp;     /* Base address of stack */
    int    ss_flags;  /* Flags */
    size_t ss_size;   /* Number of bytes in stack */
} stack_t;
```

#### 示例

```c
#include <signal.h>
#include <stddef.h> // Get NULL
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

static void segv_handler(int sig) {
    printf("sig: %d, %s\n", sig, strsignal(sig)); // UNSAFE
    abort();
}

static void overflow() {
    char a[100000];
    overflow();
}

int main() {
    stack_t ss;
    ss.ss_sp = malloc(SIGSTKSZ);
    ss.ss_size = SIGSTKSZ;
    ss.ss_flags = 0;
    sigaltstack(&ss, NULL);

    struct sigaction sa;
    sa.sa_handler = segv_handler;
    sa.sa_flags = SA_ONSTACK; // 使用专用栈
    sigfillset(&sa.sa_mask);
    sigaction(SIGSEGV, &sa, NULL);

    overflow();
}
```

输出:

```sh
sig: 11, Segmentation fault
[1]    3918 abort      ./a.out
```

### 三参数版本的信号处理器

```c
void handler(int sig, siginfo_t *info, ucontext_t *ucontext);
```

在建立信号处理器时，如果sigaction.sa_flags设置了SA_SIGINFO，那么使用三参数版本的信号处理器

```c
#include <signal.h>
#include <stddef.h> // Get NULL
#include <stdio.h>
#include <unistd.h>
#include <ucontext.h>

static void quit_handler(int sig, siginfo_t *info, void *uctx) {
    // UNSAFE
    printf("si_signo: %d\nsi_code: %d\n", info->si_signo, info->si_code);
    psiginfo(info, "siginfo");
}

int main() {
    struct sigaction sa;
    sa.sa_sigaction = quit_handler;
    sa.sa_flags = SA_SIGINFO; // 使用三参数版本的信号处理器
    sigfillset(&sa.sa_mask);
    sigaction(SIGQUIT, &sa, NULL);

    sleep(100);
}
```

```c
$ ./a.out
^\si_signo: 3
si_code: 128
siginfo: Quit (Signal sent by the kernel 0 0)

$ grep -Rw 'SI_KERNEL' /usr/include
/usr/include/asm-generic/siginfo.h:#define SI_KERNEL    0x80            /* sent by the kernel from somewhere */
/usr/include/x86_64-linux-gnu/bits/siginfo-consts.h:  SI_KERNEL = 0x80          /* Send by kernel.  */
/usr/include/x86_64-linux-gnu/bits/siginfo-consts.h:#define SI_KERNEL   SI_KERNEL
```

`SIGQUIT`信号是终端驱动发射的，所以这里`si_code`的值为`SI_KERNEL`

#### siginfo_t 结构

```c
typedef struct {
    int      si_signo;     /* Signal number */
    int      si_errno;     /* An errno value */
    int      si_code;      /* Signal code */
    int      si_trapno;    /* Trap number for hardware-generated signal (unused on most architectures) */
    pid_t    si_pid;       /* Sending process ID */
    uid_t    si_uid;       /* Real user ID of sending process */
    int      si_status;    /* Exit value or signal */
    clock_t  si_utime;     /* User time consumed */
    clock_t  si_stime;     /* System time consumed */
    sigval_t si_value;     /* Signal value */
    int      si_int;       /* POSIX.1b signal */
    void    *si_ptr;       /* POSIX.1b signal */
    int      si_overrun;   /* Timer overrun count; POSIX.1b timers */
    int      si_timerid;   /* Timer ID; POSIX.1b timers */
    void    *si_addr;      /* Memory location which caused fault */
    long     si_band;      /* Band event (was int in glibc 2.3.2 and earlier) */
    int      si_fd;        /* File descriptor */
    short    si_addr_lsb;  /* Least significant bit of address (since Linux 2.6.32) */
    void    *si_lower;     /* Lower bound when address violation occurred (since Linux 3.19) */
    void    *si_upper;     /* Upper bound when address violation occurred (since Linux 3.19) */
    int      si_pkey;      /* Protection key on PTE that caused fault (since Linux 4.6) */
    void    *si_call_addr; /* Address of system call instruction (since Linux 3.5) */
    int      si_syscall;   /* Number of attempted system call (since Linux 3.5) */
    unsigned int si_arch;  /* Architecture of attempted system call (since Linux 3.5) */
} siginfo_t;
```

这个结构包含的信息非常多，重点关注`si_code`和`si_addr`

* `si_code`: 这个值指示信号源
* `si_addr`: 硬件产生的信号(`SIGBUS`,`SIGSEGV`,`SIGILL`,`SIGFPE`)用这个字段存储相关地址(可以用来定位奔溃地址，获取backtrace等)
  * 对于`SIGBUS`和`SIGSEGV`，`si_addr`包含的是无效的变量地址
  * 对于`SIGILL`和`SIGFPE`，`si_addr`包含的是引发硬件异常的指令地址

详细信息参考神书TPLI 21.4小节及sigaction(2)手册

#### ucontext_t 参数

This structure provides so-called user-context information describing the process state prior to invocation of the signal handler, including the previous process signal mask and saved register values (e.g., program counter and stack pointer).

The `mcontext_t` type is machine-dependent and opaque.\
The `ucontext_t` type is a structure that has at least the following fields:

```c
typedef struct ucontext_t {
    struct ucontext_t *uc_link;
    sigset_t          uc_sigmask;
    stack_t           uc_stack;
    mcontext_t        uc_mcontext;
} ucontext_t;
```

`uc_mcontext` is the machine-specific representation of the saved context, that includes the calling thread's machine registers.

另外，user-context可以用来实现coroutine，即: 用户级线程

详细信息参考`getcontext(3)`, `makecontext(3)`手册

## 同步等待信号

### 方法1: sigwaitinfo

```c
#include <signal.h>
int sigwait(const sigset_t *set, int *sig);
int sigwaitinfo(const sigset_t *set, siginfo_t *info);
int sigtimedwait(const sigset_t *set, siginfo_t *info, const struct timespec *timeout);
```

示例

```c
#include <signal.h>
#include <stddef.h>  // Get NULL
#include <stdio.h>

int main(int argc, const char *argv[]) {
    sigset_t mask;
    sigfillset(&mask);
    sigprocmask(SIG_SETMASK, &mask, NULL);

    for (;;) {
        siginfo_t siginfo;
        int sig = sigwaitinfo(&mask, &siginfo);
        printf("signal: %d\n", sig);
        psiginfo(&siginfo, "siginfo");
    }
}
```

```sh
$ ./a.out
^Csignal: 2
siginfo: Interrupt (Signal sent by the kernel 0 0)
^\signal: 3
siginfo: Quit (Signal sent by the kernel 0 0)
signal: 28
siginfo: Window changed (Signal sent by the kernel 0 0)
```

### 方法2: signalfd

```c
#include <sys/signalfd.h>
int signalfd(int fd, const sigset_t *mask, int flags);
```

示例

```c
#include <signal.h>
#include <stddef.h>  // Get NULL
#include <stdio.h>
#include <sys/signalfd.h>
#include <unistd.h>

int main(int argc, const char *argv[]) {
    sigset_t mask;
    sigfillset(&mask);
    sigprocmask(SIG_SETMASK, &mask, NULL);

    int fd = signalfd(-1, &mask, 0);
    for (;;) {
        struct signalfd_siginfo siginfo;
        read(fd, &siginfo, sizeof(struct signalfd_siginfo));
        printf("signal: %d\n", siginfo.ssi_signo);
    }
}
```

```sh
$ ./a.out
^Csignal: 2
^\signal: 3
```

## 其它等待信号方法

```c
#include <unistd.h>
int pause(void);

#include <signal.h>
int sigsuspend(const sigset_t *sigmask);
```
