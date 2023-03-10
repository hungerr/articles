# Go语法

## 变量

### 声明变量
基本用法，使用`var`关键字 `var 变量名 数据类型`
```GO
var firstName string
```
一行声明多个同类型变量
```GO
var firstName, lastName string
```
使用括号一起声明
```GO
var (
    firstName, lastName string
    age int
)
```

### 初始化变量
基本用法
```GO
var (
    firstName string = "John"
    lastName  string = "Doe"
    age       int    = 32
)
```
可以不指定其类型，因为当你使用具体值初始化该变量时，Go 会推断出其类型
```GO
var (
    firstName = "John"
    lastName  = "Doe"
    age       = 32
)
```
单行声明
```GO
var (
    firstName, lastName, age = "John", "Doe", 32
)
```
在函数内使用`:=`声明：
```GO
package main

import "fmt"

func main() {
    firstName, lastName := "John", "Doe"
    age := 32
    fmt.Println(firstName, lastName, age)
}
```

使用`:=`时，要声明的变量必须是新变量。 如果使用`:=`并已经声明该变量，将不会对程序进行编译

在声明函数外的变量时，必须使用 var 关键字执行此操作

在 Go 中， 当你声明一个变量但不使用它时，Go 会抛出错误

可以使用`_`声明不使用的变量。在 Go 中，`_`意味着我们不会使用该变量的值，而是要将其忽略

## 常量

代码中加入静态值，这称为常量

常量的声明与变量类似，只不过是使用 const 关键字。

Go 中的常量就是不变量。它们在`编译时创建`，即便它们可能是函数中定义的局部变量。 常量只能是`数字、字符（符文）、字符串或布尔值`。由于编译时的限制， 定义它们的表达式必须也是可被编译器求值的常量表达式。例如 `1<<3` 就是一个常量表达式，而 `math.Sin(math.Pi/4)` 则不是，因为对 math.Sin 的函数调用在运行时才会发生。

常量可以是`字符、字符串、布尔值或数值`。

常量不能用 `:=` 语法声明。

```GO
const HTTPStatusOK = 200
```

与变量一样，Go 可以通过分配给常量的值推断出类型。 在 Go 中，常量名称通常以**混合大小写字母**或**全部大写字母**书写
```GO
const (
    StatusOK              = 0
    StatusConnectionReset = 1
    StatusOtherError      = 2
)
```

**数值常量**是高精度的值。

一个未指定类型的常量由上下文来决
```GO
package main

import "fmt"

const (
	Big = 1 << 100
	Small = Big >> 99
)

func needInt(x int) int { return x*10 + 1 }
func needFloat(x float64) float64 {
	return x * 0.1
}

func main() {
	fmt.Println(needInt(Small))
	fmt.Println(needFloat(Small))
	fmt.Println(needFloat(Big))
}
```

### iota

在一组常量声明`constant declaration`内可以使用`iota`

`iota`值是所在位置的`index`值 初始值是`0`

一行内可以使用多次`iota` 值是相同的
```GO
const (
	c0 = iota  // c0 == 0
	c1 = iota  // c1 == 1
	c2 = iota  // c2 == 2
)

const (
	a = 1 << iota  // a == 1  (iota == 0)
	b = 1 << iota  // b == 2  (iota == 1)
	c = 3          // c == 3  (iota == 2, unused)
	d = 1 << iota  // d == 8  (iota == 3)
)

const (
	u         = iota * 42  // u == 0     (untyped integer constant)
	v float64 = iota * 42  // v == 42.0  (float64 constant)
	w         = iota * 42  // w == 84    (untyped integer constant)
)

const x = iota  // x == 0
const y = iota  // y == 0

const (
	bit0, mask0 = 1 << iota, 1<<iota - 1  // bit0 == 1, mask0 == 0  (iota == 0)
	bit1, mask1                           // bit1 == 2, mask1 == 1  (iota == 1)
	_, _                                  //                        (iota == 2, unused)
	bit3, mask3                           // bit3 == 8, mask3 == 7  (iota == 3)
)
```

使用iota创建字节大小表示:
```GO
type ByteSize float64

const (
    _           = iota // 通过赋予空白标识符来忽略第一个值
    KB ByteSize = 1 << (10 * iota)
    MB
    GB
    TB
    PB
    EB
    ZB
    YB
)

func (b ByteSize) String() string {
    switch {
    case b >= YB:
        return fmt.Sprintf("%.2fYB", b/YB)
    case b >= ZB:
        return fmt.Sprintf("%.2fZB", b/ZB)
    case b >= EB:
        return fmt.Sprintf("%.2fEB", b/EB)
    case b >= PB:
        return fmt.Sprintf("%.2fPB", b/PB)
    case b >= TB:
        return fmt.Sprintf("%.2fTB", b/TB)
    case b >= GB:
        return fmt.Sprintf("%.2fGB", b/GB)
    case b >= MB:
        return fmt.Sprintf("%.2fMB", b/MB)
    case b >= KB:
        return fmt.Sprintf("%.2fKB", b/KB)
    }
    return fmt.Sprintf("%.2fB", b)
}
```

## 基本数据类型

Go 是一种**强类型**语言。 你声明的每个变量都绑定到特定的数据类型，并且只接受与此类型匹配的值。

Go 有四类数据类型：

- **基本类型**：数字、字符串和布尔值
- **聚合类型**：数组Array和结构Struct
- **引用类型**：指针Pointer、切片Slice、映射Map、函数Function和通道Channel
- **接口类型**：接口Interface

### 整数数字

一般来说，定义整数类型的关键字是 `int`。 但 Go 还提供了 `int8`、`int16`、`int32` 和 `int64` 类型，其大小分别为 8、16、32 或 64 位的整数。 使用 32 位操作系统时，如果只是使用 int，则大小通常为 `32` 位。 在 64 位系统上，int 大小通常为 `64` 位。 但是，此行为可能因计算机而不同。 可以使用 `uint`。 但是，只有在出于某种原因需要将值表示为无符号数字的情况下，才使用此类型。 此外，Go 还提供 `uint8`、`uint16`、`uint32` 和 `uint64` 类型。

int, uint 和 uintptr 在 32 位系统上通常为 32 位宽，在 64 位系统上则为 64 位宽。 当你需要一个整数值时应使用 `int` 类型，除非你有特殊的理由使用固定大小或无符号的整数类型。

需要强制转换时，你需要进行显式转换。 如果尝试在不同类型之间执行数学运算，将会出现错误

`byte` 是 `uint8` 数据类型的别名

`rune` 是 `int32` 数据类型的别名,表示一个 Unicode 码点
```GO
rune := 'G'
fmt.Println(rune)
```
单引号`'`用于表示`rune`

### 浮点数字

Go 提供两种浮点数大小的数据类型：`float32` 和 `float64`。

你可以使用 `math` 包中提供的 `math.MaxFloat32` 和 `math.MaxFloat64` 常量来查找这两种类型的限制

当需要使用十进制数时，浮点类型也很有用:
```GO
const e = 2.71828
const Avogadro = 6.02214129e23
const Planck = 6.62606957e-34
```

### 布尔型

布尔类型仅可能有两个值：`true` 和 `false`。 你可以使用关键字 `bool` 声明布尔类型。 Go 与其他编程语言不同。 在 Go 中，不能将布尔类型隐式转换为 `0` 或 `1`。 你必须显式执行此操作。

因此，你可以按如下方式声明布尔变量：

```GO
var featureFlag bool = true
```

### 字符串

在 Go 中，关键字 `string` 用于表示字符串数据类型。 若要初始化字符串变量，你需要在双引号`"`中定义值。 

```GO
var firstName string = "John"
lastName := "Doe"
fmt.Println(firstName, lastName)
```

单引号`'`用于`单个字符`以及`runes`。单引号里面是单个字符，对应的值为该字符的 `ASCII` 值
```GO
rune := 'G'
fmt.Println(rune)
```

反引号``中的字符表示其原生的意思，在反引号中的内容可以是多行内容，不支持转义。
```GO
func main() {
    a := `Hello golang\n:
I am wz.
Good.`
    fmt.Println(a)
}
```
输出：
```
$ go run main.go
Hello golang\n:
I am random_wz.
Good.
```
可以看到 `\n` 并没有被转义，而是被直接作为字符串输出

有时，你需要对字符进行转义。 为此，在 Go 中，请在字符之前使用反斜杠 `\`。 例如，下面是使用转义字符的最常见示例：

- `\n`：新行
- `\r`：回车符
- `\t`：制表符
- `\'`：单引号
- `\"`：双引号
- `\\`：反斜杠

### 默认值
在 Go 中，如果你不对变量初始化，所有数据类型都有默认值

- `int` 类型的 `0`（及其所有子类型，如 `int64`）
- `float32` 和 `float64` 类型的 `+0.000000e+000`
- `bool` 类型的 `false`
- `string` 类型的空字符串""
- 其余类型均为`nil`

### 类型转换

在 Go 中隐式**强制转换**不起作用。 接下来，需要**显式强制转换**。 Go 提供了将一种数据类型转换为另一种数据类型的一些本机方法。 例如，一种方法是对每个类型使用**内置函数**，如下所示：

```GO
var integer16 int16 = 127
var integer32 int32 = 32767
fmt.Println(int32(integer16) + integer32)
```

Go 的另一种转换方法是使用 `strconv` 包。 例如，若要将 `string` 转换为 `int`，可以使用以下代码，反之亦然：

```GO
package main

import (
    "fmt"
    "strconv"
)

func main() {
    i, _ := strconv.Atoi("-42")
    s := strconv.Itoa(-42)
    fmt.Println(i, s)
}
```

## 指针

Go 拥有指针。指针保存了值的内存地址。

类型 `*T` 是指向 `T` 类型值的指针。其零值为 `nil`。
```GO
var p *int
```

`&` 操作符会生成一个指向其操作数的指针。
```GO
i := 42
p = &i
```

`*` 操作符表示指针指向的底层值。
```GO
fmt.Println(*p) // 通过指针 p 读取 i
*p = 21         // 通过指针 p 设置 i
```

这也就是通常所说的“间接引用”或“重定向”。

与 C 不同，Go 没有指针运算。
