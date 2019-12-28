---
title: "unix time"
date: 2011-12-23T13:05:31Z
draft: true
---

# unix time

## Calendar Time

![calendar time](/image/time.png)

### 1. time

```c
 #include <time.h>
time_t time(time_t *tloc);
```

这是一个系统调用，用来获取从1970-01-01 00:00:00 +0000 (UTC)至今的秒数，称为日历时间

一般`tloc`传`NULL`

### 2. gettimeofday

```c
 #include <sys/time.h>
int gettimeofday(struct timeval *tv, struct timezone *tz);
```

这也是一个系统调用，这个系统调用获取的时间精度更高。

`timeval` 结构如下：

```c
struct timeval {
    time_t      tv_sec;     /* seconds */
    suseconds_t tv_usec;    /* microseconds */
};
```

`tv_sec` 的值跟`time`系统调用的返回值一样

`tv_usec` 表示微秒

第二个参数已经废弃，传`NULL`就行

### 3. clock_gettime

```c
#include <time.h>
int clock_gettime(clockid_t clk_id, struct timespec *tp);

struct timespec {
    time_t   tv_sec;        /* seconds */
    long     tv_nsec;       /* nanoseconds */
};
```

`clock_gettime`可以获取纳秒级时间

```c
long long get_nano_time() {
    struct timespec ts;
    clock_gettime(CLOCK_REALTIME, &ts);
    return ts.tv_sec * 1000000000 + ts.tv_nsec;
}
```

日历时间（用秒数表示的时间）很有用，但是有时候我们需要获取具体的日期时间，即所谓的Broken-Down Time

## Broken-Down Time

```c
#include <time.h>
struct tm *gmtime(const time_t *timep);
struct tm *localtime(const time_t *timep);
```

这两个函数将`time_t`表示的秒数转成日期时间(datetime)

`gmtime`返回的是UTC(GMT)日期时间

`localtime`返回的系统本地日期时间

`struct tm`结构如下：

```c
struct tm {
    int tm_sec;   /* Seconds (0-60)                     对应%S格式 */
    int tm_min;   /* Minutes (0-59)                     对应%M格式 */
    int tm_hour;  /* Hours (0-23)                       对应%H格式 */
    int tm_mday;  /* Day of the month (1-31)            对应%d格式 */
    int tm_mon;   /* Month (0-11)                       对应%m格式-1 */
    int tm_year;  /* Year - 1900                        对应%Y格式-1900 */
    int tm_wday;  /* Day of the week (0-6, Sunday = 0)  对应%w格式 */
    int tm_yday;  /* Day in the year (0-365, 1 Jan = 0) 对应%j格式-1 */
    int tm_isdst; /* Daylight saving time */
};
```

注释中指出了对应`strftime`函数的格式说明符

下面代码可以获取本地日期

```c
time_t t = time(NULL);
struct tm *tm = localtime(&t);
printf("%d-%02d-%02d %02d:%02d:%02d\n",
        1900 + tm->tm_year, tm->tm_mon + 1, tm->tm_mday, tm->tm_hour, tm->tm_min, tm->tm_sec);
// output:
// 2018-12-09 11:16:00
```

## Broken-Down Time与字符串形式的互转

### 1. struct tm转成字符串

```c
#include <time.h>
size_t strftime(char *outstr, size_t max, const char *format, const struct tm *tm)
```

下面代码可以获取本地日期

```c
time_t t = time(NULL);
struct tm *tm = localtime(&t);
char str[32];
strftime(str, sizeof(str), "%F %T", tm);
// strftime(str, sizeof(str), "%Y-%m-%d %H:%M:%S", tm);
printf("%s\n", str);
```

### 2. 字符串转struct tm

```c
#include <time.h>
char *strptime(const char *s, const char *format, struct tm *tm);
```

下面的代码将字符串表示的日期时间转换为struct tm

```c
const char *dtstr = "2018-12-09 11:16:00";
const char *format = "%Y-%m-%d %H:%M:%S";
struct tm tm;
memset(&tm, 0, sizeof(struct tm));
strptime(dtstr, format, &tm);
```

## Broken-Down Time 转日历时间(time_t)

```c
#include <time.h>
time_t mktime(struct tm *tm);
```

注意：

    1. mktime使用系统的时区信息来解释tm参数
    2. mktime根据tm->tm_isdst的值，决定是否将tm视为夏令时
        1. 当tm->tm_isdst == 0，将tm视为标准时间
        2. 当tm->tm_isdst > 0，将tm视为夏令时
        3. 当tm->tm_isdst < 0，根据系统的时区信息和tm共同确定是否将tm视为夏令时，并修改tm->tm_isdst的值
    3. 如果tm_sec、tm_min、tm_hour、tm_mday、tm_mon、tm_year等字段不在各自的范围内，mktime会调整它们到范围内，
       这个特性很有用，可以使用这个特性做日期计算
    4. 最后mktime会修改tm_wday和tm_yday，使其与其他字段匹配

下面代码将struct tm转换成日历时间(time_t)，（特意选了一个夏令时，即：实行夏时制时的时间）

```c
const char *dtstr = "1991-07-01 05:00:00";
const char *format = "%Y-%m-%d %H:%M:%S";
struct tm tm;
memset(&tm, 0, sizeof(struct tm));
strptime(dtstr, format, &tm);
tm.tm_isdst = -1; // Not set by strptime(); tells mktime() to determine if DST is in effect
time_t t = mktime(&tm);
printf("%ld\n", t); // 678312000
printf("%d\n", tm.tm_isdst); // 1 表示这个时间为夏令时
```

这里如果将 tm.tm_isdst = -1; 这行注释掉，那么转换输出的日历时间是`678315600`这是错误的

我们通过`date`命令反转这两个日历时间：

```c
date -d @678312000 '+%F %T'
date -d @678315600 '+%F %T'
// output:
// 1991-07-01 05:00:00
// 1991-07-01 06:00:00
```

从转换结果看出来差了1小时，实行夏时制的时间比不实行夏时制的时间早了1小时

## 一个示例

```c
enum option {
    year    = 1,
    month   = 2,
    day     = 3,
    hour    = 4,
    minute  = 5,
    second  = 6,
};

time_t time_add(time_t t, int x, enum option opt) {
    struct tm *tm = localtime(&t);
    switch (opt) {
    case year:
        tm->tm_year += x;
        break;
    case month:
        tm->tm_mon += x;
        break;
    case day:
        tm->tm_mday += x;
        break;
    case hour:
        tm->tm_hour += x;
        break;
    case minute:
        tm->tm_min += x;
        break;
    case second:
        tm->tm_sec += x;
        break;
    }
    return mktime(tm);
}

const char *datetime_string(time_t t) {
    static char str[32];
    struct tm *tm = localtime(&t);
    strftime(str, sizeof(str), "%F %T", tm);
    return str;
}

int main() {
    time_t t = time(NULL);
    const char *s = datetime_string(t);
    printf("%s\n", s);
    t = time_add(t, 2, day);
    t = time_add(t, 1, hour);
    t = time_add(t, 10, minute);
    t = time_add(t, -1, year);
    s = datetime_string(t);
    printf("%s\n", s);
}
// output:
// 2018-12-09 16:18:51
// 2017-12-11 17:28:51
```

## 其它无用的函数

```c
#include <time.h>
char *ctime(const time_t *timep);   // calendar time
char *asctime(const struct tm *tm); // ascii time
```

```c
time_t t = time(NULL);
const char *s1 = ctime(&t);
const char *s2 = asctime(localtime(&t));
printf("%s%s", s1, s2);
// output:
// Sun Dec  9 15:43:37 2018
// Sun Dec  9 15:43:37 2018
```

ctime(&t)与asctime(localtime(&t))等价，它们将日历时间转换为固定格式的时间字符串，因为格式固定，一般不满足我们的需求，
所以基本没什么用
