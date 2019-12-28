---
title: "SIGCHLD 信号处理器"
date: 2011-12-23T13:05:31Z
draft: true
---

# `SIGCHLD`信号处理器

## 示例

```c
#include <errno.h>
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/wait.h>
#include <unistd.h>

static void handle_sigchld(int sig) {
    int saved_errno = errno;

    for (;;) {
        int pid = waitpid(-1, NULL, WNOHANG);
        if (pid <= 0)
            break;
        fprintf(stderr, "child(%d) reaped\n", pid); // UNSAFE
    }

    errno = saved_errno;
}

int main(int argc, char *argv[]) {
    struct sigaction sa;
    sa.sa_handler = handle_sigchld;
    sa.sa_flags = 0;
    sigemptyset(&sa.sa_mask);
    sigaction(SIGCHLD, &sa, NULL);

    for (int i = 1; i < argc; i++) {
        int sec = atoi(argv[i]);
        switch (fork()) {
        case -1: // error
            exit(1);
        case 0: // child
            fprintf(stderr, "child(%d) started, sleep %ds\n", getpid(), sec);
            sleep(sec);
            _exit(0);
        default: // parent
            break;
        }
    }

    for (;;) {
        pause();
    }
}
```

```sh
$ ./a.out 1 2 3
child(4193) started, sleep 2s
child(4194) started, sleep 3s
child(4192) started, sleep 1s
child(4192) reaped
child(4193) reaped
child(4194) reaped
^C
$
```

## 说明

因为标准信号没有队列，如果连续收到`SIGCHLD`，信号处理器只会调用一次，所以在`handle_sigchld`函数里循环调用非阻塞(`WNOHANG`)的`waitpid`
