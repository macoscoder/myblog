---
title: "文件描述符"
date: 2011-12-23T13:05:31Z
draft: true
---

# 文件描述符

## 内核数据结构

内核维护三个数据结构:

* the per-process file descriptor table;
* the system-wide table of open file descriptions;
* the system-wide file system i-node table.

如图所示:

![文件描述符结构](/image/fd.png)

说明:

* 一个进程对应一张文件描述符表
* 一张所有进程共享的打开文件表
* 一张所有进程共享的inode表

### file descriptor table

文件描述符表项包含:

* 一个fd flag, 目前只支持一个: `close-on-exec` flag
* 一个打开文件表项的引用

### open file table

打开文件表项包含:

* 一个引用计数，记录引用该表项的文件描述符个数
* 当前的文件偏移
* 打开文件时指定的status flags
* 文件的访问模式(ro,wo,rw)
* inode引用

### inode table

inode table是内核维护的一张全局表，这里的inode是从文件系统的inode table中拷贝来的
除此之外还包含:

* 一个引用计数，记录引用该inode的打开文件表项数
* major and minor device id, 记录该inode是从哪个设备拷贝来的
* 该inode持有的记录锁

## 用例分析

```c
int fd1 = open("/tmp/myfile", O_RDWR|O_CLOEXEC);
```

通过open打开一个文件，执行:

* 把文件系统中对应的inode拷贝到内核的inode table中(第一次打开这个文件时)
* 在打开文件表中添加一个表项，指向inode table中对应的inode
* 在文件描述符表中添加一个表项，指向打开文件表中对应的表项

此时inode table中对应inode的引用计数为1
fd1指向的打开文件表项的引用计数为1

```c
int fd2 = open("/tmp/myfile", O_RDWR|O_CLOEXEC);
```

通过open再次打开这个文件，执行:

* 在打开文件表中添加一个表项，指向inode table中对应的inode
* 在文件描述符表中添加一个表项，指向打开文件表中对应的表项

此时inode table中对应inode的引用计数为2
fd2指向的打开文件表项的引用计数为1

```c
int fd3 = dup(fd1);
```

通过dup复制文件描述符，执行

* 在文件描述符表中添加一个表项，指向打开文件表中对应的表项(引用计数+1)

此时inode table中对应inode的引用计数为2
fd3指向的打开文件表项的引用计数为2

```c
close(fd1)
```

通过close关闭文件描述符执行:

* 删除文件描述符表中fd1指向的文件描述符表项
* fd1指向的打开文件表项的引用计数减1

```c
close(fd3)
```

* 删除文件描述符表中fd3指向的文件描述符表项
* fd3指向的打开文件表项的引用计数减1，此时引用计数已为0
* 删除fd3指向的文件表项，同时对应的inode的引用计数减1

```c
close(fd2)
```

执行类似操作后，对应inode的引用计数已为0，则从inode table中删除该inode
此时内核还会检查文件系统中的inode表中对应的inode的link count，如果也为0
则会从文件系统中删除对应的文件

例如:

```c
FILE *tmpfile(void);
```

tmpfile会打开一个临时文件，并立即调用`unlink`将文件系统中的inode的link count减1
但是此时因为内核中的inode table中对应的inode的引用计数为1，内核不会立即删除对应的文件
而是等到引用该文件的`FILE *`关闭后才会删除文件系统中的文件
