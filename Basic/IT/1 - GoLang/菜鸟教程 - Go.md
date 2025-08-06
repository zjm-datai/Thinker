
## 语言变量

变量来源于数学，是计算机语言中可以存储计算结果或能表示值抽象的概念。

变量可以通过变量名访问。

Go 语言变量名由字母、数字、下划线组成，其中首个字符不能为数字。

声明变量的一般形式是使用 var 关键字：

```go
var identifier type
```

可以一次声明多个变量

```go
var identifier1, identifier2 type
```

```go
package main
import "fmt"
func main() {
    var a string = "Runoob"
    fmt.Println(a)

    var b, c int = 1, 2
    fmt.Println(b, c)
}
```

### 变量声明

第一种，指定变量类型，如果没有初始化，则变量默认为零值

```go
var v_name v_type
v_name = value
```

零值就是变量没有做初始化时候系统默认设置的值

```go
package main

import "fmt"

func main() {
	// 声明一个变量并初始化
	var a = "RUNOOB"
	fmt.Println(a)
	// 没有初始化就是零
	var b int
	fmr.Println(b)
	// bool 零值为 false
	var c bool
	fmt.Println(c)
}
```

```go
package main

import "fmt"

func main() {
    var i int
    var f float64
    var b bool
    var s string
    fmt.Printf("%v %v %v %q\n", i, f, b, s)
}
```

第二种，根据值自行判定变量类型

```go
var v_name = value
```

第三种，如果变量已经使用 var 声明过了，再使用 `:=` 声明变量，就产生编译错误，格式：

```go
v_name := value
```

例如：

```go
var intVal int
intVal := 1 // 这时候会产生编译错误，因为 intVal 已经声明，不需要重新声明
```

直接使用下面的语句即可：

```go
intVal := 1 // 此时不会产生编译错误，因为有声明新的变量，因为 := 是一个声明语句
```

intVal := 1 相等于：

```go
var intVal int
intVal = 1
```

可以将 var f string = "Runoob" 简写为 f := "Runoob"：

```go
package main
import "fmt"

func main() {
	f := "Runoob"
	fmt.Println(f)
}
```

### 多变量声明

```go
//类型相同多个变量, 非全局变量
var vname1, vname2, vname3 type
vname1, vname2, vname3 = v1, v2, v3

var vname1, vname2, vname3 = v1, v2, v3 // 和 python 很像,不需要显示声明类型，自动推断

vname1, vname2, vname3 := v1, v2, v3 // 出现在 := 左侧的变量不应该是已经被声明过的，否则会导致编译错误


// 这种因式分解关键字的写法一般用于声明全局变量
var (
    vname1 v_type1
    vname2 v_type2
)
```

### 值类型和引用类型

所有像 int、float、bool 和 string 这些基本类型都属于值类型，使用这些类型的变量直接指向存在内存中的值。

当使用等号 = 将一个变量的值赋值给另一个变量时，如：j = i，实际上是在内存中将 i 的值进行了拷贝。

我们可以使用 &i 来获取变量 i 的内存地址，例如：0xf840000040（每次的地址都可能不一样）。

值类型变量通常存储在栈中，尤其是当它们是局部变量时。当值类型变量的值需要在函数作用域之外使用时，Go 会将其分配到堆内存中。

内存地址会根据机器的不同而有所不同，甚至相同的程序在不同的机器上指向也会有不同的内存地址。

因为每台机器可能有不同的存储器布局，并且位置分配也可能不同。

更复杂的数据通常使用多个字，这些数据一般使用引用类型保存。

一个引用类型的变量 r1 存储的是 r1 的值所在的内存地址（数字），或内存地址中第一个字所在的位置。

这个内存地址称之为指针，这个指针实际上也被存在另外的某一个值中。

同一个引用类型的指针指向的多个字可以是在连续的内存地址中（内存布局是连续的），这也是计算效率最高的一种存储形式；也可以将这些字分散存放在内存中，每个字都指示了下一个字所在的内存地址。当使用赋值语句 r2 = r1 时，只有引用地址被复制。

如果 r1 的值被改变了，那么这个值的所有引用都会指向被修改后的内容，在这个例子中，r2 也会受到影响。

---

#### 值类型：直接存储值本身

**核心特点：** 变量直接存储具体的数据值，而非地址

1. 内存中的表现

比如定义一个整数变量 i := 10

- 系统会在内存中（通常是栈内存，局部变量时）开辟一块空间，直接存放 10 个值
- 变量 i 就绑定到这块空间，访问 i 就是直接读取里面的 10 

2. 赋值时的行为：拷贝值

当执行 j := i 时：

- 系统会新开辟一块内存空间，将 i 存储的 10 完整复制一份，放到 j 对应的空间里
- 此时 i 和 j 是两块独立的内存，修改其中一个不会影响另外一个

```go
i := 10
j := i
i = 20  // 只修改i，j不受影响
fmt.Println(j)  // 输出 10
```

#### 如何查看内存地址

用 & 符号可以获取变量的内存地址。比如 &i 会返回 i 所在内存的地址

对于值类型，i 和 j 的地址一定不同（因为是两块独立空间）

### 引用类型：存储值的地址（指针）

**核心特点：** 变量不直接存值，而是存储值所在的内存地址（指针），通过地址间接访问值。

#### 内存中的表现

以 Go 中的切片为例，切片是典型的引用类型：

```go
s1 := []int{1, 2, 3}
```

系统会在内存中做两件事

- 在堆内存中开辟一块空间，存放 `[1, 2, 3]` 这个具体的切片数据
- 在栈内存中给变量 s1 开辟一块空间，存放的不是 `[1, 2, 3]` ，而是堆内存中切片数据的地址（比如  0x123456）

访问 s1 时，系统会先读取 s1 存储的地址 0x123456，再根据地址找到堆内存中的 `[1, 2, 3]` 。

#### 赋值时的行为：拷贝地址引用

当执行 s2 := s1 时：

- 系统只会复制 s1 中存储的地址（0x123456），给 s2 开辟的新空间。
- 此时 s1 和 s2 存储的地址相同，都指向堆内存中同一份 `[1,2,3]` 数据。

修改值时的影响：所有引用都会受影响

```go
s1 := []int{1, 2, 3}
s2 := s1       // s2 复制了s1的地址，和s1指向同一份数据
s1[0] = 100    // 修改s1指向的数据
fmt.Println(s2[0])  // 输出 100（s2也会读到修改后的值）
```

| 类型   | 存储内容     | 赋值行为      | 修改影响范围          | 常见例子（Go）                             |
| ---- | -------- | --------- | --------------- | ------------------------------------ |
| 值类型  | 直接存 “值”  | 拷贝完整的值    | 只影响当前变量         | int、float、bool、string、数组（array）      |
| 引用类型 | 存 “值的地址” | 只拷贝地址（引用） | 所有指向该地址的变量都会受影响 | 切片（slice）、映射（map）、指针（*T）、通道（channel） |

### 为什么需要区分这两种类型？

1. **内存效率**：
    
    - 值类型如果数据量大（如长字符串、大数组），赋值时拷贝完整数据会浪费内存和时间。
    - 引用类型赋值时只拷贝地址（通常是 8 字节，64 位系统），更高效。

2. **数据同步**：
    
    - 引用类型适合需要多变量共享同一份数据的场景（比如函数间传递大集合，避免拷贝）。
    - 值类型适合需要独立数据副本的场景（比如计算过程中不希望原数据被修改）。

### string 是特殊的 “值类型”

Go 中的 string 虽然是值类型，但行为有点特殊：

- 它存储的是字符串的 “指针 + 长度”（类似引用类型的结构），但赋值时会拷贝这两个值（指针和长度）。
- 由于字符串是**不可变的**（修改时会创建新字符串），所以即使拷贝了指针，修改后也不会影响原变量，看起来像值类型的独立行为。

[[go 中的 string]]

### 简短形式，使用 := 赋值操作符

我们知道可以在变量的初始化时省略变量的类型而由系统自动推断，声明语句写上 var 关键字其实是显得有些多余了，因此我们可以将它们简写为 a := 50 或 b := false。

a 和 b 的类型（int 和 bool）将由编译器自动推断。

这是使用变量的首选形式，但是它只能被用在函数体内，而不可以用于全局变量的声明与赋值。使用操作符 := 可以高效地创建一个新的变量，称之为初始化声明。

Go 对全局变量（包级变量）的声明有严格的语法要求：  

**全局变量必须通过  var 关键字声明，且必须明确类型（或通过初始化值推断类型，但 var 关键字不能省略）**。

Go 语言强调 “简单、清晰、无歧义”，而全局变量和局部变量的职责不同：

- **局部变量**（函数体内）：生命周期短，作用域有限，通常用于临时计算，需要简洁的语法提高开发效率。:= 的设计正是为了简化局部变量的声明，让代码更紧凑。
- **全局变量**：生命周期与包绑定，作用域覆盖整个包（甚至可导出到其他包），对代码的整体结构影响更大。Go 希望全局变量的声明更 “郑重”，通过强制使用 var 关键字，让开发者在定义全局变量时更谨慎，也让读者能快速识别出全局变量（看到 var 就知道可能是全局的）。

## Go 语言条件语句

### if 语句

Go 语言中的 if 语句支持以下的形式：

```go
if 初始化语句; 条件表达式 {
	// 代码块
}
```

这里的初始化语句是可选的，通常用于声明并初始化一个或多个局部变量（如 `r := recover()`），这些变量的作用域仅限于这个 `if` 语句块内。

```go
if const r = recover(); r != nil { 
	// 错误：非法语法 fmt.Println("Recovered from panic:", r) 
}
```

### if 语句嵌套

## 语言切片

Go 语言切片是对数组的抽象。

Go 数组的长度不可改变，在特定场景中这样的集合就不太适用，Go 中提供了一种灵活，功能强悍的内置类型切片("动态数组")，与数组相比切片的长度是不固定的，可以追加元素，在追加时可能使切片的容量增大。

### 定义切片

我们可以声明一个未指定大小的数组来定义切片：

```go
var indentifier []type
```

切片不需要说明长度。

或使用 **make()** 函数来创建切片:

```go
var slice1 []type = make([]type, len)

// 也可以简写为

slice1 := make([]type, len)
```

也可以指定容量，其中 **capacity** 为可选参数。

```go
make([]T, length, capacity)
```

这里 len 是数组的长度并且也是切片的初始长度。

## Map 集合

Map 是一种无序的键值对集合，它最重要的一点在于它是通过 key 来快速检索数据的，key 类似于索引，指向数据的值。

Map 是一种集合，所以我们可以像迭代数组和切片那样去迭代它。不过 Map 是无序的，遍历 Map 时返回的键值对的顺序是不确定的。

在获取 Map 的值时，如果键不存在，返回该类型的零值，例如 int 类型的零值是 0，string 类型的零值是 ""。

Map 是引用类型，如果将一个 Map 传递给一个函数或赋值给另一个变量，它们都指向同一个底层数据结构，因此对 Map 的修改会影响到所有引用它的变量。

### 定义 Map

可以使用内建函数 make 或使用 map 关键字来定义 Map ：

```go
map_variable := make(map[KeyType]ValueType, initialCapacity)
```

其中 KeyType 是键的类型，ValueType 是值的类型，initialCapacity 是可选的参数，用于指定 Map 的初始容量。Map 的容量是指 Map 中可以保存的键值对的数量，当 Map 中的键值对数量达到容量时，Map 会自动扩容。如果不指定 initialCapacity，Go 语言会根据实际情况选择一个合适的值。

```go
// 创建一个空的 Map
m := make(map[string]int)

// 创建一个初始容量为 10 的 Map
m := make(map[string]int, 10)
```

也可以使用字面量创建 Map：

```go
// 使用字面量创建 Map
m := map[string]int{
    "apple": 1,
    "banana": 2,
    "orange": 3,
}
```

删除元素：

```go
delete(m, "banana")
```



## Go defer 关键字

在 Go 语言里，`defer` 语句的作用是把一个函数调用推迟到当前函数执行结束后再进行。

- **函数结束时执行**：无论函数是正常返回，还是因为发生 panic 而异常退出，`defer` 后的函数都会被执行。
- **多个 defer 的执行顺序**：如果有多个 `defer` 语句，它们会按照后进先出（LIFO）的顺序执行。
- **参数值的确定**：`defer` 语句中的参数值是在 defer 语句出现时就确定下来的，而不是在函数执行结束时确定。

```go
package main

import "fmt"

func main() {
	defer fmt.Println("defer 1")
	defer fmt.Println("defer 2")
	fmt.Println("main 函数")
}
```

输出结果：

```
main 函数 
defer 2 
defer 1
```

### 常见应用场景

#### 资源释放

在进行文件操作，网络连接或者锁操作的时候，defer 可以保证资源被正确释放。

```go
func readFile() {
	file, err := os.Open("test.txt")
	if err != nil {
		log.Fatal(err)
	}

	defer file.Close() // 确保文件会被关闭

	// 对文件进行读取操作，会比关闭先执行
}
```

#### 解锁操作

在使用互斥锁（sync.Mutex）的时候，可以借助 `defer` 来确保锁一定会被释放

```go
func processData(mu *sync.Mutex) {
	mu.Lock()
	defer mu.Unlock() // 确保锁一定会被释放

	// 对共享数据进行操作
}
```

锁要发挥作用，关键在于多个 goroutine 访问的是同一个锁的实例。如果锁传递的是值（也就是 sync.Mutex 而不是 `*sync.Mutex`），函数在调用的时候会复制这个锁，这就使得每个函数调用的时候都拥有了自己的锁副本，进而导致锁无法正常的发挥作用，共享资源依然会面临竞态条件。

```go
package main

import (
	"fmt",
	"sync"
)

// 错误做法：传递锁的值

func wrongProcessData(mu sync.Mutex) {
    mu.Lock()
    defer mu.Unlock()
    fmt.Println("wrongProcessData 获取到锁")
}

// 正确做法：传递锁的指针
func rightProcessData(mu *sync.Mutex) {
    mu.Lock()
    defer mu.Unlock()
    fmt.Println("rightProcessData 获取到锁")
}

func main() {
    var mu sync.Mutex
    var wg sync.WaitGroup

    // 错误示例：多个 goroutine 使用的是锁的副本
    wg.Add(2)
    go func() {
        defer wg.Done()
        wrongProcessData(mu) // 传递的是锁的值
    }()
    go func() {
        defer wg.Done()
        wrongProcessData(mu) // 传递的是锁的值
    }()
    wg.Wait()

    fmt.Println("-------------------")

    // 正确示例：多个 goroutine 使用的是同一个锁
    wg.Add(2)
    go func() {
        defer wg.Done()
        rightProcessData(&mu) // 传递的是锁的指针
    }()
    go func() {
        defer wg.Done()
        rightProcessData(&mu) // 传递的是锁的指针
    }()
    wg.Wait()
}
```



## Go 语言递归函数

## Go 语言接口

接口是 Go 语言中的一种类型，用于定义行为的集合，它通过描述类型必须实现的方法，规定了类型的行为契约。

### 空接口

空接口是 Go 的特殊接口，表示所有类型的超集。

- 任意类型都实现了空接口
- 常用于需要存储任意类型数据的场景，如泛型容器，通用参数等

### 类型断言

类型断言是一种机制，可用于检查接口值的底层具体类型，还能从中提取出该底层的值。其语法格式为 `x.(T)` ，这里的 x 代表的是接口类型的表达式，而 T 则表示要断言的目标类型。

- **断言类型正确**：要是接口值的底层类型确实是 `T`，那么断言操作会返回该底层值，其类型为 `T`。
- **断言类型错误**：若接口值的底层类型并非 `T`，程序就会触发 `panic`。

```go
package main

import "fmt"

func main() {
	var i interface{} = "hello"

	// 断言类型正确
	s := i.(string)
	fmt.Println(s) // 输出：hello

    // 断言类型错误，会触发 panic
    // n := i.(int) 
    // 运行时 panic: interface conversion: interface {} is string, not int
}
```

#### 安全断言（带双返回值）

为了防止程序因断言失败而出现 panic ，可以采用安全断言的方式。也就是让类型断言返回两个值，第一个值是转换后的结果，第二个值是一个布尔值，用于表示断言是否成功。其语法格式为 `value, ok := x.(T)`

- 当断言成功时，ok 的值为 true，value 就是转换之后的结果
- 当断言失败时，ok 的值为 false ，value 则是 T 类型的零值，并且程序不会发生 panic 

```go
package main

import "fmt"

func main() {
	var i interface{} = "hello"

	// 安全断言
	if s, ok := i.(string); ok {
		fmt.Println("是字符串类型：", s)
	} else {
		fmt.Println("不是字符串类型")
	}

	if n, ok := i.(int); ok {
		fmt.Println("是整数类型：", n)
	} else {
		fmt.Println("不是整数类型")
	}
}
```

运行这段代码，输出结果为：

```
是字符串类型: hello 
不是整数类型
```

#### 结合类型开关（Type Switch）使用

类型断言常常会和类型开关结合起来，用于对接口值的类型进行分支判断。示例如下：

```go
package main

import "fmt"

func describe(i interface{}) {
	switch v := i.(type) {
	case int:
		fmt.Println("是整数，值为：", v)
	case string:
		fmt.Println("是字符串，值为：", v)
	default:
		fmt.Println("未知类型")
	}
}

func main() {
	describe(123)
	describe("hello")
	describe(true)
}
```

#### 常见应用场景

**从接口值中提取具体类型**：当你把具体类型的值赋给接口变量后，如果后续需要使用该值的具体类型方法，就可以借助类型断言来实现。

**处理多类型参数**：在函数需要处理多种不同类型的参数时，可将参数定义为接口类型，然后在函数内部通过类型断言对参数类型进行判断和处理。

**实现泛型功能**：在 Go 1.18 版本引入泛型之前，类型断言被广泛用于实现类似泛型的功能。

## Go 错误处理

Go 语言通过内置的错误接口提供了非常简单的错误处理机制。

Go 语言的错误处理采用显示返回错误的方式，而非传统的异常处理机制。这种设计使代码逻辑更清晰，便于开发者在编译时或运行时明确处理错误。

Go 的错误处理主要围绕以下机制展开：

1. error 接口：标准的错误表示
2. 显式返回值：通过函数的返回值返回错误
3. 自定义错误：可以通过标准库或自定义的方式创建错误
4. panic 和 recover ：处理不可恢复的严重错误

### error 接口

Go 标准库定义了一个 error 接口，表示一个错误的抽象。

error 类型是一个接口类型，这是它的定义：

```go
type error interface {
	Error() string
}
```

实现 error 接口：任何实现了 Error() 方法的类型都可以作为错误。

Error() 方法返回一个描述错误的字符串。

#### 使用 error 包创建错误

我们可以在编码中通过实现 error 接口类型来生成错误信息。创建一个简单的错误：

```python
package main

import (
	"error",
	"fmt"
)

func main() {
	err := errors.New("this is an error")
	fmt.Println(err)
}
```

函数通常在最后的返回值中返回错误信息，使用 errors.New 可返回一个错误信息：

```go
func Sqrt(f float64) (float64, error) {
    if f < 0 {
        return 0, errors.New("math: square root of negative number")
    }
    // 实现
}
```

```go
result, err:= Sqrt(-1)

if err != nil {
   fmt.Println(err)
}
```

### 显式返回错误

Go 中，错误通常作为函数的返回值返回，开发者需要显式检查并处理。

显式返回错误：

```go
package main

import (
	"errors",
	"fmt"
)

func divide(a, b int) (int, error) {
	if b == 0 {
		return 0, errors.New("division by zero")
	}
}

func main() {
	result, err := divide(10, 0)
	if err != nil {
		fmt.Println("Error:" err)
	} else {
		fmt.Println("Result:", result)
	}
}
```

输出：

```
Error: division by zero
```

### 自定义错误

通过定义自定义类型，可以扩展 error 接口。

自定义错误类型：

```go
package main

import {
	"fmt"
}

type DivideError struct {
	Dividend int
	Divisor int
}

func (e *DivideError) Error() string {
	return fmt.Sprintf("cannot divide %d by %d", e.Dividend, e.Divisor)
}

func divide(a, b int) (int, error) {  
	if b == 0 {  
		return 0, &DivideError{Dividend: a, Divisor: b}  
	}  
	return a / b, nil  
}  
  
func main() {  
	_, err := divide(10, 0)  
	if err != nil {  
		fmt.Println(err) // 输出：cannot divide 10 by 0  
	}  
}
```

## Go 并发

### 简介

并发指的是程序同时执行多个任务的能力。

Go 支持并发，通过 goroutine 和 channels 提供了一种简洁且高效的方式来实现并发。
#### Gorountines

- Go 中的并发执行单位，类似于轻量级的线程。
- Goroutine 的调度由 Go 运行时管理，用户无需手动分配线程。
- 使用 `go` 关键字启动 Goroutine。
- Goroutine 是非阻塞的，可以高效地运行成千上万个 Goroutine。

#### Channel

- Go 中用于在 Goroutine 之间通信的机制。
- 支持同步和数据共享，避免了显示的锁机制。
- 使用 chan 关键字创建，通过 `<-` 操作符号发送和接受数据。

#### Scheduler 调度器

Go 的调度器基于 GMP 模型，调度器会将 Goroutine 分配到系统线程中执行，并通过 M 和 P 的配合高效管理并发。

- **G**：Goroutine。
- **M**：系统线程（Machine）。
- **P**：逻辑处理器（Processor）。

### Gorountine

### Channel





## Go 继承

在面向对象编程 OOP 中，继承是一种机制，允许一个类（子类）从另一个类（父类）继承属性和方法。通过继承，子类可以复用父类的代码，并且可以在不修改父类的情况下扩展或修改其行为。

Go 语言并不是一种传统的面向对象编程语言，它没有类和继承的概念。

Go 使用结构体（struct）和接口（interface）来实现类似的功能。

### Go 中的继承

Go 语言没有传统面向对象语言中的类（class）和继承（inheritance）概念，而是通过组合（composition）和接口（interface）来实现类似的功能。

#### 组合

组合是 Go 中实现代码复用的主要方式。通过将一个结构体嵌入到另一个结构体中，子结构体可以继承父结构体的字段和方法。

```go
package main

import "fmt"

type Animal struct {
	Name string
}

func (a *Animal) Speak() {
	fmt.Println(a.Name, "says hello!")
}

type Dog struct {
	Animal // 嵌入 Animal 结构体
	Breed string
}

func main() {
	dog := Dog{
		Animal: Animal{Name: "Buddy"},
		Breed: "Golden Retriever",
	}

	dog.Speak()
	fmt.Println("Breed:", dog.Breed)
}
```

### 接口模拟多态

接口是 GO 中实现多态的主要方式。通过定义接口，不同的结构体可以实现相同的方法，从而实现类似继承的多态行为。

```go
package main

import "fmt"

// 定义接口
type Speaker interface {
    Speak()
}

// 父结构体
type Animal struct {
    Name string
}

// 实现接口方法
func (a *Animal) Speak() {
    fmt.Println(a.Name, "says hello!")
}

// 子结构体
type Dog struct {
    Animal
    Breed string
}

// 重写 Speak 方法
func (d *Dog) Speak() {
    fmt.Println(d.Name, "the", d.Breed, "barks!")
}

// 新增子结构体 Cat
type Cat struct {
    Animal
    Color string
}

// 实现 Cat 的 Speak 方法
func (c *Cat) Speak() {
    fmt.Println(c.Name, "the", c.Color, "cat meows!")
}

func main() {
    var speaker Speaker

    dog := Dog{
        Animal: Animal{Name: "Buddy"},
        Breed:  "Golden Retriever",
    }

    cat := Cat{
        Animal: Animal{Name: "Whiskers"},
        Color:  "Gray",
    }

    // 通过接口调用不同实现
    speaker = &dog
    speaker.Speak() // 输出: Buddy the Golden Retriever barks!

    speaker = &cat
    speaker.Speak() // 输出: Whiskers the Gray cat meows!
}
```







