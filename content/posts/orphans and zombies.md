---
title: "孤儿进程和僵尸进程"
date: 2011-12-23T13:05:31Z
draft: true
---

# 孤儿进程和僵尸进程

## 孤儿进程

当父进程死了之后，子进程变成孤儿进程，并被pid为1的进程(`init`/`systemd`)收养\
孤儿进程只是一种称谓，并不是内核中有什么数据结构去标识一个进程是孤儿进程，判断一个进程是否是孤儿进程的一个方法是:\
使用`getppid`系统调用获取父进程id，如果是1，则该进程是孤儿进程，前提是该进程的生父不是`init`\
如果该进程的生父是`init`，那么该进程永远不会变成孤儿进程，因为`init`进程是永远不死的

## 僵尸进程

默认情况下当子进程死后会变成僵尸(活着的尸体)进程\
僵尸进程的大部分资源已经被回收，只有一个进程表项仍然在内核的进程表中\
当父进程调用`wait`执行收尸操作后，该表项会从进程表中移除\
如果父进程没有收尸，自己就死了，那么该僵尸进程变成孤儿僵尸进程，由`init`进程替其收尸

僵尸进程就像电影里的僵尸一样，是杀不死的，即使用`SIGKILL`也杀不死

前面说了默认情况下当子进程死后会变成僵尸进程，但是有两个方法可以避免子进程变成僵尸进程

* 父进程显式忽略`SIGCHLD`信号
* 通过`sigaction`设置`SIGCHLD`信号处理器时，指定`SA_NOCLDWAIT`标志

这种情况下，父进程不能通过`wait`获取子进程的死因，但是`wait`会阻塞，直到所有子进程都终止后才返回且返回值为-1

## 孤儿进程示例

```c
// orphan.c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main() {
    int pid = fork();
    switch (pid) {
    case -1: // error
        exit(1);
    case 0: // child
        pause();
        _exit(0);
    default: // parent
        break;
    }
    fprintf(stderr, "child(%d)\n", pid);
    fprintf(stderr, "parent sleeping...\n");
    sleep(10);
    fprintf(stderr, "parent(%d) exited\n", getpid());
}
```

```sh
$ cc orphan.c
$ ./a.out
child(1427)
parent sleeping...
parent(1426) exited
$
```

另开一个会话，在父进程退出之前查看进程树

```sh
$ pstree -ps 1427
systemd(1)───sshd(512)───sshd(22582)───sshd(22600)───zsh(22601)───a.out(1426)───a.out(1427)
```

在父进程退出之后查看进程树

```sh
$ pstree -ps 1427
systemd(1)───a.out(1427)
```

可以看到子进程变成孤儿进程，被`systemd`进程收养了

## 僵尸进程示例

```c
// zombie.c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main() {
    int pid = fork();
    switch (pid) {
    case -1: // error
        exit(1);
    case 0: // child
        fprintf(stderr, "child(%d) exited\n", getpid());
        _exit(0);
    default: // parent
        break;
    }
    pause();
}
```

```sh
$ cc zombie.c
$ ./a.out&
[1] 1886
child(1889) exited
```

```sh
$ ps -eo ppid,pid,s,comm | grep -E 'PPID|a.out'
 PPID   PID S COMMAND
22601  1886 S a.out
 1886  1889 Z a.out <defunct>
```

从上面的演示可以看出，子进程退出后，父进程没有收尸(`wait`)，子进程变为僵尸状态(状态列`Z`和命令列的`<defunct>`)

尝试强行杀掉僵尸进程

```sh
$ kill -KILL 1889
$ ps -eo ppid,pid,s,comm | grep -E 'PPID|a.out'
 PPID   PID S COMMAND
22601  1886 S a.out
 1886  1889 Z a.out <defunct>
```

从上面的输出看出，僵尸进程还在，也就是`SIGKILL`也是杀不死僵尸进程的

杀掉父进程

```sh
$ kill 1886
[1]  + 1886 terminated  ./a.out
$ ps -e | grep a.out
$
```

从上面看到，当父进程被杀掉后，僵尸子进程不见了，僵尸子进程是被`init`进程收尸了
