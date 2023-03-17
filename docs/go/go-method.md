# 方法Method与接口Interface

## 方法

Go 中的方法是一种特殊类型的函数，但存在一个简单的区别：你必须在函数名称之前加入一个额外的参数。 此额外参数称为`接收方`。

如你希望分组函数并将其绑定到`自定义类型`，则方法非常有用。 Go 中的这一方法类似于在其他编程语言中`创建类`，因为它允许你实现面向`对象编程 (OOP)` 模型中的某些功能，例如`嵌入`、`重载`和`封装`。

### 声明方法

在声明方法之前，必须先创建结构
```GO
package main

import "fmt"

type triangle struct {
    size int
}

type square struct {
    size int
}

func (t triangle) perimeter() int {
    return t.size * 3
}

func (s square) perimeter() int {
    return s.size * 4
}

func main() {
    t := triangle{3}
    s := square{4}
    fmt.Println("Perimeter (triangle):", t.perimeter())
    fmt.Println("Perimeter (square):", s.perimeter())
}
```

编译器将根据`接收方`类型来确定要调用的函数

### 方法中的指针
- 有时，方法需要更新变量。 
- 或者，如果方法的参数太大，你可能希望避免复制它。 在遇到此类情况时，你需要使用指针传递变量的地址。 在之前的模块中，当我们在讨论指针时提到，每次在 Go 中调用函数时，Go 都会复制每个参数值以便使用。

```GO
func (t *triangle) doubleSize() {
    t.size *= 2
}

func main() {
    t := triangle{3}
    t.doubleSize()
    fmt.Println("Size:", t.size)
    fmt.Println("Perimeter:", t.perimeter())
}
```

如果方法仅可访问接收方的信息，则不需要在接收方变量中使用指针。 但是，依据 Go 的约定，如果结构的任何方法具有指针接收方，则此结构的所有方法都必须具有指针接收方。 即使此结构的某个方法不需要它也是如此。

### 声明其他类型的方法
方法的一个关键方面在于，需要为任何类型定义方法，而不只是针对自定义类型（如结构）进行定义

```GO
package main

import (
    "fmt"
    "strings"
)

type upperstring string

func (s upperstring) Upper() string {
    return strings.ToUpper(string(s))
}

func main() {
    s := upperstring("Learning Go!")
    fmt.Println(s)
    fmt.Println(s.Upper())
}
```

### 嵌入方法
可以在一个结构中使用属性，并将同一属性嵌入另一个结构中。 类似的观点也适用于方法
```GO
type coloredTriangle struct {
    triangle
    color string
}

func main() {
    t := coloredTriangle{triangle{3}, "blue"}
    fmt.Println("Size:", t.size)
    fmt.Println("Perimeter", t.perimeter())
}
```

如果你熟悉 Java 或 C++ 等 OOP 语言，则可能会认为 `triangle` 结构看起来像基类，而 coloredTriangle 是一个子类（如继承），但事实并不是如此。 实际上，Go 编译器会通过创建如下的`包装器方法`来推广 perimeter() 方法：
```GO
func (t coloredTriangle) perimeter() int {
    return t.triangle.perimeter()
}
```

### 重载方法
你可以使用一个同名的方法，只要此方法专门用于要使用的接收方即可。 利用这种区别就是重载方法的方式。

```GO
func (t coloredTriangle) perimeter() int {
    return t.size * 3 * 2
}
```

如果你仍需要从 triangle 结构调用 perimeter() 方法，则可通过对其进行显示访问来执行此操作，如下所示：
```GO
func main() {
    t := coloredTriangle{triangle{3}, "blue"}
    fmt.Println("Size:", t.size)
    fmt.Println("Perimeter (colored)", t.perimeter())
    fmt.Println("Perimeter (normal)", t.triangle.perimeter())
}
```

在 Go 中，你可以 **替代**方法，并在需要时仍访问 **原始** 方法。

### 封装
在其他编程语言中，你会将 private 或 public 关键字放在方法名称之前。 在 Go 中，只需使用**大写标识符，即可公开方法**，使用**非大写的标识符将方法设为私有方法**。

Go 中的封装仅在程序包之间有效。 换句话说，你只能隐藏来自其他程序包的实现详细信息，而不能隐藏程序包本身。

## 接口Interface
接口类似于对象应满足的蓝图或协定。 在你使用接口时，你的基本代码将变得更加灵活、适应性更强，因为你编写的代码未绑定到特定的实现

接口是一系列类型type的集合set

接口可实现动态绑定

接口有三种类型: `Basic interfaces`,`Embedded interfaces`,`General interfaces`

### Basic interfaces

只包含方法
```GO
type Shape interface {
    Perimeter() float64
    Area() float64
}
```

`Shape` 接口表示你想要考虑 `Shape` 的任何类型都需要同时具有 `Perimeter()` 和 `Area()` 方法

### 实现接口
```GO
type Square struct {
    size float64
}

func (s Square) Area() float64 {
    return s.size * s.size
}

func (s Square) Perimeter() float64 {
    return s.size * 4
}

func main() {
    var s Shape = Square{3}
    fmt.Printf("%T\n", s)
    fmt.Println("Area: ", s.Area())
    fmt.Println("Perimeter:", s.Perimeter())
}
```
输出
```
main.Square
Area:  9
Perimeter: 12
```
输出中的对象类型不涉及 `Shape` 接口的任何内容。

无论你是否使用接口，都没有任何区别
```GO
type Circle struct {
    radius float64
}

func (c Circle) Area() float64 {
    return math.Pi * c.radius * c.radius
}

func (c Circle) Perimeter() float64 {
    return 2 * math.Pi * c.radius
}

func printInformation(s Shape) {
    fmt.Printf("%T\n", s)
    fmt.Println("Area: ", s.Area())
    fmt.Println("Perimeter:", s.Perimeter())
    fmt.Println()
}

func main() {
    var s Shape = Square{3}
    printInformation(s)

    c := Circle{6}
    printInformation(c)
}
```
请注意，对于 c 对象，我们不将其指定为 `Shape` 对象。 但是，`printInformation`函数需要一个对象来实现 `Shape` 接口中定义的方法。

### Embedded interfaces
```GO
type Reader interface {
	Read(p []byte) (n int, err error)
	Close() error
}

type Writer interface {
	Write(p []byte) (n int, err error)
	Close() error
}

// ReadWriter's methods are Read, Write, and Close.
type ReadWriter interface {
	Reader  // includes methods of Reader in ReadWriter's method set
	Writer  // includes methods of Writer in ReadWriter's method set

type ReadCloser interface {
	Reader   // includes methods of Reader in ReadCloser's method set
	Close()  // illegal: signatures of Reader.Close and Close are different
}
```

### General interfaces
接口内不光只有方法，还有类型

General interfaces不能用来定义变量，只能用于泛型的类型约束中:
```GO
// An interface representing only the type int.
interface {
	int
}

// An interface representing all types with underlying type int.
interface {
	~int
}

// An interface representing all types with underlying type int that implement the String method.
interface {
	~int
	String() string
}

// An interface representing an empty type set: there is no type that is both an int and a string.
interface {
	int
	string
}

type MyInt int

interface {
	~[]byte  // the underlying type of []byte is itself
	~MyInt   // illegal: the underlying type of MyInt is not MyInt
	~error   // illegal: error is an interface
}

// The Float interface represents all floating-point types
// (including any named types whose underlying types are
// either float32 or float64).
type Float interface {
	~float32 | ~float64
}

interface {
	P                // illegal: P is a type parameter
	int | ~P         // illegal: P is a type parameter
	~int | MyInt     // illegal: the type sets for ~int and MyInt are not disjoint (~int includes MyInt)
	float32 | Float  // overlapping type sets but Float is an interface
}

var x Float                     // illegal: Float is not a basic interface

var x interface{} = Float(nil)  // illegal

type Floatish struct {
	f Float                 // illegal
}

// illegal: Bad may not embed itself
type Bad interface {
	Bad
}

// illegal: Bad1 may not embed itself using Bad2
type Bad1 interface {
	Bad2
}
type Bad2 interface {
	Bad1
}

// illegal: Bad3 may not embed a union containing Bad3
type Bad3 interface {
	~int | ~string | Bad3
}

// illegal: Bad4 may not embed an array containing Bad4 as element type
type Bad4 interface {
	[10]Bad4
}
```

### 实现字符串接口
扩展现有功能的一个简单示例是使用 `Stringer`，它是具有 `String()` 方法的接口，具体如下所示：

```GO
type Stringer interface {
    String() string
}
```
`fmt.Printf` 函数使用此接口来输出值，这意味着你可以编写自定义 `String()` 方法来打印自定义字符串，具体如下所示：
```GO
package main

import "fmt"

type Person struct {
    Name, Country string
}

func (p Person) String() string {
    return fmt.Sprintf("%v is from %v", p.Name, p.Country)
}
func main() {
    rs := Person{"John Doe", "USA"}
    ab := Person{"Mark Collins", "United Kingdom"}
    fmt.Printf("%s\n%s\n", rs, ab)
}
```

### 扩展现有实现


`io.Copy(os.Stdout, resp.Body)`源码:
```GO
func Copy(dst Writer, src Reader) (written int64, err error) {
	return copyBuffer(dst, src, nil)
}
```
参数`dst Writer`是一个接口:
```GO
type Writer interface {
	Write(p []byte) (n int, err error)
}
```
我们可以实现这个接口：
```GO
package main

import (
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "os"
)

type GitHubResponse []struct {
    FullName string `json:"full_name"`
}

type customWriter struct{}

func (w customWriter) Write(p []byte) (n int, err error) {
    var resp GitHubResponse
    json.Unmarshal(p, &resp)
    for _, r := range resp {
        fmt.Println(r.FullName)
    }
    return len(p), nil
}

func main() {
    resp, err := http.Get("https://api.github.com/users/microsoft/repos?page=15&per_page=5")
    if err != nil {
        fmt.Println("Error:", err)
        os.Exit(1)
    }

    writer := customWriter{}
    io.Copy(writer, resp.Body)
}
```

## 通用性
若某种现有的类型仅实现了一个接口，且除此之外并无可导出的方法，则该类型本身就无需导出。 **仅导出该接口**能让我们更专注于其行为而非实现，其它属性不同的实现则能镜像该原始类型的行为。 这也能够避免为每个通用接口的实例重复编写文档。

在这种情况下，构造函数应当**返回一个接口值**而非实现的类型。例如在 `hash` 库中，`crc32.NewIEEE` 和 `adler32.New` 都返回接口类型 `hash.Hash32`。要在 Go 程序中用 `Adler-32` 算法替代 `CRC-32`， 只需修改构造函数调用即可，其余代码则不受算法改变的影响。
```GO
// New creates a new hash.Hash32 computing the CRC-32 checksum using the
// polynomial represented by the Table. Its Sum method will lay the
// value out in big-endian byte order. The returned Hash32 also
// implements encoding.BinaryMarshaler and encoding.BinaryUnmarshaler to
// marshal and unmarshal the internal state of the hash.
func New(tab *Table) hash.Hash32 {
	if tab == IEEETable {
		ieeeOnce.Do(ieeeInit)
	}
	return &digest{0, tab}
}

// New returns a new hash.Hash32 computing the Adler-32 checksum. Its
// Sum method will lay the value out in big-endian byte order. The
// returned Hash32 also implements encoding.BinaryMarshaler and
// encoding.BinaryUnmarshaler to marshal and unmarshal the internal
// state of the hash.
func New() hash.Hash32 {
	d := new(digest)
	d.Reset()
	return d
}

type Hash32 interface {
    Hash
    Sum32() uint32
}
```

### 编写自定义服务器 API

为映射添加方法
```GO
package main

import (
    "fmt"
    "log"
    "net/http"
)

type dollars float32

func (d dollars) String() string {
    return fmt.Sprintf("$%.2f", d)
}

type database map[string]dollars

func (db database) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    for item, price := range db {
        fmt.Fprintf(w, "%s: %s\n", item, price)
    }
}

func main() {
    db := database{"Go T-Shirt": 25, "Go Jacket": 55}
    log.Fatal(http.ListenAndServe("localhost:8000", db))
}
```

## 接口和方法
由于几乎任何类型都能添加方法，因此几乎任何类型都能满足一个接口。一个很直观的例子就是 http 包中定义的 Handler 接口。任何实现了 Handler 的对象都能够处理 HTTP 请求。
```GO
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
```
`ResponseWriter` 接口提供了对方法的访问，这些方法需要响应客户端的请求。 由于这些方法包含了标准的 `Write` 方法，因此 `http.ResponseWriter` 可用于任何 `io.Writer` 适用的场景。`Request` 结构体包含已解析的客户端请求。

为简单起见，我们假设所有的 HTTP 请求都是 GET 方法，而忽略 POST 方法， 这种简化不会影响处理程序的建立方式。这里有个短小却完整的处理程序实现， 它用于记录某个页面被访问的次数。
```GO
// 简单的计数器服务器。
type Counter struct {
    n int
}

func (ctr *Counter) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    ctr.n++
    fmt.Fprintf(w, "counter = %d\n", ctr.n)
}
```

（紧跟我们的主题，注意 Fprintf 如何能输出到 http.ResponseWriter。） 作为参考，这里演示了如何将这样一个服务器添加到 URL 树的一个节点上。
```GO
import "net/http"
...
ctr := new(Counter)
http.Handle("/counter", ctr)
```
但为什么 Counter 要是结构体呢？一个整数就够了。（接收者必须为指针，增量操作对于调用者才可见。）
```GO
// 简单的计数器服务。
type Counter int

func (ctr *Counter) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    *ctr++
    fmt.Fprintf(w, "counter = %d\n", *ctr)
}
```
当页面被访问时，怎样通知你的程序去更新一些内部状态呢？为 Web 页面绑定个信道吧。
```GO
// 每次浏览该信道都会发送一个提醒。
// （可能需要带缓冲的信道。）
type Chan chan *http.Request

func (ch Chan) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    ch <- req
    fmt.Fprint(w, "notification sent")
}
```
最后，假设我们需要输出调用服务器二进制程序时使用的实参 /args。 很简单，写个打印实参的函数就行了。
```GO
func ArgServer() {
    fmt.Println(os.Args)
}
```
如何将它转换为 HTTP 服务器呢？我们可以将 ArgServer 实现为某种可忽略值的方法，不过还有种更简单的方法。 既然我们可以为除指针和接口以外的任何类型定义方法，同样也能为一个函数写一个方法。 http 包包含以下代码：
```GO
// HandlerFunc 类型是一个适配器，
// 它允许将普通函数用做HTTP处理程序。
// 若 f 是个具有适当签名的函数，
// HandlerFunc(f) 就是个调用 f 的处理程序对象。
type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP calls f(w, req).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, req *Request) {
    f(w, req)
}
```
HandlerFunc 是个具有 ServeHTTP 方法的类型， 因此该类型的值就能处理 HTTP 请求。我们来看看该方法的实现：接收者是一个函数 f，而该方法调用 f。这看起来很奇怪，但不必大惊小怪， 区别在于接收者变成了一个信道，而方法通过该信道发送消息。

为了将 ArgServer 实现成 HTTP 服务器，首先我们得让它拥有合适的签名。
```GO
// 实参服务器。
func ArgServer(w http.ResponseWriter, req *http.Request) {
    fmt.Fprintln(w, os.Args)
}
```
ArgServer 和 HandlerFunc 现在拥有了相同的签名， 因此我们可将其转换为这种类型以访问它的方法，就像我们将 Sequence 转换为 IntSlice 以访问 IntSlice.Sort 那样。 建立代码非常简单：
```GO
http.Handle("/args", http.HandlerFunc(ArgServer))
```
当有人访问 /args 页面时，安装到该页面的处理程序就有了值 ArgServer 和类型 HandlerFunc。 HTTP 服务器会以 ArgServer 为接收者，调用该类型的 ServeHTTP 方法，它会反过来调用 ArgServer（通过 f(c, req)），接着实参就会被显示出来。

在本节中，我们通过一个结构体，一个整数，一个信道和一个函数，建立了一个 HTTP 服务器， 这一切都是因为接口只是方法的集合，而几乎任何类型都能定义方法。
