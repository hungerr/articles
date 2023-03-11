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

要保证goroutine的完成 可以使用`Channel`

`Channel`在`goroutine`之间传递信息

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

### 非缓冲channel
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

### 缓冲channel

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

### Channel 方向

Go 中的通道具有另一个有趣的功能。 在使用通道作为函数的参数时，可以指定通道是要“发送”数据还是“接收”数据

```GO
chan<- int // it's a channel to only send data
<-chan int // it's a channel to only receive data
```
通过仅接收的 channel 发送数据时，在编译程序时会出现错误。


### select

```GO
SelectStmt = "select" "{" { CommClause } "}" .
CommClause = CommCase ":" StatementList .
CommCase   = "case" ( SendStmt | RecvStmt ) | "default" .
SendStmt   = Channel "<-" Expression .
RecvStmt   = [ ExpressionList "=" | IdentifierList ":=" ] RecvExpr .
RecvExpr   = Expression .
```
每个case可以是SendStmt或者RecvStmt

RecvStmt 可以吧RecvExpr的结果赋值给一个或两个变量

RecvExpr 必须是个receive operation

使用 select 关键字可以与多个通道同时交互

-  channel表达式会立即执行 所有被发送的表达式都会被求值
- 如果多个chan可以使用 随机选择一个接受或者发送数据
- 如果只有一个chan可以使用 选择此chan
- 如果无chan可使用 有default 执行default 无default则一直阻塞


```GO
var a []int
var c, c1, c2, c3, c4 chan int
var i1, i2 int
select {
case i1 = <-c1:
	print("received ", i1, " from c1\n")
case c2 <- i2:
	print("sent ", i2, " to c2\n")
case i3, ok := (<-c3):  // same as: i3, ok := <-c3
	if ok {
		print("received ", i3, " from c3\n")
	} else {
		print("c3 is closed\n")
	}
case a[f()] = <-c4:
	// same as:
	// case t := <-c4
	//	a[f()] = t
default:
	print("no communication\n")
}

for {  // send random sequence of bits to c
	select {
	case c <- 0:  // note: no statement, no fallthrough, no folding of cases
	case c <- 1:
	}
}

select {}  // block forever
```

### Channels of channels

`channel` is a first-class value

它可以被分配并像其它值到处传递.这种特性通常被用来实现安全、并行的多路分解。

每一个请求都能为其回应提供自己的channel
```GO
type Request struct {
    args        []int
    f           func([]int) int
    resultChan  chan int
}

func sum(a []int) (s int) {
    for _, v := range a {
        s += v
    }
    return
}

request := &Request{[]int{3, 4, 5}, sum, make(chan int)}
// Send request
clientRequests <- request
// Wait for response.
fmt.Printf("answer: %d\n", <-request.resultChan)
```

服务端:
```GO
func handle(queue chan *Request) {
    for req := range queue {
        req.resultChan <- req.f(req.args)
    }
}
```
This code is a framework for a rate-limited, parallel, non-blocking RPC system, and there's not a mutex in sight.

### Parallelization并行

多 CPU 核心上实现并行计算

如果计算过程能够被分为几块 可独立执行的过程，它就可以在每块计算结束时向信道发送信号，从而实现并行处理。

对一系列向量项进行极耗资源的操作， 而每个项的值计算是完全独立的。
```GO
type Vector []float64

// Apply the operation to v[i], v[i+1] ... up to v[n-1].
func (v Vector) DoSome(i, n int, u Vector, c chan int) {
    for ; i < n; i++ {
        v[i] += u.Op(v[i])
    }
    c <- 1    // signal that this piece is done
}
```

我们在循环中启动了独立的处理块，每个 CPU 将执行一个处理。 它们有可能以乱序的形式完成并结束，但这没有关系； 我们只需在所有 Go 协程开始后接收，并统计信道中的完成信号即可。
```GO
const numCPU = 4 // number of CPU cores

func (v Vector) DoAll(u Vector) {
    c := make(chan int, numCPU)  // Buffering optional but sensible.
    for i := 0; i < numCPU; i++ {
        go v.DoSome(i*len(v)/numCPU, (i+1)*len(v)/numCPU, u, c)
    }
    // Drain the channel.
    for i := 0; i < numCPU; i++ {
        <-c    // wait for one task to complete
    }
    // All done.
}
```

除了直接设置 numCPU 常量值以外，我们还可以向 runtime 询问一个合理的值

可以返回硬件 CPU 上的核心数量
```GO
var numCPU = runtime.NumCPU()
```

当前最大可用的 CPU 数量

`runtime.GOMAXPROCS`可以设置或者返回之前设置的最大可用的 CPU 数量。

默认情况下使用 runtime.NumCPU 的值。可以通过设置同样名称的环境变量改变，或者调用此函数并传参正整数进行改变。

传参 0 的话会直接返回当前值：

```GO
var numCPU = runtime.GOMAXPROCS(0)
```

### A leaky buffer 限流设计

客户端保存了一个空闲链表，使用一个带缓冲信道表示。若信道为空，就会分配新的缓冲区。 一旦消息缓冲区就绪，它将通过 serverChan 被发送到服务器。
```GO
var freeList = make(chan *Buffer, 100)
var serverChan = make(chan *Buffer)

func client() {
    for {
        var b *Buffer
        // Grab a buffer if available; allocate if not.
        select {
        case b = <-freeList:
            // Got one; nothing more to do.
        default:
            // None free, so allocate a new one.
            b = new(Buffer)
        }
        load(b)              // Read next message from the net.
        serverChan <- b      // Send to server.
    }
} 
```

服务器从客户端循环接收每个消息，处理它们，并将缓冲区返回给空闲列表。
```GO
func server() {
    for {
        b := <-serverChan    // Wait for work.
        process(b)
        // Reuse buffer if there's room.
        select {
        case freeList <- b:
            // Buffer on free list; nothing more to do.
        default:
            // Free list full, just carry on.
        }
    }
}
```
客户端试图从 `freeList` 中获取缓冲区；若没有缓冲区可用， 它就将分配一个新的。服务器将 `b` 放回空闲列表 `freeList` 中直到列表已满，此时缓冲区将被丢弃，并被垃圾回收器回收。（select 语句中的 default 子句在没有条件符合时执行，这也就意味着 selects 永远不会被阻塞。）依靠带缓冲的信道和垃圾回收器的记录， 我们仅用短短几行代码就构建了一个leaky buffer

## sync

### Mutex锁
Go中也有锁的概念

```GO
package main

import (
	"fmt"
	"sync"
	"time"
)

type SafeCounter struct {
	n     int
	mutex sync.Mutex
}

func (c *SafeCounter) Inc() {
	c.mutex.Lock()
	c.n += 1
	c.mutex.Unlock()
}

func main() {
	c := SafeCounter{}
	for i := 0; i < 1000; i += 1 {
		go c.Inc()
	}
	time.Sleep(time.Second)
	fmt.Print(c.n)
}
```

### WaitGroup

`WaitGroup`可以等待一组`goroutines`完成

主`goroutine`调用Add设置`goroutines` 其余各个`goroutines`完成的时候调用`Done`

主`goroutine`调用`Wait`等待其余`goroutines`完成

此种情况可以替代`chan`的通讯作用  特别是`goroutines`数目不明的时候更为方便
```GO
package main

import (
	"fmt"
	"net/http"
	"sync"
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

	wg := sync.WaitGroup{}

	for _, api := range apis {
		wg.Add(1)
		go checkAPI(api, &wg)
	}

	wg.Wait()
	elapsed := time.Since(start)
	fmt.Printf("Done! It took %v seconds!\n", elapsed.Seconds())
}

func checkAPI(api string, wg *sync.WaitGroup) {
	http.Get(api)
	defer wg.Done()
}
```

### Once

`Once` is a fairly simple primitive: the first call to `Do(func())` will cause all other concurrent calls to block until the argument of Do returns. After this happens all blocked calls and successive ones will do nothing and return immediately.

只执行一次
```GO
package main

import (
	"fmt"
	"sync"
)

func main() {
	var once sync.Once
	onceBody := func() {
		fmt.Println("Only once")
	}
	done := make(chan bool)
	for i := 0; i < 10; i++ {
		go func() {
			once.Do(onceBody)
			done <- true
		}()
	}
	for i := 0; i < 10; i++ {
		<-done
	}
}
```

### Pool

sync.Pool 是 Golang 内置的对象池技术，可用于缓存临时对象，避免因频繁建立临时对象所带来的消耗以及对 GC 造成的压力。

在许多知名的开源库中，都可以看到 sync.Pool 的大量使用。例如，HTTP 框架 Gin 用 sync.Pool 来复用每个请求都会创建的 gin.Context 对象。 在 grpc-Go、kubernates 等也都可以看到对 sync.Pool 的身影。

但需要注意的是，sync.Pool 缓存的对象随时可能被无通知的清除，因此不能将 sync.Pool 用于存储持久对象的场景。

`sync.Pool` 在初始化的时候，需要用户提供一个对象的构造函数 `New`。用户使用 `Get` 来从对象池中获取对象，使用 `Put` 将对象归还给对象池

```GO
package main

import (
	"bytes"
	"io"
	"os"
	"sync"
	"time"
)

var bufPool = sync.Pool{
	New: func() any {
		// The Pool's New function should generally only return pointer
		// types, since a pointer can be put into the return interface
		// value without an allocation:
		return new(bytes.Buffer)
	},
}

// timeNow is a fake version of time.Now for tests.
func timeNow() time.Time {
	return time.Unix(1136214245, 0)
}

func Log(w io.Writer, key, val string) {
	b := bufPool.Get().(*bytes.Buffer)
	b.Reset()
	// Replace this with time.Now() in a real logger.
	b.WriteString(timeNow().UTC().Format(time.RFC3339))
	b.WriteByte(' ')
	b.WriteString(key)
	b.WriteByte('=')
	b.WriteString(val)
	w.Write(b.Bytes())
	bufPool.Put(b)
}

func main() {
	Log(os.Stdout, "path", "/search?q=flowers")
}
```

### Cond条件

```GO
c.L.Lock()
for !condition() {
    c.Wait()
}
... make use of condition ...
c.L.Unlock()
```

## time

### func After(d Duration) <-chan Time
```GO
package main

import (
	"fmt"
	"time"
)

var c chan int

func handle(int) {}

func main() {
	select {
	case m := <-c:
		handle(m)
	case <-time.After(10 * time.Second):
		fmt.Println("timed out")
	}
}
```

### func Tick(d Duration) <-chan Time
```GO
package main

import (
	"fmt"
	"time"
)

func statusUpdate() string { return "" }

func main() {
	c := time.Tick(5 * time.Second)
	for next := range c {
		fmt.Printf("%v %s\n", next, statusUpdate())
	}
}
```

### Ticker

```GO
type Ticker struct {
    C <-chan Time // The channel on which the ticks are delivered.
    // contains filtered or unexported fields
}
```

```GO
package main

import (
	"fmt"
	"time"
)

func main() {
	ticker := time.NewTicker(time.Second)
	defer ticker.Stop()
	done := make(chan bool)
	go func() {
		time.Sleep(10 * time.Second)
		done <- true
	}()
	for {
		select {
		case <-done:
			fmt.Println("Done!")
			return
		case t := <-ticker.C:
			fmt.Println("Current time: ", t)
		}
	}
}
```