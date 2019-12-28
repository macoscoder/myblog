---
title: "终端"
date: 2011-12-23T13:05:31Z
draft: true
---

# 终端

## 简介

终端是通过RS232串口连接到计算机的一种设备，它有一个黑白显示器(24*80)和一个键盘组成\
`tty`这个词是`teletype`的缩写，称为电传打字机，这是早期的终端设备，它有一个打印机和一个键盘组成

上面所说的这种传统的终端已经灭绝了，取而代之的是虚拟终端(virtual terminal)和伪终端(pseudoterminal)

### virtual terminal

虚拟终端有时候也被称为虚拟控制台(virtual console)

虚拟终端对应的设备文件为`/dev/ttyn`，n的范围是[1-63]，通常前6个虚拟终端可以通过快捷键来切换\
在X环境下可以通过`Ctl+Alt+F[1-6]`来切换到对应的虚拟终端，在虚拟终端环境下可以通过`Alt+F[1-6]`或`Alt+<-`和`Alt+->`来切换\
其他虚拟终端可以通过如下命令来切换，以`/dev/tty10`为例:

```sh
# openvt -c 10 agetty tty10 linux ; chvt 10
```

还有三个特殊的设备文件`/dev/tty`, `/dev/tty0`, `/dev/console`

* `/dev/tty`指向进程的控制终端(如果有的话)，也就是进程的`stdin`,`stdout`,`stderr`指向的设备文件(如果没有重定向的话)
* `/dev/tty0`指向当前激活的虚拟终端
* `/dev/console` 是system console，linux内核日志输出的地方，默认情况下是指向`/dev/tty0`，也就是指向当前激活的虚拟终端

示例验证

```sh
# tty                       查看当前所在的虚拟终端
/dev/tty1
# sleep 5; echo hello > /dev/tty0
```

执行上述命令后，立即用`Alt+F2`切换到第二个虚拟终端(`/dev/tty2`)，5s后可看到终端输入hello，这证明了`/dev/tty0`是指向当前的虚拟终端

另一个查看当前`/dev/console`和`/dev/tty0`指向的终端的方法

```sh
$ cat /sys/devices/virtual/tty/console/active
tty0
$ cat /sys/devices/virtual/tty/tty0/active
tty2
```

### pseudoterminal

伪终端是一对设备，主设备文件为`/dev/ptmx`，从设备文件为`/dev/pts/n`，n的范围为[0-4096]\
最大值可通过`/proc/sys/kernel/pty/max`文件配置

主设备文件是一个多路器，全称`pty master multiplexer`，多路器是一种注册机制，可通过打开`/dev/ptmx`文件来注册\
打开`/dev/ptmx`文件返回一个主设备的文件描述符，通过此描述符可以获取注册的从设备文件名

```c
#include <fcntl.h>
#include <stdlib.h>
int ptm_fd = open("/dev/ptmx", O_RDWR | O_NOCTTY);
const char *pts_name = ptsname(ptm_fd);
```

打开从设备文件得到从设备文件的描述符，在主设备描述符上写数据，可以通过从设备描述符读出来，反之亦然。这类似双向管道

示意图

![pty](/image/pty.png)

上图中`driver program`是父进程，`terminal-oriented program`是子进程

从图中看到，伪终端对是一种进程间通信的技术，其中的从设备相当于传统的终端设备，主设备与从设备连接形成一个通信通道

进程间通信技术有很多，为什么需要使用伪终端技术来进行通信呢？

原因是已有的面向终端的程序必须在终端上使用，在不修改已有程序的的情况下要想与之通信，必须提供一种终端支持传统终端的功能，\
而且能将对终端的IO导向另一个进程，基于此原因，内核引入了伪终端对，面向终端的程序与从设备交互，`driver program`与主设备交互

典型的`driver program`有`terminal emulator`和`sshd`

ssd使用伪终端示意图

![ssh](/image/ssh.png)

示例(在mac上执行)

```sh
$ ssh -t fly@192.168.0.112 vi
fly@192.168.0.112's password:
```

上述命令在我的`debian`系统上打开vi程序

在`debian`上执行下列命令

```sh
$ ps -e | grep vi
Sl     514   514   516 ?        VBoxService
Sl     748   748  1268 ?        dconf-service
Ss+   1525  1525  1525 pts/0    vi
$ pstree -ps 1525
systemd(1)───sshd(483)───sshd(1518)───sshd(1524)───vi(1525)
```

从上述输出看到，`vi(1525)`的父进程是`sshd(1524)`，`sshd(1524)`和`vi(1525)`这两个进程通过伪终端通信，也就是上面示意图的右图

在我的debian系统桌面环境下执行下列命令

```sh
$ pstree -ps `echo $$`
systemd(1)───systemd(579)───gnome-terminal-(1627)───bash(1633)───pstree(1646)
```

`gnome-terminal-`和`bash(1633)`这对父子进程也是通过伪终端通信的

上述的`gnome-terminal-`显示不全，全称为`gnome-terminal-server`

```sh
$ ps -e -o cmd | grep gnome-terminal
/usr/lib/gnome-terminal/gnome-terminal-server
grep --color=auto gnome-terminal
```

下面是我手绘的一张图，包含了终端模拟器

![terimal emulator](/image/terminal.png)

M表示伪终端主设备，S表示伪终端从设备

左侧的S是bash和ssh的控制终端，gnome-terminal没有控制终端，它是一个gui应用\
右侧的S是bash的控制终端，sshd没有控制终端，它是一个daemon
