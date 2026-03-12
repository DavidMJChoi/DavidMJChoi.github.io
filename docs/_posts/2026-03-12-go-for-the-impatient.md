---
layout: post
title: Go for the Impatient
date: 2026-03-12 16:42 +0800
---

> 出色的 Go 程序员：大道至简。
我：大道智减。
> 
    

> The only way to learn a new programming language is by writing programs in it. — *The C Programming Language*
> 

# 1 语言结构

Go 源码由以下部分组成：

- 包声明
- 导入声明
- 函数
- 变量声明
- 语句和表达式
- 注释

```go
package main

import "fmt"

var globalVar int = 42

func main() {
	/*
		comment block
	*/
	localVar := 720
	// single line comment
	p(localVar)
	fmt.Println(globalVar)
}

func p(v int) {
	fmt.Println(v)
}
```

## 1.1 执行 Go 程序

Go 是编译型语言。每个程序以 `main` 包为入口。最简单的执行方式为 `go run`：

```go
go run main.go
```

也可以使用 `go build` 生成可执行文件：

```go
go build -o _output/main main.go && ./_output/main
```

## 1.2 目录结构

同一目录下，具有同一个包名的若干个 `.go` 文件组成一个包。

- 同一目录下的所有 `.go` 文件必须属于同一个包（`_test` 包除外），否则导入这个目录后，会导致编译错误（在 `import` 语句处）。
- 包名不必与目录名一致，但这是惯例做法，也是**最佳实践**。使用 `import` 语句导入包时，使用的导入路径以目录名为准。若包名与目录名不一致，使用包中导出的实体时，仍然使用包名而非目录名。

一个包定义了一个独立的命名空间。

包也提供代码**封装**。任何标识符大写字母开头的实体将会被**导出**（export），可在包外使用；其他实体则为包私有。

# 2 变量

任何的变量声明有以下的通用形式：

```go
var name type = expression
```

可以省略 `type`，此时类型由等号右侧的表达式中推断；

或者可以省略表达式，此时变量未进行显式初始化，会默认进行**零值**（zero value）**初始化**：

```go
var a = 5 // a is an int
var b int // b is 0
```

- 数字类型初始化为 `0`
- 布尔类型初始化为 `false`
- 字符串初始化为 `""`
- 接口和引用类型初始化为 `nil`
    - 引用类型包括切片、指针、映射、通道、函数等
- 聚合类型（数组或结构体）对其所有元素或字段进行上述的零值初始化

上述的初始化规则已经能够保证所有声明的变量都有合法的值。Go 中不存在“**未初始化的变量**”。

可以在一条语句中声明多个变量，如下所示：

```go
var i,j,k int
var b, f, s = true, 2.3, "four"
var f, err = os.Open(name)
```

局部变量一旦声明则必须被使用，否则编译时报错。全局变量没有这个限制。

## 2.1 短变量声明

短变量声明可用以声明**局部变量**：

```go
a := 5
f, err = os.Open(file)
```

- 简短、易用、且灵活，因此短变量声明被用以声明绝大多数局部变量。
- 使用 `var` 的变量声明一般仅用作所需类型与推断类型不一致时，例如 `var boiling float64 = 100`。


<img src="https://www.notion.so/icons/science_gray.svg" alt="https://www.notion.so/icons/science_gray.svg" width="40px" />

Go 中任何在函数外的语句必须以关键字开头，因此短变量声明不能用在函数外。



注意 `:=` 是声明操作符，其左侧必须出现至少一个未声明变量：

```go
var x int = 10
var y int = 20

x, y := 30, 40 
// error: no new variables on left side of :=
```

对于出现在其左侧的已声明变量，`:=` 的表现与赋值操作符 `=` 一致：

```go
var x int = 10
x, y := 20, 30 // legal. x is 20 afterward.
```

任何定义在当前语义块之外的同名变量会被忽略：

```go
var x int = 10
func main() {
	var y int = 20
	x, y := 30,40
}
```

- 全局的 `x` 在 `main()` 内被忽略了，否则因 `:=` 左边并无未声明变量，`x,y:=30,40` 应导致编译错误。

赋值时会先运算 `=` 右侧，在逐一赋值给 `=` 左侧，因此可如下交换两个变量：

```go
a, b = b, a
```

一些操作会返回多个值，如 map 查询和返回多个值的函数，有时候我们不需要使用这些返回值，但声明后不使用又会报错。因此可以使用只写变量 `_` 来抛弃这些值：

```go
_, ok := m[key]
```

## 2.2 变量的生命周期

**全局变量**：程序存活时间

**局部变量**：不**内存逃逸**则为函数存活时间

# 3 常量

**常量**是编译时求值的表达式。

常量的底层类型必须是基本类型之一：数字、布尔、字符串

```go
const ans int = 42
```

```go
// grouped. the type can be inferred from RHS operand.
const (
	IPv4Len = 4
	IPv6Len = 16
)
```

```go
// omitting implies repeating
const (
	a = 1
	b
	c = 2
	d
) // b is 1, d is 2
```

常量允许的操作：所有算数、逻辑、关系操作；类型转换；特定内置函数 `len` `cap` `real` `img` `complex` `unsafe.Sizeof`。操作发生在编译时，结果也为常数。

## 3.1 iota

常量生成器 `iota` 用于隐式地创建一系列相关的值。`iota` 在 `const` 出现时被重置为 `0`，`const` 块中每新增一行，`iota` 递增 `1`。`iota` 可以用来创建**枚举**：

```go
const (
	sun Weekday = iota
	mon
	tue
	wed
	thu
	fri
	sat
) // sun -> 0, mon -> 1, tue -> 2, ...
```

或者创建**位掩码**：

```go
const (
	FlagUp           Flags = 1 << iota // is up
	FlagBroadcast                      // supports broadcast access capability
	FlagLoopback                       // is a loopback interface
	FlagPointToPoint                   // belongs to a point-to-point link
	FlagMulticast                      // supports multicast access capability
)
```

# 4 操作符

二元的算术、逻辑，以及关系操作符的五个**优先级**如下表所示：

| 1 | `*` `/` `%` `<<` `>>` `&` `&^` |
| --- | --- |
| 2 | `+` `-` `|` `^` |
| 3 | `==` `≠` `<` `≤` `>` `≥` |
| 4 | `&&` |
| 5 | `||` |
- 一元操作符 `!` 为逻辑非
- 同一优先级的操作符向左结合
- 上表中首两行的操作符均有其对应的赋值操作符如 `+=`。
- `%` 仅用于整数，而其结果（即余数）的符号总与**被除数**一致。
    
    ```go
    fmt.Println(-5 % 3)  // -2
    fmt.Println(-5 % -3) // -2
    fmt.Println(5 % -3)  // 2
    fmt.Println(5 % 3)   // 2
    ```
    
- `/` 的表现取决于其操作数。二者均为整数时才进行整数出发。如下例
    
    ```go
    fmt.Println(5.0 / 4.0)  // 1.25
    fmt.Println(5 / 4.0)    // 1.25
    fmt.Println(5.0 / 4)    // 1.25
    fmt.Println(5 / 4)      // 1
    ```
    

算术操作溢出时**回绕**（wrap around）。

下列为二元按位操作符：

| `&` | bitwise AND |
| --- | --- |
| `|` | bitwise OR |
| `^` | bitwise XOR |
| `&^` | bitwise AND NOT (bit clear) |
| `<<` | left shift |
| `>>` | right shift |
- `^` 作为一元操作符时为按位取反。
- `x &^ y` 即 x AND (NOT y)，按照 `y` 中的 1 所在位置，清除（设为 0）`x` 中的对应位置。
- `fmt` 的 `Printf()` 系列函数提供格式化输出。格式化动词 `%b` 表示以二进制输出给定数值。
- 左移和无符号数的右移以 `0` 填充；有符号数的右移以符号位填充。

# 5 结构体

结构体可以用于定义复合数据类型：

```go
type Student struct {
	ID int
	Name string
	Age int
	Score int
}
```

结构体不是引用类型，其零值不为 `nil`，而是其中各个字段各自零值初始化的一个结构体实例。

**声明**：

```go
// an empty struct. each field is zero-value init.
var dmc Employee

// a struct pointer
var pdmc *Employee = &dmc
pdmc := &dmc

// initialize with struct literal. not specified fields are zero-value init.
dmc := Employee{
	ID: 1,
	Name: dmc,
}

// must list all fields in order
dmc := Employee{1, dmc}
```

结构体不能包含自身作为字段，但可以包含指向自身的指针作为字段：

```go
// a binary tree node
type node struct {
	value int
	left, right *node
}
```

访问字段：

```go
// directly
dmc.ID = 2

// indirectly via a pointer
(&dmc).ID = 2
```

若其所有字段均可比较，则结构体可比较。`==` 等关系操作符会执行逐字段比较。

# 6 数组和切片

切片 `[]T` 是 `T` 类型元素的变长连续序列。切片是一个底层数组的抽象视图，表示该数组的一个连续子序列。一个切片由三部分组成：指向子序列首元素的指针；长度，即切片中元素数量，由 `len()` 返回；容量，从切片首元素到底层数组的最后一个元素总计的元素数，由 `cap()` 返回。



**类型**

数组类型为 `[n]T`，而切片类型为 `[]T`**。**

切片只可与 `nil` 比较。

切片类型的零值 `nil`。`nil` 切片不指向任何底层数组，在其上的任何下标操作会导致 panic，但 `len` 和 `cap` 返回 0。

- 处理 `nil` 切片
    
    按照惯例，Go 函数应对所有长度为 0 的切片一视同仁，包括 `nil` 切片。例如 `len` 函数在遭遇 `nil` 切片时正常返回不会 panic。
    




**声明**

```go
// a slice. type is []int
a := []int{1,2,3,4,5}

// an empty slice: len 0, cap 0
s := []int{}

// a nil slice: len 0, cap 0
s := []int(nil)
var s []int

// create a zero-value initialized slice. must at least specify length.
s := make([]T, len, cap)
s := make([]T, cap)[:len]
```





**下标和切片**

下标操作根据索引获取切片中元素：`s[i]`，其中 `i`  的范围为 0 到 `len(s)-1`。

切片操作返回另一个切片：`s[i:j]` 中包括元素 `s[i]` 直到 `s[j-1]`（即其索引范围为半开半闭区间 $[i,j)$）。长度为 `j-i`，容量与 `s` 相同。上述操作中忽略 `i` 则默认为 `0`；忽略 `j` 则默认为 `len(s)`。`i` 应大于 0，而 `j` 应小于 `cap(s)`（对于数组和字符串，则为 `len(s)`）。

如上所述，切片操作可以用于数组和字符串。用于字符串时产生新的字符串。

切片操作可以用来拓展（extend）当前切片：

```go
array := [5]int{1, 3, 5, 7, 9}
s1 := array[0:3]     // slice an array
s2 := s1[:1]         // slice another slice
s3 := s2[:len(s2)+1] // extending (no exceed cap)
```

由于数组必须指定长度，数组越界访问可以在编译时被捕获；而且切片的越界访问只能为运行时 panic。

切片操作还可以附加容量参数：`s[i:j:max]`。返回的新切片的容量为 `max-i`。





**遍历**

```go
for i, v := range s { /* ... */ } 
```





**作为参数传递**

注意 Go 的参数总为按值传递，因此将数组作为参数传递时会创建其一份拷贝。当然也可以手动传递指针：

```go
func zero(ptr *[32]byte)
```

而切片本身就是一个引用，直接传递切片就有类似按引用传递的效果。



# 7 控制结构

## 7.1 分支

```go
if cond {
	// ...
} else if {
	// ...
} else {
	// ...
}
```

```go
if num, ok := dic[key]; ok {
	// ...
}
if ret, err := f(); err != nil {
	// ...
}
```

```go
switch coinFlip() {
	case "head":
		heads++
	case "tail":
		tails++
	default:
		fmt.Println("landed on edge!")
}
```

- 由上至下逐个 `case` 检查是否匹配。无一匹配时落入 `default` 分支。
    - `default` 分支可选。
- 当有一 `case` 匹配时，则自动 `break`，而不会像 C 语言中 “fallthrough”。
    - 可以使用 `fallthrough` 关键字以覆盖默认的自动 `break`。

`switch` 语句的操作数不是必须的，例如：

```go
switch {
	case x > 0:
		return 1
	case x < 0:
		return -1
	default:
		return 0
}
```

- 相当于 `switch true`
- 多分支 `if-else if-else` 的可选写法

## 7.2 循环

```go
for {
	// infinite loop
}
```

```go
for cond {
	// ...
}
```

```go
for init; cond; post {
	// ...
}
```

```go
for k, v := range m {
	// ...
}
for k := range m { /* ... */ }
for _, v := range m { /* ... */ }
```

## 7.3 打乱控制流

`break`、`continue` 以及 `goto` 语句用来修改控制流：

- `break` 完全终止一层 `for` 或 `switch` 或 `select`；
- `continue` 跳过 `for` 循环的当前迭代，马上开始下一次迭代；
- 可以为语句附加标签，而后 `break` 和 `continue` 可直接跳转至标签处。该特性可用来一次性跳出多重循环。
- **Go** 语言中有 `goto` 语句，但其本意是在机器生成的代码中使用，而非直接使用。

# 8 函数

函数声明格式如下：

```go
func funcName(parameterList) (returnList) {
	// function body
}
```

**参数列表**定义了参数名和类型：

```go
func f(x int, y float64, s, t string)
```

- 每个**参数**（parameter）都是函数作用域内的局部函数，函数调用时由**实际参数**（argument）初始化。

**返回列表**：

```go
func f(x int, y float64, s, t string) (a, b bool)
```

- 支持多个返回值
- 返回值可以命名，函数调用时零值初始化，函数体中可以直接使用
- **Bare return**: 返回值预先命名时，不带任何操作数的 `return` 语句会根据定义的顺序直接返回。
- Example of `return` without operands
    
    ```go
    func f(x int) (a bool, b int) {
    
    	a = x == 0
    	b = x + 1
    
    	return
    }
    
    // in main()
    a, b := f(1)
    ```
    

**函数调用**

函数调用操作符 `()`，以及其中的实际参数列表用以调用一个函数：

```go
f(1)
```

- 上述表达式的结果为函数的返回值。如果不将返回值赋值给变量，则返回值被丢弃。
- 若函数没有返回值，上述表达式的结果为一个特殊的 `(no value)`。若将其用作一个值则会导致编译错误。
- Go 中没有“默认参数”。任何函数调用必须提供完整的实际参数列表。
- Go 的参数总按值传递。

## 8.1 函数类型和函数值

函数为**一等值**（first-class values） 。函数有类型，可以被赋给变量，可以当作参数或返回值传递。

一个函数值可以由 `()` 调用：

```go
// an example of being lazy. not a good practice.
var p func(a ...any) (n int, err error) = fmt.Println
// ... in main()
p("hello world")  // equivalent to fmt.Println("hello world")
```

## 8.2 匿名函数和闭包
[TODO: ...]