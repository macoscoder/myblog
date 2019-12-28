---
title: "git快速上手"
date: 2011-12-23T13:05:31Z
draft: true
---

# git快速上手

## 配置命令

```sh
git config [--local] --edit  // edit .git/config
git config --global --edit // edit $HOME/.gitconfig
git config --system --edit // edit /etc/gitconfig
git config --list // List all variables set in config file, along with their values.
```

## 配置用户名，邮箱

```sh
git config --global user.name 'fly'
git config --global usr.email '723937936@qq.com'
```

## 配置命令别名

```sh
git config --global alias.st 'status --short'
git config --global alias.ci 'commit'
git config --global alias.co 'checkout'
git config --global alias.br 'branch'
```

## 仓库初始化

```sh
cd /path/to/your/project
git init    // Create an empty Git repository
```

此时会在project目下创建一个.git目录，这个目录就是git的仓库

## 添加文件

```sh
git add file // Add file contents to the index
git add dir
```

## 撤销添加

```sh
git reset HEAD file
git reset HEAD dir
```

## 恢复工作树中的文件

```sh
git checkout -- file
git checkout -- dir
```

## 恢复工作树和暂存区中的文件

```sh
git checkout HEAD file
git checkout HEAD dir
```

## 提交

```sh
git commit -m '日志信息'
```

## 撤销提交

```sh
git reset --solft HEAD^ // 相当于没有做过上次提交
git reset --hard HEAD^  // 撤销到上次提交状态，清除自上次提交之后的所有修改
```

## rebase

```sh
git fetch
git rebase
```

## 合并

```sh
git pull
```

## 检出分支

```sh
git checkout branch
```

## 显示状态

```sh
git status --short --branch
```

## 查看日志

```sh
git log
```

## diff

```sh
git diff            // 工作树和暂存区比较
git diff --cached   // 暂存区和HEAD比较
git diff HEAD       // 工作树和HEAD比较
```

## 显示当前分支

```sh
git branch
```

## 显示.git所在目录的绝对路径

```sh
git rev-parse --show-toplevel
```

## 概念说明

`tree` 目录

`blob` 文件

`HEAD` 是一个引用，他指向另一个引用：refs/heads/master

`master` 指向master分支的最新提交
