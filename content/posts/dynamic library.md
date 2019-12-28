---
title: "动态库"
date: 2011-12-23T13:05:31Z
draft: true
---

# 动态库

## 创建动态库

```sh
$ ls
a.c b.c main.c
$ gcc -c -fPIC a.c b.c
$ gcc -shared -o libfoo.so.1.0.0 a.o b.o
```

## 使用动态库

```sh
$ gcc -c -fPIC main.c
$ gcc -o main main.o libfoo.so.1.0.0
$ ls main
main
```

## 运行

```sh
$ LD_LIBRARY_PATH=. ./main
foo
```

## soname

创建动态库时可以指定别名，称作`soname`

```sh
$ gcc -shared -Wl,-soname,libfoo.so.1 -o libfoo.so.1.0.0 a.o b.o
$ ls libfoo.so
libfoo.so
```

上述方法创建的动态库的`soname`为`libfoo.so.1`

查看动态库的`soname`

```sh
$ readelf -d libfoo.so.1.0.0 | grep SONAME
0x000000000000000e (SONAME)             Library soname: [libfoo.so.1]
```

创建可执行程序

```sh
$ gcc -o main main.o libfoo.so.1.0.0
$ LD_LIBRARY_PATH=. ./main
./main: error while loading shared libraries: libfoo.so.1: cannot open shared object file: No such file or directory
```

报错了，找不到`libfoo.so.1`

这也就是说，如果创建动态库时指定了`soname`，创建可执行程序时，可执行程序里存的是`soname`\
如果创建动态库时没有指定`soname`，创建可执行程序时，可执行程序里存的是动态库的文件名，称为`realname`

`soname`提供了一种间接性，当动态库版本更新时，只要`soname`不变，那么已有的可执行程序就可以链接到新版本的动态库

创建符号链接

```sh
$ ln -s libfoo.so.1.0.0 libfoo.so.1
$ LD_LIBRARY_PATH=. ./main
$ ./main
foo
```

## linker name

通常会再创建一个符号链接指向`soname`，称为`linker name`，目的是在链接程序时不需要指定版本号

```sh
$ ln -s libfoo.so.1 libfoo.so
$ gcc -o main main.o -L. -lfoo
$ LD_LIBRARY_PATH=. ./main
foo
```

## 动态库命名约定

* realname - libname.so.major-id.minor-id
* soname - libname.so.major-id
* linker name - libname.so

其中`major-d`为一个数字，`minor-id`一般为用点分开的两个数字，例如:

* realname - `libname.so.1.0.0`
* soname - `libname.so.1`
* linker name - `libname.so`

当动态库minor-id变化时，一般是需要更新`libname.so.major-id`这个符号链接，使其指向最新的版本

## 常用命令

* ldd - List dynamic dependencies
* objdump - 反汇编
* readelf - Displays information about ELF files.
* nm - List symbols from object files

## 标准库目录

* /lib 系统启动必须的标准库安装目录
* /usr/lib 大多数标准款安装目录

除此之外还有配置文件`/etc/ld.so.conf`指定的目录，比如:

* /usr/local/lib 非标准库安装目录
* /lib/x86_64-linux-gnu 64位库
* /usr/lib/x86_64-linux-gnu 64位库

## dynamic linker/loader

* ld.so 早期`a.out`格式的动态链接器
* ld-linux.so.1 `libc5`的动态链接器
* ld-linux.so.2 `glibc2`的动态链接器

## ldconfig 命令

动态链接器搜索动态库的顺序是

* LD_LIBRARY_PATH 环境指定的目录
* DT_RUNPATH 属性指定的目录，通过-rpath链接选项设置
* /etc/ld.so.cache 缓存文件指定的库路径
* /lib 和 /usr/lib 目录

`ldconfig`命令的其中一个功能是维护`/etc/ld.so.cache`文件，另一个功能是更新`soname`符号链接

`ldconfig -p`选项列出`/etc/ld.so.cache`文件中缓存的所有动态库路径

```sh
$ /sbin/ldconfig -p | wc -l
1043
```

`ldconfig -n`选项创建指定目录的`soname`

```sh
$ ls libdemo*
libdemo.so.1.0.0
$ /sbin/ldconfig -n .
$ ls libdemo*
libdemo.so.1  libdemo.so.1.0.0
```

一般当有新库被安装到标准目录，或旧库被删除，或`/etc/ld.so.conf`文件更新，都需要运行`ldconfig`

## -rpath linker option

```sh
$ gcc -Wl,-rpath,'$ORIGIN' -o main main.o libdemo.so
$ ./main
foo
bar
```

查看`main`的`DT_RUNPATH`属性

```sh
$ readelf -d main | grep PATH
0x000000000000001d (RUNPATH)            Library runpath: [$ORIGIN]
```

`$ORIGIN` 指定运行时搜索当前目录，也就是可执行程序`main`所在的目录

这样做的好处是开箱即用，不需要使用`LD_LIBRARY_PATH`环境变量，也不需要安装动态库到标准目录

## –Bsymbolic linker option

默认情况下，主程序通过定义与动态库同名的的全局变量或全局函数，可以实现注入功能\
也就是动态库中本来调用自己定义的全局函数，现在却调用了主程序的版本

要避免这种情况，可以用链接器选项`–Bsymbolic`来创建动态库，让动态库调用自己定义的版本

```sh
gcc -shared -Wl,-Bsymbolic -Wl,-soname,libdemo.so.1 -o libdemo.so.1.0.0 foo.o bar.o
```

## 静态链接

在链接程序时通过-L和-l指定库，如果同时存在动态库和静态库，那么默认使用动态库

下列方法可以显式链接静态版本

* -static 选项
* 命令行显示指定静态库全路径

一个简单的`hello world`程序，链接静态库和动态库的区别

```c
// hello.c
#include <stdio.h>
int main() {
    printf("hello world\n");
}
```

```sh
$ cc -o hello.shared hello.c
$ cc -static -o hello.static hello.c
$ ll -h hello.shared hello.static
-rwxr-xr-x 1 fly fly 8.5K Feb 26 22:31 hello.shared
-rwxr-xr-x 1 fly fly 792K Feb 26 22:31 hello.static
$ ldd hello.shared
    linux-vdso.so.1 (0x00007ffe973c6000)
    libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007fa4c4a78000)
    /lib64/ld-linux-x86-64.so.2 (0x00007fa4c5019000)
$ ldd hello.static
    not a dynamic executable
```
