---
title: "stdio buffer"
date: 2011-12-23T13:05:31Z
draft: true
---

# stdio buffer

## 缓冲模式

* _IONBF unbuffered
* _IOLBF line buffered
* _IOFBF fully buffered

## 库函数

```c
#include <stdio.h>
int setvbuf(FILE *stream, char *buf, int mode, size_t size);
```

```c
#include <stdio.h>
void setbuf(FILE *stream, char *buf);
void setbuffer(FILE *stream, char *buf, size_t size);
void setlinebuf(FILE *stream);
```

后面这三个函数是`setvbuf()`的别名

* `setbuf(stream, buf)` 等价于:

  ```c
  setvbuf(stream, buf, buf ? _IOFBF : _IONBF, BUFSIZ);
  ```

* `setbuffer(stream, buf, size)` 等价于:

  ```c
  setvbuf(stream, buf, buf ? _IOFBF : _IONBF, size);
  ```

* `setlinebuf(stream)` 等价于:

  ```c
  setvbuf(stream, NULL, _IOLBF, 0);
  ```

## 示例

* 全缓冲

  ```c
  setvbuf(stream, NULL, _IOFBF, 0)
  ```

* 行缓冲

  ```c
  setvbuf(stream, NULL, _IOLBF, 0)
  setlinebuf(stream)
  ```

* 无缓冲

  ```c
  setvbuf(stream, NULL, _IONBF, 0)
  setbuf(stream, NULL)
  setbuffer(stream, NULL, 0)
  ```

## flush

```c
#include <stdio.h>
int fflush(FILE *stream);
```
