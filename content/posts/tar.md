---
title: "tar 命令"
date: 2011-12-23T13:05:31Z
draft: true
---

# tar 命令

## 打包

```sh
tar -czf /path/to/ARCHIVE FILE
tar -czf /path/to/ARCHIVE -C /path/from/ FILE
```

## 解包

```sh
tar -xf /path/from/ARCHIVE
tar -xf /path/from/ARCHIVE -C /path/to/
```

## list

```sh
tar -tf /path/from/ARCHIVE
```

## 选项说明

```sh
-c, --create
        Create a new archive.
-x, --extract, --get
        Extract files from an archive.
-f, --file=ARCHIVE
        Use archive file or device ARCHIVE.
-t, --list
        List the contents of an archive.
-z, --gzip, --gunzip, --ungzip
        Filter the archive through gzip(1).
-a, --auto-compress
        Use archive suffix to determine the compression program.
-C, --directory=DIR
        Change to DIR before performing any operations. This option is order-sensitive, i.e. it affects all options that follow.
```
