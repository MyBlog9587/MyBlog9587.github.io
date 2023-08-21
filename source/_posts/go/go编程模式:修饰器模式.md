---
title: go编程模式:修饰器模式
date: 2023-07-24 18:31:55
tags:
- go
- private
categories:
- go
- 函数式编程
- Decorator
---

装饰器函数包装了目标函数，使能够在目标函数执行前后加入额外的逻辑


## http demo

### http demo
其中 `WithServerHeader()` 函数就是一个 Decorator，其传入一个 `http.HandlerFunc`，然后返回一个改写的版本

```go
package main

import (
	"fmt"
	"log"
	"net/http"
)
// WithServerHeader() 函数就是一个 Decorator，其传入一个 http.HandlerFunc，然后返回一个改写的版本
func WithServerHeader(h http.HandlerFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		log.Println("--->WithServerHeader()")
		w.Header().Set("Server", "HelloServer v0.0.1")
		h(w, r)
	}
}

func hello(w http.ResponseWriter, r *http.Request) {
	log.Printf("Recieved Request %s from %s\n", r.URL.Path, r.RemoteAddr)
	fmt.Fprintf(w, "Hello, World! "+r.URL.Path)
}

// http://127.0.0.1:8080/v1/hello --> Hello, World! /v1/hello
func main() {
	http.HandleFunc("/v1/hello", WithServerHeader(hello))
	err := http.ListenAndServe(":8080", nil)
	if err != nil {
		log.Fatal("ListenAndServe: ", err)
	}
}
```

### 延伸
仿照上述示例可以增加 写 HTTP 响应头的，写认证 Cookie 的，检查认证Cookie的，打日志的……

```go
func WithAuthCookie(h http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        log.Println("--->WithAuthCookie()")
        cookie := &http.Cookie{Name: "Auth", Value: "Pass", Path: "/"}
        http.SetCookie(w, cookie)
        h(w, r)
    }
}
func WithBasicAuth(h http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        log.Println("--->WithBasicAuth()")
        cookie, err := r.Cookie("Auth")
        if err != nil || cookie.Value != "Pass" {
            w.WriteHeader(http.StatusForbidden)
            return
        }
        h(w, r)
    }
}
func WithDebugLog(h http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        log.Println("--->WithDebugLog")
        r.ParseForm()
        log.Println(r.Form)
        log.Println("path", r.URL.Path)
        log.Println("scheme", r.URL.Scheme)
        log.Println(r.Form["url_long"])
        for k, v := range r.Form {
            log.Println("key:", k)
            log.Println("val:", strings.Join(v, ""))
        }
        h(w, r)
    }
}

func main() {
    http.HandleFunc("/v1/hello", WithServerHeader(WithAuthCookie(hello)))
    http.HandleFunc("/v2/hello", WithServerHeader(WithBasicAuth(hello)))
    http.HandleFunc("/v3/hello", WithServerHeader(WithBasicAuth(WithDebugLog(hello))))
    err := http.ListenAndServe(":8080", nil)
    if err != nil {
        log.Fatal("ListenAndServe: ", err)
    }}
```

### 优化
在使用上，需要对函数一层层的套起来，看上去好像不是很好看，如果需要 decorator 比较多的话，代码会比较难看了。重构一下用来遍历并调用各个 decorator

```go
type HttpHandlerDecorator func(http.HandlerFunc) http.HandlerFunc

func Handler(h http.HandlerFunc, decors ...HttpHandlerDecorator) http.HandlerFunc {
    for i := range decors {
        d := decors[len(decors)-1-i] // iterate in reverse
        h = d(h)
    }
    return h
}

http.HandleFunc("/v4/hello", Handler(hello,
                WithServerHeader, WithBasicAuth, WithDebugLog))
```


## func demo

```go
// 装饰器模式

package main

import (
	"fmt"
	"reflect"
)

/*
允许在不修改现有代码的情况下，动态地向对象添加新的行为：

	decoPtr 是一个指向装饰器函数的指针
	fn 是一个函数，表示要装饰的目标函数
*/
func Decorator(decoPtr, fn interface{}) (err error) {
	var decoratedFunc, targetFunc reflect.Value

	decoratedFunc = reflect.ValueOf(decoPtr).Elem()
	targetFunc = reflect.ValueOf(fn)

	v := reflect.MakeFunc(targetFunc.Type(),
		func(in []reflect.Value) (out []reflect.Value) {
			fmt.Println("before")
			out = targetFunc.Call(in)
			fmt.Println("after")
			return
		})

	decoratedFunc.Set(v)
	return
}

// TargetFunc 是想要装饰的函数
func TargetFunc(name string) {
	fmt.Println("TargetFunc Hello：", name)
}

// DecoratorFunc 是修饰的函数
func DecoratorFunc(fn func(string)) func(string) {
	return func(name string) {
		fmt.Println("DecoratorFunc Decorating...")
		fn(name)
		fmt.Println("DecoratorFunc Decoration done.")
	}
}

func main() {
	// 定义目标函数和装饰函数
	var targetFn, decoratorFn func(string)
	
	targetFn = TargetFunc
	decoratorFn = DecoratorFunc(targetFn)

	// 将装饰器应用于目标函数
	err := Decorator(&targetFn, decoratorFn)
	if err != nil {
		fmt.Println("Error:", err)
		return
	}

	// 调用修饰过的目标函数
	targetFn("John")

	/*
		Output:
			before
			DecoratorFunc Decorating...
			TargetFunc Hello： John
			DecoratorFunc Decoration done.
			after
	*/
}
```


定义了 targetFn 和 decoratorFn 两个函数变量，分别代表目标函数和装饰器函数。接着通过调用 Decorator 函数将装饰器应用到目标函数上；

`targetFn("John")` 调用 `DecoratorFunc` 的过程：

- 在 main 函数中，创建了 targetFn 和 decoratorFn 两个函数变量，分别为 TargetFunc 和 DecoratorFunc(targetFn)。
- 在 Decorator 函数内部，首先使用反射库获取 targetFn 和 decoratorFn 的反射值。
- 然后 使用 reflect.MakeFunc 创建了一个新的函数值 v，该函数值执行装饰器逻辑（在调用目标函数之前和之后输出信息）并调用 targetFn。这里的 v 就是一个结合了装饰器逻辑的新函数。
- 最后，使用 decoratedFunc.Set(v) 将创建的新函数 v 设置为 targetFn，从而将装饰器逻辑应用到 targetFn 上。

当调用 targetFn("John") 时，实际上是调用了经过装饰后的函数 v
