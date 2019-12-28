---
title: "find 命令"
date: 2011-12-23T13:05:31Z
draft: true
---

# find 命令

## SYNOPSIS

```sh
find DIR... EXPRESSION
```

## EXPRESSION

表达式返回一个布尔值(true或false)

表达式由以下操作组成:

* TESTS
* ACTIONS
* OPERATORS

### TESTS

TESTS: 测试文件的属性，返回true或false

常用的TESTS有:

```sh
-name pattern
        Base of file name (the path with the leading directories removed) matches shell pattern pattern.
-iname pattern
        Like -name, but the match is case insensitive.
-path pattern
        File name matches shell pattern pattern.
-ipath pattern
        Like -path. but the match is case insensitive.
-regex pattern
        File name matches regular expression pattern.
-iregex pattern
        Like -regex, but the match is case insensitive.
-type c
        File is of type c:
        b block (buffered) special
        c character (unbuffered) special
        d directory
        p named pipe (FIFO)
        f regular file
        l symbolic  link
        s socket
        D door (Solaris)
-readable
        Matches files which are readable by the current user.
-writable
        Matches files which are writable by the current user.
-executable
        Matches files which are executable and directories which are searchable.
```

### ACTIONS

ACTIONS: 执行指定动作，返回true或false

常用的ACTIONS有:

```sh
-ls
        True; list current file in ls -dils format on standard output.
-print
        True; print the full file name on the standard output, followed by a newline.
-print0
        True; print the full file name on the standard output, followed by a null character.
-printf format
        True; print format on the standard output, interpreting '\' escapes and '%' directives.
-fprint file
        True; print the full file name into file file.
-fprint0 file
        True; like -print0 but write to file like -fprint.
-fprintf file format
        True; like -printf but write to file like -fprint.
-prune
        True; if the file is a directory, do not descend into it.
```

### OPERATORS

OPERATORS: 连接不同操作，返回true或false

常用的OPERATORS有:

```sh
expr1 expr2
        Two expressions in a row are taken to be joined with an implied -a; expr2 is not evaluated if expr1 is false.
expr1 -a expr2
        Same as expr1 expr2.
expr1 -o expr2
        Or; expr2 is not evaluated if expr1 is true.
! expr
        True if expr is false.
( expr )
        Force  precedence.
```

## EXAMPLES

```sh
$ find /usr/include/ -name 'signal.h'
/usr/include/asm-generic/signal.h
/usr/include/linux/signal.h
/usr/include/x86_64-linux-gnu/sys/signal.h
/usr/include/x86_64-linux-gnu/asm/signal.h
/usr/include/signal.h
```

```sh
$ find $GOROOT/src/ -path '*syslog*.go'
/usr/lib/go-1.10/src/log/syslog/syslog_test.go
/usr/lib/go-1.10/src/log/syslog/doc.go
/usr/lib/go-1.10/src/log/syslog/syslog.go
/usr/lib/go-1.10/src/log/syslog/syslog_unix.go
/usr/lib/go-1.10/src/log/syslog/example_test.go
```
