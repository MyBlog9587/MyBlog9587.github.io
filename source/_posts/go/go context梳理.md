---
title: go context梳理
date: 2023-06-09 17:01:42
categories:
- go
tags:
- go
- context
---


## 超时控制 

```go
package main

import (
	"context"
	"fmt"
	"time"
)

func main() {
    // 创建一个 context
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

    // // 设置超时时间 3s
	timeoutCtx, timeoutCancel := context.WithTimeout(ctx, 3*time.Second) 
	defer timeoutCancel()

	doneCh := make(chan string)

    // 启动一个 goroutine 处理中间数据并发起数据库查询操作
	go processAndQueryData(timeoutCtx, doneCh)

    // 通过 select 监听任务完成情况
	select {
	case <-timeoutCtx.Done():
		fmt.Println("超时, 任务未能完成")
	case result := <-doneCh:
		fmt.Println("任务结果：", result)
	}
}

// processAndQueryData 处理中间数据并发起数据库查询操作
func processAndQueryData(ctx context.Context, doneCh chan string) {
	// 处理中间数据
	processDuration := 2 * time.Second

    // 模拟数据处理 耗时 2s
	select {
	case <-time.After(processDuration):
		fmt.Println("数据处理完成")
	case <-ctx.Done():
		fmt.Println("处理数据中途超时，直接取消")
		doneCh <- "任务取消"
		return
	}

	// 查询数据库
	queryDuration := 1 * time.Second
	if err := queryDatabase(ctx, queryDuration); err != nil {
		// 根据超时情况处理错误
		fmt.Println("查询超时，请稍后重试:", err)
		doneCh <- "查询超时"
		return
	} else {
		doneCh <- "查询成功"
	}
}

// 模拟查询数据库操作, 假设这里的 ctx 实际上用于数据库查询
func queryDatabase(ctx context.Context, duration time.Duration) error {
	select {
	case <-time.After(duration):
		fmt.Println("数据库查询完成")
		return nil
	case <-ctx.Done():
		return ctx.Err()
	}
}
```


## context 传值 并类型转换
```go
package main

import (
	"context"
	"fmt"
)

type key int

const (
	usernameKey key = iota
)

func main() {
	// 创建一个包含值的上下文
	ctx := context.WithValue(context.Background(), usernameKey, "Alice")

	// 从上下文中获取键对应的值
	value := ctx.Value(usernameKey)

	// 将值转换回真实类型
	if username, ok := value.(string); ok {
		fmt.Println("Username:", username)
	} else {
		fmt.Println("Username not found or has invalid type")
	}
}
```


