---
title: "ss"
date: 2011-12-23T13:05:31Z
draft: true
---

# ss

ss is used to dump socket statistics.

## SYNOPSIS

```sh
ss [OPTIONS] [STATE-FILTER] [ADDRESS-FILTER]
```

## OPTIONS

```sh
-n, --numeric
    Do not try to resolve service names.
-r, --resolve
    Try to resolve numeric address/ports.
-a, --all
    Display both listening and non-listening sockets.
-l, --listening
    Display only listening sockets.
-p, --processes
    Show process using socket.
-K, --kill
    Attempts to forcibly close sockets.
-4, --ipv4
    Display only IP version 4 sockets (alias for -f inet).
-6, --ipv6
    Display only IP version 6 sockets (alias for -f inet6).
-0, --packet
    Display PACKET sockets (alias for -f link).
-t, --tcp
    Display TCP sockets.
-u, --udp
    Display UDP sockets.
-d, --dccp
    Display DCCP sockets.
-w, --raw
    Display RAW sockets.
-x, --unix
    Display Unix domain sockets (alias for -f unix).
-S, --sctp
    Display SCTP sockets.
-f FAMILY, --family=FAMILY
    Display sockets of type FAMILY. Currently the following families are supported: unix, inet, inet6, link, netlink.
```

## STATE-FILTER

ss allows to filter socket states, using keywords `state` and `exclude`, followed by some `state identifier`.

`exclude` 可以简写成 `excl`

状态过滤器格式:

`state|exclude stateid1 state|exclude stateid2 ...`

`state identifier` 有:

* established
* syn-sent
* syn-recv
* fin-wait-1
* fin-wait-2
* time-wait
* closed
* close-wait
* last-ack
* listening
* closing
* all
* connected
* synchronized
* bucket
* big

默认情况下不加任何选项相当于

`state all exclude listening exclude syn-recv exclude time-wait exclude closed`

`-a`选项相当于 `state all`

debian 9 ss(8)手册有个错误，`listen`应为`listening`

see [http://man7.org/](http://man7.org/linux/man-pages/man8/ss.8.html)

## ADDRESS-FILTER

地址过滤器形成一个布尔表达式，格式如下:

`predicate1 OP predicate2 ...`

`predicate` 有

* dst ADDRESS_PATTERN - matches remote address and port
* src ADDRESS_PATTERN - matches local address and port
* dport RELOP PORT - compares remote port to a number
* sport RELOP PORT - compares local port to a number
* autobound - checks that socket is bound to an ephemeral port

逻辑操作符(`OP`) 有

* and
* or
* not

关系操作符(`RELOP`) 有

* le
* ge
* eq
* ne
* gt
* lt

`ADDRESS_PATTERN`格式

1. IPv4地址格式
    ip:port
2. IPv6地址格式
    [ip]:port
3. unix地址格式
    path

上述的ip或port可省略其一

## EXAMPLES

### STATE-FILTER 示例

```sh
ss -t state listening state time-wait
ss -x state listening
ss -t excl listening
```

### ADDRESS-FILTER 示例

```sh
$ ss -a src :443
Netid State      Recv-Q Send-Q                  Local Address:Port                                   Peer Address:Port
tcp   LISTEN     0      128                                :::https                                            :::*
tcp   ESTAB      0      0               ::ffff:67.216.219.154:https                          ::ffff:1.180.238.165:14694
```

443是https服务

```sh
$ ss -a src [::ffff:67.216.219.154]
Netid State      Recv-Q Send-Q                  Local Address:Port                                   Peer Address:Port
tcp   ESTAB      0      0               ::ffff:67.216.219.154:https                          ::ffff:1.180.238.165:24022
tcp   LAST-ACK   0      1               ::ffff:67.216.219.154:9999                           ::ffff:1.180.238.165:43113
tcp   LAST-ACK   0      1               ::ffff:67.216.219.154:9999                           ::ffff:1.180.238.165:28346
tcp   ESTAB      0      0               ::ffff:67.216.219.154:9999                           ::ffff:1.180.238.165:11740
tcp   ESTAB      0      0               ::ffff:67.216.219.154:https                          ::ffff:1.180.238.165:14694
tcp   ESTAB      0      0               ::ffff:67.216.219.154:http                           ::ffff:1.180.238.165:21362
tcp   ESTAB      0      0               ::ffff:67.216.219.154:9999                           ::ffff:1.180.238.165:40387
tcp   LAST-ACK   0      1               ::ffff:67.216.219.154:9999                           ::ffff:1.180.238.165:34051
tcp   LAST-ACK   0      1               ::ffff:67.216.219.154:9999                           ::ffff:1.180.238.165:18873
```

```sh
ss -a sport eq http or sport eq https
```

上述命令列出所有http和https socket

```sh
ss -ax src '/run/systemd/journal/*'
```

上述的路径需要加引号，防止被shell扩展

## 参考资料

```sh
$ sudo apt install iproute2-doc
$ dpkg-query -L iproute2-doc | grep ss.html
/usr/share/doc/iproute2-doc/ss.html
```
