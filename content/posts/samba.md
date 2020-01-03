---
title: "samba服务器搭建"
date: 2011-12-23T13:05:31Z
draft: true
---

# samba服务器搭建

## 安装

```
$ sudo apt install samba
```

## 添加用户

```
$ sudo smbpasswd -a username
```

其中`username`为系统用户名

## 配置

```
$ sudo nano /etc/samba/smb.conf
```

找到 `[homes]` 下面的 `read only = no` 改为 `read only = yes`

## 重启

```
$ sudo systemctl restart smbd.service
```
