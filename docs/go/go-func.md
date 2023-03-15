# 函数

## 参数传递

Go里边函数传参只有值传递一种方式

值传递是指在调用函数时将实际参数复制一份传递到函数中，这样在函数中如果对参数进行修改，将不会影响到实际参数。

Go中函数调用只有值传递，但是类型引用有引用类型，他们是：`slice`、`map`、`channel`

为什么我们在函数内部能够修改 `slice` 的原因，因为当它作为参数传入函数时，虽然 `slice` 本身是值拷贝，但是它内部引用了对应数组的结构

`slice`是一个结构体，他的第一个元素是一个`指针`类型:
```GO
type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}
```

## main 函数
Go 中的所有`可执行程序`都具有此函数，因为它是程序的起点。 你的程序中只能有一个 main() 函数。 如果创建的是 `Go 包`，则无需编写 main() 函数

main() 函数没有任何参数，并且不返回任何内容

如要访问 Go 中的命令行参数，可以使用用于保存传递到程序的所有参数的 `os` 包 和 `os.Args` 变量来执行操作
```GO
package main

import (
    "fmt"
    "os"
    "strconv"
)

func main() {
    number1, _ := strconv.Atoi(os.Args[1])
    number2, _ := strconv.Atoi(os.Args[2])
    fmt.Println("Sum:", number1+number2)
}
```

运行程序
```bash
go run main.go 3 5
```

## 自定义函数
```GO
func name(parameters) (results) {
    body-content
}
```
可以指定零个或多个参数。 你还可以定义函数的返回类型，也可以是零个或多个
```GO
func sum(number1 string, number2 string) int {
    int1, _ := strconv.Atoi(number1)
    int2, _ := strconv.Atoi(number2)
    return int1 + int2
}
```

当连续两个或多个函数的已命名形参类型相同时，除最后一个类型以外，其它都可以省略
```GO
func add(x, y int) int {
	return x + y
}
```

在 Go 中，你还可以为函数的**返回值**设置名称，将其当作一个**变量**，它们会被视作定义在函数顶部的变量
```GO
func sum(number1 string, number2 string) (result int) {
    int1, _ := strconv.Atoi(number1)
    int2, _ := strconv.Atoi(number2)
    result = int1 + int2
    return
}
```

你还可以在函数中使用该变量，并且只需在末尾添加 `return` 行.Go 将返回这些返回变量的当前值

返回多个值:
```GO
func calc(number1 string, number2 string) (sum int, mul int) {
    int1, _ := strconv.Atoi(number1)
    int2, _ := strconv.Atoi(number2)
    sum = int1 + int2
    mul = int1 * int2
    return
}
```
直接返回语句应当仅用在这样的短函数中。在长的函数中它们会影响代码的可读性。

## 更改函数参数值（指针）

Go 传递**变量的值**，而不是变量本身

`指针`是包含另一个`变量的内存地址`的变量。 向函数发送指针时，不是传递值，而是**传递内存地址**。 因此，对该变量所做的每个更改都会影响调用方。

在 Go 中，有两个运算符可用于处理指针：

- `&` 使用其后对象的地址。
- `*` 引用指针。 你可以前往指针中包含的地址访问其中的对象。

```GO
package main

import "fmt"

func main() {
    firstName := "John"
    updateName(&firstName)
    fmt.Println(firstName)
}

func updateName(name *string) {
    *name = "David"
}
```

## 参数使用...

如果函数参数使用`p of type ...T`，在函数内部p等同于`type []T`
```GO
func Greeting(prefix string, who ...string)
Greeting("nobody")
Greeting("hello:", "Joe", "Anna", "Eileen")

s := []string{"James", "Jasmine"}
Greeting("goodbye:", s...)
```

## 函数值

函数也是值。它们可以像其它值一样传递。

函数值可以用作函数的参数或返回值。

函数表达式function literal代表一个匿名函数
```GO
FunctionLit = "func" Signature FunctionBody .
func(a, b int, z float64) bool { return a*b < int(z) }
f := func(x, y int) int { return x + y }
func(ch chan int) { ch <- ACK }(replyChan)
```

函数表达式不能有类型参数

函数表达式可以赋值给变量或者直接执行

```GO
package main

import (
	"fmt"
	"math"
)

func compute(fn func(float64, float64) float64) float64 {
	return fn(3, 4)
}

func main() {
	hypot := func(x, y float64) float64 {
		return math.Sqrt(x*x + y*y)
	}
	fmt.Println(hypot(5, 12))

	fmt.Println(compute(hypot))
	fmt.Println(compute(math.Pow))
}
```

## 函数的闭包
Go 函数可以是一个闭包。闭包是一个函数值，它引用了其函数体之外的变量。该函数可以访问并赋予其引用的变量的值，换句话说，该函数被这些变量“绑定”在一起。

```GO
package main

import "fmt"

func adder() func(int) int {
	sum := 0
	return func(x int) int {
		sum += x
		return sum
	}
}

func main() {
	pos, neg := adder(), adder()
	for i := 0; i < 10; i++ {
		fmt.Println(
			pos(i),
			neg(-2*i),
		)
	}
}
```

## init 函数
每个源文件都可以通过定义自己的无参数 `init` 函数来设置一些必要的状态。 （其实每个文件都可以拥有多个 `init` 函数。）而它的结束就意味着初始化结束： 只有该包中的所有变量声明都通过它们的初始化器求值后 `init` 才会被调用， 而包中的变量只有在所有已导入的包都被初始化后才会被求值。

除了那些不能被表示成声明的初始化外，`init` 函数还常被用在程序真正开始执行前，检验或校正程序的状态。
```GO
func init() {
    if user == "" {
        log.Fatal("$USER not set")
    }
    if home == "" {
        home = "/home/" + user
    }
    if gopath == "" {
        gopath = home + "/go"
    }
    // gopath 可通过命令行中的 --gopath 标记覆盖掉。
    flag.StringVar(&gopath, "gopath", gopath, "override default GOPATH")
}
```