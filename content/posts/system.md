---
title: "调用sh(1)执行命令"
date: 2011-12-23T13:05:31Z
draft: true
---

# 调用sh(1)执行命令

`system()`使用`fork(2)`创建子进程，然后调用`execl(3)`执行`/bin/sh`

`execl("/bin/sh", "sh", "-c", command, (char *) 0);`

然后调用`waitpid(2)`等待子进程(`/bin/sh`)终止

## 函数原型

```c
#include <stdlib.h>
int system(const char *command);
```

## 返回值

* 如果参数`command`为`NULL`，`system()`返回非0表示`sh(1)`可用，返回0表示`sh(1)`不可用
* 如果`fork(2)`或`waitpid(2)`调用失败，返回-1
* 如果`execl(3)`调用失败，子进程用参数127调用`_exit(2)`
* 如果都成功，返回`waitpid(2)`的第二个参数，即`wait status`

有三种情况下`system()`返回`0x7f00`

1. `execl(3)`调用失败
2. `command`找不到
3. `command`调用`_exit(127);`

## 示例

```c
// system.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/types.h>
#include <sys/wait.h>

static void print_wait_status(int wstatus) {
    printf("wstatus: 0x%04x\n", wstatus);
    if (WIFEXITED(wstatus)) {
        printf("child exited, status=%d\n", WEXITSTATUS(wstatus));
        return;
    }
    if (WIFSIGNALED(wstatus)) {
        int sig = WTERMSIG(wstatus);
        printf("child killed by signal %d (%s)\n", sig, strsignal(sig));
        if (WCOREDUMP(wstatus))
            printf("core dumped\n");
        return;
    }
    if (WIFSTOPPED(wstatus)) {
        int sig = WSTOPSIG(wstatus);
        printf("child stopped by signal %d (%s)\n", sig, strsignal(sig));
        return;
    }
    if (WIFCONTINUED(wstatus)) {
        printf("child continued\n");
        return;
    }
    exit(1);
}

int main() {
    char line[1024];
    for (;;) {
        printf("==> ");
        char *r = fgets(line, sizeof(line), stdin);
        if (r == NULL)
            break;
        int status = system(line);
        if (status == -1)
            exit(1);
        print_wait_status(status);
    }
}
```

### 演示

```sh
$ cc -o system system.c
$ ./system
==> xxx
sh: 1: xxx: not found           // command找不到情况
wstatus: 0x7f00
child exited, status=127
==> exit 127                    // command调用_exit(127)情况
wstatus: 0x7f00
child exited, status=127
==> sudo mv /bin/sh /bin/sh2    // 改名
wstatus: 0x0000
child exited, status=0
==> ls                          // execl调用失败情况
wstatus: 0x7f00
child exited, status=127
==> ^C
$ sudo mv /bin/sh2 /bin/sh
$
```

```sh
$ ./system
==> sleep 1000
^Z
[1]  + 715 suspended  ./system
$ ps
  PID TTY          TIME CMD
  477 pts/0    00:00:00 zsh
  715 pts/0    00:00:00 system
  716 pts/0    00:00:00 sh
  717 pts/0    00:00:00 sleep
  726 pts/0    00:00:00 ps
$ pstree -ps 717
systemd(1)───sshd(395)───sshd(467)───sshd(476)───zsh(477)───system(715)───sh(716)───sleep(717)
$ kill 717
$ fg
[1]  + 715 continued  ./system
Terminated
wstatus: 0x8f00
child exited, status=143        // 128+15
==> sleep 1000
^Z
[1]  + 715 suspended  ./system
$ ps
  PID TTY          TIME CMD
  477 pts/0    00:00:00 zsh
  715 pts/0    00:00:00 system
  772 pts/0    00:00:00 sh
  773 pts/0    00:00:00 sleep
  782 pts/0    00:00:00 ps
$ kill 772
$ fg
[1]  + 715 continued  ./system
wstatus: 0x000f
child killed by signal 15 (Terminated)
==>
```

从上面的输出看出

* 当kill掉sleep时，sh的退出状态为`_exit(128+SIGTERM);`
* 当kill掉sh时，`waitpid`获取的等待状态为0x000f，这表示sh被`SIGTERM`终止

也就是说`system(3)`的返回值反映了了`sh(1)`的终止状态，`sh(1)`的终止状态反映了`command`的终止状态

## 不要在set-user-ID和set-group-ID程序中使用`system()`

```sh
$ id
uid=1000(fly) gid=1000(fly) groups=1000(fly),24(cdrom),25(floppy),27(sudo),29(audio),30(dip),44(video),46(plugdev),108(netdev),999(vboxsf)
$ ls -n ./system
-rwxr-xr-x 1 1000 1000 8984 Jan 26 12:56 system
$ chmod u+s system                                  // 设置成set-user-ID程序
$ ls -n ./system
-rwsr-xr-x 1 1000 1000 8984 Jan 26 12:56 system
$ touch foo
$ ls -n ./foo
-rw-r--r-- 1 1000 1000    0 Jan 26 12:56 foo        // 只有所有者可写
$ su jhm                                            // 切换用户
Password:
$ id
uid=1001(jhm) gid=1001(jhm) groups=1001(jhm),27(sudo)
$ ./system
==> id
uid=1001(jhm) gid=1001(jhm) euid=1000(fly) groups=1001(jhm),27(sudo)
wstatus: 0x0000
child exited, status=0
==> echo xxx >> foo                                 // 用户jhm写foo文件
wstatus: 0x0000
child exited, status=0
==> cat foo
xxx
wstatus: 0x0000
child exited, status=0
==>
```

从上面的演示看出，使用了`system()`函数的set-user-ID程序，可以执行用户`fly`可以执行的所有操作

下面看下`bash(1)`与`sh(1)`的不同之处

```sh
$ su jhm
Password:
jhm@debian:/tmp$ id
uid=1001(jhm) gid=1001(jhm) groups=1001(jhm),27(sudo)
jhm@debian:/tmp$ ./system
==> id
uid=1001(jhm) gid=1001(jhm) euid=1000(fly) groups=1001(jhm),27(sudo)
wstatus: 0x0000
child exited, status=0
==> bash
bash-4.4$ id                                                // bash重置了euid/egid为ruid/rgid
uid=1001(jhm) gid=1001(jhm) groups=1001(jhm),27(sudo)
bash-4.4$ echo yyy >> foo
bash: foo: Permission denied
```

在我的系统(debian)上,/bin/sh是/bin/dash的符号链接

```sh
$ ls -l /bin/sh
lrwxrwxrwx 1 root root 4 Jan 24  2017 /bin/sh -> dash
```

dash是一个符合POSIX标准的轻量级的shell实现，执行效率比bash高，通常用来执行脚本。

## system()对信号的处理

以下面的进程树说明

```sh
system───sh───sleep
```

### 阻塞SIGCHLD

假如`system`进程建立了`SIGCHLD`信号处理器，并在信号处理器中调用`wait(2)`，那么当`sh`进程结束时，`system`进程建立的信号处理器会执行\
并收集了`sh`进程的终止状态，之后`system()`函数里调用的`waitpid()`函数会返回-1并设置`errno`为`EINTR`，指示被信号中断，在这种情况下\
`system()`会重启`waitpid()`调用，然而`waitpid()`会再次返回-1并设置`errno`为`ECHILD`，指示子进程不存在(已在信号处理器里`wait`)

基于这个原因，`system()`函数必须第一时间阻塞`SIGCHLD`，从而确保`waitpid()`可以收集到`sh`的终止状态

### 忽略SIGINT和SIGQUIT

因为`system`, `sh`, `sleep`三个进程在同一个前台进程组中

当在终端上输入`Ctl-C`或`Ctl-\`时，信号会被分别发送到这三个进程

`sh`对`SIGINT`和`SIGQUIT`的处理比较复杂，姑且认为`sh`是忽略这两个信号

忽略`SIGINT`和`SIGQUIT`的意义在于，在终端上键入`Ctl-C`或`Ctl-\`时，这三个进程可以同步\
首先`sleep`进程收到信号后退出,`sh`进程从`wait()`返回退出，`system`进程从`waitpid()`返回继续下一次循环
