---
title: "文件介绍"
date: 2011-12-23T13:05:31Z
draft: true
---

# 文件介绍

## 文件系统结构

![file system](/image/fs.png)

## 文件

* 文件是由inode+data blocks组成
* inode存储在inode table中
* inode在inode table中的索引称为inode number
* 每个文件系统有一张inode table

inode存储文件状态(元)信息

* file type
* uid, gid
* access permisstion
* last access time, last modify time, last inode change time
* inode link(ref) count
* file size
* data block count
* data block pointers

注意：inode中并不包含文件名，文件名和inode number存储在directory entry中

如果用面向对象的思想来描述文件，那么把inode看成对象，inode number就是对象地址，
link count就是对象的引用计数，directory entry是对象的引用

## 创建文件

```c
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
int open(const char *pathname, int flags);
int open(const char *pathname, int flags, mode_t mode);
```

open函数创建inode，然后创建一个引用(directory entry)，其中包含inode number和引用名，此时该inode的引用计数为1
`pathname`为新的引用名

## 增加引用计数

```c
#include <unistd.h>
int link(const char *oldpath, const char *newpath);
```

link函数用来增加inode的引用计数，并创建一个新的引用(directory entry)，其中包含inode number和新的引用名
引用名`oldpath`指向已有的inode对象，`newpath`为新建的引用的名称

## 减少引用计数

```c
#include <unistd.h>
int unlink(const char *pathname);
```

unlink将引用名`pathname`指向的inode对象的引用计数减1，如果对象的引用计数减为0，则内核将删除inode和对应的data blocks

`rm(1)`命令调用`unlink`函数
