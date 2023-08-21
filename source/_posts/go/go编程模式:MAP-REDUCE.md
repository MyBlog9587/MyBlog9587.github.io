---
title: go编程模式:MAP-REDUCE
date: 2023-06-27 22:59:05
tags:
- go
- private
categories:
- go
- MAP-REDUCE式编程
- MAP-REDUCE
---


函数式编程的中非常重要的 Map、Reduce、Filter 的三种操作，这三种操作可以非常方便灵活地进行一些数据处理

# demo 样例

## Map示例
```go
func MapStrToStr(arr []string, fn func(s string) string) []string {
    var newArray = []string{}
    for _, it := range arr {
        newArray = append(newArray, fn(it))
    }
    return newArray
}

func MapStrToInt(arr []string, fn func(s string) int) []int {
    var newArray = []int{}
    for _, it := range arr {
        newArray = append(newArray, fn(it))
    }
    return newArray
}
```

这两个函数需要两个参数:
- 一个是字符串数组 `[]string`，说明需要处理的数据一个字符串
- 另一个是一个函数 `func(s string) string` 或 `func(s string) int`

函数体都是在遍历第一个参数的数组，然后，调用第二个参数的函数，然后把其值组合成另一个数组返回

使用方式如下：

```go
var list = []string{"aaa", "cccc", "abcdefg"}

x := MapStrToStr(list, func(s string) string {
    return strings.ToUpper(s)
})
//["AAA", "CCCC", "ABCDEFG"]
fmt.Printf("%v\n", x)


y := MapStrToInt(list, func(s string) int {
    return len(s)
})
//[3, 4, 7]
fmt.Printf("%v\n", y)
```


## Reduce 示例

```go
func Reduce(arr []string, fn func(s string) int) int {
    sum := 0
    for _, it := range arr {
        sum += fn(it)
    }
    return sum
}

var list = []string{"aaa", "cccc", "abcdefg"}

x := Reduce(list, func(s string) int {
    return len(s)
})
// 14
fmt.Printf("%v\n", x)
```

## Filter示例
```go
func Filter(arr []int, fn func(n int) bool) []int {
    var newArray = []int{}
    for _, it := range arr {
        if fn(it) {
            newArray = append(newArray, it)
        }
    }
    return newArray
}

var intset = []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
out := Filter(intset, func(n int) bool {
   return n%2 == 1
})
fmt.Printf("%v\n", out)

out = Filter(intset, func(n int) bool {
    return n > 5
})
fmt.Printf("%v\n", out)
```


# 实践

```go
type Employee struct {
    Name     string
    Age      int
    Vacation int
    Salary   int
}

var list = []Employee{
    {"Hao", 44, 0, 8000},
    {"Bob", 34, 10, 5000},
    {"Alice", 23, 5, 9000},
    {"Jack", 26, 0, 4000},
    {"Tom", 48, 9, 7500},
    {"Marry", 29, 0, 6000},
    {"Mike", 32, 8, 4000},
}
```

对应的 Reduce/Fitler 函数

```go
// 满足条件的个数
func EmployeeCountIf(list []Employee, fn func(e *Employee) bool) int {
    count := 0
    for i, _ := range list {
        if fn(&list[i]) {
            count += 1
        }
    }
    return count
}

// 过滤后的列表
func EmployeeFilterIn(list []Employee, fn func(e *Employee) bool) []Employee {
    var newList []Employee
    for i, _ := range list {
        if fn(&list[i]) {
            newList = append(newList, list[i])
        }
    }
    return newList
}

// 满足条件的总和
func EmployeeSumIf(list []Employee, fn func(e *Employee) int) int {
    var sum = 0
    for i, _ := range list {
        sum += fn(&list[i])
    }
    return sum
}
```

## 各种统计示例
1. 统计有多少员工大于40岁
```go
old := EmployeeCountIf(list, func(e *Employee) bool {
    return e.Age > 40
})
fmt.Printf("old people: %d\n", old)
//old people: 2
```

2. 统计有多少员工薪水大于6000
```go
high_pay := EmployeeCountIf(list, func(e *Employee) bool {
    return e.Salary >= 6000
})
fmt.Printf("High Salary people: %d\n", high_pay)
//High Salary people: 4
```

3. 列出有没有休假的员工
```go
no_vacation := EmployeeFilterIn(list, func(e *Employee) bool {
    return e.Vacation == 0
})
fmt.Printf("People no vacation: %v\n", no_vacation)
//People no vacation: [{Hao 44 0 8000} {Jack 26 0 4000} {Marry 29 0 6000}]
```

4. 统计所有员工的薪资总和
```go
total_pay := EmployeeSumIf(list, func(e *Employee) int {
    return e.Salary
})
fmt.Printf("Total Salary: %d\n", total_pay)
//Total Salary: 43500
```

5. 统计30岁以下员工的薪资总和
```go
younger_pay := EmployeeSumIf(list, func(e *Employee) int {
    if e.Age < 30 {
        return e.Salary
    } 
    return 0
})
```

可以看到，上面的 `Map-Reduce` 都因为要处理数据的类型不同而需要写出不同版本的 `Map-Reduce`

## 通过 `interface{} + reflect` 实现通用的 `Map-Reduce`

```go
func Map(data interface{}, fn interface{}) []interface{} {
    vfn := reflect.ValueOf(fn)
    vdata := reflect.ValueOf(data)
    result := make([]interface{}, vdata.Len())

    for i := 0; i < vdata.Len(); i++ {
        result[i] = vfn.Call([]reflect.Value{vdata.Index(i)})[0].Interface()
    }
    return result
}
```

实现不同类型的数据可以使用相同逻辑的 Map()代码，对于这种用interface{} 的“过度泛型”，就需要自己来做类型检查 

```go
square := func(x int) int {
  return x * x
}
nums := []int{1, 2, 3, 4}

squared_arr := Map(nums,square)
//[1 4 9 16]
fmt.Println(squared_arr)



upcase := func(s string) string {
  return strings.ToUpper(s)
}
strs := []string{{"aaa", "cccc", "abcdefg"}
upstrs := Map(strs, upcase);
//["AAA", "CCCC", "ABCDEFG"]
fmt.Println(upstrs)
```

## 完善版 Map
```go
// Transform函数可以对slice中的元素进行操作，返回操作后的slice
func Transform(slice, function interface{}) interface{} {
  return transform(slice, function, false)
}

// TransformInPlace函数可以对slice中的元素进行操作，直接在原slice上进行修改
func TransformInPlace(slice, function interface{}) interface{} {
  return transform(slice, function, true)
}

// transform函数可以对slice中的元素进行操作，返回操作后的slice或直接在原slice上进行修改
func transform(slice, function interface{}, inPlace bool) interface{} {
 
  // 检查slice的类型是否为Slice
  sliceInType := reflect.ValueOf(slice)
  if sliceInType.Kind() != reflect.Slice {
    panic("transform: 不是slice类型")
  }

  // 检查函数的签名是否正确
  fn := reflect.ValueOf(function)
  elemType := sliceInType.Type().Elem()
  if !verifyFuncSignature(fn, elemType, nil) {
    panic("trasform: 函数必须是 func(" + sliceInType.Type().Elem().String() + ") outputElemType")
  }

  sliceOutType := sliceInType
  if !inPlace {
    sliceOutType = reflect.MakeSlice(reflect.SliceOf(fn.Type().Out(0)), sliceInType.Len(), sliceInType.Len())
  }
  for i := 0; i < sliceInType.Len(); i++ {
    sliceOutType.Index(i).Set(fn.Call([]reflect.Value{sliceInType.Index(i)})[0])
  }
  return sliceOutType.Interface()

}

// verifyFuncSignature函数用于检查函数签名是否正确
func verifyFuncSignature(fn reflect.Value, types ...reflect.Type) bool {

  // 检查是否为函数类型
  if fn.Kind() != reflect.Func {
    return false
  }
  // 检查输入参数和输出参数的数量是否正确
  if (fn.Type().NumIn() != len(types)-1) || (fn.Type().NumOut() != 1) {
    return false
  }
  // 检查输入参数的类型是否正确
  for i := 0; i < len(types)-1; i++ {
    if fn.Type().In(i) != types[i] {
      return false
    }
  }
  // 检查输出参数的类型是否正确
  outType := types[len(types)-1]
  if outType != nil && fn.Type().Out(0) != outType {
    return false
  }
  return true
}

```

1. 可以用于字符串数组
```go
list := []string{"1", "2", "3", "4", "5", "6"}
result := Transform(list, func(a string) string{
    return a +a +a
})
//{"111","222","333","444","555","666"}
```

2. 可以用于整形数组
```go
list := []int{1, 2, 3, 4, 5, 6, 7, 8, 9}
TransformInPlace(list, func (a int) int {
  return a*3
})
//{3, 6, 9, 12, 15, 18, 21, 24, 27}
```

3. 可以用于结构体
```go
var list = []Employee{
    {"Hao", 44, 0, 8000},
    {"Bob", 34, 10, 5000},
    {"Alice", 23, 5, 9000},
    {"Jack", 26, 0, 4000},
    {"Tom", 48, 9, 7500},
}
result := TransformInPlace(list, func(e Employee) Employee {
    e.Salary += 1000
    e.Age += 1
    return e
})
```

## 完善版 Reduce

```go
// Reduce函数可以对slice中的元素进行累加操作，返回累加结果
func Reduce(slice, pairFunc, zero interface{}) interface{} {
  sliceInType := reflect.ValueOf(slice)
  if sliceInType.Kind() != reflect.Slice {
    panic("reduce: 错误的类型，不是slice")
  }

  len := sliceInType.Len()
  if len == 0 {
    return zero
  } else if len == 1 {
    return sliceInType.Index(0)
  }

  elemType := sliceInType.Type().Elem()
  fn := reflect.ValueOf(pairFunc)
  if !verifyFuncSignature(fn, elemType, elemType, elemType) {
    t := elemType.String()
    panic("reduce: 函数必须是 func(" + t + ", " + t + ") " + t)
  }

  var ins [2]reflect.Value
  ins[0] = sliceInType.Index(0)
  ins[1] = sliceInType.Index(1)
  out := fn.Call(ins[:])[0]

  for i := 2; i < len; i++ {
    ins[0] = out
    ins[1] = sliceInType.Index(i)
    out = fn.Call(ins[:])[0]
  }
  return out.Interface()
}

```


## 完善版 Filter

```go
// Filter函数可以根据传入的函数对slice进行过滤，返回符合条件的元素组成的新slice
func Filter(slice, function interface{}) interface{} {
  result, _ := filter(slice, function, false)
  return result
}

// FilterInPlace函数可以根据传入的函数对slice进行过滤，将符合条件的元素保留在原slice中
func FilterInPlace(slicePtr, function interface{}) {
  in := reflect.ValueOf(slicePtr)
  if in.Kind() != reflect.Ptr {
    panic("FilterInPlace: wrong type, " +
      "not a pointer to slice")
  }
  _, n := filter(in.Elem().Interface(), function, true)
  in.Elem().SetLen(n)
}

// boolType是bool类型的反射值
var boolType = reflect.ValueOf(true).Type()

// filter函数是Filter和FilterInPlace的核心函数，用于实现过滤逻辑
func filter(slice, function interface{}, inPlace bool) (interface{}, int) {

  // 检查slice的类型是否为Slice
  sliceInType := reflect.ValueOf(slice)
  if sliceInType.Kind() != reflect.Slice {
    panic("filter: wrong type, not a slice")
  }

  // 检查传入的函数是否符合要求
  fn := reflect.ValueOf(function)
  elemType := sliceInType.Type().Elem()
  if !verifyFuncSignature(fn, elemType, boolType) {
    panic("filter: function must be of type func(" + elemType.String() + ") bool")
  }

  // 记录符合条件的元素的下标
  var which []int
  for i := 0; i < sliceInType.Len(); i++ {
    if fn.Call([]reflect.Value{sliceInType.Index(i)})[0].Bool() {
      which = append(which, i)
    }
  }

  // 根据inPlace参数决定是否创建新的slice
  out := sliceInType
  if !inPlace {
    out = reflect.MakeSlice(sliceInType.Type(), len(which), len(which))
  }

  // 将符合条件的元素复制到新的slice中
  for i := range which {
    out.Index(i).Set(sliceInType.Index(which[i]))
  }

  return out.Interface(), len(which)
}

```












> [参考](https://coolshell.cn/articles/21164.html)