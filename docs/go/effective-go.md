# Effective Go

## 格式化

### gofmt
在 Go 中我们另辟蹊径，让机器来处理大部分的格式化问题。`gofmt` 程序（也可用 `go fmt`，它以包为处理对象而非源文件）将 Go 程序按照标准风格缩进、 对齐，保留注释并在需要时重新格式化。

### 缩进

我们使用制表符`TAB`缩进，`gofmt` 默认也使用它。在你认为确实有必要时再使用空格。

### 行的长度

Go 对行的长度没有限制，别担心打孔纸不够长。如果一行实在太长，也可进行折行并插入适当的 tab 缩进。

### 括号

比起 C 和 Java，Go 所需的括号更少：控制结构（if、for 和 switch）在语法上并不需要圆括号。此外，操作符优先级处理变得更加简洁，因此

## 注释

Go 语言支持 C 风格的块注释 `/* */` 和 C++ 风格的行注释 `//`。 行注释更为常用，而块注释则主要用作包的注释，当然也可在禁用一大段代码时使用。

`godoc` 既是一个程序，又是一个 Web 服务器，它对 Go 的源码进行处理，并提取包中的文档内容。 **出现在顶级声明之前，且与该声明之间没有空行的注释**，将与该声明一起被提取出来，作为该条目的说明文档。 这些注释的类型和风格决定了 godoc 生成的文档质量。

每个包都应包含一段包注释，即放置在包子句前的一个块注释。对于包含多个文件的包， 包注释只需出现在其中的任一文件中即可。包注释应在整体上对该包进行介绍，并提供包的相关信息。 它将出现在 godoc 页面中的最上面，并为紧随其后的内容建立详细的文档

若某个包比较简单，包注释同样可以简洁些。
```GO
// path 包实现了一些常用的工具，
// 以便于操作用反斜杠分隔的路径.
```

Go 的声明语法允许成组声明。单个文档注释应介绍一组相关的常量或变量。 由于是整体声明，这种注释往往较为笼统。
```GO
// 表达式解析失败后返回错误代码
var (
    ErrInternal      = errors.New("regexp: internal error")
    ErrUnmatchedLpar = errors.New("regexp: unmatched '('")
    ErrUnmatchedRpar = errors.New("regexp: unmatched ')'")
    ...
)
```

即便是对于私有名称，也可通过成组声明来表明各项间的关系，例如某一组由互斥体保护的变量。
```GO
var (
    countLock   sync.Mutex
    inputCount  uint32
    outputCount uint32
    errorCount  uint32
)
```

## 命名规则

### 包名
当一个包被导入后，包名就会成了内容的访问器。在以下代码
```GO
import "bytes"
```
之后，被导入的包就能通过 `bytes.Buffer` 来引用了。 若所有人都以相同的名称来引用其内容将大有裨益， 这也就意味着包应当有个恰当的名称：其名称应该简洁明了而易于理解。

**包应当以小写**的单个单词来命名，且不应使用下划线或驼峰记法。

不必担心引用次序的冲突。**包名就是导入时所需的唯一默认名称**， 它并不需要在所有源码中保持唯一，即便在少数发生冲突的情况下， **也可为导入的包选择一个别名**来局部使用。 无论如何，通过文件名来判定使用的包，都是不会产生混淆的。

另一个约定就是包名应为其源码目录的基本名称。在 `src/encoding/base64` 中的包应作为 "`encoding/base64`" 导入，其包名应为 `base64`， 而非 `encoding_base64` 或 `encodingBase64`

包的导入者可通过包名来引用其内容，因此包中的可导出名称可以此来避免冲突。例如，`bufio` 包中的缓存读取器类型叫做 `Reader` 而非 `BufReader`，因为用户将它看做 bufio.Reader，这是个清楚而简洁的名称。 

`bufio.Reader` 不会与 `io.Reader` 发生冲突。同样，用于创建 `ring.Ring` 的`新实例`的函数（这就是 Go 中的构造函数）一般会称之为 `NewRing`，但由于 Ring 是该包所导出的唯一类型，且该包也叫 ring，因此它可以只叫做 `New`，它跟在包的后面，就像 `ring.New`。使用包结构可以帮助你选择好的名称。

另一个简短的例子是 `once.Do`，once.Do(setup) 表述足够清晰， 使用 `once.DoOrWaitUntilDone(setup)` 完全就是画蛇添足。 长命名并不会使其更具可读性。一份有用的说明文档通常比额外的长名更有价值。

### 获取器
Go 并不对获取器getter和设置器setter提供自动支持。 你应当自己提供获取器和设置器，通常很值得这样做，但若要将 Get 放到获取器的名字中，既不符合习惯，也没有必要。若你有个名为 `owner` （小写，未导出）的字段，其获取器应当名为 `Owner`（大写，可导出）而非 `GetOwner`。大写字母即为可导出的这种规定为区分方法和字段提供了便利。 若要提供设置器方法，`SetOwner` 是个不错的选择。两个命名看起来都很合理：
```GO
owner := obj.Owner()
if owner != user {
    obj.SetOwner(user)
}
```

### 接口命名
按照约定，只包含一个方法的接口应当以该方法的名称加上 - `er` 后缀来命名，如 R`eader、Writer、 Formatter、CloseNotifier` 等。

诸如此类的命名有很多，遵循它们及其代表的函数名会让事情变得简单。 `Read、Write、Close、Flush、 String` 等都具有典型的签名和意义。为避免冲突，请不要用这些名称为你的方法命名， 除非你明确知道它们的签名和意义相同。

反之，若你的类型实现了的方法， 与一个众所周知的类型的方法拥有相同的含义，那就使用相同的命名。 请将字符串转换方法命名为 `String` 而非 ToString。

### 驼峰命名  非下划线
最后，Go 中的约定是使用 `MixedCaps` 或 `mixedCaps` 而不是下划线来编写多个单词组成的命名

## 分号

和 C 一样，Go 的正式语法使用分号来结束语句，和 C 不同的是，这些分号并不在源码中出现。 取而代之，词法分析器会使用一条简单的规则来自动插入分号，因此源码中基本就不用分号了。

规则是这样的：若在新行前的最后一个标记为标识符（包括 int 和 float64 这类的单词）、数值或字符串常量之类的基本字面或以下标记之一
```GO
break continue fallthrough return ++ -- ) }
```

则词法分析将始终在该标记后面插入分号。这点可以概括为： “如果新行前的标记为语句的末尾，则插入分号”。

分号也可在闭括号之前直接省略，因此像
```GO
    go func() { for { dst <- <-src } }()
```
这样的语句无需分号。通常 Go 程序只在诸如 `for 循环子句`这样的地方使用分号， 以此来将初始化器、条件及增量元素分开。如果你在一行中写多个语句，也需要用分号隔开。

**警告**：无论如何，你都不应将一个控制结构（if、for、switch 或 select）的左大括号放在下一行。如果这样做，就会在大括号前面插入一个分号，这可能引起不需要的效果。 你应该这样写
```GO
if i < f() {
    g()
}
```
而不是这样写
```GO
if i < f()  // 错误！
{           //  错误！
    g()
}
```

## 声明和分配

```GO
f, err := os.Open(name)
```
该语句声明了两个变量 f 和 err。在几行之后，又通过：
```GO
d, err := f.Stat()
```

调用了 `f.Stat`。它看起来似乎是声明了 `d` 和 `err`。 注意，尽管两个语句中都出现了 `err`，但这种重复仍然是合法的：`err` 在第一条语句中被声明，但在第二条语句中只是被再次赋值罢了。也就是说，调用 `f.Stat` 使用的是前面已经声明的 `err`，它只是被重新赋值了而已。

在满足下列条件时，已被声明的变量 `v` 可出现在`:=` 声明中：

- 本次声明与已声明的 v 处于同一作用域中（若 v 已在外层作用域中声明过，则此次声明会创建一个新的变量 §），
- 在初始化中与其类型相应的值才能赋予 v，且在此次声明中**至少另有一个变量是新声明的**。

这个特性简直就是纯粹的实用主义体现，它使得我们可以很方便地只使用一个 `err` 值，例如，在一个相当长的 `if-else` 语句链中， 你会发现它用得很频繁。

值得一提的是，即便 Go 中的函数形参和返回值在词法上处于大括号之外， 但它们的作用域和该函数体仍然相同。

## Data

### new

Go 提供了两种分配原语，即内建函数 `new` 和 `make`

`new`是个用来分配内存的内建函数， 但与其它语言中的同名函数不同，它不会`初始化`内存，只会将内存`置零`。 也就是说，`new(T)` 会为类型为 `T` 的新项分配`已置零的内存空间`， 并返回它的`地址`，也就是一个类型为 `*T` 的值。用 Go 的术语来说，它返回一个`指针`， 该指针**指向新分配的，类型为T的零值**。

既然 new 返回的内存已`置零`，那么当你设计数据结构时， 每种类型的零值就不必进一步初始化了，这意味着该数据结构的使用者只需用 new 创建一个新的对象就能`正常工作`。

例如，`bytes.Buffer` 的文档中提到 “零值的 Buffer 就是已准备就绪的缓冲区。” 同样，`sync.Mutex` 并没有显式的构造函数或 Init 方法， 而是零值的 `sync.Mutex` 就已经被定义为已解锁的互斥锁了。

`零值属性`可以带来各种好处。考虑以下类型声明。
```GO
type SyncedBuffer struct {
    lock    sync.Mutex
    buffer  bytes.Buffer
}
```
`SyncedBuffer` 类型的值也是在声明时就分配好内存就绪了。后续代码中， `p` 和 `v` 无需进一步处理即可正确工作。
```GO
p := new(SyncedBuffer)  // type *SyncedBuffer
var v SyncedBuffer      // type  SyncedBuffer
```

### 构造函数和复合字面量
有时零值还不够好，这时就需要一个初始化`构造函数`，如来自 os 包中的这段代码所示。
```GO
func NewFile(fd int, name string) *File {
    if fd < 0 {
        return nil
    }
    f := new(File)
    f.fd = fd
    f.name = name
    f.dirinfo = nil
    f.nepipe = 0
    return f
}
```
这里显得代码过于冗长。我们可通过`复合字面`来简化它， 该表达式在每次求值时都会创建新的实例。
```GO
func NewFile(fd int, name string) *File {
    if fd < 0 {
        return nil
    }
    f := File{fd, name, nil, 0}
    return &f
}
```

请注意，返回一个局部变量的地址完全没有问题，这点与 C 不同。该局部变量对应的数据 在函数返回后依然有效。实际上，每当获取一个复合字面的地址时，都将为一个新的实例分配内存， 因此我们可以将上面的最后两行代码合并：
```GO
    return &File{fd, name, nil, 0}
```
复合字面的字段必须按顺序全部列出。但如果以 字段:值 对的形式明确地标出元素，初始化字段时就可以按任何顺序出现，未给出的字段值将赋予零值。 因此，我们可以用如下形式：
```GO
    return &File{fd: fd, name: name}
```
少数情况下，若复合字面不包括任何字段，它将创建该类型的零值。表达式 `new(File)` 和 `&File{}` 是等价的。

复合字面同样可用于创建数组、切片以及映射，字段标签是索引还是映射键则视情况而定。 在下例初始化过程中，无论 Enone、Eio 和 Einval 的值是什么，只要它们的标签不同就行。
```GO
a := [...]string   {Enone: "no error", Eio: "Eio", Einval: "invalid argument"}
s := []string      {Enone: "no error", Eio: "Eio", Einval: "invalid argument"}
m := map[int]string{Enone: "no error", Eio: "Eio", Einval: "invalid argument"}
```

### make
再回到内存分配上来。内建函数 `make(T, args)` 的目的不同于 new(T)。它只用于创建 `slice`、`map` 和 `channel`，并返回类型为 `T`（而`非 *T`）的一个`已初始化` （而非置零）的值。 

出现这种用差异的原因在于，这三种类型本质上为`引用数据类型`，它们在使用前`必须初始化`。 例如，切片是一个具有`三项内容的描述符`，包含一个指向（数组内部）数据的`指针`、`长度`以及`容量`， 在这三项被初始化之前，该切片为 `nil`。对于 slice、map 和 channel，make 用于初始化其内部的数据结构并准备好将要使用的值。例如，
```GO
make([]int, 10, 100)
```
会分配一个具有 `100` 个 `int` 的数组空间，接着创建一个长度为 `10`， 容量为 `100` 并指向该数组中前 10 个元素的切片结构。（生成切片时，其容量可以省略，更多信息见切片一节。） 与此相反，`new([]int)` 会返回一个指向新分配的，已`置零`的切片结构， 即一个指向 `nil` 切片值的指针。

下面的例子阐明了 new 和 make 之间的区别：
```GO
var p *[]int = new([]int)       // 分配切片结构；*p == nil；很少用到
var v  []int = make([]int, 100) // 切片 v 现在引用了一个具有 100 个 int 元素的新数组

// 没必要的复杂用法:
var p *[]int = new([]int)
*p = make([]int, 100, 100)

// 常规用法:
v := make([]int, 100)
```
请记住，`make` 只适用于 `map`、`切片`和 `channel` 且**不返回指针**。若要获得明确的指针， 请使用 `new` 分配内存。

### 打印Printing

Go 采用的格式化打印风格和 C 的 `printf` 族类似，但却更加丰富而通用。 这些函数位于 `fmt` 包中，且函数名首字母均为大写：如 `fmt.Printf`、`fmt.Fprintf`，`fmt.Sprintf` 等。 字符串函数（`Sprintf` 等）会返回一个`字符串`，而非填充给定的缓冲区。

你无需提供一个格式字符串。每个 `Printf`、`Fprintf` 和 `Sprintf` 都分别对应另外的函数，如 `Print` 与 `Println`。 这些函数并不接受格式字符串，而是为每个实参生成一种默认格式。`Println` 系列的函数还会在实参中插入空格，并在输出时`追加一个换行符`，而 `Print` 版本仅在操作数两侧都没有字符串时才添加空白。以下示例中各行产生的输出都是一样的。
```GO
fmt.Printf("Hello %d\n", 23)
fmt.Fprint(os.Stdout, "Hello ", 23, "\n")
fmt.Println("Hello", 23)
fmt.Println(fmt.Sprint("Hello ", 23))
```

若你只想要默认的转换，如使用十进制的整数，你可以使用通用的格式 `%v`（对应 “值”）；其结果与 `Print` 和 `Println` 的输出完全相同。此外，这种格式还能打印任意值，甚至包括数组、结构体和映射

对于映射，`Printf` 会自动对映射值按照键的字典顺序排序。

当然，映射中的键可能按任意顺序输出。当打印结构体时，改进的格式 `%+v` 会为结构体的每个字段添上字段名，而另一种格式 `%#v` 将完全按照 `Go` 的语法打印值。
```GO
type T struct {
    a int
    b float64
    c string
}
t := &T{ 7, -2.35, "abc\tdef" }
fmt.Printf("%v\n", t)
fmt.Printf("%+v\n", t)
fmt.Printf("%#v\n", t)
fmt.Printf("%#v\n", timeZone)
```
将打印
```GO
&{7 -2.35 abc   def}
&{a:7 b:-2.35 c:abc     def}
&main.T{a:7, b:-2.35, c:"abc\tdef"}
map[string]int{"CST":-21600, "EST":-18000, "MST":-25200, "PST":-28800, "UTC":0}
```

当遇到 `string` 或 `[]byte` 值时， 可使用 `%q` 产生带引号的字符串；而格式 `%#q` 会尽可能使用反引号。 （`%q` 格式也可用于整数和符文，它会产生一个带单引号的符文常量。） 此外，`%x` 还可用于字符串、字节数组以及整数，并生成一个很长的`十六进制字符串`， 而带空格的格式（`% x`）还会在字节之间插入空格。

另一种实用的格式是 `%T`，它会打印某个值的类型。

## 初始化

### 常量

Go 中的常量就是不变量。它们在编译时创建，即便它们可能是函数中定义的局部变量。 常量只能是数字、字符（符文）、字符串或布尔值。由于编译时的限制， 定义它们的表达式必须也是可被编译器求值的常量表达式。例如 `1<<3` 就是一个常量表达式，而 `math.Sin(math.Pi/4)` 则不是，因为对 `math.Sin` 的函数调用在运行时才会发生。

### 变量

变量能像常量一样初始化，而且可以初始化为一个可在运行时得出结果的普通表达式。
```GO
var (
    home   = os.Getenv("HOME")
    user   = os.Getenv("USER")
    gopath = os.Getenv("GOPATH")
)
```

### init 函数

最后，每个源文件都可以通过定义自己的无参数 init 函数来设置一些必要的状态。 （其实每个文件都可以拥有多个 init 函数。）而它的结束就意味着初始化结束： 只有该包中的所有变量声明都通过它们的初始化器求值后 init 才会被调用， 而包中的变量只有在所有已导入的包都被初始化后才会被求值。

除了那些不能被表示成声明的初始化外，init 函数还常被用在程序真正开始执行前，检验或校正程序的状态。
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

## 空白标识符 `_`

若某次赋值需要匹配多个左值，但其中某个变量不会被程序使用， 那么用空白标识符来代替该变量可避免创建无用的变量，并能清楚地表明该值将被丢弃。 例如，当调用某个函数时，它会返回一个值和一个错误，但只有错误很重要， 那么可使用空白标识符来丢弃无关的值。
```GO
if _, err := os.Stat(path); os.IsNotExist(err) {
    fmt.Printf("%s does not exist\n", path)
}
```

### 未使用的导入和变量

而在程序开发过程中，经常会产生未使用的导入和变量。虽然以后会用到它们， 但为了完成编译又不得不删除它们才行，这很让人烦恼。空白标识符就能提供一个工作空间。

```GO
package main

import (
    "fmt"
    "io"
    "log"
    "os"
)

var _ = fmt.Printf  // 用于调试，结束时删除。
var _ io.Reader    // 用于调试，结束时删除。

func main() {
    fd, err := os.Open("test.go")
    if err != nil {
        log.Fatal(err)
    }
    // TODO: use fd.
    _ = fd
}
```

### 为辅助作用而导入

像前例中 fmt 或 io 这种未使用的导入总应在最后被使用或移除： 空白赋值会将代码标识为工作正在进行中。但有时导入某个包只是为了其辅助作用， 而没有任何明确的使用。例如，在 net/http/pprof 包的 init 函数中记录了 HTTP 处理程序的调试信息。它有个可导出的 API， 但大部分客户端只需要该处理程序的记录和通过 Web 页面访问数据。只为了其辅助作用来导入该包， 只需将包重命名为空白标识符：
```GO
import _ "net/http/pprof"
```

这种导入格式能明确表示该包是为其辅助作用而导入的，因为没有其它使用该包的可能： 在此文件中，它没有名字。（若它有名字而我们没有使用，编译器就会拒绝该程序。）

### 接口检查
就像我们在前面接口中讨论的那样， 一个类型无需显式地声明它实现了某个接口。取而代之，该类型只要实现了某个接口的方法， 其实就实现了该接口。在实践中，大部分接口转换都是静态的，因此会在编译时检测。 例如，将一个 *os.File 传入一个需要 io.Reader 的函数将不会被编译，除非该 *os.File 实现了 io.Reader 接口。

尽管有些接口检查会在运行时进行。encoding/json 包中就有个实例它定义了一个 Marshaler 接口。当 JSON 编码器接收到一个实现了该接口的值，那么该编码器就会调用该值的编组方法， 将其转换为 JSON，而非进行标准的类型转换。 编码器在运行时通过类型断言检查其属性，就像这样：
```GO
m, ok := val.(json.Marshaler)
```

若只需要判断某个类型是否是实现了某个接口，而不需要实际使用接口本身 （可能是错误检查部分），就使用空白标识符来忽略类型断言的值：
```GO
if _, ok := val.(json.Marshaler); ok {
    fmt.Printf("value %v of type %T implements json.Marshaler\n", val, val)
}
```

当需要确保某个包中实现的类型一定满足该接口时，就会遇到这种情况。 若某个类型（例如 json.RawMessage） 需要一种自定义的 JSON 表现时，它应当实现 json.Marshaler， 不过现在没有静态转换可以让编译器去自动验证它。若该类型通过忽略转换失败来满足该接口， 那么 JSON 编码器仍可工作，但它却不会使用自定义的实现。为确保其实现正确， 可在该包中用空白标识符声明一个全局变量：
```GO
var _ json.Marshaler = (*RawMessage)(nil)
```

在此声明中，我们调用了一个 `*RawMessage` 转换并将其赋予了 `Marshaler`，以此来要求 `*RawMessage` 实现 `Marshaler`，这时其属性就会在编译时被检测。 若 `json.Marshaler` 接口被更改，此包将无法通过编译， 而我们则会注意到它需要更新。

在这种结构中出现空白标识符，即表示该声明的存在只是为了类型检查。 不过请不要为满足接口就将它用于任何类型。作为约定， 只有当代码中不存在静态类型转换时才能使用这种声明，毕竟这是种非常罕见的情况。

## 内嵌

Go 并不提供典型的，类型驱动的子类化概念，但通过将**类型内嵌到结构体或接口**中， 它就能 “借鉴” 部分实现。

接口内嵌非常简单:
```GO
type Reader interface {                       //定义读取的接口类型
    Read(p []byte) (n int, err error)       //定义函数   传入[]byte类型  返回一个整型和err
}

type Writer interface {                         //定义写入的接口类型
    Write(p []byte) (n int, err error)       //定义函数   传入[]byte类型  返回一个整型和err
}

// ReadWriter 接口结合了 Reader 接口 和 Writer 接口
type ReadWriter interface {
    Reader
    Writer
}
```

同样的基本想法可以应用在结构体中，但其意义更加深远:
```GO
// ReadWriter 存储了指向 Reader 和 Writer 的指针。
// 它实现了 io.ReadWriter。
type ReadWriter struct {
    *Reader  // *bufio.Reader
    *Writer  // *bufio.Writer
}
```

## 错误

按照约定，错误的类型通常为 error，这是一个内置的简单接口。
```GO
type error interface {
    Error() string
}
```
库的开发者可以自由地用更丰富的模型实现这个接口，这样不仅可以看到错误，还可以提供一些上下文。如前所述，除了通常的 `*os.File` 返回值外，`os.Open` 还返回一个错误值。如果文件被成功打开，错误将为 nil，但是当出现问题时，它将返回一个 `os.PathError` 的错误，就像这样：
```GO
// PathError 记录错误、执行的操作和文件路径
type PathError struct {
    Op string    // "open", "unlink" 等等对文件的操作
    Path string  // 相关文件的路径
    Err error    // 由系统调用返回
}

func (e *PathError) Error() string {
    return e.Op + " " + e.Path + ": " + e.Err.Error()
}
```

## Test

创建测试文件`*_test.go`:
```GO
package bank

import "testing"

func TestAccount(t *testing.T) {

}
```
使用以下命令在详细模式下运行测试：
```
go test -v
```
Go 将查找所有 `*_test.go` 文件来运行测试，因此你应该会看到以下输出：
```
=== RUN   TestAccount
--- PASS: TestAccount (0.00s)
PASS
ok      github.com/msft/bank    0.391s
```
