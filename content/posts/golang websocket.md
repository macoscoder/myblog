---
title: "github.com/gorilla/websocket包的使用"
date: 2011-12-23T13:05:31Z
draft: true
---

# github.com/gorilla/websocket包的使用

## Server

### 定义请求消息格式(以下代码在ws包中定义)

```go
// Request websocket 请求消息
type Request struct {
    Action string          `json:"action"`
    Data   json.RawMessage `json:"data"`
}
```

### 定义Server结构，实现http.Handler接口，维护活跃的连接

```go
// Handler 消息处理器接口
type Handler interface {
    ServeWS(r *Request, hr *http.Request)
}

// HandlerFunc 处理器函数
type HandlerFunc func(r *Request, hr *http.Request)

// ServeWS 实现Handler接口
func (f HandlerFunc) ServeWS(r *Request, hr *http.Request) {
    f(r, hr)
}

// Server websocket server
type Server struct {
    activeConns map[string]*websocket.Conn
    mutex       sync.Mutex
    hander      Handler
}

// NewServer 用这个方法来创建server
func NewServer(h Handler) *Server {
    return &Server{activeConns: make(map[string]*websocket.Conn), hander: h}
}

// ServeHTTP 实现http.Handler接口，接受连接
func (s *Server) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    up := websocket.Upgrader{
        CheckOrigin: func(r *http.Request) bool {
            return true // for debug
        },
    }
    conn, err := up.Upgrade(w, r, nil)
    if err != nil {
        log.Print(err)
        return
    }
    // 这里要求客户端传一个设备id
    deviceID := r.FormValue("device_id")

    s.mutex.Lock()
    s.activeConns[deviceID] = conn
    s.mutex.Unlock()

    // 启动一个goroutine服务连接
    go s.serve(r, conn)
}
```

```go
// 这个函数的生命周期就是连接的生命周期
// 这个函数循环读取请求消息，如果发生错误，则关闭连接
func (s *Server) serve(hr *http.Request, conn *websocket.Conn) {
    defer func() {
        if err := recover(); err != nil {
            log.Print(err)
        }
        conn.Close()
        s.mutex.Lock()
        delete(s.activeConns, hr.FormValue("device_id"))
        s.mutex.Unlock()
    }()
    for {
        r, err := doRead(conn)
        if err != nil {
            log.Print(err)
            return
        }
        s.hander.ServeWS(r, hr)
    }
}

// 读取下一个请求消息
func doRead(conn *websocket.Conn) (*Request, error) {
    err := conn.SetReadDeadline(time.Now().Add(time.Second * 60))
    if err != nil {
        return nil, err
    }
    _, msg, err := conn.ReadMessage()
    if err != nil {
        return nil, err
    }
    r := &Request{}
    err = json.Unmarshal(msg, r)
    if err != nil {
        return nil, err
    }
    return r, nil
}
```

```go

```

### 使用方法(以下代码在main包中)

```go
var wsServer = ws.NewServer(ws.HandlerFunc(handleRequest))

func main() {
    http.Handle("/ws", wsServer)
    http.ListenAndServe(":8081", nil)
}

func handleRequest(r *ws.Request, hr *http.Request) {
    fmt.Println(r.Action)
}
```

### 多路器Mux，实现Handler接口(以下代码在ws包中定义)

```go
// Mux 多路器
type Mux struct {
    handlers map[string]Handler
    mutex    sync.Mutex
}

// NewMux 创建多路器
func NewMux() *Mux {
    return &Mux{handlers: make(map[string]Handler)}
}

// Handle 注册处理器
func (mux *Mux) Handle(pattern string, h Handler) {
    mux.mutex.Lock()
    mux.handlers[pattern] = h
    mux.mutex.Unlock()
}

// HandleFunc 注册处理器函数
func (mux *Mux) HandleFunc(pattern string, f HandlerFunc) {
    mux.Handle(pattern, f)
}

// ServeWS 实现Handler接口
func (mux *Mux) ServeWS(r *Request, hr *http.Request) {
    mux.mutex.Lock()
    h, ok := mux.handlers[r.Action]
    mux.mutex.Unlock()
    if !ok {
        log.Print("动作不存在: ", r.Action)
        return
    }
    h.ServeWS(r, hr)
}
```

### 多路器使用方法

```go
var (
    mux      = ws.NewMux()
    wsServer = ws.NewServer(mux)
)

func main() {
    mux.HandleFunc("ping", ping)
    mux.HandleFunc("echo", echo)
    http.Handle("/ws", wsServer)
    http.ListenAndServe(":8081", nil)
}

func ping(r *ws.Request, hr *http.Request) {
    fmt.Println(r.Action)
}

func echo(r *ws.Request, hr *http.Request) {
    fmt.Println(r.Action)
}
```

### 广播

```go
// BCMessage 广播消息
type BCMessage struct {
    Action string      `json:"action"`
    Data   interface{} `json:"data"`
}

// Broadcast 广播
func (s *Server) Broadcast(msg *BCMessage, deviceIDs ...string) {
    bytes, _ := json.Marshal(msg)
    s.mutex.Lock()
    defer s.mutex.Unlock()
    if len(deviceIDs) == 0 {
        for _, conn := range s.activeConns {
            go doWrite(conn, bytes)
        }
    } else {
        for _, id := range deviceIDs {
            conn, ok := s.activeConns[id]
            if !ok {
                log.Printf("id: %q, no conn", id)
                continue
            }
            go doWrite(conn, bytes)
        }
    }
}

func doWrite(conn *websocket.Conn, msg []byte) error {
    _ = conn.SetWriteDeadline(time.Now().Add(time.Second * 60))
    return conn.WriteMessage(websocket.TextMessage, msg)
}
```

### 广播的使用

```go
var (
    mux      = ws.NewMux()
    wsServer = ws.NewServer(mux)
)

func main() {
    mux.HandleFunc("ping", ping)
    http.Handle("/ws", wsServer)
    http.ListenAndServe(":8081", nil)
}

func ping(r *ws.Request, hr *http.Request) {
    fmt.Println(r.Action)
    msg := ws.BCMessage{
        Action: "pong",
        Data:   "",
    }
    deviceID := hr.FormValue("device_id")
    wsServer.Broadcast(&msg, deviceID)
}
```

### 对指定用户的推送

对指定用户的推送需要维护一张设备用户关联表，通过用户ID查询对应的设备ID，通过Broadcast接口实现推送

### 测试工具

[Smart Websocket Client](https://chrome.google.com/webstore/detail/smart-websocket-client/omalebghpgejjiaoknljcfmglgbpocdp)

用例:

```json
url:
ws://localhost:8081/ws?device_id=xxxxxxxx

request:
{
  "action":"ping"
}

response:
{
  "action": "pong",
  "data": ""
}
```
