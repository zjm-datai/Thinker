
如果说 gorountine 是 Go 语言程序的并发体的话，那么 channels 是他们之间通信机制。他可以让一个 gorountine 通过它给另一个 gorountine 发送值消息。每个 channel 都有一个特殊的类型，也就是 channels 可发送数据的类型。一个可以发送 int 类型数据的 chammel 一般写作 chan int 。

使用内置的 make 函数，我们可以创建一个 channel：

```go
ch := make(chan int)
```

和 map 类似，channel 也对应一个 make 创建的底层数据结构的引用。当我们复制一个 channel 或用于函数参数传递的时候，我们只是拷贝了一个 channel 引用，因此调用者和被调用者将引用同一个 channel 对象。和其他的引用类型一样，channel 的零值也是 nil 。

一个 channel 有发送和接受两个主要的操作，都是通信行为。一个发送语句将一个值从一个 goroutine 通过 channel 发送到另外一个执行接受操作的 goroutine 。

发送和接受两个操作都是用 `<-` 运算符。在发送语句中，<- 运算符分割 channel 和要发送的值。在接受语句中，<- 运算符写在 channel 对象之前。一个不使用接收结果的接收操作也是合法的。

```go
ch <- x // a send statement
x = <-ch // a receive expression in an assignment statement
<-ch // a receive statement; result is discarded
```

Channel 还支持 close 操作，用于关闭 channel，随后对基于该 channel 的任何发送操作都将导致 panic 异常。对一个已经被 close 过的 channel 进行接收操作依然可以接受到之前已经成功发送的数据；如果 channel 中已经没有数据的话，将产生一个零值的数据。

使用内置的 close 函数就可以关闭一个 channel ：

```go
close(ch)
```

我们也可以指定 make 函数的第二个整型参数，对应 channel 的容量。如果 channel 的容量大于零，那么这个 channel 就是带缓存的 channel 。

```go
ch = make(chan int)
ch = make(chan int, 0)
ch = make(chan int, 3)
```

### 不带缓存的 channels 

