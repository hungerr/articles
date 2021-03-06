## 异常控制流

程序计数器从一个指令到下一个指令的过渡称之为控制转移(control transfer)。这样的控制转移序列叫做处理器的控制流(control flow)。

最简单的控制流是一个平滑的序列，上一条指令与下一条指令在内存中都是相邻的。这种平滑流的突变(指令不相邻)通常是由诸如**跳转、调用和返回**这样一些熟悉的程序指令造成。这些指令使得程序能够对由**程序变量表示的内部程序状态**中的变化做出反应。

但是系统也必须能够对**系统状态的变化**做出反应，这些系统状态不是被内部程序变量捕获的，而且也不一定要和程序的执行相关：

- 一个硬件定时器定期产生信号，这个事件必须得到处理。
- 包到达网络适配器后，必须放在内存中。
- 程序向磁盘请求数据，然后休眠，直到被通知说数据已就绪。
- 子进程终止时，父进程必须得到通知。

现代系统通过使控制流发生突变来应对这些情况，这些突变称之为**异常控制流**(Exceptional Control Flow，ECF)。**ECF**发生在计算机系统的各个层次：

- 硬件层，硬件监测到的事件会触发控制突然转移到异常处理程序。
- 操作系统层，内核通过上下文切换将控制从一个用户进程转移到另一个用户进程。
- 应用层，一个进程可以发信号到另一个进程，接收者会将控制转移到它的一个信号处理程序。
- 一个程序可以通过回避通常的栈规则，并执行到其他函数中任意位置的非本地跳转来对错误做出反应。

ECF的意义：

- 理解操作系统：ECF是操作系统用来实现I/O、进程和虚拟内存的基本机制。
- 理解应用程序如何与操作系统交互：应用程序通过使用一个叫做陷阱(trap)或者**系统调用**(system call)的ECF形式，向操作系统请求服务。比如向磁盘写入数据、从网络读取数据、创建一个新进程，以及终止当前进程。
- 编写有趣的新应用程序：操作系统为应用程序提供了强大的ECF机制，用来创建新进程、等待进程终止、通知其他进程系统中的异常事件，以及检测和响应这些事件。
- 理解并发：ECF是计算机系统实现**并发**的基本机制。在运行中并发的例子：中断应用程序执行的异常处理程序，在时间上重叠执行的进程和线程，以及中断应用程序执行的信号处理程序。
- 理解软件异常如何工作。try、catch以及throw。非本地跳转(即违反通常的调用/返回栈规则的跳转)来响应错误情况。非本地跳转是一种应用层ECF，在C中通过setjmp和longjmp函数提供。