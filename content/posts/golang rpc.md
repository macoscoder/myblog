---
title: "封装net.Server"
date: 2011-12-23T13:05:31Z
draft: true
---

# 封装net.Server

## 定义demo/server包

```go
// Package server socket级别的server
//
// 使用方法：
// srv := New(HandlerFunc(handleConn))
// err := srv.ListenAndServe("tcp", ":8888")
// if err != nil {
//     // 处理错误
// }
//
// func handleConn(rw io.ReadWriter) {
//     // 执行io操作
// }
package server

import (
    "io"
    "log"
    "net"
)

// Handler 连接处理器
type Handler interface {
    ServeConn(rw io.ReadWriter)
}

// HandlerFunc 连接处理函数
type HandlerFunc func(rw io.ReadWriter)

// ServeConn 实现Handler接口
func (f HandlerFunc) ServeConn(rw io.ReadWriter) {
    f(rw)
}

// Server 服务器
type Server struct {
    Handler Handler
}

// ListenAndServe 监听并服务
func (s *Server) ListenAndServe(network, address string) error {
    ln, err := net.Listen(network, address)
    if err != nil {
        return err
    }
    defer ln.Close()
    for {
        conn, err := ln.Accept()
        if err != nil {
            return err
        }
        go s.serve(conn)
    }
}

func (s *Server) serve(conn net.Conn) {
    defer func() {
        if err := recover(); err != nil {
            log.Print("panic: ", err)
        }
        conn.Close()
    }()
    s.Handler.ServeConn(conn)
}

// New 创建服务器
func New(h Handler) *Server {
    return &Server{
        Handler: h,
    }
}
```

## 定义demo/rpcserver包

```go
// Package rpcserver 简单封装rpc server
//
// 使用方法：
// srv := New()
// srv.Register(&Service1{})
// srv.Register(&Service2{})
// err := srv.ListenAndServe("tcp", ":8888")
// if err != nil {
//     // 处理错误
// }
//
// type Service1 struct {}
// func (*Service1) S1F1(arg *int, reply *int) error {}
// func (*Service1) S1F2(arg *int, reply *int) error {}
//
// type Service2 struct {}
// func (*Service2) S2F1(arg *int, reply *int) error {}
// func (*Service2) S2F2(arg *int, reply *int) error {}
package rpcserver

import (
    "io"
    "net/rpc"

    "demo/server"
)

type nopCloser struct {
    io.ReadWriter
}

func (nopCloser) Close() error {
    return nil
}

// RPCServer wrap *rpc.Server
type RPCServer struct {
    rpcServer *rpc.Server
}

// ServeConn 实现server.Handler接口
func (s *RPCServer) ServeConn(rw io.ReadWriter) {
    s.rpcServer.ServeConn(nopCloser{rw})
}

// Register 注册服务
func (s *RPCServer) Register(rcvr interface{}) error {
    return s.rpcServer.Register(rcvr)
}

// ListenAndServe 监听并服务
func (s *RPCServer) ListenAndServe(network, address string) error {
    srv := server.New(s)
    return srv.ListenAndServe(network, address)
}

// New 创建rpc server
func New() *RPCServer {
    return &RPCServer{
        rpcServer: rpc.NewServer(),
    }
}
```

## 示例

### demo/cmd/rpcsrv

```go
package main

import (
    "fmt"
    "log"

    "demo/rpcserver"
)

// RPC rpc服务
type RPC struct {
}

// Ping ping
func (*RPC) Ping(arg string, reply *string) error {
    fmt.Println(arg)
    *reply = "pong"
    return nil
}

func main() {
    server := rpcserver.New()
    server.Register(&RPC{})
    err := server.ListenAndServe("tcp", ":8888")
    if err != nil {
        log.Fatal(err)
    }
}
```

### demo/cmd/rpccli

```go
package main

import (
    "fmt"
    "log"
    "net/rpc"
)

func main() {
    cli, err := rpc.Dial("tcp", ":8888")
    if err != nil {
        log.Fatal(err)
    }
    var reply string
    err = cli.Call("RPC.Ping", "ping", &reply)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println(reply)
}
```
