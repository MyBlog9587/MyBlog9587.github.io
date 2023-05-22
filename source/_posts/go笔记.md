---
title: go笔记
date: 2023-05-11 23:43:06
tags: ['golang']
---

{% asset_img back.png 背景图片 %}

### go 注意点汇总：

#### go build 注意点
使用`go build`时有一个地方需要注意，对外发布编译文件如果不希望被人看到源代码，请使用`go build -ldflags` 命令，设置编译参数`-ldflags "-w -s"` 再编译后发布。避免使用`gdb`来调试而清楚看到源代码。

#### init 函数
1. init()函数是⽤于程序执⾏前做包的初始化的函数，⽐如初始化包⾥的变量等；
2. ⼀个包可以出现多个init()函数，⼀个源⽂件也可以包含多个init()函数；
3. 同⼀个包中多个init()函数的执⾏顺序没有明确的定义，但是不同包的init函数是根据包导⼊的依赖
关系决定的；
4. init函数在代码中不能被显示调⽤、不能被引⽤（赋值给函数变量），否则出现编译失败；
5. **⼀个包被引⽤多次，如A import B， C import B， A import C， B被引⽤多次，但B包只会初始化⼀
次；**
6. 引⼊包，不可出现死循环。即A import B， B import A，这种情况下编译失败；

目录结构：
```bash
├── a
│   └── init.go
├── b
│   └── init.go
└── main.go
```
a/init.go
```go
package a

import (
	_ "b"
	"fmt"
)

func init() {
	fmt.Println("inti from a")
}
```

b/init.go
```go
package b

import "fmt"

func init()  {
	fmt.Println("init from b")
}
```

main.go
```go
package main

import (
	_ "a"
	_ "b"
	"fmt"
)

func init() {
	fmt.Println("main init")
}

// inti from b 
// inti from a
// main init
func main() {

}
```


#### 值引用、指针引用类型：

- 值引用：
- 指针引用：


#### 声明
- 已经声明，不能使⽤ `:=`
- 当多值赋值时， `:=` 左边的变量⽆论声明与否都可以

```go
package main

import "fmt"

func f() (int, error) {
	return 0, nil
}

func main() {
	var x int
	// 错误
	x, _ := f()
	// 正确
	x, _ = f()
	// 正确
	x, y := f()
	// 错误
	x, y = f()

	fmt.Println(x, y)
}
```


#### 变量赋值：

`var`：

- 作用于全局变量

 `:=` : 

- 不能提供数据类型，编译器会⾃动推导；
- 只能在函数内部使⽤简短模式；
- 不能⽤于结构体字段赋值.

```go
var (
	// := 只能在函数内部使用
	size := 1024
	max_size = size * 2
)
func main() {
	fmt.Println(size, max_size)
}
```

多重赋值分为两个步骤，有先后顺序：计算等号左边的索引表达式和取址表达式，接着计算等号右边的表达式；赋值；
```go
func main() {
	i := 1
	s := []string{"A", "B", "C"}
	i, s[i-1] = 2, "Z"
	// s: [Z,B,C]
	fmt.Printf("s: %v \n", s) 
  
  var a []int = nil
  // 运行时错误 a[0] 不存在
  a, a[0] = []int{1, 2}, 9
  fmt.Println(a)
}
```


#### 变量作用域：
- 如果新变量与同名已定义的变量不在同⼀个作⽤域中，那么 Go 会新定义这个变量

```go
package main

import (
   "fmt"
)

var p *int

func foo() (*int, error) {
    var i int = 5
    return &i, nil
}
func bar() {
    //use p
    fmt.Println(*p)
}

func main() {
	// main() 函数⾥的 p 是新定义的变量(:=)，会遮住全局变量 p，导致执⾏到 bar() 时程序，全局变量 p 依然还是 nil，程序随即 Crash
	p, err := foo() // invalid memory address or nil pointer dereference
	// 优化
//  var err error
//  p, err = foo()
	if err != nil {
	fmt.Println(err)
	return
	}
	bar()
	fmt.Println(*p)
}
```

```go
func main() {
  x := 1
  fmt.Println(x)
  {
    fmt.Println(x)
    // 如果符号左边有多个变量，只需要保证⾄少有⼀个变量是新声明的，并对已定义的变量尽进⾏赋值操作。
    i,x := 2,2
    fmt.Println(i,x)
  }
  // 但如果出现作⽤域之后，就会导致变量隐藏的问题，
  fmt.Println(x) // print 1
}
```



#### make 和 new 的区别：

- new(T) 和 make(T, args) 是Go语⾔内建函数，⽤来分配内存，但适⽤的类型不⽤。
- new(T) 会为了 T 类型的新值分配已置零的内存空间，并返回地址（指针），即类型为 *T 的值。换句话说就是，返回⼀个指针，该指针指向新分配的、类型为 T 的零值。适⽤于值类型，如数组 、 结构体 等。
- make(T, args) 返回初始化之后的T类型的值，也不是指针 *T ，是经过初始化之后的T的引⽤。**make() 只适⽤于 slice 、 map 和 channel** 。

```go
/*
	不能通过编译
	new([]int) 之后的 list 是⼀个 *int[] 类型的指针，不能对指针执⾏ append 操作。可以使⽤ make() 初始化之后再⽤。同样的， map 和 channel 建议使⽤ make() 或字⾯量的⽅式初始化，不要⽤ new 
*/
func main() {
	list := new([]int)
	// panic
	list = append(list, 1)
	fmt.Println(list)
}
```

####  类型别名 与 类型定义 的区别：

```go
package main
import "fmt"

// 基于类型 int 创建了新类型 MyInt1
type MyInt1 int
// 创建了int的类型别名 MyInt2
// 类型别名的定义是 =
type MyInt2 = int

func main() {
	var i int = 0
	// 将int类型的变量赋值给MyInt1类型的变量， Go是强类型语⾔，编译不通过
	var i1 MyInt1 = i
	// 使⽤强制类型转换 进行赋值
	// var i1 MyInt1 = MyInt1(i)

	// MyInt2只是int的别名，本质上还是int，可以赋值
	var i2 MyInt2 = i
	fmt.Println(i1, i2)
}
```

#### 类型选择
- **只有接⼝类型才能使⽤类型选择**
- 类型选择的语法形如： i.(type)，其中**i是接⼝**， **type是固定关键字**，需要注意的是，只有接⼝类型才可以使⽤类型选择。

```go
func GetValue() int {
	return 1
}

// 编译失败
func main() {
	i := GetValue()	// 断言只能是针对接口
	switch i.(type) {
	case int:
		fmt.Println("int")
	case string:
		fmt.Println("string")
	case interface{}:
		fmt.Println("interface")
	default:
		fmt.Println("unknown")
	}
}
```

#### 类型转换
**语法**：
`<结果类型> := <目标类型> ( <表达式> )`
```go
var i int = 1
var j float64 = 0.01
f = float64(i)
a = int(f)

// 非基础类型 转换异常
// s := []int(i) // cannot convert i (type int) to type []int
```

#### 类型断言
当一个函数的形参是 interface{}，那么在函数中，需要对形参进行断言，从而得到它的真实类型; 

类型转换和类型断言有些相似，不同之处，在于类型断言是对接口进行的操作.

**断言的语法**：

`<目标类型的值>，<布尔参数> := <表达式>.( 目标类型 ) // 安全类型断言`
`<目标类型的值> := <表达式>.( 目标类型 )　　//非安全类型断言`

```go
func main() {
    var i interface{} = new(Student)
    s, ok := i.(Student)	// 断言
    if ok {
        fmt.Println(s)
    }
}
```

> 即使断言失败也不会 panic

#### 类型检查
```go
var _ io.Writer = (*myWriter)(nil)
```
编译器会由此检查 `*myWriter` 类型是否实现了 `io.Writer` 接口

```go
package main
import "io"

type myWriter struct {
}

/*func (w myWriter) Write(p []byte) (n int, err error) {
    return
}*/

func main() {
    // 检查 *myWriter 类型是否实现了 io.Writer 接口
    var _ io.Writer = (*myWriter)(nil)
    // 检查 myWriter 类型是否实现了 io.Writer 接口
    var _ io.Writer = myWriter{}
}
```

注释掉为 `myWriter` 定义的 `Write` 函数后，运行程序报错信息：`*myWriter/myWriter` 未实现 `io.Writer` 接口，也就是未实现 `Write` 方法

解除注释后，运行程序不报错


#### 比较

- go 是强类型，不同类型之间不能直接操作，需要强制转换

- 直接比较必须相同类型才可以进行，数组的⻓度也是数组类型的组成部分，所以 a 和 b 是不同的类型，是不能⽐较的，所以编译错误
- 不能直接⽐较的有 slice、 map、函数

```go
// 编译错误
func main() {
	a := 5
	b := 8.1
	// a 的类型是 int ， b 的类型是 float ，两个不同类型的数值不能相加，编译报错
	fmt.Println(a + b)
}
```

```go
// 编译错误
func main() {
	a := [2]int{5, 6}
	b := [3]int{5, 6}
	if a == b {
	fmt.Println("equal")
	} else {
	fmt.Println("not equal")
	}
}
```

#### ⾃增⾃减操作:

​	i++ 和 i-- 在 Go 语⾔中是语句，不是表达式，因此**不能赋值给另外的变量**。此外没有 ++i 和 --i。

#### 包类型作用域：

- 基于类型创建的⽅法必须定义在同⼀个包内。

```go
// 基于 int 类型创建了 PrintInt() ⽅法，由于 int类型和⽅法 PrintInt() 定义在不同的包内，所以编译出错。
func (i int) PrintInt () {
	fmt.Println(i)
}

// 编译出错
func main() {
	var i int = 1
	i.PrintInt()
}

/* 修改：定义⼀种新的类型 */
type Myint int

func (i Myint) PrintInt () {
	fmt.Println(i)
}

func main() {
	var i Myint = 1
	i.PrintInt()
}
```

#### 常量：

- 常量不同于变量的在运⾏期分配内存，常量通常会被编译器在**预处理阶段**直接展开，作为指令数据使⽤，所以**常量⽆法寻址**。
- 常量未使⽤是能编译通过。
- 常量是⼀个简单值的标识符，在程序运⾏时，不会被修改的量。
- 常量组中如不指定类型和初始化值，则与上⼀⾏⾮空常量右值相同.

```go
const i = 100

const (
  x uint16 = 120
  y 						// uint16 120
  s	= "abc"
  z							// string abc
)

func main() {
	// cannot take the address of i 
	fmt.Println(&i, i)
}
```



#### array:

- 数组⻓度是数组类型的⼀部分
```go
// 编译错误
func main() {
	// […]int{1} 和 [2]int{1}是两种不同的类型（数组⻓度不同），不能⽐较
	fmt.Println([...]int{1} == [2]int{1})
}
```

- 数组和切片的注意点
```go
package main

import (
   "fmt"
)


func main() {
//  var a = [5]int{1, 2, 3, 4, 5}
    var a = []int{1, 2, 3, 4, 5}
    var r [5]int
    for i, v := range a {
        if i == 0 {
            a[1] = 12
            a[2] = 13
        }
        r[i] = v
    }
    fmt.Println("r = ", r)
    fmt.Println("a = ", a)

    //r = [1 2 3 4 5] 	// 数组发生拷贝 不影响原本的
    //r = [1 12 13 4 5]	// slice发生拷贝，但是指向数据的指针不变，会影响原数据
    
    //a = [1 12 13 4 5]
}
```

#### slice：

- 长度len：[n,m] **左闭右开**规则，len = m-n
- 容量cap：创建时未指定cap 即`s := make([]int, n)` 则cap = len；如果基于数组或其他切片创建，则 cap = 基于数组或切片的长度或cap 减去当前新切片的起始位置，比如`s4 := s3[3:6], cap(s3) = 8`. 则`len(s4) = 6-3, cap(s4) = cap(s3) - 3`; 并且s4看不到s3前3个元素内容。
- 截取之后获得的切⽚的⻓度和容量的计算⽅法： ⻓度： j-i，容量： k-i （**假如底层数组的⼤⼩为 k**）
- 截取操作符还可以有第三个参数，形如 `[i,j,k]`，第三个参数 k ⽤来限制新切⽚的容量，但不能超过原数组（切⽚）的底层数组⼤⼩。截取获得的切⽚的⻓度和容量分别是： `j-i`、 `k-i`。
- 不能比较
- 截取符号 `[i:j]`，如果 j 省略，默认是原切⽚或者数组的**⻓度**

```go
// nil切片，不会分配内存
var s1 []int
// 空切片, 空切⽚和 nil 不相等，表示⼀个空的集合
var s2 = []int{}

fmt.Printf("ptr:%t\tcap:%d\tlen:%d\n", s1 == nil, cap(s1), len(s1)) // ptr:true		cap:0	len:0
fmt.Printf("ptr:%t\tcap:%d\tlen:%d\n", s2 == nil, cap(s2), len(s2)) // ptr:false	cap:0	len:0
```

- 初始化：
```go
func SliceInit()  {
	// len = cap
	s1 := []int{1, 2}
	s2 := []int{1, 2, 3}
	s3 := []int{1, 2, 3, 4}
	s4 := []int{1, 2, 3, 4, 5}
/*	s1: 2   2       [1 2]
	s2: 3   3       [1 2 3]
	s3: 4   4       [1 2 3 4]
	s4: 5   5       [1 2 3 4 5]*/
	fmt.Printf("s1: %d\t%d\t%v\n", len(s1), cap(s1), s1)
	fmt.Printf("s2: %d\t%d\t%v\n", len(s2), cap(s2), s2)
	fmt.Printf("s3: %d\t%d\t%v\n", len(s3), cap(s3), s3)
	fmt.Printf("s4: %d\t%d\t%v\n", len(s4), cap(s4), s4)

	// 初始化为 [0,0], 未指明cap 则len = cap
	s11 := make([]int,2)
	s12 := make([]int,2)
	s13 := make([]int, 2, 2)
	s14 := make([]int, 2, 3)
/*	s1: 2   2       [0 0]
	s2: 2   2       [0 0]
	s3: 2   2       [0 0]
	s4: 2   3       [0 0]*/
	fmt.Printf("s1: %d\t%d\t%v\n", len(s1), cap(s1), s11)
	fmt.Printf("s2: %d\t%d\t%v\n", len(s2), cap(s2), s12)
	fmt.Printf("s3: %d\t%d\t%v\n", len(s3), cap(s3), s13)
	fmt.Printf("s4: %d\t%d\t%v\n", len(s4), cap(s4), s14)

	// len=cap=0, 默认值未[]
	s21 := *new([]int)
	// s2: 0   0       []
	fmt.Printf("s2: %d\t%d\t%v\n", len(s2), cap(s2), s21)
}
```

- 赋值
```go
func SliceRise(s []int)  {
	// s: 0xc0000b0000 [1 2]
	/*
		在未扩容的前提下, 为什么对实参没有改变？
		答：该还是为值传递，形参s 为实参的一个副本。在未扩容的前提下，append真是改变了临时副本数据；
			在发生扩容时，产生底层产生新数组指针，也不影响实参
	*/

	s =append(s, 0)
	// s: 0xc0000b6020 [1 2 0]
	for i := range s {
		/*
			在未扩容的前提下, 为什么改变实参？
			答：因为slice 实参和形参 指向数组的指针时同一个。所以改变形参会改变实参
		*/
		s[i]++
	}
	// s: 0xc0000b6020 [2 3 1]
}

func SlicePrint()  {
	s1 := []int{1, 2}
	// s2>>>>  0xc0000b0000    2       2       [1 2]
	s2 := s1

	// s1>>>>  0xc0000b0000    2       2       [1 2]
	// s2>>>>  0xc0000b6000    3       4       [1 2 3]
	s2 = append(s2, 3)

	SliceRise(s1)
	SliceRise(s2)
	// 0xc0000b0000    2       2       [1 2]
	fmt.Printf("s1<<<<  %p\t%d\t%d\t%v\n", s1, len(s1), cap(s1), s1)
	// s2 没有打印出[2 3 4 1] 是因为len=3, 所以纸打印前三个，对比SliceRisePointer()
	// 0xc0000b6000    3       4       [2 3 4]
	fmt.Printf("s2<<<<  %p\t%d\t%d\t%v\n", s2, len(s2), cap(s2), s2)
	//p := unsafe.Pointer(&s2)
	//fmt.Printf("%d\n", *((*int)(unsafe.Pointer(uintptr(p) + unsafe.Offsetof(8)))))
}
```
```go
func SliceRisePointer(s *[]int)  {
	fmt.Printf("1111  %p %v\n", s, s)
	// 0xc00000c030 &[1 2 3]
	*s =append(*s, 0)

	// 0xc000018100 [1 2 3 0]
	fmt.Printf("2222  %p %v\n", *s, *s)
	for i := range *s {
		(*s)[i]++
	}
	// 0xc000018100 [2 3 4 1]
	fmt.Printf("3333  %p %v\n--------------\n", *s, *s)
}

func main() {
	s1 := []int{1, 2}
	s2 := s1
	// 0xc0000140e0-0xc0000140e0
	fmt.Printf("%p: %v-%p: %v\n", s1, s1, s2, s2)

	s2 = append(s2, 3)
	// 0xc0000140e0-0xc000018100
	fmt.Printf("%p: %v-%p: %v\n", s1, s1, s2, s2)

	SliceRisePointer(&s2)
	// 0xc0000140e0: [1 2]-0xc000018100: [2 3 4 1]
	fmt.Printf("%p: %v-%p: %v\n", s1, s1, s2, s2)
	// s1: 4   4       [2 3 4 1]
	fmt.Printf("s2: %d\t%d\t%v\n", len(s2), cap(s2), s2)
}
```

- fmt.Println 的坑（print 打印的是slice[0:len]）

```go
package main

import "fmt"

func main() {
	sl := make([]int, 0, 10)
	// 此处为值传递
	var appenFunc = func(s []int) {
		// 改变了s 的 len 0->3
		s = append(s, 10, 20, 30)
		fmt.Println(len(s), cap(s), s)	// 3 10 [10 20 30]
	}
	
	fmt.Println(len(sl), cap(sl), sl)	// 0 10 []
	
	appenFunc(sl)
	// print 打印的是 [0:len]  所以为空
	fmt.Println(len(sl), cap(sl), sl) 	// 0 10 []
	
	// sl 的len、cap 没有改变
	fmt.Println(len(sl), cap(sl), sl[:10]) 	// 0 10 [10 20 30 0 0 0 0 0 0 0]
}
```

- 作为参数传递

```go
func main() {
	a := []int{7, 8, 9}
	// 0xc00008e000-[7 8 9] - 3
	fmt.Printf("%p - %+v - %d\n", a, cap(a))

	ap(a)
	// 0xc00008e000-[7 8 9] - 3
	fmt.Printf("%p - %+v - %d\n", a, cap(a))

	app(a)
	// 0xc00008e000-[1 8 9] - 3
	fmt.Printf("%p - %+v - %d\n", a, cap(a))
}

// append导致底层数组重新分配内存了， append中的a这个alice的底层数组和外⾯不是⼀个，并没有改变外⾯的。
func ap(a []int) {
	// 0xc00008e000-[7 8 9]
	fmt.Printf("append %p - %+v\n", a)
	a = append(a, 10)
	// append 0xc000080030-[7 8 9 10]
	fmt.Printf("append end: %p - %+v\n", a)
}

// 数据地址是相同的，所以外面的改变
func app(a []int) {
	// 0xc00008e000-[7 8 9]
	fmt.Printf("=== %p - %+v\n", a)
	a[0] = 1
	// 0xc00008e000-[1 8 9]
	fmt.Printf("=== end: %p - %+v\n", a)
}
```

截取时省略的坑：
```go
package main
func main() {
	x := make([]int, 2, 10)
	_ = x[6:10]
	
	// 省略了j，默认是原切⽚或者数组的⻓度；x 的⻓度是 2，⼩于起始下标 6 ，所以 panic。
	_ = x[6:] 	// panic: runtime error: slice bounds out of range [6:2]
	
	_ = x[2:]	// [2:len(x)] --> len=2-2=0, cap=cap(x)-2=8
}
```

#### map：

- Go 语言字典的**键类型**不可以是函数类型、字典类型和切片类型。因为在**键类型的值之间必须可以施加操作符`==`和`!=`。换句话说，键类型的值必须要支持判等操作**。由于函数类型、字典类型和切片类型的值并不支持判等操作，所以字典的键类型不能是这些类型。
- 如果**键的类型是接口类型的**，那么键值的实际类型也不能是上述三种类型，否则在程序运行过程中会引发 panic（即运行时恐慌）。
- **在值为`nil`的字典上执行读写操作会成功吗**：写操作会引起panic，其他操作不会异常。
- map并发读写需要加锁，map操作不是并发安全的。（判断一个操作是否是原子的可以使用 `go run race` 命令做数据的竞争检测）
- 删除 map 不存在的键值对时，不会报错，相当于没有任何作⽤；获取不存在的减值对时，返回值类型对应的零值，所以返回 0。
- **map 的 value 本身是不可寻址的。**

```go
package main

import (
	"fmt"
)

type persion struct {
	name string
}

func main() {
	var m map[persion]int // 只声明了map m ,并没有分配内存空间，不能直接赋值，需要使⽤ make()
	p := persion{"make"}
	// 打印⼀个map中不存在的值时，返回元素类型的零值。
	// 这个例⼦中， m的类型是map[person]int，因为m中 不存在p，所以打印int类型的零值，即0
	fmt.Println(m[p])

	m[p] = 1
	// panic: assignment to entry in nil map
	fmt.Println(m)
}

/*修改*/
var m map[persion]int
m = make(map[persion]int)
m[p] = 1
```

**map 的 value 本身是不可寻址**

举例1:

```go
type Math struct {
	x, y int
}

var m1 = map[string]Math{
	"foo": Math{2, 3},
}

var m2 = map[string]*Math{
	"foo": &Math{2, 3},
}

func map_value() {

	// error: value 不可直接寻址操作
	m1["foo"].x = 4 // cannot assign to struct field m["foo"].x in map
	fmt.Println(m1["foo"].x)

	// 1. 使⽤临时变量：
	tmp := m1["foo"]
	tmp.x = 4
	m1["foo"] = tmp
	fmt.Println(m1["foo"].x)

	// 2. 修改数据结构：
	m2["foo"].x = 4
	fmt.Println(m2["foo"].x)
	fmt.Printf("%#v", m2["foo"])
}
```

举例2:

```go
type T struct {
	n int
}

func main() {
  m := make(map[int]T)
  // map[key]struct 中 struct 是不可寻址的，所以⽆法直接赋值
  m[0].n = 1
  // 以下修改
  //t := T{1}
	//m[0] = t
  fmt.Println(m[0].n)
}
```



#### struct：

- 控制各字段的访问权限：
- 如果一个字段的声明中只有字段的类型名而没有字段的名称，那么它就是一个**嵌入字段**，也可以被称为**匿名字段**；嵌入字段的类型既是类型也是名称

```go
type AnimalCategory struct {
	genus  string // 注意后面不能有逗号
	species string
}

func (ac AnimalCategory) String() string {
	return fmt.Sprintf("%s%s",ac.genus, ac.species)
}

type Animal struct {
	scientificName string
	AnimalCategory // 匿名字段
}

category := AnimalCategory{species: "cat"}

animal := Animal{
	scientificName: "American Shorthair",
	AnimalCategory: category,
}

// 使用fmt.Printf函数和%s占位符试图打印animal的字符串表示形式，相当于调用animal的String方法。虽然还没有为Animal类型编写String方法，但在这里，嵌入字段AnimalCategory的String方法会被当做animal的方法调用
// 如果Animal类型也有一个String方法，那么 animal 的String方法会被调用；即只要名称相同，无论这两个方法的签名是否一致，被嵌入类型的方法都会“屏蔽”掉嵌入字段的同名方法，只能通过链式表达式引用（同名的字段同样道理）。
fmt.Printf("The animal: %s\n", animal)
```

- 结构体的⽐较，有⼏个需要注意的地⽅：
  1. 结构体只能⽐较是否相等，但是不能⽐较⼤⼩；

  2. 同类型的结构体才能进⾏⽐较，结构体是否相同不但与属性类型有关，还与属性顺序相关；

  3. 如果struct的所有成员都可以⽐较，则该struct就可以通过==或!=进⾏⽐较是否相同，⽐较时逐个项进⾏⽐较，如果每⼀项都相等，则两个结构体才相等，否则不相等；

    

  那有什么是可以⽐较的呢？
  	- 常⻅的有bool、数值型、字符、指针、数组等

  不能⽐较的有
  	- slice、 map、函数

```go
func main() {
	// 结构体初始化
	sn1 := struct {
		age int
		name string
	}{age: 11, name: "qq"} // 下面的}提到这儿来可以省略逗号
	/*
	p := sn1{
		name: "taozs", 
		age:  18, //注意后面要加逗号 或 者下面的}提到这儿来可以省略逗号
	}
	*/

	sn2 := struct {
		age int
		name string
	}{age: 11, name: "11"}

	if sn1 == sn2 {
		fmt.Println("sn1 == sn2")
	}	
  
	// 包含map 不能做比较 
	sm1 := struct {
		age int
		m map[string]string
	}{age: 11, m: map[string]string{"a": "1"}}

	sm2 := struct {
		age int
		m map[string]string
	}{age: 11, m: map[string]string{"a": "1"}}

	// invalid operation: sm1 == sm2
	if sm1 == sm2 {
		fmt.Println("sm1 == sm2")
	}
}
```

不可寻址举例：

```go
type T struct {
	n int
}
func (t *T) Set(n int) {
	t.n = n
}
// T{} 是不可寻址的，不能直接调⽤⽅法
func getT() T {
	return T{}
}
func main() {
  // getT().Set(1) // 异常
  t := getT()
  t.Set(2)
  fmt.Println(t.n)
}
```

**逃逸引起的误解**
```go
type People struct {
}

func f1() {
	a := &People{} // &People{} does not escape
	b := &People{}
	println(a, b, a == b) // 0xc00010feef 0xc00010feef false

	c := &People{} // &People{} escapes to heap
	d := &People{}
	fmt.Printf("%p\n", c) // 0x8cede0	... argument does not escape
	fmt.Printf("%p\n", d) // 0x8cede0	... argument does not escape
	fmt.Println(c == d)   // true		... argument does not escape	c == d escapes to heap  

	c1 := make(chan int)
	c2 := make(chan int)
	fmt.Println(c1 == c2) // false		... argument does not escape	c1 == c2 escapes to heap
}
```
针对上述经过 print 出现两种不同结果，运行 go run -gcflags="-m -l"  进行逃逸分析可得

```bash
# go run -gcflags="-m -l" main.go
# command-line-arguments
./main.go:21:7: &People{} does not escape
./main.go:22:7: &People{} does not escape
./main.go:25:7: &People{} escapes to heap
./main.go:26:7: &People{} escapes to heap
./main.go:27:12: ... argument does not escape
./main.go:28:12: ... argument does not escape
./main.go:29:13: ... argument does not escape
./main.go:29:16: c == d escapes to heap
./main.go:33:13: ... argument does not escape
./main.go:33:17: c1 == c2 escapes to heap
0xc000086f57 0xc000086f57 false
0x5510b0
0x5510b0
true
false
```
关键原因是因为调用了 fmt.Println 方法，该方法内部是涉及到大量的反射相关方法的调用，会造成逃逸行为，也就是分配到堆上; 

为什么逃逸后，两个空 struct 会是相等？
主要与 Go runtime 的一个优化细节有关
```bash
// runtime/malloc.go
var zerobase uintptr
```
变量 zerobase 是所有 0 字节分配的基础地址。换句话说就是空（0字节）在进行了逃逸分析后，往堆分配的都会指向 zerobase 这一个地址。所以 **空 struct** 在逃逸后本质上指向了 zerobase，其两者比较就是相等的，返回了 true； 为什么没逃逸前，两个空 struct 比较不相等，是因为go 设计特性决定减少对使用影响



#### iota:

- iota :特殊常量，可以认为是一个可以被编译器修改的常量,每当 iota 在新的一行被使用时，它的值都会自动加 1.在每一个const关键字出现时，被重置为 0.
- 前面的操作数如果不指定，默认使用上一个的。

```go
const (
	a = iota   //0
	b          //1
	c          //2
	d = "ha"   //独立值，iota += 1
	e          //"ha"   iota += 1
	f = 100    //iota +=1
	g          //100  iota +=1
	h = iota   //7,恢复计数
	i          //8
)


const (
	i=1<<iota
	j=3<<iota
	k
	l
)
i=1<<0        0001    i= 1
j=3<<1        0011    j= 6
k=3<<2        0011    k= 12
l=3<<3        0011    l= 24
// 前面的操作数如果不指定，默认使用上一个的.

const (
	name = "name" // iota = 0
	c = iota // 1
	d = iota // 2
)

const (
	mutexLocked      = 1 << iota // 1
	mutexWoken                   // 2
	mutexStarving                // 4
	mutexWaiterShift = iota      // 3
)

```


#### for ... rang:

- for range 循环的时候会创建每个元素的**副本，⽽不是每个元素的引⽤**。
- 循环次数在循环**开始前**就已经确定，**循环内改变切⽚的⻓度，不影响循环次数**。
- for range 使⽤短变量声明(:=)的形式迭代变量; 变量 i、 v 在每次循环体中都会**被重⽤，⽽不是重新声明**。
- for 循环不⽀持以逗号为间隔的多个赋值语句，必须使⽤平⾏赋值的⽅式来初始化多个变量.

```go
package main

import "fmt"

func main() {
	slice := []int{0, 1, 2, 3}
	m := make(map[int]*int)
	for key, val := range slice {
		// m[key] = &val 取的都是变量val的地址，所以最后 map 中的所有元素的值都是变量 val 的地址，
		// 因为最后 val 被赋值为3，所有输出的都是3
		m[key] = &val
	}

	for k, v := range m {
		fmt.Println(k, "->", *v)
	}
}
/*
0 -> 3
1 -> 3
2 -> 3
3 -> 3
*/
```

各个 goroutine 中输出的 i、 v 值都是 for range 循环结束后的 i、 v 最终值，⽽不是各个goroutine启动时的i, v值。**可以理解为闭包引⽤，使⽤的是上下⽂环境的值**。

```go
func main() {
	var m = [...]int{1, 2, 3}
	for i, v := range m {
		go func() {
			fmt.Println(i, v)
		}()
	}
	/*
	2 3
	2 3
	2 3
	*/
	time.Sleep(time.Second * 3)
}

/*优化*/
// 方案1. 使⽤函数传递：
for i, v := range m {
	go func(i,v int) {
		fmt.Println(i, v)
	}(i,v)
}
// 方案2. 使⽤临时变量保留当前值：
for i, v := range m {
	i := i // 这⾥的 := 会重新声明变量，⽽不是重⽤
	v := v
	go func() {
		fmt.Println(i, v)
	}()
}
```


正常结束，不会无限循环（循环次数在循环开始前就已经确定）
```go
func slice_add() {
	v := []int{1, 2, 3}
	for i := range v {
		v = append(v, i)
	}
	// [1 2 3 0 1 2]
}
```

关于map动态增删的影响
```go
func map_del() {
	var m = map[string]int{
		"A": 21,
		"B": 22,
		"C": 23,
	}
	counter := 0
	for k, v := range m {
		if counter == 0 {
			delete(m, "A")
		}
		counter++
		fmt.Println(k, v)
	}
	// for range map 是⽆序的，如果第⼀次循环到 A，则输出 3；否则输出 2。
	fmt.Println("counter is ", counter)
}

func map_add() {
	var m = map[string]int{
		"A": 21,
		"B": 22,
		"C": 23,
	}
	counter := 0
	for k, v := range m {
		if counter == 0 {
			m["D"] = 24
		}
		counter++
		fmt.Println(k, v)
	}
	// 4
	fmt.Println("counter is ", counter)
}
```
映射上的迭代顺序没有指定，也不能保证从一个迭代到下一个迭代是相同的。如果在迭代过程中删除了尚未到达的映射条目，则不会产生相应的迭代值。如果在迭代过程中创建了映射条目，那么该条目可能在迭代过程中生成，也可能被跳过。对于创建的每个条目以及从一个迭代到下一个迭代，选择可能会有所不同。如果映射为nil，则迭代次数为0

> The iteration order over maps is not specified and is not guaranteed to be the same from one iteration to the next. If a map entry that has not yet been reached is removed during iteration, the corresponding iteration value will not be produced. If a map entry is created during iteration, that entry may be produced during the iteration or may be skipped. The choice may vary for each entry created and from one iteration to the next. If the map is nil, the number of iterations is 0.

参考 [http://weekly.golang.org/ref/spec#For_statements](https://go.dev/ref/spec#For_statements)  - For statements with range clause



- 注意输出位置

```go
func main() {
  x := []string{"a", "b", "c"}
  for v := range x {
  	fmt.Print(v) // 0 1 2 
  }
  for _, v := range x {
  	fmt.Print(v) // a b c
  }
}
```



#### select:

select {} : 阻塞

for {}: 线程独占cpu



#### append:

- `append()` 的第⼆个参数不能直接使⽤ slice ，需使⽤ `...` 操作符，将⼀个切⽚追加到另⼀个切⽚上： `append(s1, s2...)` 。或者直接跟上元素，形如： `append(s1, 1, 2, 3)`

```go
func main() {
  s1 := []int{1, 2, 3}
  s2 := []int{4, 5}
  // panic
  s1 = append(s1, s2)
//  s1 = append(s1, s2...)
  fmt.Println(s1)
}
```

#### defer

- defer 关键字后⾯的函数或者⽅法想要执⾏必须**先注册**， return 之后的 defer 是不能注册的， 也就不能执⾏后⾯的函数或⽅法。
- 函数的参数在执⾏ defer 语句的时候会保存⼀份**副本**，在实际调⽤函数时使⽤

```go
// 1
func df1() (r int) {
	// 类似闭包 r 作为全局变量 内部传的是指针
	defer func() {
		r++
	}()
	return 0
}

// 0
func df2() (r int) {
	// defer 中 r是局部变量
	defer func(r int) {
		r++
	}(r)
	return r
}

// 5
func df3() int {
	t := 5
	// 闭包: 相当于 返回值 a = t, t = t+5 , return a
	defer func() {
		t = t + 5
	}()
	return t
}

// 10
func df4() (r int) {
	t := 5
	// 闭包 返回值 r = r, r = t+5 , return r
	defer func() {
		r = t + 5
	}()
	return r
}

// 0
func df5() (r int) {
	t := 5
	defer func(r int) {
		// 局部变量 不影响外部 r
		r = t + 5
	}(t)
	return r // 返回值 r =0， return r
}

// 1
func df6() (r int) {
	defer func(r int) {
		// 局部变量 不影响外部 r
		r = r + 5
	}(r)
	return 1 // 返回值 r =1， return r
}

// 15
func df6() (r int) {
	i := 5
	
	defer func(i int) {
		// 局部变量 
		i++
		fmt.Printf("def i: %d\n", i) // 6
	}(i)

	i = i + 10
	// 返回值 r = i = 15, return r
	return i // 15
}
```

区别函数和闭包
```go
type Person struct {
	age int
}

func F(n int) func() int {
	return func() int {
		n++
		return n
	}
}

func f1() {
	person := &Person{28}
	fmt.Printf("28 person addr： %p\n", person)
	// 1. 这里虽然 defer 是在 return 前执行，但是在定义得时候， 已经将defer要执行得函数压入栈中，所以传递给 fmt打印的是 person.age 的值，
	// 即 person.age 此时是将 28 当做 defer 函数的参数，会把 28 缓存在栈中，等到最后执⾏该 defer 语句的时候取出，即输出 28；
	defer fmt.Printf("111 ptr: %p\t%d\n", person, person.age)

	// 2. 这里是闭包， 将 func 压入栈中，最后通过指针访问变量（变量相对于该闭包来说相当于全局变量）
	// 情况一 defer 缓存的是结构体 Person{28} 的地址，最终 Person{28} 的 age 被重新赋值为 29
	// 情况二 defer 缓存的是结构体 Person{28} 的地址，这个地址指向的结构体没有被改变
	defer func(p *Person) {
		fmt.Printf("222 ptr: %p\t%d\n", p, p.age)
	}(person)

	// 3. 闭包引⽤
	// 情况二 person 的值已经被改变，指向结构体 Person{29}
	defer func() {
		fmt.Printf("333 ptr: %p\t%d\n", person, person.age)
	}()

	// 情况一 29 29 28
	// person.age = 29 // 变量没有变化
	// fmt.Printf("29 person addr： %p\n", person)

	// 情况二 29 28 28 
	/*
		28 person addr： 0xc0000160d0
		29 person addr： 0xc0000160d8
		333 ptr: 0xc0000160d8	29
		222 ptr: 0xc0000160d0	28
		111 ptr: 0xc0000160d0	28
	*/
	person = &Person{29} // 重新创建了一个变量
	fmt.Printf("29 person addr： %p\n", person)
}

func f2() {
	f := F(5)

	// 此时是将 f() 当做 defer 函数的参数，会把 f() 缓存在栈中，等到最后执⾏该 defer 语句的时候取出，即输出
	defer fmt.Println(f()) // 6
	
	// 在执⾏ defer 语句的时候会保存⼀份副本，在实际调⽤ func() 函数时⽤
	defer func() {
		fmt.Println(f()) // 9 
	}()

	defer fmt.Println(f()) // 7 

	i := f()
	fmt.Println(i) // 8 
}
```

注意点:

```go
// increase(1) > 2
// 先将d=1 ，然后 return ret= d = 1; 再计算defer改变ret，最后得到返回值ret++ => 2
func increase(d int) (ret int) {
    defer func() {
        ret++
    }()
    return d
}

// 1
/* 先将0 赋值给返回值result, 再执行defer, 改变返回值result,最后函数返回 */
func f() (result int) {
    defer func() {
        result++

    }()
    return 0

}

// 5
/* 先将t 复制给返回值r，再运行defer改变t，但是没有影响到r，所以不改变*/
func r() (r int) {
    t := 5
    defer func() {
        t = t + 5

    }()
    return t

}

// 10  
/*   区别上一例子*/
func rr() (r int) {
    t := 5
    defer func() {
        //        r = r + 5
        r = t + 5

    }()
    return t

}

// 1
/* defer内的r 和函数返回r只是同名，但是属于局部变量，不会影响到整体返回值*/
func s() (r int) {
    defer func(r int) {
        r = r + 5
    }(r)
    return 1
}

或者

func s() (r int) {
    defer func(a int) {
        a= a + 5
        a= r + 5  
    }(r)
    return 1
}

//6
/*区别上一例子*/
func s() (r int) {
    defer func(a int) {
        r = r + 5  // 改变返回值
    }(r)
    return 1
}


/**********************************************************************************************/
type message struct {
    content string
}

func (p *message) set(c string) {
    p.content = c
}

func (p *message) print() string {
    return p.content
}

// 输出为 world hello
/*在 defer 中， fmt.Print 被推迟到函数返回后执行，可是 m.print() 的值在当时就已经被求出，因此， m.print() 会返回 "Hello" ，这个值会保存一直到外围函数返回*/
func defer_print() {
    m := &message{content: "Hello"}
    defer fmt.Println(m.print())
    m.set("World")
    fmt.Println(m.content) // 打印world
    /* 
        defer fmt.Print(m.print()) 
        被延迟的函数在这里执行
    */
}

/**********************************************************************************************/
// 当代码执行的时候，被延迟的函数会查看当时 i 的值，这是因为，当 defer 出现的时候， Go 的运行时会保存 i 的地址，在 for循环结束之后 i 的值变成了 3。 因此，当延迟语句运行的时候，所有的延迟栈中的语句都会去查看 i 地址中的值，也就是 3
//（被当做循环结束后的当时值）。

func defer_for() {
    for i := 0; i < 3; i++ {
        defer func() {
            fmt.Println(i)
        }()
    }
}

// 解决方案
func result_for() {
    for i := 0; i < 3; i++ {
        defer func(i int) {
            fmt.Println(i)

        }(i)

    }

    for i := 0; i < 3; i++ {
        i := i
        defer func() {
            fmt.Println(i)

        }()

    }

    for i := 0; i < 3; i++ {
        defer fmt.Println(i)

    }
}

/**********************************************************************************************/
// 延迟调用含有闭包的函数

type database struct{}

func (db *database) connect() (disconnect func()) {
    fmt.Println("connect")

    return func() {
        fmt.Println("disconnect")
    }
}

func connect_test() {
    db := &database{}
    defer db.connect()

    fmt.Println("query db ..")
}

/* 输出结果
query db...
connect
*/

// 解决方案
// db.connect() 返回了一个函数，然后我们再对这个函数使用 defer 接收闭包函数
func result_connect_test() {
    db := &database{}
    close := db.connect()
    defer close()

    fmt.Println("query db...")
    /*
        或者 (不建议采用)
        db := &database{}
        defer db.connect()()
        ...
    */
}

/**********************************************************************************************/
// 没有使用指针作为接收者

type Car struct {
    model string
}

func (c Car) PrintModel() {
    fmt.Println("非指针:", c.model)

}

func (c *Car) PrintModel_pointer() {
    fmt.Println("指针:", c.model)

}

// 外围函数还没有返回的时候，Go 的运行时就会立刻将传递给延迟函数的参数保存起来
// 1 发生一次拷贝，所以后面改变不会影响结果，2 使用指针，指向同一地址
func pointer_defer() {
    c := Car{model: "DeLorean DMC-12"}
    // 1
    defer c.PrintModel()
    c.model = "Chevrolet Impala"
    // 2
    defer c.PrintModel_pointer()
    c.model = "Chevrolet Impala"
}
```

#### recover：

- recover() 必须在 defer() 函数中直接调⽤才有效。
- 下⾯其他⼏种情况调⽤都是⽆效的：
  - 直接调⽤recover()
  - 在 defer() 中直接调⽤ recover() 
  - defer() 调⽤时多层嵌套

```go
package main

import (
	"fmt"
)

func f(n int) (r int) {
	// 3. r = 7, n =3
	defer func() {
		r += n
		recover() // 接住了 defer f() -> nil 的panic
	}()
	// f = nil
	var f func()
	// 2. def panic 此时入栈的是nil 下面赋值也没影响,执行得时候发啥panic
	defer f()
	// 没有被调用
	f = func() {
		r += 2
	}
	// 1. r= 4, n = 3
	return n + 1
}
func main() {
	// 7
	fmt.Println(f(3))
}
```

#### interface
`iface`包含两个字段：`tab` 是接口表指针，指向类型信息；`data` 是数据指针，则指向具体的数据。它们分别被称为动态类型和动态值。而接口值包括动态类型和动态值

当且仅当接⼝的动态值和动态类型都为 nil 时，接⼝类型值才为 nil
```go
func main() {
	var i interface{}
	// nil
	if i == nil {
		fmt.Println("nil")
	return
	}
	fmt.Println("not nil")
}
```

**⽅法集**：
```go
type Integer int
func (a Integer) Add(b Integer) Integer {
	return a + b
}

func (a *Integer) Add(b Integer) Integer {
	return *a + b
}

func main() {
	var a Integer = 1
	var b Integer = 2
	var i interface{} = &a
	sum := i.(*Integer).Add(b)
	fmt.Println(sum)
}
```

**多态**：
```go
package main
import "fmt"

// 定义接口列表
type Person interface {
    job()
    growUp()
}

// 接口函数
func whatJob(p Person) {
    p.job()
}
func growUp(p Person) {
    p.growUp()
}

// 实现接口的实例
type Student struct {
    age int
}
func (p Student) job() {
    fmt.Println("I am a student.")
    return
}
func (p *Student) growUp() {
    p.age += 1
    return
}
type Programmer struct {
    age int
}
func (p Programmer) job() {
    fmt.Println("I am a programmer.")
    return
}
func (p Programmer) growUp() {
    // 程序员老得太快 ^_^
    p.age += 10
    return
}

func main() {
    qcrao := Student{age: 18}
    whatJob(&qcrao)
    growUp(&qcrao)
    fmt.Println(qcrao)
    stefno := Programmer{age: 100}
    whatJob(stefno)
    growUp(stefno)
    fmt.Println(stefno)
}
```

**接口转换**：
当判定一种类型是否满足某个接口时，Go 使用类型的方法集和接口所需要的方法集进行匹配，如果类型的方法集完全包含接口的方法集，则可认为该类型实现了该接口。

例如某类型有 m 个方法，某接口有 n 个方法，则很容易知道这种判定的时间复杂度为 O(mn)，Go 会对方法集的函数按照函数名的字典序进行排序，所以实际的时间复杂度为 O(m+n)



#### channel：

- 通道相当于一个先进先出（FIFO）的**队列**

- 通道类型是**引用类型**，所以它的零值就是`nil`。换句话说，当只声明该类型的变量但**没有用`make`函数对它进行初始化时，该变量的值就会是`nil`**。

- **元素值从外界进入通道时会被复制**，发送操作包括了 **"复制元素值" 和 "放置副本到通道内部" 这两个步骤。在这两个步骤完全完成之前，发起这个发送操作的那句代码会 一直阻塞**在那里。

- 元素值从通道进入外界时会被移动。这个移动操作实际上包含了两步，第一步是生成正在通道中的这个元素值的副本，并准备给到接收方，第二步是删除在通道中的这个元素值；接收操作通常包含了 **"复制通道内的元素值"、"放置副本到接收方"、"删掉原值"** 三个步骤。在所有这些步骤完全完成之前，发起该操作的代码也会 **一直阻塞**。

- **缓冲通道**则在用**异步**的方式传递数据。

- **非缓冲通道** 无论是发送操作还是接收操作，一开始执行就会被阻塞，直到配对的操作也开始执行，才会继续传递。是在用**同步**的方式传递数据。也就是说，只有收发双方对接上了，数据才会被传递。

- **对于值为`nil`的通道，不论它的具体类型是什么，对它的发送操作和接收操作都会永久地处于阻塞状态**。它们所属的 goroutine 中的任何代码，都不再会被执行。

- 已初始化并未关闭的通道收发操作一定不会引发 `panic`，通道一旦**关闭**，再对它进行**发送**操作，就会引发 `panic`。

- **关闭一个已经关闭**了的通道，也会引发 panic；接收表达式的结果同时赋给两个变量时，第二个变量的类型就是一定`bool`类型。它的值如果为`false`就说明通道已经关闭，并且再没有元素值可取了。如果里面还有元素值未被取出，那么接收表达式的第一个结果，仍会是通道中的某一个元素值，而第二个结果值一定会是`true`。所以**要通过接收表达式的第二个结果值来判断通道是否已经关闭来避免使用过程中通道关闭造成的异常问题**。

- 有⽅向的 channel 不可以被关闭。

  

  汇总即：

  1. 给⼀个 nil channel 发送数据，造成永远阻塞
  2. 从⼀个 nil channel 接收数据，造成永远阻塞
  3.  给⼀个已经关闭的 channel 发送数据，引起 panic
  4.  从⼀个已经关闭的 channel 接收数据，如果缓冲区中为空，则返回⼀个零值

  

  举例1 **阻塞**：

  ```go
  // intChan2 = make(chan int, 3)
  for elem := range intChan2 {
  	fmt.Printf("The element in intChan2: %v\n", elem)
  }
  ```

  - `for`语句会不断地尝试从`intChan2`种取出元素值，即使`intChan2`被关闭，它也会在取出所有剩余的元素值之后再结束执行。
  - 当`intChan2`中没有元素值时，它会被阻塞在有`for`关键字的那一行，直到有新的元素值可取。
  - 假设`intChan2`的值为`nil`，那么它会被永远阻塞在有`for`关键字的那一行。

  

  举例2 **随机遍历**：

  ```go
  intChannels := [3]chan int{
  	make(chan int, 1),
  	make(chan int, 1),
  	make(chan int, 1),
  }
  
  select {
    case <-intChannels[0]:
  		fmt.Println("The first candidate case is selected.")
  	case <-intChannels[1]:
      fmt.Println("The second candidate case is selected.")
    case elem := <-intChannels[2]:
      fmt.Printf("The third candidate case is selected, the element is %d.\n", elem)
    default:
      fmt.Println("No candidate case is selected!")
  }
  ```

  - 如果包含 `default` 分支，那么`select`语句永远都不会被阻塞。其他情况没有满足，会走默认分支。
  - 如果没有 `default` 分支，一旦所有的`case`表达式都没有满足求值条件，那么`select`语句就会被阻塞。直到至少有一个`case`表达式满足条件为止。
  - `select`语句只能对其中的每一个`case`表达式各求值**一次**。想**连续或定时**地操作其中的通道的话，就往往需要通过在`for`语句中嵌入`select`语句的方式实现。但这时要注意，简单地在`select`语句的分支中使用`break`语句，只能结束当前的`select`语句的执行，而并不会对外层的`for`语句产生作用。这种错误的用法可能会让这个`for`语句无休止地运行下去。

  举例3: **不能关闭有方向的chan**

  ```go
  // invalid operation: close(stop) (cannot close receive-only channel)
  func Stop(stop <-chan bool) {
  	close(stop)
  }
  ```

  

#### 函数：

- 传入函数的是一个值类型的参数值时，如果这个参数值中的**某个元素是引用类型**的，那么要小心

```go
package main

import "fmt"

type A struct {
	a [3]string
}

type B struct {
	a []string
}

// 数组值传递 函数内部只是拷贝的副本，不会对原数据产生影响
func modifyArray_value(a [3]string) [3]string {
	fmt.Printf("The array value: %p\n", &a)
	a[1] = "x"
	return a
}
// 切片值传递 结果改变是因为slice是引用类型而非值类型，a中保存的是slice的指针，所以即使创建了副本(值传递)，a中仍然保存的是指针；并发时panic
func modifySlice_value(a []string) []string {
	fmt.Printf("The slice value: %p\n", &a)
	a[1] = "x"
	return a
}
// 切片指针传递
func modifySlice_Ptr(a *[]string) *[]string {
	fmt.Printf("The slice ptr: %p\n", a)
	(*a)[1] = "x"
	return a
}

/*
The original array: 0xc000098180: {[d e f]}
The original slice value: 0xc0000a6018: {[d e f]}
The original slice ptr: 0xc0000a6030: {[d e f]}
The array value: 0xc000098270
The slice value: 0xc0000a6090
The slice ptr: 0xc0000a6030
The modified array: 0xc000098240: {[d e f]}
The modified slice value: 0xc0000a6078: [d x f]
The modified slice ptr: 0xc0000a6030: &[d x f]
*/
func main() {
	array := A{a: [3]string{"d", "e", "f"}}
	slice := B{a: []string{"d", "e", "f"}}
	slicePtr := B{a: []string{"d", "e", "f"}}

	fmt.Printf("The original array: %p: %v\n", &array, array)
	fmt.Printf("The original slice value: %p: %v\n", &slice, slice)
	fmt.Printf("The original slice ptr: %p: %v\n", &slicePtr, slicePtr)

	array1 := modifyArray_value(array.a)
	slice1 := modifySlice_value(slice.a)
	slice2 := modifySlice_Ptr(&slicePtr.a)
	fmt.Printf("The modified array: %p: %v\n", &array1, array)
	fmt.Printf("The modified slice value: %p: %v\n", &slice1, slice1)
	fmt.Printf("The modified slice ptr: %p: %v\n", slice2, slice2)
}
```

```go
package main

import (
    "fmt"
)

func hello() []string {
    fmt.Println("aaaaaaaaaa")
    return nil
}
func main() {
    h := hello
    // 是将 hello() 赋值给变量h，⽽不是函数的返回值，所以输出 not nil
    if h == nil {
    	fmt.Println("nil")
    } else {
    	// not nil
    	fmt.Println("not nil")
    }
    
    // aaaaaaaaaa
    h()
}
```



#### 闭包：

- 闭包引⽤相同变量

```go
// x 相当于两个闭包函数的全局变量
func test(x int) (func(), func()) {
  return func() {
    println(x)
    x += 10
  }, func() {
  	println(x)
  }
}
func main() {
  a, b := test(100)
  a()	// 100
  b()	// 110
}
```

闭包延迟求值

```go
func test() []func() {
    var funs []func()
  	//  for 循环局部变量 i，匿名函数每⼀次使⽤的都是同⼀个变量
    for i := 0; i < 2; i++ {
        funs = append(funs, func() {
            println(&i, i)
        })
    }
    return funs
}

// 0xc000064000 2
// 0xc000064000 2
func main() {
    funs := test()
    for _, f := range funs {
      f()
    }
}
```



#### 方法：

```go
package main

import "fmt"

// AnimalCategory 代表动物分类学中的基本分类法。
type AnimalCategory struct {
	kingdom string // 界。
	phylum string // 门。
	class  string // 纲。
	order  string // 目。
	family string // 科。
	genus  string // 属。
	species string // 种。
}

func (ac AnimalCategory) String() string {
	fmt.Printf("AnimalCategory string func\n")
	return fmt.Sprintf("%s%s%s%s%s%s%s",ac.kingdom, ac.phylum, ac.class, ac.order,ac.family, ac.genus, ac.species)
}

/*
AnimalCategory string func
The animal category: cat
*/
func main() {
	category := AnimalCategory{species: "cat"}
	fmt.Printf("The animal category: %s\n", category)
}
```

在调用`fmt.Printf`函数时，使用占位符`%s`和`category`值本身就可以打印出后者的字符串表示形式，而无需显式地调用它的`String`方法。

同一个方法集合中的方法不能出现重名。并且，如果它们所属的是一个结构体类型，那么它们的名称与该类型中任何字段的名称也不能重复。

只要名称相同，无论这两个方法的签名是否一致，被嵌入类型的方法都会“屏蔽”掉嵌入字段的同名方法。

```go
type People interface {
	Speak(string) string
}

type Student struct{}
// 值类型 Student 没有实现接⼝的 Speak() ⽅法，⽽是指针类型 *Student 实现该⽅法。
func (stu *Student) Speak(think string) (talk string) {
  if think == "speak" {
  	talk = "speak"
  } else {
  	talk = "hi"
  }
  return
}
func main() {
  var peo People = Student{}
  // 修改方法 取指针
  // var peo People = &Student{}
  think := "speak"
  // 编译错误：Student does not implement People (Speak method has pointer receiver)
  fmt.Println(peo.Speak(think))
}
```

通过类型引⽤的⽅法表达式会被还原成普通函数样式，接收者是第⼀个参数，调⽤时显示传参。类型可以是 T 或 *T，只要⽬标⽅法存在于该类型的⽅法集中就可以
```go
type N int

func (n N) test(){
	fmt.Println(n)
}
func main() {
	var n N = 10
	fmt.Println(n)	// 10
	
	n++
	f1 := N.test
	f1(n)		// 11
	
	n++
	f2 := (*N).test
	f2(&n)		// 12
	
	n++
	N.test(n)	// 13
	
	n++
	(*N).test(&n)	// 14
}
```

指针传递和值传递：
```go
package main

import (
   "fmt"
)

type N int
// 当指针值赋值给变量或者作为函数参数传递时，会⽴即计算并复制该⽅法执⾏所需的接收者对象，与其绑定，以便在稍后执⾏时，能隐式第传⼊接收者参数。
func (n N) test(){
	fmt.Printf("%p-%v\n", &n, n)
}

// 13 11 12 
func test_1() {
	var n N = 10
	p := &n
	fmt.Printf("%p-%v\n", &n, n) // 0xc42000e228-10

	n++
	f1 := n.test
	fmt.Printf("%p-%v\n", &n, n) // 0xc42000e228-11

	n++
	f2 := p.test
	fmt.Printf("%p-%v\n", &n, n) // 0xc42000e228-12

	n++
	fmt.Printf("%p-%v\n", &n, n) // 0xc42000e228-13

	// 0xc42000e280-11
	f1()
	// 0xc42000e290-12
	f2()
}

// 当⽬标⽅法的接收者是指针类型时，那么被复制的就是指针。
func (n *N) test2(){
	fmt.Println(*n)
}
func test_2() {
	var n N = 10
	p := &n
	n++
	f1 := n.test		// 13
	n++
	f2 := p.test		// 13
	n++
	fmt.Println(n)		// 13
	f1()
	f2()
}

func main() {
	// 值传递
	test_1() // 13 11 12
	// 指针传递
	test_2() // 13 13 13
}
```

#### 参数：

- 函数参数为 interface{} 时可以接收任何类型的参数，包括⽤户⾃定义类型等，即使是接收指针类型也⽤interface{}，⽽不是使⽤ *interface{}。

```go
type S struct {
}
func f(x interface{}) {
	fmt.Println(x)
}
func g(x *interface{}) {
	fmt.Println(x)
}
func main() {
	s := S{}
	p := &s
	f(s) // 正确 {}
	g(s) // 错误 cannot use s (type S) as type *interface {} in function argument:*interface {} is pointer to interface, not interface
	f(p) // 正确 &{}
	g(p) // 错误 cannot use p (type *S) as type *interface {} in function argument:*interface {} is pointer to interface, not interface
}
```

#### 返回值：

- 在函数有多个返回值时，只要有⼀个返回值有命名，其他的也必须命名。如果有多个返回值必须加上括号();如果只有⼀个返回值且命名也需要加上括号()

```go
// 第⼆个返回值没有命名
func funcMui(x, y int) (sum int, error) {
	return x + y, nil
}

// 返回时不指明返回变量（也可正确返回）
func Get() (value interface{}, ok bool) {
  	// <nil> false
	return
}
```

- nil 可以⽤作 interface、 function、 pointer、 map、 slice 和 channel 的“空值”; `nil` 不能作为`string`类型的默认返回值。

#### cap():

- cap() 函数适⽤于数组、数组指针、 slice 和 channel，不适⽤于 map，可以使⽤ len() 返回 map 的元素个数。
  - array 返回数组的元素个数
  - slice 返回slice的最⼤容量
  - channel 返回 channel的容量


#### sizeof()

返回类型 x 所占据的字节数, 但是却不包括x所引用的内存. 例如: 对于一个指针，函数返回的大小为8字节（64位机上），一个slice的大小则为slice header的大小

```go
func main() {
	var a int32 = 10
	t := unsafe.Sizeof(a)
	// 4 
	fmt.Println(t)	

	var b string = "test"
	// 16 Sizeof(string)占计算string header的大小, struct string { uint8 *str; int len;}
	fmt.Println(unsafe.Sizeof(b))

	type aa struct {
		B bool
		C uint64
	}
	c := aa{}
	// 16 内存对齐
	fmt.Println(unsafe.Sizeof(c))

	type bb struct {
		B *bool
	}
	c1 := bb{}
	// 8 指针占8个字节
	fmt.Println(unsafe.Sizeof(c1))
}
```


#### 指针 和 unsafe.Pointer
##### 指针 `pointer `
- Go 的指针不能进行数学运算(相对于c)
- 不同类型的指针不能相互转换
- 不同类型的指针不能使用 `==` 或 `!=` 比较
- 不同类型的指针变量不能相互赋值
```go
func main() {
	a := 5
	p := &a
	p++	// 不能对指针做数学运算

	i := int(100)
	var f *float64
	f = &i	// 不同类型不能转换、复制
	
	arr := [4]int{1, 2, 3, 4}
	// ptr := &arr
	// ptr++ // 不能对指针做数学运算

	// 利用转换进行偏移运算
	ptr := (*int)(unsafe.Pointer(&arr))
	fmt.Printf("arr = %p\tp = %p\n", &arr, ptr)

	// 2 3
	fmt.Println(*ptr+1, *ptr+2)
	
}
```

##### `unsafe.Pointer `
- 任何类型的指针和 `unsafe.Pointer` 可以相互转换。
- `uintptr` 类型和 `unsafe.Pointer` 可以相互转换。

<img src="./img/指针.png">


所以，pointer 不能直接进行数学运算，但可以把它转换成 uintptr，对 uintptr 类型进行数学运算，再转换成 pointer 类型
```go
// uintptr 是一个整数类型，它足够大，可以存储
type uintptr uintptr
```

`uintptr` 所指向的对象会被 `gc` 无情地回收。

`unsafe.Pointer` 有指针语义，可以保护它所指向的对象在**有用**的时候不会被垃圾回收


##### 利用 `unsafe.Pointer` 修改自定义类型成员值

- 通过 `unsafe.Offsetof()` 函数可以获取结构体成员的偏移量，进而获取成员的地址
- 通过 `unsafe.Sizeof()` 函数可以获取成员大小，进而计算出成员的地址，直接修改内存

```go
package main

import (
	"fmt"
	"unsafe"
)

type Programmer struct {
	name     string
	language string
	age      int
}

func main() {
	p := Programmer{"test", "go", 18}
	fmt.Println(p) // {test go 18}

	name := (*string)(unsafe.Pointer(&p))
	*name = "aaaa"
	// 利用地址偏移修改对应位成员内容
	lang := (*string)(unsafe.Pointer(uintptr(unsafe.Pointer(&p)) + unsafe.Offsetof(p.language)))
	*lang = "C++"
	fmt.Println(p) // {aaaa C++ 18}

	age := (*int)(unsafe.Pointer(uintptr(unsafe.Pointer(&p)) + unsafe.Sizeof(string("")) + unsafe.Sizeof(string(""))))
	*age = 20
	fmt.Println(p) // {aaaa C++ 20}
}
```

##### 零拷贝实现 `string` 和 `byte[]` 转换
利用类型底层结构进行转换

```go
type StringHeader struct {
    Data uintptr
    Len  int
}
type SliceHeader struct {
    Data uintptr
    Len  int
    Cap  int
}


func string2bytes(s string) []byte {
    stringHeader := (*reflect.StringHeader)(unsafe.Pointer(&s))
    bh := reflect.SliceHeader{
        Data: stringHeader.Data,
        Len:  stringHeader.Len,
        Cap:  stringHeader.Len,
    }
    return *(*[]byte)(unsafe.Pointer(&bh))
}
func bytes2string(b []byte) string{
    sliceHeader := (*reflect.SliceHeader)(unsafe.Pointer(&b))
    sh := reflect.StringHeader{
        Data: sliceHeader.Data,
        Len:  sliceHeader.Len,
    }
    return *(*string)(unsafe.Pointer(&sh))
}
```



#### 值接收 和 指针接收
##### 区别
实现了接收者是值类型的方法，相当于自动实现了接收者是指针类型的方法；而实现了接收者是指针类型的方法，不会自动生成对应接收者是值类型的方法

```go
package main
import "fmt"
type coder interface {
    code()
    debug()
}
type Gopher struct {
    language string
}
func (p Gopher) code() {
    fmt.Printf("I am coding %s language\n", p.language)
}
func (p *Gopher) debug() {
    fmt.Printf("I am debuging %s language\n", p.language)
}
func main() {
	var c coder = &Gopher{"Go"}
	c.code()	// I am coding Go language
	c.debug()	// I am debuging Go language


	var d coder = Gopher{"Go"} 
	/* command-line-arguments 
		cannot use Gopher literal (type Gopher) as type coder in assignment:
	 Gopher does not implement coder (debug method has pointer receiver)
	*/
	d.code()
	d.debug()
}
```
##### 应用场景
如果方法的接收者是值类型，无论调用者是对象还是对象指针，修改的都是对象的副本，不影响调用者；如果方法的接收者是指针类型，则调用者修改的是指针指向的对象本身。

使用指针作为方法的接收者的理由：
- 方法能够修改接收者指向的值。
- 避免在每次调用方法时复制该值，在值的类型为大型结构体时，这样做会更加高效。


#### atomic.Value 原子操作、yaml 加载、ip 检测

```yaml
# 测试配置文件
server:
    listen_addr: ":9090"
    allowed_networks: ["1.2.3.4/24"]
```

```go
/*
   从yaml文件中解析ip配置，然后检测任意给定ip 是否在范围内（实现 ACL 限制）
*/
package main

import (
	"fmt"
	"io/ioutil"
	"net"
	"os"
	"strings"
	"sync"
	"sync/atomic"

	yaml "gopkg.in/yaml.v2"
)

const entireIPv4 = "0.0.0.0/0"

var (
	clientip string = "10.2.3.4:9090"
)

type Networks []*net.IPNet

type Server struct {
	ListenAddr      string   `yaml:"listen_addr"`
	AllowedNetworks []string `yaml:"allowed_networks,omitempty"`

	// Catches all undefined fields and must be empty after parsing.
	XXX map[string]interface{} `yaml:",inline"`
}

type cfg struct {
	Server Server
}

func (n Networks) Contains(addr string) bool {
	if len(n) == 0 {
		panic(fmt.Sprintf("BUG: network list is 0!"))
		//return true
	}

	h, _, err := net.SplitHostPort(addr)
	if err != nil {
		panic(fmt.Sprintf("BUG: unexpected error while parsing RemoteAddr: %s", err))
	}

	ip := net.ParseIP(h)
	if ip == nil {
		panic(fmt.Sprintf("BUG: unexpected error while parsing IP: %s", h))
	}

	fmt.Printf("string: %v -> ip: %v\n", h, ip)
	for _, ipnet := range n {
		if ipnet.Contains(ip) {
			return true
		}
	}

	return false
}

func stringToIPnet(s string) (*net.IPNet, error) {
	if s == entireIPv4 {
		return nil, fmt.Errorf("suspicious mask specified \"0.0.0.0/0\". " +
			"If you want to allow all then just omit `allowed_networks` field")
	}
	ip := s
	if !strings.Contains(ip, `/`) {
		ip += "/32"
	}
	_, ipnet, err := net.ParseCIDR(ip)
	if err != nil {
		return nil, fmt.Errorf("wrong network group name or address %q: %s", s, err)
	}
	return ipnet, nil
}

// load yaml config file
func loadconfig(path string) (*Server, error) {
	var sy sync.Mutex
	content, err := ioutil.ReadFile(path)
	if err != nil {
		return nil, err
	}

	cfg := &cfg{}
	sy.Lock()
	defer sy.Unlock()

	if err := yaml.Unmarshal([]byte(content), cfg); err != nil {
		return nil, err
	}

	fmt.Printf("cfg: %+v\n", cfg)
	return &cfg.Server, nil
}

func main() {
	var an *Networks
	var allowedNetworksHTTPS atomic.Value
	var list Networks

	// load config
	cfg, err := loadconfig("./config.yaml")
	if err != nil {
		fmt.Println(err)
		os.Exit(1)
	}
	fmt.Printf("AllowedNetworks: %v\n", cfg.AllowedNetworks)

	for _, ips := range cfg.AllowedNetworks {
		ip, err := stringToIPnet(ips)
		if err != nil {
			continue
		}
		list = append(list, ip)
	}

  // store value
	allowedNetworksHTTPS.Store(&list)

	// load value
	an = allowedNetworksHTTPS.Load().(*Networks)

	// ip check
	if !an.Contains(clientip) {
		fmt.Printf("%s 不属于该网络\n", clientip)
		os.Exit(1)
	}
	fmt.Println("ip check success!!!")
}
```



#### string()方法:

- 如果类型实现 String() ⽅法，使⽤ Printf()、 Print() 、 Println() 、Sprintf() 等格式化输出时会⾃动使⽤ String() ⽅法; 在该类型的String() ⽅法内使⽤格式化输出，导致递归调⽤，最后抛错。

```go
type ConfigOne struct {
	Daemon string
}
func (c *ConfigOne) String() string {
	return fmt.Sprintf("print: %v", c)
}
func main() {
  // 如果是c := ConfigOne{} 则不会掉用string 方法，因为String()是指针⽅法，⽽不是值⽅法
	c := &ConfigOne{}
	// runtime: goroutine stack exceeds 1000000000-byte limit
	// fatal error: stack overflow
	c.String()
}
```

#### 静态类型 、动态类型：
##### 静态类型
Go 语言中，每个变量都有一个**静态类型**，在**编译阶段**就确定了的，比如 `int`, `float64`, `[]int` 等等。注意，**这个类型是声明时候的类型，不是底层数据类型**。

```go
type MyInt int

var i int
var j MyInt

var m interface{}	// 给 m 声明了 interface{} 类型，所以 i 的静态类型就是 interface{}

m = 18			// 变量 m 赋一个 int 类型的值时，它的静态类型还是 interface{}，这是不会变的，但是它的动态类型此时变成了 int 类型
m = "hello wold"	// 变量 m 赋一个 string 类型的值时，它的静态类型还是 interface{}，它还是不会变，但是它的动态类型此时又变成了 string 类型
```

尽管 `i`，`j` 的底层类型都是 `int`，但我他们是不同的静态类型，除非进行类型转换，否则，`i` 和 `j` **不能同时出现在等号两侧**。`j` 的静态类型就是 `MyInt`


##### 动态类型
动态类型（即 concrete type，也叫具体类型）是 程序**运行时**系统才能看见的类型，从源码里可以看到：iface包含两个字段：tab 是接口表指针，指向类型信息；data 是数据指针，则指向具体的数据。它们分别被称为**动态类型和动态值**。而接口值包括动态类型和动态值。

**接口类型和 nil 作比较：**
接口值的零值是指动态类型和动态值都为 nil。当仅且当这两部分的值都为 nil 的情况下，这个接口值就才会被认为 `接口值 == nil`。

```go
type Coder interface {
	code()
}

type Gopher struct {
	name string
}

func (g Gopher) code() {
	fmt.Printf("%s is coding\n", g.name)
}

func main() {
	var c Coder
	// true
	fmt.Println(c == nil)
	// c: <nil>, <nil>
	fmt.Printf("c: %T, %v\n", c, c)

	var g *Gopher
	// true 
	fmt.Println(g == nil)

	c = g
	// false
	fmt.Println(c == nil)
	// c: *main.Gopher, <nil>
	fmt.Printf("c: %T, %v\n", c, c)
}
```
一开始，c 的 动态类型和动态值都为 nil，g 也为 nil，当把 g 赋值给 c 后，c 的动态类型变成了 *main.Gopher，仅管 c 的动态值仍为 nil，但是当 c 和 nil 作比较的时候，结果就是 false 了


举例：
```go
package main

func main() {
	var x interface{}
	var y interface{} = []int{3, 5} // 动态类型发生改变
	fmt.Println(x == x) // true

	fmt.Println(x == y) // false
	fmt.Println(y == y) // panic: runtime error: comparing uncomparable type []int
}
```

```go
var r io.Reader	// 声明 r 的类型是 io.Reader，注意，这是 r 的静态类型，此时它的动态类型为 nil，并且它的动态值也是 nil。
tty, err := os.OpenFile("/Users/qcrao/Desktop/test", os.O_RDWR, 0)
if err != nil {
    return nil, err
}
r = tty	// 将 r 的动态类型变成 *os.File，动态值则变成非空，表示打开的文件对象。这时，r 可以用<value, type>对来表示为： <tty, *os.File>
```

<img src="./img/动态类型.png">

虽然 `fun` 所指向的函数只有一个 `Read` 函数，其实 `*os.File` 还包含 `Write` 函数，也就是说 `*os.File` 其实还实现了 `io.Writer` 接口。因此下面的断言语句可以执行

```go
var w io.Writer
w = r.(io.Writer)
```

之所以用断言，而**不能直接赋值**，是因为 `r` 的静态类型是 `io.Reader`，并**没有实现** `io.Writer` 接口。断言能否成功，看 `r` 的动态类型是否符合要求; 

这样，`w` 也可以表示成 `<tty, *os.File>`，仅管它和 `r` 一样，但是 `w` 可调用的函数取决于它的静态类型 `io.Writer`，也就是说它只能有这样的调用形式： `w.Write()` 。`w` 的内存形式如下图：

<img src="./img/动态类型2.png">

和 r 相比，仅仅是 fun 对应的函数变了：Read -> Write。

> 对于空接口,所有的类型都实现了它，可以直接赋给它，不需要执行断言操作


#### Go 代码断⾏规则：

Go语法是使用（英文）分号`;`做为结尾标识符的。 但是，我们很少在Go代码中使用和看到分号。为什么呢？原因是大多数分号都是可选的，因此它们常常被省略。 在编译时刻，Go编译器会自动插入这些省略的分号。

自动插入分号的规则：

1. 在Go代码中，注释除外，如果一个代码行的最后一个语法词段（token）为下列所示之一，则一个分号将自动插入在此字段后（即行尾）：
   - 一个[标识符](https://gfw.go101.org/article/keywords-and-identifiers.html#identifier)；
   - 一个整数、浮点数、虚部、码点或者字符串[字面量](https://gfw.go101.org/article/basic-types-and-value-literals.html#basic-value-literals)；
   - 这几个跳转关键字之一：`break`、`continue`、`fallthrough`和`return`；
   - 自增运算符`++`或者自减运算符`--`；
   - 一个右括号：`)`、`]`或`}`。
2. 为了允许一条复杂语句完全显示在一个代码行中，分号可能被插入在一个右小括号`)`或者右大括号`}`之前。

```go
func alwaysFalse() bool {
	return false
}

// true
func main() {
  switch alwaysFalse()
  // 注意这个大括号位置
  {
  case true:
  	println(true)
  case false:
  	println(false)
  }
  
/*
	等价于
	switch alwaysFalse(); true {
	case true: fmt.Println("true")
	case false: fmt.Println("false")
	}
*/  
}
```



#### yaml、 json 解析

![7A4AAD19-8A7A-4B15-B84D-5E5D940E2C6F](https://user-images.githubusercontent.com/32731294/156387936-6aecd359-449e-4ff3-9a06-22dc45eb1cc3.png)


Config.yaml

```yam
# 测试配置文件
server:
  listen_addr: ":9090"
  allowed_networks: ["1.2.3.4/24"]
```

Main.go

```go
type Server struct {
  // 首字母大写  `` 里指明配置文件格式以及对应的名称
	ListenAddr      string   `yaml:"listen_addr"`
	AllowedNetworks []string `yaml:"allowed_networks,omitempty"`

	XXX map[string]interface{} `yaml:",inline"`
}

// 字段名称首字母必须大写，并且和配置文件中名称保持一致（不能使用其它名称或者小写格式server）
// 结构体访问控制，⾸字⺟如果是⼩写，导致其他包不能访问
type cfg struct {
	Server Server
}

func load(path string) (*Server, error) {
	content, err := ioutil.ReadFile(path)
	if err != nil {
		return nil, err
	}
	cfg := &cfg{}
	if err := yaml.Unmarshal([]byte(content), cfg); err != nil {
		return nil, err
	}   
	return &cfg.Server, nil
}
```

#### 反射 reflect
```go
var x float64 = 3.4
v := reflect.ValueOf(x)
v.SetFloat(7.1) // Error: will panic.
```
反射变量 v 不能代表 x 本身，为什么？因为调用 `reflect.ValueOf(x)` 这一行代码的时候，传入的参数在函数内部只是一个拷贝，是值传递，所以 v 代表的只是 x 的一个拷贝，因此对 v 进行操作是被禁止的。


可以使用 `canset` 进行判断
```go
var x float64 = 3.4
p := reflect.ValueOf(&x)
fmt.Println("type of p:", p.Type())		// type of p: *float64
fmt.Println("settability of p:", p.CanSet())	// settability of p: false
```

p 还不是代表 x，`p.Elem()` 才真正代表 x，这样就可以真正操作 x 了
```go
v := p.Elem()
v.SetFloat(7.1)
fmt.Println(v.Interface())	// 7.1
fmt.Println(x)			// 7.1
```

> 想要操作原变量，反射变量 Value 必须要 hold 住原变量的地址才行

##### 使用场景
- 不能明确接口调用哪个函数，需要根据传入的参数在运行时决定。
- 不能明确传入函数的参数类型，需要在运行时处理任意对象。

##### 利用反射比较两个对象是否相同
如果是不同的类型，即使是底层类型相同，相应的值也相同，那么两者也不是“深度”相等。

```go
func DeepEqual(x, y interface{}) bool
```

举例:
```go
type MyInt int
type YourInt int

func main() {
    m := MyInt(1)
    y := YourInt(1)
    fmt.Println(reflect.DeepEqual(m, y)) // false
}
```
m, y 底层都是 int，而且值都是 1，但是两者静态类型不同，前者是 MyInt，后者是 YourInt，因此两者不是“深度”相等。

| 类型 | 深度相等情形 |
|---|---|
|array|相同索引处的元素“深度”相等|
|Struct|相应字段，包含导出和不导出，“深度”相等|
|Func|只有两者都是 nil 时|
|Interface|两者存储的具体值“深度”相等|
|Map|1、都为 nil；2、非空、长度相等，指向同一个 map 实体对象，或者相应的 key 指向的 value “深度”相等|
|Pointer|1、使用 == 比较的结果相等；2、指向的实体“深度”相等|
|Slice|1、都为 nil；2、非空、长度相等，首元素指向同一个底层数组的相同元素，即 &x[0] == &y[0] 或者 相同索引处的元素“深度”相等|
|numbers, bools, strings, and channels|使用 == 比较的结果为真|



#### 泛型：
https://www.ardanlabs.com/blog/2020/07/generics-01-basic-syntax.html



#### 调度器:
https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part2.html




### 使用技巧举例：

1. 返回值

```go

/*
	不指定返回变量：
*/

package main

import "fmt"


func Get() (value interface{}, ok bool) {
	return
}

func main() {
    v, ok := Get()
    // <nil> false
    fmt.Println(v, ok)
}
```



2. goto 使用
```go
/*
	hello world!
	go to LOOP before!
	tag go to LOOP.
	LOOP end!!!
	see you later!!!
*/

package main

import (
	"fmt"
	"time"
)


func Run(ch <-chan bool) {
	fmt.Println("go to LOOP before!")
LOOP:
	for {
/*		select {
		case <- ch:
			fmt.Println("case go to LOOP.")
			break LOOP
		default:
		}*/

		if <- ch {
			fmt.Println("tag go to LOOP.")
			break LOOP
		} else {
			continue
		}
	}

	fmt.Println("LOOP end!!!")
}

func main() {
	fmt.Println("hello world!")
	ch := make(chan bool)
	go Run(ch)

	time.Sleep(time.Duration(time.Second) * 2)
	ch <- true

	time.Sleep(time.Duration(time.Second) * 4)
	fmt.Println("see you later!!!")
}
```
