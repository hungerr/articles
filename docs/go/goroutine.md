# Go并发goroutine

通常，编写并发程序时最大的问题是在进程之间共享数据。 Go 采用不同于其他编程语言的通信方式，因为 Go 是通过 `channel` 来回传递数据的。 此方法意味着只有一个 (`goroutine`) 有权访问数据，设计上不存在争用条件。

Go 的方法可以总结为以下口号：“不是通过共享内存通信，而是**通过通信共享内存**。”

这种方法意义深远。例如，引用计数通过为整数变量添加互斥锁来很好地实现。 但作为一种高级方法，通过`channel`来控制访问能够让你写出更简洁，正确的程序。

Golang 通过编译器运行时（Runtime），从语言上原生支持了并发编程。

## 并发与并行
学习 go 并发编程之前，我们需要弄清并发、并行的概念。

由于 CPU 同一时间只能执行一个进程/线程，在下文的概念中，我们把进程/线程统称为任务。不同的场景下，任务所指的可能是进程，也可能是线程。

### 并发（Concurrency）

并发是指计算机在同一时间段内执行多个任务。

并发的概念比较宽泛，它单纯是指计算机能够同时执行多个任务。比如我们当前是一个单核的 CPU，但是我们有5个任务，计算机会通过某种算法将 CPU 资源合理的分配给多个任务，从用户角度来看的话就是多个任务在同时执行。前面说的的算法比如“时间片轮转”。

### 并行（Parallelism）

并行是指在同一时刻执行多个任务。
当我们有多个核心的 CPU 的时候，我们同时执行两个任务，就不需要通过“时间片轮转”的方式让多个任务交替执行了，一个 CPU 负责一个任务，同一时刻，多个任务同时执行，这就是并行。

## 协程（Coroutine）

- 轻量级的线程：作用和线程差不多，都是并发执行一些任务的。
- 非抢占式多任务处理，即由协程主动交出控制权。这里需要了解一下抢占式和非抢占式的区别：

    - 抢占式：以线程为例，线程在任何时候都可能被操作系统进行切换，所以线程就叫做抢占式任务处理，即线程没有控制权，任务即使做到一半，哪怕没有做完，也会被操作系统给抢占了，然后切换到其他任务去。
    - 非抢占式：非抢占式的代表就是协程了，协程在任务做到一半的时候可以**主动的交出任务的控制权**，控制权是由协程内部决定，也正是因为这一特性，协程才是轻量级的。需要注意的是，当一个协程不主动交出控制权的时候，可能会造成死锁，也就是说控制权会一直在这个协程内部，程序将长时间等待，无法跳出。

- 编译器/解释器/虚拟机层面的多任务，**非操作系统层面**的，操作系统层面的多任务就只有进程/线程。
- 多个协程可能在一个或多个线程上运行，大多数情况下由调度器决定。
- 子程序（函数调用，比如 func a() {}）是协程的一个特例。

## Goroutine

goroutine 是轻量线程中的并发活动，而不是在操作系统中进行的传统活动

A goroutine has a simple model: it is a function executing concurrently with other goroutines in the same address space

它是轻量级的， 所有消耗几乎就只有栈空间的分配。而且栈最开始是非常小的，所以它们很廉价， 仅在需要时才会随着堆空间的分配（和释放）而变化。

Go 协程在多线程操作系统上可实现多路复用，因此若一个线程阻塞，比如说等待 I/O， 那么其它的线程就会运行。Go 协程的设计隐藏了线程创建和管理的诸多复杂性。

程序执行的第一个 goroutine 是 main() 函数。 如果要创建其他 goroutine，则必须在调用该函数之前使用 go 关键字

When the call completes, the goroutine exits, silently. (The effect is similar to the Unix shell's & notation for running a command in the background.)

```GO
go list.Sort()  // run list.Sort concurrently; don't wait for it.
```
使用匿名函数
```GO
func Announce(message string, delay time.Duration) {
    go func() {
        time.Sleep(delay)
        fmt.Println(message)
    }()  // Note the parentheses - must call the function.
}
```
实践 don't wait for it.
```GO
package main

import (
	"fmt"
	"net/http"
	"time"
)

func main() {
	start := time.Now()

	apis := []string{
		"https://management.azure.com",
		"https://dev.azure.com",
		"https://api.github.com",
		"https://outlook.office.com/",
		"https://api.somewhereintheinternet.com/",
		"https://graph.microsoft.com",
	}

	for _, api := range apis {
		go checkAPI(api)
	}

	elapsed := time.Since(start)
	fmt.Printf("Done! It took %v seconds!\n", elapsed.Seconds())
}

func checkAPI(api string) {
	_, err := http.Get(api)
	if err != nil {
		fmt.Printf("ERROR: %s is down!\n", api)
		return
	}

	fmt.Printf("SUCCESS: %s is up and running!\n", api)
}
```
输出
```
Done! It took 0 seconds!
```
即使看起来 checkAPI 函数没有运行，它实际上是在运行。 它只是**没有时间完成**

在循环之后添加一个睡眠计时器：

```GO
for _, api := range apis {
    go checkAPI(api)
}

time.Sleep(3 * time.Second)
```
输出
```
ERROR: https://api.somewhereintheinternet.com/ is down!
SUCCESS: https://management.azure.com is up and running!
SUCCESS: https://api.github.com is up and running!
SUCCESS: https://graph.microsoft.com is up and running!
SUCCESS: https://outlook.office.com/ is up and running!
SUCCESS: https://dev.azure.com is up and running!
Done! It took 3.0031011 seconds!
```

在`goroutine`之间传递信息需要`Channel`

## Channel

`channel`需要`make`创建

有两种类型 无缓冲和有缓冲
```GO
ch := make(chan int)            // unbuffered channel of integers
cj := make(chan int, 0)         // unbuffered channel of integers
cs := make(chan *os.File, 100)  // buffered channel of pointers to Files
```

**接收者在收到数据前会一直阻塞**。若信道是不带缓冲的，那么在接收者收到值前， 发送者会一直阻塞；若信道是带缓冲的，则发送者仅在值被复制到缓冲区前阻塞； 若缓冲区已满，发送者会一直等待直到某个接收者取出一个值为止。

操作符`<-`位于channel之前是从channel接收数据

操作符`<-`位于channel之后是向channel发送数据

```GO
ch <- x // sends (or writes ) x through channel ch 阻塞
x = <-ch // x receives (or reads) data sent to the channel ch
<-ch // receives data, but the result is discarded
```

使用`chan`阻塞特性, 等待`goroutine`完成:
```GO
c := make(chan int)  // Allocate a channel.
// Start the sort in a goroutine; when it completes, signal on the channel.
go func() {
    list.Sort()
    c <- 1  // Send a signal; value does not matter.
}()
doSomethingForAWhile()
<-c   // Wait for sort to finish; discard sent value.
```
若要关闭通道，使用内置的 close() 函数：
```GO
close(ch)
```

### 非缓冲
若信道是不带缓冲的，那么在接收者收到值前， 发送者会一直阻塞

使用无缓冲 channel 时，可以控制可并发运行的 goroutine 的数量。 例如，你可能要对 API 进行调用，并且想要控制每秒执行的调用次数。 否则，你可能会被阻止。

```GO
package main

import (
	"fmt"
	"net/http"
	"time"
)

func main() {
	start := time.Now()

	apis := []string{
		"https://management.azure.com",
		"https://dev.azure.com",
		"https://api.github.com",
		"https://outlook.office.com/",
		"https://api.somewhereintheinternet.com/",
		"https://graph.microsoft.com",
	}

	ch := make(chan string)
	for _, api := range apis {
		go checkAPI(api, ch)
	}
	time.Sleep(3 * time.Second)
	elapsed := time.Since(start)
	fmt.Printf("Done! It took %v seconds!\n", elapsed.Seconds())
}

func checkAPI(api string, ch chan string) {
	_, err := http.Get(api)
	if err != nil {
		ch <- fmt.Sprintf("ERROR: %s is down!\n", api)
		fmt.Printf("ERROR: %s is down!\n", api)
		return
	}

	ch <- fmt.Sprintf("SUCCESS: %s is up and running!\n", api)
	fmt.Printf("ERROR: %s is down!\n", api)
}
```
输出
```
Done! It took 3.0024393 seconds!
```
ch接收数据一直在阻塞

接收两次 发送两次
```GO
package main

import (
	"fmt"
	"net/http"
	"time"
)

func main() {
	start := time.Now()

	apis := []string{
		"https://management.azure.com",
		"https://dev.azure.com",
		"https://api.github.com",
		"https://outlook.office.com/",
		"https://api.somewhereintheinternet.com/",
		"https://graph.microsoft.com",
	}

	ch := make(chan string)
	for _, api := range apis {
		go checkAPI(api, ch)
	}
	fmt.Print(<-ch)
	fmt.Print(<-ch)
	elapsed := time.Since(start)
	fmt.Printf("Done! It took %v seconds!\n", elapsed.Seconds())
}

func checkAPI(api string, ch chan string) {
	_, err := http.Get(api)
	if err != nil {
		ch <- fmt.Sprintf("ERROR: %s is down!\n", api)
		return
	}

	ch <- fmt.Sprintf("SUCCESS: %s is up and running!\n", api)
}
```
输出
```
ERROR: https://api.somewhereintheinternet.com/ is down!
SUCCESS: https://graph.microsoft.com is up and running!
Done! It took 1.0784033 seconds!
```
保证接收全部数据
```GO
package main

import (
	"fmt"
	"net/http"
	"time"
)

func main() {
	start := time.Now()

	apis := []string{
		"https://management.azure.com",
		"https://dev.azure.com",
		"https://api.github.com",
		"https://outlook.office.com/",
		"https://api.somewhereintheinternet.com/",
		"https://graph.microsoft.com",
	}

	ch := make(chan string)

	for _, api := range apis {
		go checkAPI(api, ch)
	}

	for i := 0; i < len(apis); i++ {
		fmt.Print(<-ch)
	}
	elapsed := time.Since(start)
	fmt.Printf("Done! It took %v seconds!\n", elapsed.Seconds())
}

func checkAPI(api string, ch chan string) {
	_, err := http.Get(api)
	if err != nil {
		ch <- fmt.Sprintf("ERROR: %s is down!\n", api)
		return
	}

	ch <- fmt.Sprintf("SUCCESS: %s is up and running!\n", api)
}
```

### 有缓冲

有缓冲 channel 将发送和接收操作解耦。 它们不会阻止程序，但你必须小心使用，因为可能最终会导致死锁（如前文所述）

即使未从ch接收数据 也可以向ch发送数据;
```GO
package main

import (
	"fmt"
	"net/http"
	"time"
)

func main() {
	start := time.Now()

	apis := []string{
		"https://management.azure.com",
		"https://dev.azure.com",
		"https://api.github.com",
		"https://outlook.office.com/",
		"https://api.somewhereintheinternet.com/",
		"https://graph.microsoft.com",
	}

	ch := make(chan string, 10)
	for _, api := range apis {
		go checkAPI(api, ch)
	}
	time.Sleep(3 * time.Second)
	elapsed := time.Since(start)
	fmt.Printf("Done! It took %v seconds!\n", elapsed.Seconds())
}

func checkAPI(api string, ch chan string) {
	_, err := http.Get(api)
	if err != nil {
		ch <- fmt.Sprintf("ERROR: %s is down!\n", api)
		fmt.Printf("ERROR: %s is down!\n", api)
		return
	}

	ch <- fmt.Sprintf("SUCCESS: %s is up and running!\n", api)
	fmt.Printf("SUCCESS: %s is up and running!\n", api)
}
```

带缓冲的信道可被用作信号量，例如限制吞吐量

进入的请求会被传递给 handle，它向信道内发送一个值，处理请求后将值从信道中取回，以便让该 “信号量” 准备迎接下一次请求。信道缓冲区的容量决定了同时调用 process 的数量上限，因此我们在初始化时首先要填充至它的容量上限

```GO
var sem = make(chan int, MaxOutstanding)

func handle(r *Request) {
    sem <- 1    // Wait for active queue to drain.。上限MaxOutstanding
    process(r)  // May take a long time.
    <-sem       // Done; enable next request to run.
}

func Serve(queue chan *Request) {
    for {
        req := <-queue
        go handle(req)  // 无需等待 handle 结束。
    }
}
```

一旦有 MaxOutstanding 个处理程序正在执行 process，缓冲区已满的信道的操作都暂停接收更多操作，直到至少一个程序完成并从缓冲区接收。

然而，它却有个设计问题：尽管只有 MaxOutstanding 个 Go 协程能同时运行，**但 Serve 还是为每个进入的请求都创建了新的 Go 协程**。其结果就是，若请求来得很快， 该程序就会**无限地消耗资源**。为了弥补这种不足，我们可以通过修改 Serve 来限制创建 Go 协程，这是个明显的解决方案，但要当心我们修复后出现的 Bug。
```GO
func Serve(queue chan *Request) {
    for req := range queue {
        sem <- 1
        go func() {
            process(req) // Buggy; see explanation below.
            <-sem
        }()
    }
}
```

Bug 出现在 Go 的 for 循环中，该循环变量在每次迭代时会被重用，因此 `req` 变量会在所有的 Go 协程间共享，这不是我们想要的。我们需要确保 req 对于每个 Go 协程来说都是唯一的。有一种方法能够做到，就是将 req 的值作为`实参`传入到该 Go 协程的闭包中：
```GO
func Serve(queue chan *Request) {
    for req := range queue {
        sem <- 1
        go func(req *Request) {
            process(req)
            <-sem
        }(req)
    }
}
```
比较前后两个版本，观察该闭包声明和运行中的差别。 另一种解决方案就是以相同的名字创建新的变量，如例中所示：
```GO
func Serve(queue chan *Request) {
    for req := range queue {
        req := req // 为该Go协程创建 req 的新实例。
        sem <- 1
        go func() {
            process(req)
            <-sem
        }()
    }
}
```
它的写法看起来有点奇怪
```GO
req := req
```
但在 Go 中这样做是合法且常见的。你用相同的名字获得了该变量的一个新的版本， 以此来局部地刻意屏蔽循环变量，使它对每个 Go 协程保持唯一。

回到编写服务器的一般问题上来。另一种管理资源的好方法就是启动固定数量的 handle Go 协程，一起从请求信道中读取数据。Go 协程的数量限制了同时调用 process 的数量。Serve 同样会接收一个通知退出的信道， 在启动所有 Go 协程后，它将阻塞并暂停从信道中接收消息。
```GO
func handle(queue chan *Request) {
    for r := range queue {
        process(r)
    }
}

func Serve(clientRequests chan *Request, quit chan bool) {
    // 启动处理程序
    for i := 0; i < MaxOutstanding; i++ {
        go handle(clientRequests)
    }
    <-quit  // 等待通知退出。
}
```