---
title: "创建临时文件"
date: 2011-12-23T13:05:31Z
draft: true
---

# 创建临时文件

## 推荐的函数

```c
#include <stdio.h>
FILE *tmpfile(void);
```

这个函数创建的临时文件在文件被关闭或进程退出时自动删除

## 过时的函数

```c
#include <stdio.h>
char *tmpnam(char *s);
char *tempnam(const char *dir, const char *pfx);
```
