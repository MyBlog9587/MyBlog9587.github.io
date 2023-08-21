---
title: go编程模式:函数式
date: 2023-06-16 15:52:56
tags:
- go
- private
categories:
- go
- 函数式编程
- Functional Options
---


## demo

如果有如下配置，需要创建一个 Server 对象

```go
type Server struct {
    Addr     string
    Port     int
    Protocol string
    Timeout  time.Duration
    MaxConns int
    TLS      *tls.Config
}
```

针对于上述这样的配置，需要有多种不同的创建不同配置 Server 的函数签名，如下所示：

```go
func NewDefaultServer(addr string, port int) (*Server, error) {
  return &Server{addr, port, "tcp", 30 * time.Second, 100, nil}, nil
}

func NewTLSServer(addr string, port int, tls *tls.Config) (*Server, error) {
  return &Server{addr, port, "tcp", 30 * time.Second, 100, tls}, nil
}

func NewServerWithTimeout(addr string, port int, timeout time.Duration) (*Server, error) {
  return &Server{addr, port, "tcp", timeout, 100, nil}, nil
}

func NewTLSServerWithMaxConnAndTimeout(addr string, port int, maxconns int, timeout time.Duration, tls *tls.Config) (*Server, error) {
  return &Server{addr, port, "tcp", 30 * time.Second, maxconns, tls}, nil
}
```

因为Go语言不支持重载函数，所以，需要用不同的函数名来应对不同的配置选项, 就会变成下面繁重的调用：

**整体代码如下：**
```go
package main

import (
	"crypto/tls"
	"fmt"
	"time"
)

// 配置结构体
type Server struct {
	Addr     string
	Port     int
	Protocol string
	Timeout  time.Duration
	MaxConns int
	TLS      *tls.Config
}

// 配置函数
func NewDefaultServer(addr string, port int) (*Server, error) {
	return &Server{addr, port, "tcp", 30 * time.Second, 100, nil}, nil
}

func NewTLSServer(addr string, port int, tls *tls.Config) (*Server, error) {
	return &Server{addr, port, "tcp", 30 * time.Second, 100, tls}, nil
}

func NewServerWithTimeout(addr string, port int, timeout time.Duration) (*Server, error) {
	return &Server{addr, port, "tcp", timeout, 100, nil}, nil
}

func NewTLSServerWithMaxConnAndTimeout(addr string, port int, maxconns int, timeout time.Duration, tls *tls.Config) (*Server, error) {
	return &Server{addr, port, "tcp", 30 * time.Second, maxconns, tls}, nil
}

func main() {
	addr := "127.0.0.1"
	port := 8123
	tls := tls.Config{
		Time: func() time.Time {
			return time.Now()
		},
	}
	timeout := 30 * time.Second

	// 针对不同情况需要单独定义
	s1, _ := NewDefaultServer(addr, port)
	s2, _ := NewTLSServer(addr, port, &tls)
	s3, _ := NewServerWithTimeout(addr, port, timeout)
	s4, _ := NewTLSServerWithMaxConnAndTimeout(addr, port, 10, timeout, &tls)

    // s1: &{Addr:127.0.0.1 Port:8123 Protocol:tcp Timeout:30s MaxConns:100 TLS:<nil>} s2: &{Addr:127.0.0.1 Port:8123 Protocol:tcp Timeout:30s MaxConns:100 TLS:0xc0000d0000}    s3: &{Addr:127.0.0.1 Port:8123 Protocol:tcp Timeout:30s MaxConns:100 TLS:<nil>}   s4: &{Addr:127.0.0.1 Port:8123 Protocol:tcp Timeout:30s MaxConns:10 TLS:0xc0000d0000}
	fmt.Printf("s1: %+v\ts2: %+v\ts3: %+v\ts4: %+v\n", s1, s2, s3, s4)
}
```


## 优化方案1: 配置对象

把那些非必输的选项都移到一个结构体里

```go
// 拆分配置 - 非必输的选项
type Config struct {
    Protocol string
    Timeout  time.Duration
    Maxconns int
    TLS      *tls.Config
}
```

于是 Server 对象变成了：

```go
// 主体
type Server struct {
    Addr string
    Port int
	// 可变配置
    Conf *Config
}
```
于是，只需要一个 NewServer() 的函数了，在使用前需要构造 Config 对象:

```go
func NewServer(addr string, port int, conf *Config) (*Server, error) {
    //...
}

// 使用默认配置项
srv1, _ := NewServer("localhost", 9000, nil) 

// 使用自定义配置项
conf := Config{Protocol:"tcp", Timeout: 60*time.Duration}
srv2, _ := NewServer("locahost", 9000, &conf)
```

这段代码算是不错了，大多数情况下，可能就止步于此了。但是其中有一点不好的是, Config 并不是必需的，所以，需要判断是否是 nil 或是 Empty 
> Config{} 这让代码感觉还是有点不是很干净

**完整代码如下：**
```go
// 函数式编程demo
package main

import (
	"crypto/tls"
	"fmt"
	"time"
)

// 拆分配置 - 非必输的选项
type ServerConfig struct {
	Protocol string
	Timeout  time.Duration
	Maxconns int
	TLS      *tls.Config
}

// 主体
type Server struct {
	Addr string
	Port int
	Conf *ServerConfig
}

// 构造函数
func NewServer(addr string, port int, conf *ServerConfig) (*Server, error) {
	s := Server{
		Addr: addr,
		Port: port,
		Conf: conf,
	}
	return &s, nil
}

func main() {
	conf := ServerConfig{Protocol: "tcp", Timeout: 60 * time.Second}

	// 使用默认配置项
	s1, _ := NewServer("localhost", 9000, nil)

	// 使用自定义配置项
	s2, _ := NewServer("locahost", 9000, &conf)

	// s1: &{Addr:localhost Port:9000 Conf:<nil>}      s2: &{Addr:locahost Port:9000 Conf:0xc000098660}
	fmt.Printf("s1: %+v\ts2: %+v\n", s1, s2)
}
```

## 优化方案2: Builder模式

builder 这样的方式不需要额外的 `Config` 类，使用**链式**的函数调用的方式来构造一个对象，只需要多加一个 `Builder` 类，这个 `Builder` 类似乎有点多余，似乎可以直接在 `Server` 上进行这样的 `Builder` 构造，**但是在处理错误的时候可能就有点麻烦（需要为Server结构增加一个error 成员，破坏了Server结构体的"纯洁"）**，不如一个包装类更好一些

```go
package main

import (
	"crypto/tls"
	"fmt"
	"time"
)

type Server struct {
	Addr     string
	Port     int
	Protocol string
	Timeout  time.Duration
	Maxconns int
	TLS      *tls.Config
}

// 使用一个builder类来做包装
type ServerBuilder struct {
	Server
}

func (sb *ServerBuilder) Create(addr string, port int) *ServerBuilder {
	sb.Server.Addr = addr
	sb.Server.Port = port
	//其它代码设置其它成员的默认值
	return sb
}

func (sb *ServerBuilder) WithProtocol(protocol string) *ServerBuilder {
	sb.Server.Protocol = protocol
	return sb
}

func (sb *ServerBuilder) WithMaxConn(maxconn int) *ServerBuilder {
	sb.Server.Maxconns = maxconn
	return sb
}

func (sb *ServerBuilder) WithTimeOut(timeout time.Duration) *ServerBuilder {
	sb.Server.Timeout = timeout
	return sb
}

func (sb *ServerBuilder) WithTLS(tls *tls.Config) *ServerBuilder {
	sb.Server.TLS = tls
	return sb
}

func (sb *ServerBuilder) Build() Server {
	return sb.Server
}

func main() {
	// 使用链式的方式创建对象
	sb := ServerBuilder{}
	server := sb.Create("127.0.0.1", 8080).
		WithProtocol("udp").
		WithMaxConn(1024).
		WithTimeOut(30 * time.Second).
		Build()

	// server: {Addr:127.0.0.1 Port:8080 Protocol:udp Timeout:30s Maxconns:1024 TLS:<nil>}
	fmt.Printf("server: %+v\n", server)
}
```

## 优化方法3: Functional Options

想省掉这个包装的结构体

### 先定义一个函数类型：
```go
type Option func(*Server)
```

### 使用函数式的方式定义一组如下的函数：

```go
func Protocol(p string) Option {
    return func(s *Server) {
        s.Protocol = p
    }
}
func Timeout(timeout time.Duration) Option {
    return func(s *Server) {
        s.Timeout = timeout
    }
}
func MaxConns(maxconns int) Option {
    return func(s *Server) {
        s.MaxConns = maxconns
    }
}
func TLS(tls *tls.Config) Option {
    return func(s *Server) {
        s.TLS = tls
    }
}
```
这组代码传入一个参数，然后返回一个函数，返回的这个函数会设置自己的 `Server` 参数。例如：
- 当调用其中的一个函数用 MaxConns(30) 时,其返回值是一个 `func(s* Server) { s.MaxConns = 30 }` 的函数

### 再定一个 NewServer()的函数，
其中，有一个可变参数 `options` 其可以传出多个上面上的函数，然后使用一个 `for-loop` 来设置 `Server` 对象

```go
func NewServer(addr string, port int, options ...func(*Server)) (*Server, error) {
	srv := Server{
		Addr:     addr,
		Port:     port,
		Protocol: "tcp",
		Timeout:  30 * time.Second,
		MaxConns: 1000,
		TLS:      nil,
	}
	for _, option := range options {
		option(&srv)
	}
	//...
	return &srv, nil
}
```

### 在创建 Server 对象

```go
s1, _ := NewServer("localhost", 1024)
s2, _ := NewServer("localhost", 2048, Protocol("udp"))
s3, _ := NewServer("0.0.0.0", 8080, Timeout(300*time.Second), MaxConns(1000))
```
实现代码高度的整洁和优雅。不但解决了使用 `Config` 对象方式需要有一个config参数，但在不需要的时候，是放 `nil` 还是放 `Config{}` 的选择困难，也不需要引用一个 `Builder` 的控制对象，直接使用函数式编程的试，在代码阅读上也很优雅

```go
package main

import (
	"crypto/tls"
	"fmt"
	"time"
)

type Server struct {
	Addr     string
	Port     int
	Protocol string
	Timeout  time.Duration
	Maxconns int
	TLS      *tls.Config
}

// 定义一个函数类型
type Option func(*Server)

// 使用函数式的方式定义一组函数
func Protocol(p string) Option {
	return func(s *Server) {
		s.Protocol = p
	}
}
func Timeout(timeout time.Duration) Option {
	return func(s *Server) {
		s.Timeout = timeout
	}
}
func MaxConns(maxconns int) Option {
	return func(s *Server) {
		s.Maxconns = maxconns
	}
}
func TLS(tls *tls.Config) Option {
	return func(s *Server) {
		s.TLS = tls
	}
}

// 定义一个可变参函数 动态设置配置
func NewServer(addr string, port int, options ...func(*Server)) (*Server, error) {
	srv := Server{
		Addr:     addr,
		Port:     port,
		Protocol: "tcp",
		Timeout:  30 * time.Second,
		Maxconns: 1000,
		TLS:      nil,
	}
	// 循环配置
	for _, option := range options {
		option(&srv)
	}
	return &srv, nil
}

func main() {
	// 默认
	s1, _ := NewServer("localhost", 1024)
	// 使用函数式编程
	s2, _ := NewServer("localhost", 2048, Protocol("udp"))
	// 使用函数式编程
	s3, _ := NewServer("0.0.0.0", 8080, Timeout(300*time.Second), MaxConns(1000))

	// s1: &{Addr:localhost Port:1024 Protocol:tcp Timeout:30s Maxconns:1000 TLS:<nil>}
	// s2: &{Addr:localhost Port:2048 Protocol:udp Timeout:30s Maxconns:1000 TLS:<nil>}
	// s3: &{Addr:0.0.0.0 Port:8080 Protocol:tcp Timeout:5m0s Maxconns:1000 TLS:<nil>}
	fmt.Printf("s1: %+v\ts2: %+v\ts3: %+v\n", s1, s2, s3)
}
```



> 参考：https://coolshell.cn/articles/21146.html

