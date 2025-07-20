
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

