# 控制流

## if/else
最基本的控制流语句

- 你不需要在条件中使用括号
- 可以使用 else if 语句
- 条件语句中可以初始化变量。并且该变量在所有 if 分支中可用。 但是，如果尝试在 if 块之外输出 num 变量的值，则会出现错误

```GO
package main

import "fmt"

func givemeanumber() int {
    return -1
}

func main() {
    if num := givemeanumber(); num < 0 {
        fmt.Println(num, "is negative")
    } else if num < 10 {
        fmt.Println(num, "has only one digit")
    } else {
        fmt.Println(num, "has multiple digits")
    }
}
```

## switch
可以使用 `switch` 语句来避免链接多个 `if` 语句

基本用法：
```GO
package main

import (
    "fmt"
    "math/rand"
    "time"
)

func main() {
    sec := time.Now().Unix()
    rand.Seed(sec)
    i := rand.Int31n(10)

    switch i {
    case 0:
        fmt.Print("zero...")
    case 1:
        fmt.Print("one...")
    case 2:
        fmt.Print("two...")
    }
    default:
        fmt.Print("no match...")
        fmt.Println("ok")
}
```

`case` 语句可包含多个表达式 请使用逗号`,`来分隔表达式
```GO
package main

import "fmt"

func location(city string) (string, string) {
    var region string
    var continent string
    switch city {
    case "Delhi", "Hyderabad", "Mumbai", "Chennai", "Kochi":
        region, continent = "India", "Asia"
    case "Lafayette", "Louisville", "Boulder":
        region, continent = "Colorado", "USA"
    case "Irvine", "Los Angeles", "San Diego":
        region, continent = "California", "USA"
    default:
        region, continent = "Unknown", "Unknown"
    }
    return region, continent
}
func main() {
    region, continent := location("Irvine")
    fmt.Printf("John works in %s, %s\n", region, continent)
}
```

switch 还可以调用函数
```GO
package main

import (
    "fmt"
    "time"
)

func main() {
    switch time.Now().Weekday().String() {
    case "Monday", "Tuesday", "Wednesday", "Thursday", "Friday":
        fmt.Println("It's time to learn some Go.")
    default:
        fmt.Println("It's the weekend, time to rest!")
    }

    fmt.Println(time.Now().Weekday().String())
```

可以在 `switch` 语句中省略条件。此模式类似于比较 `true` 值，就像强制 `switch` 语句一直运行一样。
```GO
package main

import (
    "fmt"
    "math/rand"
    "time"
)

func main() {
    rand.Seed(time.Now().Unix())
    r := rand.Float64()
    switch {
    case r > 0.1:
        fmt.Println("Common case, 90% of the time")
    default:
        fmt.Println("10% of the time")
    }
}
```

在 `Go` 中，当逻辑进入某个 `case` 时，它会退出 `switch` 块，除非你显式停止它。 若要使逻辑进入到下一个紧邻的 `case`，请使用 `fallthrough` 关键字
```GO
package main

import (
    "fmt"
)

func main() {
    switch num := 15; {
    case num < 50:
        fmt.Printf("%d is less than 50\n", num)
        fallthrough
    case num > 100:
        fmt.Printf("%d is greater than 100\n", num)
        fallthrough
    case num < 200:
        fmt.Printf("%d is less than 200", num)
    }
}
```

## for 表达式循环遍历数据
`for` 循环表达式不需要括号。 但是，大括号`{}`是必需的。

分号 (`;`) 分隔 `for` 循环的三个组件：
- 在第一次迭代之前执行的初始语句（可选）。
- 在每次迭代之前计算的条件表达式。 该条件为 false 时，循环会停止。
- 在每次迭代结束时执行的后处理语句（可选）。

```GO
func main() {
    sum := 0
    for i := 1; i <= 100; i++ {
        sum += i
    }
    fmt.Println("sum of 1..100 is", sum)
}
```

`Go` 没有 `while` 关键字。 但可以改用 `for` 循环，并利用 `Go` 使预处理语句和后处理语句可选这一事实
```GO
package main

import (
    "fmt"
    "math/rand"
    "time"
)

func main() {
    var num int64
    rand.Seed(time.Now().Unix())
    for num != 5 {
        num = rand.Int63n(15)
        fmt.Println(num)
    }
}
```
可以在 Go 中编写的另一种循环模式是无限循环
```GO
package main

import (
    "fmt"
    "math/rand"
    "time"
)

func main() {
    var num int32
    sec := time.Now().Unix()
    rand.Seed(sec)

    for {
        fmt.Print("Writing inside the loop...")
        if num = rand.Int31n(10); num == 5 {
            fmt.Println("finish!")
            break
        }
        fmt.Println(num)
    }
}
```

- 若要使逻辑退出循环，请使用 `break` 关键字
- 可以使用 `continue` 关键字跳过循环的当前迭代。

`for` 循环的 `range` 形式可遍历切片或映射。

当使用 `for` 循环遍历切片时，每次迭代都会返回两个值。第一个值为当前元素的下标，第二个值为该下标所对应元素的一份副本。
```GO
package main

import "fmt"

var pow = []int{1, 2, 4, 8, 16, 32, 64, 128}

func main() {
	for i, v := range pow {
		fmt.Printf("2**%d = %d\n", i, v)
	}
}
```

可以将下标或值赋予 `_` 来忽略它。
```GO
for i, _ := range pow
for _, value := range pow
```

若你只需要索引，忽略第二个变量即可。
```GO
for i := range pow
```

## defer

在 Go 中，`defer` 语句会推迟函数（包括任何参数）的运行，直到包含 `defer` 语句的函数完成。 通常情况下，当你想要避免忘记任务（例如关闭文件或运行清理进程）时，可以推迟某个函数的运行。

可以根据需要推迟任意多个函数。 `defer` 语句按逆序运行，先运行最后一个，最后运行第一个。

延迟调用输出的顺序相反（**后进先出**），因为它们是从栈中弹出的。

```GO
package main

import "fmt"

func main() {
    for i := 1; i <= 4; i++ {
        defer fmt.Println("deferred", -i)
        fmt.Println("regular", i)
    }
}
```
TO:
```
regular 1
regular 2
regular 3
regular 4
deferred -4
deferred -3
deferred -2
deferred -1
```

defer 函数的一个典型用例是在使用完文件后将其关闭。 下面是一个示例：
```GO
package main

import (
    "io"
    "os"
    "fmt"
)

func main() {
    newfile, error := os.Create("learnGo.txt")
    if error != nil {
        fmt.Println("Error: Could not create file.")
        return
    }
    defer newfile.Close()

    if _, error = io.WriteString(newfile, "Learning Go!"); error != nil {
	    fmt.Println("Error: Could not write to file.")
        return
    }

    newfile.Sync()
}
```

## panic 函数

内置 `panic()` 函数可以停止 Go 程序中的正常控制流。 当你使用 `panic` 调用时，**任何延迟的函数调用都将正常运行**。 进程会在堆栈中继续，直到所有函数都返回。 然后，程序会崩溃并记录日志消息。 此消息包含错误信息和堆栈跟踪，有助于诊断问题的根本原因。
```GO
package main

import "fmt"

func highlow(high int, low int) {
    if high < low {
        fmt.Println("Panic!")
        panic("highlow() low greater than high")
    }
    defer fmt.Println("Deferred: highlow(", high, ",", low, ")")
    fmt.Println("Call: highlow(", high, ",", low, ")")

    highlow(high, low + 1)
}

func main() {
    highlow(2, 0)
    fmt.Println("Program finished successfully!")
}
```
输出如下：

```
Call: highlow( 2 , 0 )
Call: highlow( 2 , 1 )
Call: highlow( 2 , 2 )
Panic!
Deferred: highlow( 2 , 2 )
Deferred: highlow( 2 , 1 )
Deferred: highlow( 2 , 0 )
panic: highlow() low greater than high

goroutine 1 [running]:
main.highlow(0x2, 0x3)
	/tmp/sandbox/prog.go:13 +0x34c
main.highlow(0x2, 0x2)
	/tmp/sandbox/prog.go:18 +0x298
main.highlow(0x2, 0x1)
	/tmp/sandbox/prog.go:18 +0x298
main.highlow(0x2, 0x0)
	/tmp/sandbox/prog.go:18 +0x298
main.main()
	/tmp/sandbox/prog.go:6 +0x37

Program exited: status 2.
```

## recover

Go 提供内置 `recover()` 函数，让你可以在程序崩溃之后重新获得控制权。 

你只会在你同时调用 `defer` 的函数中调用 `recover`。 

如果调用 `recover()` 函数，则在正常运行的情况下，它会返回 `nil`，没有任何其他作用。
```GO
func main() {
    defer func() {
	    handler := recover()
        if handler != nil {
            fmt.Println("main(): recover", handler)
        }
    }()

    highlow(2, 0)
    fmt.Println("Program finished successfully!")
}
```
运行程序时，输出应该如下所示：

```
Call: highlow( 2 , 0 )
Call: highlow( 2 , 1 )
Call: highlow( 2 , 2 )
Panic!
Deferred: highlow( 2 , 2 )
Deferred: highlow( 2 , 1 )
Deferred: highlow( 2 , 0 )
main(): recover from panic highlow() low greater than high

Program exited.
```

不再看到堆栈跟踪错误。

`panic` 和 `recover` 函数的组合是 `Go` 处理异常的惯用方式。 其他编程语言使用 `try/catch` 块。 `Go` 首选此处所述的方法。
