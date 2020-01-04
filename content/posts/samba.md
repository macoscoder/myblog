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

找到 `[homes]` 修改：

```
read only = no
create mask = 0644      # rw-r--r--
directory mask = 0755   # rwxr-xr-x
```

## 重启

```
$ sudo systemctl restart smbd.service
```
