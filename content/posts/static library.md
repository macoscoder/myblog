---
title: "静态库"
date: 2011-12-23T13:05:31Z
draft: true
---

# 静态库

## 编译生成对象文件

```sh
$ ls
a.c  b.c  c.c  d.c
$ cc -c *.c
$ ls *.o
a.o  b.o  c.o  d.o
```

## 创建静态库

```sh
$ ar -r libdemo.a *.o
$ ls libdemo.a
libdemo.a
```

## 查看静态库里的对象模块

```sh
$ ar -t libdemo.a
a.o
b.o
c.o
d.o
```

## 使用静态库

### 方法1

```sh
$ rm *.o
$ echo 'int main() {}' > main.c
$ ls
a.c  b.c  c.c  d.c  libdemo.a  main.c
$ cc -c main.c
$ cc -o main main.o libdemo.a
$ ls main
main
```

### 方法2

```sh
$ rm main
$ cc -o main main.o -L. -ldemo
$ ls main
main
```

## ar命令

### SYNOPSIS

```sh
ar options archive object-file...
```

### OPTIONS

#### operation code

```sh
d Delete modules from the archive.
r Insert an object file into the archive, replacing any previous object file of the same name.
t Display a table listing the contents of archive.
x Extract members (named member) from the archive.
```

#### modifiers

```sh
v This modifier requests the verbose version of an operation.
```
