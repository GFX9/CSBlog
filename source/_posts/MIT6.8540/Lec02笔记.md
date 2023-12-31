---
title: "MIT6.8540(6.824) Lec02笔记: Go: 同步原语、通道、Context和RPC)"
date: 2023-12-25 17:01:51
category: 
- 'CS课程笔记'
- 'MIT6.5840(6.824) 2023'
- 课程笔记
tags:
- '分布式系统'
- 'Go'
---

这节课主要是对分布式整体的概念和`Go`的一些相关介绍, 包括线程、RPC、同步原语、管道等，相对简单， 简单记录一下， 顺带附上我进行Lab1实验时相关知识的补充

# 1 同步原语
课程举了一个投票的例子:
```go
package main

import "sync"
import "time"
import "math/rand"

func main() {
	rand.Seed(time.Now().UnixNano())

	count := 0
	finished := 0
	var mu sync.Mutex
	cond := sync.NewCond(&mu)

	for i := 0; i < 10; i++ {
		go func() {
			vote := requestVote()
			mu.Lock()
			defer mu.Unlock()
			if vote {
				count++
			}
			finished++
			cond.Broadcast()
		}()
	}

	mu.Lock()
	for count < 5 && finished != 10 {
		cond.Wait()
	}
	if count >= 5 {
		println("received 5+ votes!")
	} else {
		println("lost")
	}
	mu.Unlock()
}

func requestVote() bool {
	time.Sleep(time.Duration(rand.Intn(100)) * time.Millisecond)
	return rand.Int() % 2 == 0
}
```
# 1.1 **锁**
整体而言`go`中锁的语法和`C/C++`区别不大, 但这里介绍一下`defer`:
- `defer` 语句后面跟着的是一个函数调用，这个调用不会立即执行。而是延迟到包含它的函数执行完毕时才执行
- 如果有多个 `defer` 语句，它们的调用顺序是后进先出的，即最后一个 `defer` 的函数调用将会第一个被执行
- **重点:`defer` 的调用不会在每次循环结束时执行，而是会在包围 defer 的函数返回时才执行** => 如果有锁需要释放的话, 需要在循环体内手动释放?

# 1.2 **条件变量**
还是相同的例子:
```go
package main

import "sync"
import "time"
import "math/rand"

func main() {
	rand.Seed(time.Now().UnixNano())

	count := 0
	finished := 0
	var mu sync.Mutex
	cond := sync.NewCond(&mu)

	for i := 0; i < 10; i++ {
		go func() {
			vote := requestVote()
			mu.Lock()
			defer mu.Unlock()
			if vote {
				count++
			}
			finished++
			cond.Broadcast()
		}()
	}

	mu.Lock()
	for count < 5 && finished != 10 {
		cond.Wait()
	}
	if count >= 5 {
		println("received 5+ votes!")
	} else {
		println("lost")
	}
	mu.Unlock()
}

func requestVote() bool {
	time.Sleep(time.Duration(rand.Intn(100)) * time.Millisecond)
	return rand.Int() % 2 == 0
}
```
没啥好讲的...
条件变量的三个主要方法是：
- `Wait`：调用这个方法会阻塞调用协程，直到其他协程在相同的条件变量上调用 Signal 或 Broadcast。
- `Signal`：唤醒等待该条件变量的一个协程（如果存在的话）。
- `Broadcast`：唤醒等待该条件变量的所有协程。

## 1.3 `WaitGroup`协程同步
`sync.WaitGroup `用于等待一组协程的完成。一个 `WaitGroup` 等待一系列的事件，主要的用法包括三个方法：
- `Add` 方法: 在启动协程之前，使用 `Add` 方法来设置要等待的事件数量。通常这个值设置为即将启动的协程的数量。
- `Done` 方法: 当协程的工作完成时，调用 `Done` `方法。Done` 方法减少 `WaitGroup` 的内部计数器，通常等价于 `Add(-1)`。
- `Wait` 方法: 使用 Wait 方法来阻塞，直到所有的事件都已通过调用 `Done` 方法来报告完成。

```go
// 假设我们有三个并行的任务需要执行
    tasks := 3

    // 使用Add方法来设置WaitGroup的计数器
    wg.Add(tasks)

    for i := 1; i <= tasks; i++ {
        // 启动一个协程
        go func(i int) {
            defer wg.Done() // 确保在协程的末尾调用Done来递减计数器
            time.Sleep(2 * time.Second) // 模拟耗时任务
            fmt.Printf("Task %d finished\n", i)
        }(i)
    }

    // Wait会阻塞，直到WaitGroup的计数器减为0
    wg.Wait()
```
**注意**:
1. 不要复制 `WaitGroup`。如果需要将 WaitGroup 传递给函数，应使用指针。
2. 避免在协程内部调用 `Add` 方法，因为这可能会导致计数器不准确。最好在启动协程之前添加所需的计数。
3. 使用 `Done` 方法是减少 `WaitGroup` 计数器的推荐方式，它等价于 `Add(-1)`。

# 2 通道
## 2.1 案例
还是投票...
`go`的通道可以实现无锁的并发访问, 核心在于其保证通道写入在不同协程间不会冲突
```go
package main

import "time"
import "math/rand"

func main() {
	rand.Seed(time.Now().UnixNano())

	count := 0
	ch := make(chan bool)
	for i := 0; i < 10; i++ {
		go func() {
			ch <- requestVote()
		}()
	}
	for i := 0; i < 10; i++ {
		v := <-ch
		if v {
			count += 1
		}
	}
	if count >= 5 {
		println("received 5+ votes!")
	} else {
		println("lost")
	}
}

func requestVote() bool {
	time.Sleep(time.Duration(rand.Intn(100)) * time.Millisecond)
	return rand.Int()%2 == 0
}
```
## 2.2 带缓冲的通道
另外, 通道还支持带缓冲:
```go
bufferedCh := make(chan int, 2) // 缓冲大小为2
```
两种通道区别如下:
- 不带缓冲的通道在发送操作和接收操作之间进行同步：发送会阻塞，直到有另一个协程来接收数据；接收会阻塞，直到有另一个协程发送数据。
- 带缓冲的通道有一个固定大小的缓冲区。发送操作只在缓冲区满时阻塞，接收操作只在缓冲区空时阻塞。

## 2.3 SELECT和通道
`select`允许协程在多个通道操作上等待。select 会阻塞，直到其中一个通道操作可以执行：
```go
select {
case msg := <-ch1:
    // 从ch1接收消息
case ch2 <- msg:
    // 发送消息到ch2
default:
    // 如果以上都不可执行，则执行默认操作（非阻塞）
}
```

# 3 Context控制上下文
## 3.1 Context 接口
`Context 类型`用于创建和操纵上下文的函数,用于定义截止日期、取消信号以及其他请求范围的值的接口。它设计用来传递给请求范围的数据、取消信号和截止时间到不同的协程中，以便于管理它们的生命周期。先来看`Context 接口`:
`Context` 接口定义了四个方法：
1. `Deadline`：返回 `Context` 被取消的时间，也就是完成工作的截止时间（如果有的话）。
2. `Done`：返回一个 `Channel`，这个 `Channel` 会在当前工作应当被取消时关闭。
3. `Err`：返回 `Context` 结束的原因，它只会在 `Done` 返回的 `Channel` 被关闭后返回非空值。
4. `Value`：从 `Context` 中检索键对应的值。

### 3.2 操纵上下文的函数
`context` 包提供了几个用于创建和操纵上下文的函数：
1. `context.Background`：返回一个空的 `Context`。这个 `Context` 通常被用作整个程序或请求的顶层 `Context`。
2. `context.TODO`：不确定应该使用哪个 `Context` 或者还没有可用的 `Context` 时，使用这个函数。这在编写初始化代码或者不确定要使用什么上下文时特别有用。
3. `context.WithCancel`：创建一个新的 `Context`，这个 `Context` 会包含一个取消函数，可用于取消这个 `Context` 及其子树。
4. `context.WithDeadline`：创建一个新的 `Context`，这个 `Context` 会在指定的时间到达时自动取消。
5. `context.WithTimeout`：创建一个新的 `Context`，这个 `Context` 会在指定的时间段后自动取消。

## 3.3 案例 
### 3.3.1 简单案例
```go
package main

import (
	"context"
	"fmt"
	"time"
)

func operation(ctx context.Context, duration time.Duration) {
	select {
	case <-time.After(duration): // 模拟一个耗时操作
		fmt.Println("Operation done.")
	case <-ctx.Done(): // 检测 context 的取消事件
		fmt.Println("Operation canceled:", ctx.Err())
	}
}

func main() {
	// 创建一个可取消的 context
	ctx, cancel := context.WithCancel(context.Background())

	// 在一个新的协程中运行 operation 函数
	go operation(ctx, 5*time.Second)

	// 模拟在 operation 运行一段时间后取消操作
	time.Sleep(2 * time.Second)
	cancel() // 取消 context

	// 给 operation 一些时间来处理取消事件
	time.Sleep(1 * time.Second)
}
```
这个案例中, 将`ctx`显式传递给在子协程, 使其可以受外部的协程控制。

### 3.3.2 复杂案例`socks5代理`
这里给出一个`字节跳动后端青训营`实现的`socks5代理`中对`context` 的使用, 完整代码看[这里](https://github.com/wangkechun/go-by-example/tree/master/proxy/v4)：
```go
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	go func() {
		_, _ = io.Copy(dest, reader)
		cancel()
	}()
	go func() {
		_, _ = io.Copy(conn, dest)
		cancel()
	}()

	<-ctx.Done()
```
> **任务需求**:
有两个 goroutine 分别用于从客户端读取数据并写入目的端，以及从目的端读取数据并写入客户端, 要求一旦有一个方向的拷贝操作出现错误, 将另一个操作也取消

- 方案:使用`WithCancel`和`<-ctx.Done()`:
- 问题？我第一次看到这个代码时， 有这样的问题
  `ctx`并没有被显式地传递给2个goroutine, 2个`goroutine`调用`cancel`取消的是`WithCancel`返回的`ctx`, 而不是自己, 所以这为什么能工作?
- 答案:
  `cancel` 函数与 `ctx` 相关联，而 `cancel` 被闭包捕获并在多个 `goroutine` 中使用。这就是为什么调用` cancel()` 会影响所有这些 goroutine 的原因，不管 `ctx` 是否被显式传递。这种行为是 `context` 包设计的一部分，允许协调不同 `goroutine` 之间的取消事件。cancel() 被调用时会取消 `ctx` 上下文，而与这个 `ctx` 相关联的所有操作（在这个例子中是两个 `io.Copy` 调用）都会接收到取消通知，即使它们在不同的 `goroutine` 中执行，且 `ctx` 没有显式地传递给它们。


## 3.4 注意事项

- 不应该把 `Context` 存储在结构体中，它应该通过参数传递。
- `Context` 是协程安全的，你可以把一个 `Context` 传递给多个协程，每个协程都可以安全地读取和监听它。
- 一旦一个 `Context` 被取消，它的所有子 `Context` 都会被取消。
- `Context` 的 `Value` 方法应该被用于传递请求范围的数据，而不是函数的可选参数。

`Context` 在处理跨越多个协程的取消信号、超时以及传递请求范围数据时起到了关键作用，是 Go 并发编程中的重要组件。

# 4 RPC
> 这本来是在`Lab 1`过程中补的知识, 但介于这里写了这么多`Go`, 就放在一起了

Go 标准库中的 `net/rpc` 包提供了创建 `RPC` 客户端和服务器的机制。
RPC 允许客户端程序调用在服务器上运行的程序的函数或方法，就好像它是本地可用的一样。客户端和服务器之间的通信通常是透明的，客户端程序仅需知道需要调用的远程方法和必须提供的参数。
 `net/rpc` 包使用了 Go 的编码和解码接口，允许使用 `encoding/gob` 包来序列化和反序列化数据（尽管也可以使用其他编码，如 `JSON`）。`RPC` 调用默认是通过 `TCP` 或者 `UNIX` 套接字传输的，但是你可以实现自己的传输方式。

## 4.1 服务器端
要创建一个 `Go RPC` 服务器，你需要创建一些方法，这些方法必须满足以下条件：

1. 方法是导出的（首字母大写）。~~Lab1 被坑惨了~~
2. 方法有两个参数，都是导出类型或内建类型。
3. 方法的第二个参数是指针, 相当于写出。
4. 方法返回一个 `error` 类型。

然后，将这个类型的实例注册为 RPC 服务：

```go
type Arith int

func (t *Arith) Multiply(args *Args, reply *int) error {
    *reply = args.A * args.B
    return nil
}

func main() {
    arith := new(Arith)
    rpc.Register(arith)
    rpc.HandleHTTP()
    err := http.ListenAndServe(":1234", nil)
    if err != nil {
        log.Fatal("ListenAndServe: ", err)
    }
}
```

## 4.2 客户端
客户端需要使用 `rpc.Dial` 函数连接到 RPC 服务器，然后可以通过 `Call` 方法进行同步调用或 `Go` 方法进行异步调用：
```go
client, err := rpc.Dial("tcp", "server address")
if err != nil {
    log.Fatal("dialing:", err)
}

// Synchronous call
args := &Args{7, 8}
var reply int
err = client.Call("Arith.Multiply", args, &reply)
if err != nil {
    log.Fatal("arith error:", err)
}
fmt.Printf("Arith: %d*%d=%d", args.A, args.B, reply)
```

## 4.3 缺点
`net/rpc` 包的文档提到，该包已经被标记为[“冻结”（frozen）](https://golang.org/s/go1.7-rpc)并不推荐使用。这意味着该包不会有新的发展，尽管它仍然是可用的。因此，应该考虑使用更现代的解决方案，如 [gRPC](https://grpc.io/)，它支持多种语言，提供了更复杂的特性，例如双向流和集成认证。