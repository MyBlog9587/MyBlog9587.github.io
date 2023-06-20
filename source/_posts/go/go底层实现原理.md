---
title: go底层实现原理
date: 2023-05-11 17:23:03
categories:
- go
tags: ['go', 'golang', 'private']
---

{% asset_img back.jpg 测试图片 %}




# GDB 查看程序入口
编译好的可执⾏⽂件真正执⾏⼊⼜并⾮我们所写的 main.main 函数，因为编译器总是会插⼊⼀段引导代码，完成诸如命令⾏参数、运⾏时初始化等⼯作，然后才会进⼊⽤户逻辑。 

要从 src/runtime ⽬录下的⼀堆⽂件中找到真正的⼊⼜，其实很容易。随便准备⼀个编译好的⽬标⽂件，⽐如 "Hello, World!"。

## 测试程序：test.go
```go
package main

import "fmt"

func main() { 
	fmt.Println("hello, world!") 
} 
```

## 编译，然后⽤ GDB 查看
`go build -gcflags "-N -l" -o test test.go `

> debug 时使⽤ -gcflags "-N -l" 参数关闭编译器代码优化和函数内联，避免断点和单步执⾏⽆法准确对应源码⾏，避免⼩函数和局部变量被优化掉。
如果在平台使⽤交叉编译（Cross Compile），需要设置 GOOS 环境变量。

```shell
# gdb test
(gdb) info files
Symbols from "/root/test".
Local exec file:
	`/root/test', file type elf64-x86-64.
	Entry point: 0x45c200
	0x0000000000401000 - 0x000000000047e337 is .text
	0x000000000047f000 - 0x00000000004b3eac is .rodata
	0x00000000004b4040 - 0x00000000004b4518 is .typelink
	0x00000000004b4520 - 0x00000000004b4578 is .itablink
	0x00000000004b4578 - 0x00000000004b4578 is .gosymtab
	0x00000000004b4580 - 0x000000000050cdf0 is .gopclntab
	0x000000000050d000 - 0x000000000050d020 is .go.buildinfo
	0x000000000050d020 - 0x000000000051d5e0 is .noptrdata
	0x000000000051d5e0 - 0x0000000000524df0 is .data
	0x0000000000524e00 - 0x0000000000553d08 is .bss
	0x0000000000553d20 - 0x0000000000559080 is .noptrbss
	0x0000000000400f9c - 0x0000000000401000 is .note.go.buildid
(gdb) b *0x45c200
Breakpoint 2 at 0x45c200: file /usr/local/go/src/runtime/rt0_linux_amd64.s, line 8.
```

在 src/runtime ⽬录下有很多不同平台的⼊⼜⽂件，都由汇编实现
```bash
# cd /usr/local/go/src/runtime && ls rt0_*
rt0_aix_ppc64.s      rt0_darwin_amd64.s     rt0_freebsd_arm64.s  rt0_js_wasm.s      rt0_linux_mips64x.s  rt0_linux_s390x.s   rt0_openbsd_386.s     rt0_plan9_386.s      rt0_windows_amd64.s
rt0_android_386.s    rt0_darwin_arm64.s     rt0_freebsd_arm.s    rt0_linux_386.s    rt0_linux_mipsx.s    rt0_netbsd_386.s    rt0_openbsd_amd64.s   rt0_plan9_amd64.s    rt0_windows_arm64.s
rt0_android_amd64.s  rt0_dragonfly_amd64.s  rt0_illumos_amd64.s  rt0_linux_amd64.s  rt0_linux_ppc64le.s  rt0_netbsd_amd64.s  rt0_openbsd_arm64.s   rt0_plan9_arm.s      rt0_windows_arm.s
rt0_android_arm64.s  rt0_freebsd_386.s      rt0_ios_amd64.s      rt0_linux_arm64.s  rt0_linux_ppc64.s    rt0_netbsd_arm64.s  rt0_openbsd_arm.s     rt0_solaris_amd64.s
rt0_android_arm.s    rt0_freebsd_amd64.s    rt0_ios_arm64.s      rt0_linux_arm.s    rt0_linux_riscv64.s  rt0_netbsd_arm.s    rt0_openbsd_mips64.s  rt0_windows_386.s
```

查看 rt0_linux_amd64.s, line 8 内容：
```c
  1 // Copyright 2009 The Go Authors. All rights reserved.
  2 // Use of this source code is governed by a BSD-style
  3 // license that can be found in the LICENSE file.
  4 
  5 #include "textflag.h"
  6 
  7 TEXT _rt0_amd64_linux(SB),NOSPLIT,$-8
  8         JMP     _rt0_amd64(SB)
  9 
 10 TEXT _rt0_amd64_linux_lib(SB),NOSPLIT,$0
 11         JMP     _rt0_amd64_lib(SB)
```

⽤ GDB 设置断点命令看看这个 _rt0_amd64 在哪
``` bash
(gdb) b _rt0_amd64
Breakpoint 3 at 0x458840: file /usr/local/go/src/runtime/asm_amd64.s, line 16.
```

查看 asm_amd64.s, line 16 内容
```c
  11 // _rt0_amd64 is common startup code for most amd64 systems when using
  12 // internal linking. This is the entry point for the program from the
  13 // kernel for an ordinary -buildmode=exe program. The stack holds the
  14 // number of arguments and the C-style argv.
  15 TEXT _rt0_amd64(SB),NOSPLIT,$-8
  16         MOVQ    0(SP), DI       // argc
  17         LEAQ    8(SP), SI       // argv
  18         JMP     runtime·rt0_go(SB)
			...
  81 TEXT runtime·rt0_go(SB),NOSPLIT|TOPFRAME,$0
  82         // copy arguments forward on an even stack
  83         MOVQ    DI, AX          // argc
  84         MOVQ    SI, BX          // argv
			...
// 创建初始化函数 
 210         CALL    runtime·args(SB)
 211         CALL    runtime·osinit(SB)
 212         CALL    runtime·schedinit(SB)
 213 
 214         // create a new goroutine to start program
// 创建 main goroutine 用于执行 runtime.main
 215         MOVQ    $runtime·mainPC(SB), AX         // entry
 216         PUSHQ   AX
 217         PUSHQ   $0                      // arg size
 218         CALL    runtime·newproc(SB)
 219         POPQ    AX
 220         POPQ    AX
 221 
 222         // start this M
 // 让当前线程开始i执行 main.goroutine
 223         CALL    runtime·mstart(SB)
			...
 232 
 233 // mainPC is a function value for runtime.main, to be passed to newproc.
 234 // The reference to runtime.main is made via ABIInternal, since the
 235 // actual function (not the ABI0 wrapper) is needed by newproc.
 236 DATA    runtime·mainPC+0(SB)/8,$runtime·main<ABIInternal>(SB)
 237 GLOBL   runtime·mainPC(SB),RODATA,$8 
```

⾄此，由汇编针对特定平台实现的引导过程就全部完成。后续内容基本上都是由 Golang 代码实现。

```bash
(gdb) b runtime.main
Breakpoint 3 at 0x423250: file /usr/local/go/src/runtime/proc.go, line 28.
```

## 初始化

```bash
(gdb)  b runtime.args
Breakpoint 6 at 0x43fb60: runtime.args. (2 locations)
(gdb) b *0x43fb60
Note: breakpoint 6 also set at pc 0x43fb60.
Breakpoint 7 at 0x43fb60: file /usr/local/go/src/runtime/runtime1.go, line 61.
```

```bash
(gdb) b runtime.osinit 
Breakpoint 8 at 0x42c580: runtime.osinit. (2 locations)
(gdb) b *0x42c580
Note: breakpoint 8 also set at pc 0x42c580.
Breakpoint 9 at 0x42c580: file /usr/local/go/src/runtime/os_linux.go, line 301.
```

```bash
(gdb) b runtime.schedinit 
Breakpoint 10 at 0x433160: runtime.schedinit. (2 locations)
(gdb) b *0x433160
Note: breakpoint 10 also set at pc 0x433160.
Breakpoint 11 at 0x433160: file /usr/local/go/src/runtime/proc.go, line 654.
```

### 函数 args 整理命令⾏参数
runtime1.go 
```go
 61 func args(c int32, v **byte) {
 62         argc = c
 63         argv = v
 64         sysargs(c, v)
 65 }
```

### 函数 osinit 确定 CPU Core 数量
os_linux.go
```go
301 func osinit() {
302         ncpu = getproccount()
303         physHugePageSize = getHugePageSize()
304         if iscgo {
305                 // #42494 glibc and musl reserve some signals for
306                 // internal use and require they not be blocked by
307                 // the rest of a normal C runtime. When the go runtime
308                 // blocks...unblocks signals, temporarily, the blocked
309                 // interval of time is generally very short. As such,
310                 // these expectations of *libc code are mostly met by
311                 // the combined go+cgo system of threads. However,
312                 // when go causes a thread to exit, via a return from
313                 // mstart(), the combined runtime can deadlock if
314                 // these signals are blocked. Thus, don't block these
315                 // signals when exiting threads.
316                 // - glibc: SIGCANCEL (32), SIGSETXID (33)
317                 // - musl: SIGTIMER (32), SIGCANCEL (33), SIGSYNCCALL (34)
318                 sigdelset(&sigsetAllExiting, 32)
319                 sigdelset(&sigsetAllExiting, 33)
320                 sigdelset(&sigsetAllExiting, 34)
321         }
322         osArchInit()
323 }
```

### 运⾏时环境初始化构造
proc.go
```go
 646 // The bootstrap sequence is:
 647 //
 648 //      call osinit
 649 //      call schedinit
 650 //      make & queue new G
 651 //      call runtime·mstart
 652 //
 653 // The new G calls runtime·main.
 654 func schedinit() {
		 ...
 681 
 // 最大系统线程数限制， 参考标准库 runtime/debug.SetMaxThreads 
 682         sched.maxmcount = 10000
 683 
 684         // The world starts stopped.
 685         worldStopped()
 686 
 687         moduledataverify()
 // 栈、内存分配器、调度器相关初始化
 688         stackinit()
 689         mallocinit()
 690         fastrandinit() // must run before mcommoninit
 691         mcommoninit(_g_.m, -1)
 692         cpuinit()       // must run before alginit
 693         alginit()       // maps must not be used before this call
 694         modulesinit()   // provides activeModules
 695         typelinksinit() // uses maps, activeModules
 696         itabsinit()     // uses activeModules
 697 
 698         sigsave(&_g_.m.sigmask)
 699         initSigmask = _g_.m.sigmask
 700 
 701         if offset := unsafe.Offsetof(sched.timeToRun); offset%8 != 0 {
 702                 println(offset)
 703                 throw("sched.timeToRun not aligned to 8 bytes")
 704         }
 705 
 // 处理命令行参数和环境变量
 706         goargs()
 707         goenvs()
 // 处理GODEBUG、GOTRACEBACK 调试相关的环境变量设置
 708         parsedebugvars()
 // 垃圾回收器初始化
 709         gcinit()
 710 
 711         lock(&sched.lock)
 712         sched.lastpoll = uint64(nanotime())
 // 通过 CPU core 和 GOMAXPROCS 环境变量 确定 P 数量
 713         procs := ncpu
 714         if n, ok := atoi32(gogetenv("GOMAXPROCS")); ok && n > 0 {
 715                 procs = n
 716         }
 // 调整 P 数量
 717         if procresize(procs) != nil {
 718                 throw("unknown runnable goroutine during bootstrap")
 719         }
 720         unlock(&sched.lock)
		...
 746 }
```

接下来要执⾏的是 `runtime.main`，⽽不是⽤户逻辑⼊⼜函数 `main.main`。 
proc.go 
```go
 144 // The main goroutine.
 145 func main() {
 146         g := getg()

 150         g.m.g0.racectx = 0
 // 执行栈最大限制 1GB on 64-bit, 250MB on 32-bit
 155         if sys.PtrSize == 8 {
 156                 maxstacksize = 1000000000
 157         } else {
 158                 maxstacksize = 250000000
 159         }
 
 164         maxstackceiling = 2 * maxstacksize
 165 
 166         // Allow newproc to start new Ms.
 167         mainStarted = true
 168 
 169         if GOARCH != "wasm" { // no threads on wasm yet, so no sysmon
 170                 // For runtime_syscall_doAllThreadsSyscall, we
 171                 // register sysmon is not ready for the world to be
 172                 // stopped.
 173                 atomic.Store(&sched.sysmonStarting, 1)
 // 启动系统后台监控(定期垃圾回收 以及 并发任务调度相关) 
 174                 systemstack(func() {
 175                         newm(sysmon, nil, -1)
 176                 })
 177         }
 // 执行 runtime 包内所有初始化函数 init
 204         doInit(&runtime_inittask) // Must be before defer.

 // 启动垃圾回收器 后台操作
 214         gcenable()

 // 执行所有的用户包(包含标准库) 初始化函数 init
 238         doInit(&main_inittask)
 
 // 执行用户逻辑入口 main.mian 函数
 254         fn := main_main // make an indirect call, as the linker doesn't know the address of the main package when laying down the runtime
 255         fn()
 256         if raceenabled {
 257                 racefini()
 258         }
 // 执行结束 返回退出状态码
 277         exit(0)
 278         for {
 279                 var x *int32
 280                 *x = 0
 281         }
 282 }
```

注意：
- 所有 init 函数都在同⼀个 goroutine 内执⾏。
- 所有 init 函数结束后才会执⾏ main.main 函数。 


# string
## 底层结构
在 `src/runtime/string.go` 中定义：

```go
type stringStruct struct {
    str unsafe.Pointer	// 字符串指向的底层字节数组
    len int         	// 数组长度
}
```

## 复制
复制操作也就是结构体的复制过程；并**不会涉及底层字节数组的复制**。

<img src="字符串布局.png" alt="string" style="zoom:100%;" />

字符串虽然不是切片，但是支持切片操作，不同位置的切片底层访问的是**同一块内存数据**（因为字符串是只读的，所以相同的字符串面值常量通常对应同一个字符串常量）。

## 考察点有哪些：

1. string和[]byte的区别：
	- string是不可变的，而`[]byte`是可变的。同时，string是Unicode字符序列，而`[]byte`是UTF-8编码的字节序列。 

2. 如何遍历一个字符串：
	- 可以使用 `range` 循环遍历字符串，它会返回Unicode字符和对应的索引。

3. 如何获取字符串的长度：
	- 可以使用`len()`函数获取字符串的字节数，也可以使用`utf8.RuneCountInString()`函数获取字符串的Unicode字符数。

4. 字符串是否支持切片：
	- 是的，可以使用类似于`[]byte`的切片语法来获取字符串的子串。需要注意的是，**切片得到的子串仍然指向原始字符串的内存地址**。

5. 如何连接字符串：
	- 可以使用`"+"`运算符连接两个字符串，也可以使用 `strings.Join()` 函数将多个字符串连接起来。

6. 如何将字符串转换为字节数组：
	- 可以使用强制类型转换将字符串转换为byte数组。例如，对于字符串`s := "hello"`，我们可以使用`b := []byte(s)`将其转换为一个byte数组`[]byte{'h', 'e', 'l', 'l', 'o'}`。

7. 如何将字节数组转换为字符串：
	- 可以使用 `string()` 函数将字节数组转换为对应的字符串。需要注意的是，如果字节数组中包含非UTF-8编码的数据，转换后的字符串可能会出现乱码。

8. 字符串是如何表示和存储的？
	- 字符串（string）是一种值类型（value type），底层结构是一个只读byte数组。由于字符串是只读的，因此可以避免在并发访问时出现数据竞争等问题。另外，由于字符串底层结构是一个byte数组，因此可以通过索引获取到字符串中某个字节的值。

9. 


# 数组
## 底层结构
连续的、固定长度的内存块，数组中的元素类型相同，所以它们占用的字节数相同，这也使得数组支持通过索引快速访问和修改任何元素的值。

## 考察点有哪些：

1. 数组是什么？与切片有何区别？
	- 数组是一种固定长度且存储相同类型元素的序列，而切片则是动态数组。
	- 数组长度在声明时就已经确定，并且不能更改；而切片可以根据需要动态扩展。
	- 数组通常用于存储定长对象，例如表示星期几的字符串或者一年12个月的数字列表。而切片通常用于存储可变长对象，例如读取的文件内容。

2. 如何定义和初始化一个数组？
	- 定义数组可以使用`var`关键字，格式如下：`var variable_name [size]variable_type`。
	- 初始化数组可以在声明时指定每个元素的值，格式如下：`var array_name = [size]variable_type{element1, element2, ...}`，也可以不指定元素的值，例如`var array_name [size]variable_type`，此时数组的所有元素都为该类型的零值（例如整型数组的所有元素都为0）。

3. 如何访问和修改数组中的元素？
	- 可以通过索引访问数组中的元素，索引从0开始，格式如下：`array_name[index]`。
	- 可以通过索引修改数组中的元素，例如：`array_name[index] = new_value`。

4. 数组值传递还是引用传递？
	- 在Go语言中，数组是值类型，所以在函数调用时会进行值拷贝。即使作为参数传递，也不会改变原数组的值。如果想在函数内直接改变原数组的值，可以传递指向数组的指针。

5. 如何将数组作为函数参数传递？
	可以将数组作为参数传递给函数，也可以传递数组的指针。例如：
	```go
	func printArray_value(arr [5]int) {
		for _, val := range arr {
			fmt.Println(val)
		}
	}

	func printArray_ptr(arr *[5]int) {
		for _, val := range arr {
			fmt.Println(val)
		}
	}

	func main() {
		a := [5]int{1, 2, 3, 4, 5}
		printArray_value(a)
		printArray_ptr(&a)
	}
	```

6. Go语言中的数组有哪些特点？
	- 数组长度不可变，定义时需要指定长度。
	- 数组是值类型，赋值和传递时会复制整个数组的数据。
	- Go语言中的数组支持多维数组（比如二维数组和三维数组）。


# 切片
## 底层结构
在 `src/runtime/slice.go` 中定义：

```go 
type SliceHeader struct {
    array unsafe.Pointer // 指向底层数组的指针
    len int         // 切片中元素的个数
    cap int         // 切片指向的内存空间的最大 容量（对应元素的个数，而不是字节数）
}
```

<img src="切片布局.png" alt="slice">

和数组一样，内置的 `len()` 函数返回切片中**有效**元素的长度，内置的 `cap()` 函数返回切片**容量**大小，**容量必须大于或等于切片的长度**.

切片可以和 `nil` 进行比较，只有当切片底层数据指针为空时切片本身才为 `nil`，这时候切片的长度和容量信息将是无效的.

## 复制
复制切片头信息（reflect.SliceHeader），而不会复制底层的数据。

如果 len 和 cap 都为 0 的话，则变成一个真正的空切片，虽然它并不是一个 nil 的切片。**在判断一个切片是否为空时，一般通过 len 获取切片的长度来判断**，一般很少将切片和 nil 做直接的比较。


## 扩容机制
- 如果 cap 大于两倍需求大小，则不处理
- 如果 cap 小于1024 那么 双倍 扩容
- 如果 cap 大于1024 那么每次扩大 1/4 容量

底层实现逻辑如下：
```go
func growslice(et *_type, old slice, cap int) slice {
    // ...
	newcap := old.cap
	doublecap := newcap + newcap
	if cap > doublecap {
		newcap = cap
	} else {
		if old.cap < 1024 {
			newcap = doublecap
		} else {
			// Check 0 < newcap to detect overflow
			// and prevent an infinite loop.
			for 0 < newcap && newcap < cap {
				newcap += newcap / 4
			}
			// Set newcap to the requested cap when
			// the newcap calculation overflowed.
			if newcap <= 0 {
				newcap = cap
			}
		}
	}
    // ...
}
```


## 避免切片内存泄漏
切片操作并不会复制底层的数据。底层的数组会被保存在内存中，直到它不再被引用。由于切片引用了整个原始数组，导致垃圾回收器不能及时释放底层数组的空间。

如果引用切片中的一部分数据，导致整个切片底层数组都被占用的情况。要解决这个问题，可以将感兴趣的数据复制到一个新的切片中，释放多余的空间。


## 考察点有哪些：
1. 切片和数组有什么区别？
	- 数组的长度是固定的，不能改变；而切片的长度是可变的，可以根据需要动态调整。另外，传递数组时通常会复制整个数组，而传递切片只会复制切片对象本身，而不会复制底层数组。

2. 切片是否thread-safe？
	- Go语言的切片在并发访问时是**非线程安全**的。因为切片可能会在底层数组大小不变的情况下分配新的底层数组，这就存在数据竞争问题。如果需要在并发环境下使用切片，可以使用同步机制，例如互斥锁等来保证线程安全。

3. 如何创建一个切片？
	- 可以使用`make()`函数或简单的切片字面量语法来创建一个切片。例如，使用`make([]int, 5)`可以创建一个长度为5的int类型切片，使用`[]int{1, 2, 3}`可以创建一个包含3个元素的int类型切片.

4. 切片的共享底层数组： 
	- 当多个切片共享同一个底层数组时，对其中任何一个切片的修改都会影响其他所有共享底层数组的切片

5. 切片的拷贝： 
	- 可以使用 `copy()` 函数将一个切片的元素拷贝到另一个切片中，如果目标切片长度不足，拷贝操作会截断源切片。

6. 切片的传递： 
	- 切片是引用类型，当将一个切片传递给函数时，函数接收的是切片的地址。因此，在函数内部修改切片的元素会对原始切片产生影响。

7. 切片的比较： 
	- 切片是无法使用 `==` 运算符进行比较的。要比较两个切片是否相等，需要逐个比较它们的元素。可以使用循环或 `reflect.DeepEquals()` 函数来比较切片。


# map
## 底层结构

`src/runtime/map.go`

```go
// A header for a Go map.
type hmap struct {
	// map 大小 由 len() 使用
	count     int // # live cells == size of map.  Must be first (used by len() builtin)
	// map 标记状态:
    // 1. key和value是否包指针
    // 2. 是否正在扩容
    // 3. 是否是同样大小的扩容
    // 4. 是否正在 `range` 方式访问当前的buckets
    // 5. 是否有 `range` 方式访问旧的bucket
	flags     uint8
	// 当前哈希表持有的 buckets 数量，但是因为哈希表中桶的数量都 2 的倍数，所以该字段会存储对数，也就是 len(buckets) == 2^B
	B         uint8  // log_2 of # of buckets (can hold up to loadFactor * 2^B items)
	// 溢出桶里bmap大致的数量
	noverflow uint16 // approximate number of overflow buckets; see incrnoverflow for details
	// hash 种子, 它能为哈希函数的结果引入随机性，这个值在创建哈希表时确定，并在调用哈希函数时作为参数传入
	hash0     uint32 // hash seed

	// 桶的地址，指向一个数组(连续内存空间)，数组的类型为 make(type) 中的type，如果count == 0，这里的值为 nil
	buckets    unsafe.Pointer // array of 2^B Buckets. may be nil if count==0.
	// 在扩容时用于保存之前 buckets 的字段，仅当在扩容的时候不为nil
	olduckets unsafe.Pointer // previous bucket array of half the size, non-nil only when growing
	// 扩容时已经移到新的map中的bucket数量
	nevacuate  uintptr        // progress counter for evacuation (buckets less than this have been evacuated)
	
	// 当key和value的类型不包含指针的时候，key和value就会做inline处理
    // 保证overflow的bucket活着，不被gc回收
	// 溢出桶结构，正常桶里面某个bmap存满了，会使用这里面的内存空间存放键值对
	extra *mapextra // optional fields
}

// mapextra holds fields that are not present on all maps.
type mapextra struct {
	overflow    *[]*bmap
	oldoverflow *[]*bmap

	// nextOverflow holds a pointer to a free overflow bucket.
	// 保存一个指向空闲溢出桶的指针
	nextOverflow *bmap
}

// A bucket for a Go map.
type bmap struct {
	// 存储了键的哈希的高 8 位，通过比较不同键的哈希的高 8 位可以减少访问键值对次数以提高性能
	tophash [bucketCnt]uint8
}
```

hmap 举例：
<img src="map 布局.png">


## map 创建
```go
func makemap(t *maptype, hint int, h *hmap) *hmap {
	mem, overflow := math.MulUintptr(uintptr(hint), t.bucket.size)
	if overflow || mem > maxAlloc {
		hint = 0
	}

	// initialize Hmap
	if h == nil {
		h = new(hmap)
	}
	h.hash0 = fastrand()

	// Find the size parameter B which will hold the requested # of elements.
	// For hint < 0 overLoadFactor returns false since hint < bucketCnt.
	B := uint8(0)
	for overLoadFactor(hint, B) {
		B++
	}
	h.B = B

	// allocate initial hash table
	// if B == 0, the buckets field is allocated lazily later (in mapassign)
	// If hint is large zeroing this memory could take a while.
	if h.B != 0 {
		var nextOverflow *bmap
		h.buckets, nextOverflow = makeBucketArray(t, h.B, nil)
		if nextOverflow != nil {
			h.extra = new(mapextra)
			h.extra.nextOverflow = nextOverflow
		}
	}

	return h
}

func makeBucketArray(t *maptype, b uint8, dirtyalloc unsafe.Pointer) (buckets unsafe.Pointer, nextOverflow *bmap) {
	base := bucketShift(b)
	nbuckets := base
	// For small b, overflow buckets are unlikely.
	// Avoid the overhead of the calculation.
	if b >= 4 {
		// Add on the estimated number of overflow buckets
		// required to insert the median number of elements
		// used with this value of b.
		nbuckets += bucketShift(b - 4)
		sz := t.bucket.size * nbuckets
		up := roundupsize(sz)
		if up != sz {
			nbuckets = up / t.bucket.size
		}
	}

	if dirtyalloc == nil {
		buckets = newarray(t.bucket, int(nbuckets))
	} else {
		// dirtyalloc was previously generated by
		// the above newarray(t.bucket, int(nbuckets))
		// but may not be empty.
		buckets = dirtyalloc
		size := t.bucket.size * nbuckets
		if t.bucket.ptrdata != 0 {
			memclrHasPointers(buckets, size)
		} else {
			memclrNoHeapPointers(buckets, size)
		}
	}

	if base != nbuckets {
		nextOverflow = (*bmap)(add(buckets, base*uintptr(t.bucketsize)))
		last := (*bmap)(add(buckets, (nbuckets-1)*uintptr(t.bucketsize)))
		last.setoverflow(t, (*bmap)(buckets))
	}
	return buckets, nextOverflow
}

// newarray 分配类型为 typ 的 n 个元素的数组
func newarray(typ *_type, n int) unsafe.Pointer {
	if n == 1 {
		return mallocgc(typ.size, typ, true)
	}
	mem, overflow := math.MulUintptr(typ.size, uintptr(n))
	if overflow || mem > maxAlloc || n < 0 {
		panic(plainError("runtime: allocation size out of range"))
	}
	return mallocgc(mem, typ, true)
}
```

## 分类
- hmap
- sync.map

### sync.map
`src/sync/map.go`

```go
// map 是空的并且可以被使用，但是第一次使用以后就不能被复制，因为包含原子操作
type Map struct {
	mu Mutex

	// read 包含map内容中对并发访问是安全的部分（有或没有 mu 持有）。 读取字段本身始终可以安全加载，但只能在保持 mu 的情况下存储。 存储在 read 中的条目可以在没有 mu 的情况下同时更新，但更新先前清除的条目需要将该条目复制到脏映射并在保留 mu 的情况下取消清除。
	read atomic.Value // readOnly

	// dirty 包含需要保留 mu 的映射内容部分。 为了确保dirty map可以快速提升为read map，它还包括read map中所有未清除的entry。
	// 已删除的条目不存储在dirty map中。 必须先取消清除干净map中已删除的条目并将其添加到dirty映射中，然后才能将新值存储到其中。
	// 如果dirty映射为 nil，则对映射的下一次写入将通过制作干净映射的浅表副本来初始化它，忽略过时的条目。
	dirty map[interface{}]*entry

	// miss计算自从上次read map更新后，需要锁定mu来确定键是否存在的加载次数。一旦发生了足够多的未命中以覆盖复制 dirty map 的成本，dirty map将被提升到read map(处于未修改的状态)，map的下一个存储将生成一个新的dirty副本。 
	misses int
}
```

## 是线程安全的么

## 扩容规则


# struct 
## 底层结构

## 空struct作用

## 大小

## 比较

# interface


# 反射 reflect
## `types` 和 `interface`
### 变量静态类型 和 动态类型
Go 语言中，每个变量都有一个**静态类型**，在**编译阶段**就确定了的，比如 `int`, `float64`, `[]int` 等等。注意，这个类型是声明时候的类型，**不是底层数据类型**。

```go
type MyInt int

var i int
var j MyInt
```

尽管 `i`，`j` 的底层类型都是 `int`，但我他们是不同的静态类型，除非进行类型转换，否则，`i` 和 `j` **不能同时出现在等号两侧**。`j` 的静态类型就是 `MyInt`


# 匿名函数 和 闭包
## 匿名函数

## 闭包

# channel
## 底层结构

runtime/chan.go:

```go
type hchan struct {
    // chan 里元素数量
    qcount   uint
    // chan 底层循环数组的长度
    dataqsiz uint
    // 指向底层循环数组的指针
    // 只针对有缓冲的 channel
    buf      unsafe.Pointer
    // chan 中元素大小
    elemsize uint16
    // chan 是否被关闭的标志
    closed   uint32
    // chan 中元素类型
    elemtype *_type // element type
    // 已发送元素在循环数组中的索引
    sendx    uint   // send index
    // 已接收元素在循环数组中的索引
    recvx    uint   // receive index
    // 等待接收的 goroutine 队列
    recvq    waitq  // list of recv waiters
    // 等待发送的 goroutine 队列
    sendq    waitq  // list of send waiters
    // 保护 hchan 中所有字段
    lock mutex
}
```
- `buf` 指向底层循环数组，只有缓冲型的 `channel` 才有。
- `sendx`，`recvx` 均指向底层循环数组，表示当前可以发送和接收的元素位置索引值（相对于底层数组）。
- `sendq`，`recvq` 分别表示被阻塞的 `goroutine`，这些 `goroutine` 由于尝试读取 `channel` 或向 `channel` 发送数据而被阻塞。
- `waitq` 是 `sudog` 的一个双向链表，而 `sudog` 实际上是对 `goroutine` 的一个封装：

```go
type waitq struct {
    first *sudog
    last  *sudog
}
```

`lock` 用来保证每个读 `channel` 或写 `channel` 的操作都是原子的。

创建一个容量为 6 的，元素为 `int` 型的 `channel` `ch := make(chan int, 6)` 数据结构如下: 

<img src="chan 底层结构.png">


最终创建 `chan` 的函数是 `makechan`：
```go
func makechan(t *chantype, size int64) *hchan
```

从函数原型来看，创建的 `chan` 是一个**指针**。所以能在函数间直接传递 `channel`，而不用传递 `channel` 的指针.


底层实现:
```go
// 早期版本
const hchanSize = unsafe.Sizeof(hchan{}) + uintptr(-int(unsafe.Sizeof(hchan{}))&(maxAlign-1))
func makechan(t *chantype, size int64) *hchan {
    elem := t.elem
    // 省略了检查 channel size，align 的代码
    // ……
    var c *hchan
    // 如果元素类型不含指针 或者 size 大小为 0（无缓冲类型）
    // 只进行一次内存分配
    if elem.kind&kindNoPointers != 0 || size == 0 {
        // 如果 hchan 结构体中不含指针，GC 就不会扫描 chan 中的元素
        // 只分配 "hchan 结构体大小 + 元素大小*个数" 的内存
        c = (*hchan)(mallocgc(hchanSize+uintptr(size)*elem.size, nil, true))
        // 如果是缓冲型 channel 且元素大小不等于 0（大小等于 0的元素类型：struct{}）
        if size > 0 && elem.size != 0 {
            c.buf = add(unsafe.Pointer(c), hchanSize)
        } else {
            // race detector uses this location for synchronization
            // Also prevents us from pointing beyond the allocation (see issue 9401).
            // 1. 非缓冲型的，buf 没用，直接指向 chan 起始地址处
            // 2. 缓冲型的，能进入到这里，说明元素无指针且元素类型为 struct{}，也无影响
            // 因为只会用到接收和发送游标，不会真正拷贝东西到 c.buf 处（这会覆盖 chan的内容）
            c.buf = unsafe.Pointer(c)
        }
    } else {
        // 进行两次内存分配操作
        c = new(hchan)
        c.buf = newarray(elem, int(size))
    }
    c.elemsize = uint16(elem.size)
    c.elemtype = elem
    // 循环数组长度
    c.dataqsiz = uint(size)
    // 返回 hchan 指针
    return c
}

// 1.17版本
func makechan(t *chantype, size int) *hchan {
	elem := t.elem

	// 编译器会检查这一点，但要安全.
	if elem.size >= 1<<16 {
		// chann 元素类型无效
		throw("makechan: invalid channel element type")
	}
	if hchanSize%maxAlign != 0 || elem.align > maxAlign {
		// 错误的对齐方式
		throw("makechan: bad alignment")
	}

	mem, overflow := math.MulUintptr(elem.size, uintptr(size))
	if overflow || mem > maxAlloc-hchanSize || size < 0 {
		// 内存不足
		panic(plainError("makechan: size out of range"))
	}

	// Hchan does not contain pointers interesting for GC when elements stored in buf do not contain pointers.
	// buf points into the same allocation, elemtype is persistent.
	// SudoG's are referenced from their owning thread so they can't be collected.
	// TODO(dvyukov,rlh): Rethink when collector can move allocated objects.
	var c *hchan
	switch {
	case mem == 0:
		// 队列或元素大小为零.
		c = (*hchan)(mallocgc(hchanSize, nil, true))
		// Race 检查器使用此位置进行同步.
		c.buf = c.raceaddr()
	case elem.ptrdata == 0:
		// 元素不包含指针.
		// 在一个调用中分配hchan和buf.
		c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
		c.buf = add(unsafe.Pointer(c), hchanSize)
	default:
		// 元素包含指针.
		c = new(hchan)
		c.buf = mallocgc(mem, elem, true)
	}

	c.elemsize = uint16(elem.size)
	c.elemtype = elem
	c.dataqsiz = uint(size)
	lockInit(&c.lock, lockRankHchan)

	if debugChan {
		print("makechan: chan=", c, "; elemsize=", elem.size, "; dataqsiz=", size, "\n")
	}
	return c
}
```

[channel 设计文档](https://speakerd.s3.amazonaws.com/presentations/10ac0b1d76a6463aa98ad6a9dec917a7/GopherCon_v10.0.pdf)


`channel` 的发送和接收操作本质上都是 “值的拷贝”

## 泄漏
`Channel` 可能会引发 `goroutine` 泄漏。

泄漏的原因是 `goroutine` 操作 `channel` 后，处于发送或接收阻塞状态，而 `channel` 处于满或空的状态，一直得不到改变。同时，垃圾回收器也不会回收此类资源，进而导致 `gouroutine` 会一直处于等待队列中，不见天日。

另外，程序运行过程中，对于一个 `channel`，如果没有任何 `goroutine` 引用了，`gc` 会对其进行回收操作，不会引起内存泄漏。


## 如何优雅地关闭 channe
根据 sender 和 receiver 的个数，分下面几种情况：

1. 一个 sender，一个 receiver
2. 一个 sender， M 个 receiver
3. N 个 sender，一个 reciver
4. N 个 sender， M 个 receiver

对于 1，2，只有一个 `sender` 的情况直接从 `sender` 端关闭

第 3 种情形下, 增加一个传递关闭信号的 `channel`，`receiver` 通过信号 `channel` 下达关闭数据 `channel` 指令。`senders` 监听到关闭信号后，停止接收数据
```go
func main() {
    rand.Seed(time.Now().UnixNano())
    const Max = 100000
    const NumSenders = 1000
    dataCh := make(chan int, 100)
    stopCh := make(chan struct{})
    // senders
    for i := 0; i < NumSenders; i++ {
        go func() {
            for {
                select {
                case <- stopCh:
                    return
                case dataCh <- rand.Intn(Max):
                }
            }
        }()
    }
    // the receiver
    go func() {
        for value := range dataCh {
            if value == Max-1 {
                fmt.Println("send stop signal to senders.")
                close(stopCh)
                return
            }
            fmt.Println(value)
        }
    }()
    select {
    case <- time.After(time.Hour):
    }
}
```

最后一种情况，还是采取第 3 种解决方案，由 `receiver` 直接关闭 `stopCh` 的话，就会重复关闭一个 `channel`，导致 `panic`。因此需要增加一个中间人，M 个 `receiver` 都向它发送关闭 `dataCh` 的“请求”，中间人收到第一个请求后，就会直接下达关闭 `dataCh` 的指令（通过关闭 `stopCh`，这时就不会发生重复关闭的情况，因为 `stopCh` 的发送方只有中间人一个）。另外，这里的 N 个 `sender` 也可以向中间人发送关闭 `dataCh` 的请求
```go
func main() {
    rand.Seed(time.Now().UnixNano())
    const Max = 100000
    const NumReceivers = 10
    const NumSenders = 1000
    dataCh := make(chan int, 100)
    stopCh := make(chan struct{})
    // It must be a buffered channel.
    toStop := make(chan string, 1)
    var stoppedBy string
    // moderator
    go func() {
        stoppedBy = <-toStop
        close(stopCh)
    }()
    // senders
    for i := 0; i < NumSenders; i++ {
        go func(id string) {
            for {
                value := rand.Intn(Max)
                if value == 0 {
                    select {
                    case toStop <- "sender#" + id:
                    default:
                    }
                    return
                }
                select {
                case <- stopCh:
                    return
                case dataCh <- value:
                }
            }
        }(strconv.Itoa(i))
    }
    // receivers
    for i := 0; i < NumReceivers; i++ {
        go func(id string) {
            for {
                select {
                case <- stopCh:
                    return
                case value := <-dataCh:
                    if value == Max-1 {
                        select {
                        case toStop <- "receiver#" + id:
                        default:
                        }
                        return
                    }
                    fmt.Println(value)
                }
            }
        }(strconv.Itoa(i))
    }
    select {
    case <- time.After(time.Hour):
    }
}
```

## 应用场景
- 停止信号
关闭某个 `channel` 或者向 `channel` 发送一个元素，使得接收 `channel` 的那一方获知道此信息，进而做一些其他的操作

- 任务定时
与 `timer` 结合
```go
select {
    case <-time.After(100 * time.Millisecond):
    case <-s.stopc:
        return false
}
```


- 解耦生产方和消费方
分生产者 消费者处理同一组 `chan`


- 控制并发数
通过 `chan` 控制数量增减


# 内存逃逸

# 内存管理

# GMP模型

# GC 原理












