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

接口可实现动态绑定

### 声明接口

Go 中的接口类似于蓝图。 一种抽象类型，只包括具体类型必须拥有或实现的方法。
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