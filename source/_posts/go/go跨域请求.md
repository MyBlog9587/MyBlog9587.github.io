---
title: go跨域请求
date: 2023-07-24 18:34:12
tags:
- go
- CORS
- 跨域
categories:
- go
---




```go
// 跨域请求

package main

import (
	"fmt"
	"net/http"
	"strings"
)

// 验证请求是否合法
func validateRequest(r *http.Request) bool {
	// 在此处添加验证逻辑
	if r.Method != "POST" {
		return false
	}

	if r.Header.Get("Content-Type") != "application/json" {
		return false
	}

	if r.URL.Path != "/cors" {
		return false
	}

	return true
}

// 设置允许跨域的来源为 http://example1.com
func cors1(path string, mux *http.ServeMux) {
	mux.HandleFunc(path, func(w http.ResponseWriter, r *http.Request) {
		// 设置允许跨域的来源为 http://example1.com
		w.Header().Set("Access-Control-Allow-Origin", "http://example1.com")
		w.Header().Set("Access-Control-Allow-Methods", "GET")
		w.Header().Set("Access-Control-Allow-Headers", "Content-Type")

		// 处理请求
		if r.Method == "OPTIONS" {
			// 如果是预检请求（OPTIONS 请求），直接返回
			return
		}

		// 返回响应
		w.Write([]byte("Hello from path1, CORS!"))
	})
}

// 不设置跨域响应头，允许同源请求
func cors2(path string, mux *http.ServeMux) {
	mux.HandleFunc(path, func(w http.ResponseWriter, r *http.Request) {
		// 不设置跨域响应头，允许同源请求

		// 处理请求
		w.Write([]byte("Hello from path3!"))
	})
}

// 根据请求路径、方法等进行跨域
func cors3(path string, mux *http.ServeMux) {
	http.HandleFunc(path, func(w http.ResponseWriter, r *http.Request) {
		// 根据请求路径进行跨域
		if strings.HasPrefix(r.URL.Path, "/cors") {
			// 设置响应头部，允许跨域请求
			w.Header().Set("Access-Control-Allow-Origin", "*")
			w.Header().Set("Access-Control-Allow-Methods", "POST, GET, OPTIONS, PUT, DELETE")
			w.Header().Set("Access-Control-Allow-Headers", "Accept, Content-Type, Content-Length, Accept-Encoding, X-CSRF-Token, Authorization")
		}
		// 判断请求方法是否为OPTIONS
		if r.Method == "OPTIONS" {
			// 如果是预检请求（OPTIONS 请求），直接返回
			return
		}
		// 添加验证函数
		if !validateRequest(r) {
			http.Error(w, "Invalid request", http.StatusBadRequest)
			return
		}
		// 输出响应内容
		fmt.Fprintf(w, "Hello, World!")
	})
}


// 主函数: 添加了路径 path1、path2 和 path3，并设置不同的跨域策略
func main() {
	mux := http.NewServeMux()

	// 路径 path1
    cors1("/path1", mux)

	// 路径 path2
    cors2("/path2", mux)

	// 处理请求
     cors3("/cors", mux)

	// 启动 HTTP 服务
	if err := http.ListenAndServe(":8080", mux); err != nil {
		panic(err)
	}
}
```