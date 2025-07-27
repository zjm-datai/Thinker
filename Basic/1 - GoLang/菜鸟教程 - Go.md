
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







