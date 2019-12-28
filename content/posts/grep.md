---
title: "grep 命令"
date: 2011-12-23T13:05:31Z
draft: true
---

# grep 命令

## SYNOPSIS

```sh
grep [OPTIONS] PATTERN [FILE...]
grep [OPTIONS] -R PATTERN [FILE_OR_DIR...]
grep [OPTIONS] -e PATTERN ... [FILE...]
```

## OPTIONS

```sh
-E, --extended-regexp
        Interpret PATTERN as an extended regular expression.
-e PATTERN, --regexp=PATTERN
        Use PATTERN as the pattern.
-i, --ignore-case
        Ignore case distinctions, so that characters that differ only in case match each other.
-v, --invert-match
        Invert the sense of matching, to select non-matching lines.
-w, --word-regexp
        Select only those lines containing matches that form whole words.
-n, --line-number
        Prefix each line of output with the 1-based line number within its input file.
-l, --files-with-matches
        Suppress normal output.
--include=GLOB
        Search only files whose base name matches GLOB.
-R, --dereference-recursive
        Read all files under each directory, recursively. Follow all symbolic links.
```

## EXAMPLES

```sh
$ grep -nR 'SIG_DFL' /usr/include
/usr/include/asm-generic/signal-defs.h:24:#define SIG_DFL ((__sighandler_t)0) /* default signal handling */
/usr/include/x86_64-linux-gnu/bits/sigaction.h:64:#define SA_RESETHAND 0x80000000 /* Reset to SIG_DFL on entry to handler.  */
/usr/include/x86_64-linux-gnu/bits/signum-generic.h:29:#define SIG_DFL ((__sighandler_t)0) /* Default action.  */
```

```sh
$ grep -nRw 'printf' /usr/include/stdio.h
318:extern int printf (const char *__restrict __format, ...);
```

```sh
$ grep -nRi 'func readall' $GOROOT/src
/usr/lib/go-1.10/src/io/ioutil/ioutil.go:18:func readAll(r io.Reader, capacity int64) (b []byte, err error) {
/usr/lib/go-1.10/src/io/ioutil/ioutil.go:44:func ReadAll(r io.Reader) ([]byte, error) {
```

```sh
$ grep -nR --include='file*' 'func Open(' $GOROOT/src
/usr/lib/go-1.10/src/debug/plan9obj/file.go:98:func Open(name string) (*File, error) {
/usr/lib/go-1.10/src/debug/pe/file.go:32:func Open(name string) (*File, error) {
/usr/lib/go-1.10/src/debug/elf/file.go:196:func Open(name string) (*File, error) {
/usr/lib/go-1.10/src/debug/macho/file.go:198:func Open(name string) (*File, error) {
/usr/lib/go-1.10/src/os/file.go:249:func Open(name string) (*File, error) {
```
