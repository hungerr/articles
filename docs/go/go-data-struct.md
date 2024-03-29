# 数据结构

## 数组

数组是一种**特定类型**且**长度固定**的数据结构。 它们可具有零个或多个元素，你必须在声明或初始化它们时定义大小。 此外，它们一旦创建，就**无法调整大小**。 鉴于这些原因，数组在 Go 程序中并不常用，但它们是`切片`和`映射`的基础。

### 声明数组

必须定义其元素的**数据类型**以及该数组可容纳的**元素数目**
```GO
package main

import "fmt"

func main() {
    var a [3]int
    a[1] = 10
    fmt.Println(a[0])
    fmt.Println(a[1])
    fmt.Println(a[len(a)-1])
}
```

### 初始化数组

使用大括号`{}`来初始化数组
```GO
package main

import "fmt"

func main() {
    cities := [5]string{"New York", "Paris", "Berlin", "Madrid"}
    fmt.Println("Cities:", cities)
}
```
数组长度为5 最新位置包含一个空的字符串

### 数组中的省略号

如果你不知道你将需要多少个位置，但知道你将具有多少数据，那么还有一种声明和初始化数组的方法是使用省略号 (...)
```GO
package main

import "fmt"

func main() {
    cities := [...]string{"New York", "Paris", "Berlin", "Madrid"}
    fmt.Println("Cities:", cities)
}
```
数组长度为4

另一种有趣的数组初始化方法是使用省略号并仅为最新的位置指定值：
```GO
package main

import "fmt"

func main() {
    numbers := [...]int{99: -1}
    fmt.Println("First Position:", numbers[0])
    fmt.Println("Last Position:", numbers[99])
    fmt.Println("Length:", len(numbers))
}
```

### 数组在 Go 和 C 中的主要区别

- 数组是值。将一个数组赋予另一个数组会复制其所有元素。
- 特别地，若将某个数组传入某个函数，它将接收到该数组的一份副本而非指针。
- 数组的大小是其类型的一部分。类型 [10]int 和 [20]int 是不同的。

数组为值的属性很有用，但代价高昂；若你想要 C 那样的行为和效率，你可以传递一个指向该数组的指针。
```GO
func Sum(a *[3]float64) (sum float64) {
    for _, v := range *a {
        sum += v
    }
    return
}

array := [...]float64{7.0, 8.5, 9.1}
x := Sum(&array)  // Note the explicit address-of operator
```
但这并不是 Go 的习惯用法，切片才是。

### 多维数组

```GO
package main

import "fmt"

func main() {
    var twoD [3][5]int
    for i := 0; i < 3; i++ {
        for j := 0; j < 5; j++ {
            twoD[i][j] = (i + 1) * (j + 1)
        }
        fmt.Println("Row", i, twoD[i])
    }
    fmt.Println("\nAll at once:", twoD)
}
```
输出:
```
Row 0 [1 2 3 4 5]
Row 1 [2 4 6 8 10]
Row 2 [3 6 9 12 15]

All at once: [[1 2 3 4 5] [2 4 6 8 10] [3 6 9 12 15]]
```

## 切片

与数组一样，`切片`也是 Go 中的一种数据类型，它表示一系列类型相同的元素。 不过，与数组更重要的区别是切片的**大小是动态**的，不是固定的。

切片是数组或另一个切片之上的数据结构。 我们将源数组或切片称为**基础数组**。 通过切片，可访问整个基础数组，也可仅访问部分元素。

切片保存了对底层数组的引用，若你将某个切片赋予另一个切片，它们会引用同一个数组。 若某个函数将一个切片作为参数传入，则它对该切片元素的修改对调用者而言同样可见， 这可以理解为传递了底层数组的指针

切片只有 3 个组件：

- 指向基础数组中第一个可访问元素的指针。 此元素不一定是数组的第一个元素
- 切片的长度。 切片中的元素数目。`len()`
- 切片的容量。 切片开头与基础数组结束之间的元素数目。`cap()`

### 声明和初始化切片
```GO
package main

import "fmt"

func main() {
    months := []string{"January", "February", "March", "April", "May", "June", "July", "August", "September", "October", "November", "December"}
    quarter1 := months[0:3]
    quarter2 := months[3:6]
    quarter3 := months[6:9]
    quarter4 := months[9:12]
    fmt.Println(quarter1, len(quarter1), cap(quarter1))
    fmt.Println(quarter2, len(quarter2), cap(quarter2))
    fmt.Println(quarter3, len(quarter3), cap(quarter3))
    fmt.Println(quarter4, len(quarter4), cap(quarter4))
}
```
与数组区别是不指定长度

输出
```
[January February March] 3 12
[April May June] 3 9
[July August September] 3 6
[October November December] 3 3
```

切片的零值是 `nil`。

`nil` 切片的长度和容量为 0 且没有底层数组
```GO
package main

import "fmt"

func main() {
	var s []int
	fmt.Println(s, len(s), cap(s))
	if s == nil {
		fmt.Println("nil!")
	}
}
```

`make` 函数会分配一个元素为零值的数组并返回一个引用了它的切片：
```GO
a := make([]int, 5)  // len(a)=5
```

要指定它的容量，需向 make 传入第三个参数：
```GO
b := make([]int, 0, 5) // len(b)=0, cap(b)=5

b = b[:cap(b)] // len(b)=5, cap(b)=5
b = b[1:]      // len(b)=4, cap(b)=4
```

### 切片项

Go 支持切片运算符 s[i:p]

切片容量cap是切片可扩展的程度 与长度len不同

## 追加项

切片的大小不是固定的，而是动态的。 创建切片后，可向其添加更多元素，这样切片就会扩展

Go 提供了内置函数 `func append(s []T, vs ...T) []T`，便于你向切片添加元素。 将要修改的切片和要追加的元素作为值发送给该函数。 然后，`append` 函数会返回一个新的切片，将其存储在变量中
```GO
package main

import "fmt"

func main() {
    var numbers []int
    for i := 0; i < 10; i++ {
        numbers = append(numbers, i)
        fmt.Printf("%d\tcap=%d\t%v\n", i, cap(numbers), numbers)
    }
}
```
输出:
```
0       cap=1   [0]
1       cap=2   [0 1]
2       cap=4   [0 1 2]
3       cap=4   [0 1 2 3]
4       cap=8   [0 1 2 3 4]
5       cap=8   [0 1 2 3 4 5]
6       cap=8   [0 1 2 3 4 5 6]
7       cap=8   [0 1 2 3 4 5 6 7]
8       cap=16  [0 1 2 3 4 5 6 7 8]
9       cap=16  [0 1 2 3 4 5 6 7 8 9]
```

直到第 3 次迭代，此时容量变为 4，切片中只有 3 个元素。 在第 5 次迭代中，容量又变为 8，第 9 次迭代时变为 16

当切片容量不足以容纳更多元素时，切片的**容量将翻倍**

 Go 会自动扩充容量。 需要谨慎操作。 有时，一个切片具有的容量可能比它需要的多得多，这样你将会浪费内存。

## 删除项

 Go 没有内置函数用于从切片中删除元素。 可使用上述切片运算符 `s[i:p]` 来新建一个仅包含所需元素的切片。

 ```GO
 package main

import "fmt"

func main() {
    letters := []string{"A", "B", "C", "D", "E"}
    remove := 2

	if remove < len(letters) {
		fmt.Println("Before", letters, "Remove ", letters[remove])
		letters = append(letters[:remove], letters[remove+1:]...)
		fmt.Println("After", letters)
	}

}
```

## 复制切片

Go 具有内置函数 `copy(dst, src []Type) `用于创建切片的副本

 更改切片中的元素时，基础数组将随之更改

 若要解决此问题，你需要创建一个切片副本，它会在后台生成新的基础数组

 ```GO
 package main

import "fmt"

func main() {
    letters := []string{"A", "B", "C", "D", "E"}
    fmt.Println("Before", letters)

    slice1 := letters[0:2]

    slice2 := make([]string, 3)
    copy(slice2, letters[1:4])

    slice1[1] = "Z"

    fmt.Println("After", letters)
    fmt.Println("Slice2", slice2)
}
```
输出
```
Before [A B C D E]
After [A Z C D E]
Slice2 [B C D]
```

## 映射

Go 中的`映射`是一个`哈希表`，是`键值对`的集合。 映射中所有的键都必须具有`相同的类型`，它们的值也是如此。 不过，可对键和值使用不同的类型

其`键`可以是任何`相等性操作符支持`的类型， 如`整数`、`浮点数`、`复数`、`字符串`、`指针`、`接口`（只要其动态类型支持相等性判断）、`结构`以及`数组`。 **切片不能用作映射键**，因为它们的相等性还未定义。与切片一样，映射也是引用类型。 若将映射传入函数中，并更改了该映射的内容，则此修改对调用者同样可见。

### 声明和初始化映射

若要声明映射，需要使用 `map` 关键字。 然后，定义键和值类型，如下所示：`map[T]T`

```GO
package main

import "fmt"

func main() {
    studentsAge := map[string]int{
        "john": 32,
        "bob":  31,
    }
    fmt.Println(studentsAge)
}
```
输出
```
map[bob:31 john:32]
```

如果不想使用项来初始化映射，可使用内置函数 `make()` 创建空映射：
```GO
studentsAge := make(map[string]int)
```

### 集合

试图通过映射中不存在的键来取值，就会返回与该映射中项的类型对应的零值

集合可实现成一个值类型为 bool 的映射。将该映射中的项置为 true 可将该值放入集合中，此后通过简单的索引操作即可判断是否存在。
```GO
attended := map[string]bool{
    "Ann": true,
    "Joe": true,
    ...
}

if attended[person] { // person不在集合中，返回 false
    fmt.Println(person, "was at the meeting")
}
```

### 添加项

要添加项，无需像对切片一样使用内置函数。 映射更加简单。 你只需定义键和值即可。 如果没有键值对，则该项会添加到映射中。

让我们使用 make 函数重写之前用于创建映射的代码。 然后，将项添加到映射中。 可以使用以下代码：

```GO
package main

import "fmt"

func main() {
    studentsAge := make(map[string]int)
    studentsAge["john"] = 32
    studentsAge["bob"] = 31
    fmt.Println(studentsAge)
}
```
运行代码时，你会获得与之前相同的输出：

输出

```
map[bob:31 john:32]
```

映射的零值为 nil。 如果尝试使用 nil 映射执行相同操作，会出现错误。 例如，以下代码将不起作用：

```GO
package main

import "fmt"

func main() {
    var studentsAge map[string]int
    studentsAge["john"] = 32
    studentsAge["bob"] = 31
    fmt.Println(studentsAge)
}
```
运行上述代码时，会出现以下错误：
```
panic: assignment to entry in nil map

goroutine 1 [running]:
main.main()
        /Users/johndoe/go/src/helloworld/main.go:7 +0x4f
exit status 2
```

若要避免在将项添加到映射时出现问题，请确保使用 `make` 函数（如我们在上述代码片段中所示）创建一个`空映射`（而不是 `nil` 映射）。 此规则仅适用于添加项的情况。 如果在 `nil` 映射中运行`查找`、`删除`或`循环`操作，Go 不会执行 panic。 

### 访问项

若要访问映射中的项，可使用常用的下标表示法 `m[key]`，就像操作数组或切片一样。 下面是一个有关如何访问项的简单示例：

```GO
package main

import "fmt"

func main() {
    studentsAge := make(map[string]int)
    studentsAge["john"] = 32
    studentsAge["bob"] = 31
    fmt.Println("Bob's age is", studentsAge["bob"])
}
```
在映射中使用下标表示法时，即使映射中没有键，你也总会获得响应。 当你访问不存在的项时，Go 不会执行 panic。 此时，你会获得默认值。 可使用以下代码来确认该行为：

试图通过映射中不存在的键来取值，就会返回与该映射中项的类型对应的零值

```GO
package main

import "fmt"

func main() {
    studentsAge := make(map[string]int)
    studentsAge["john"] = 32
    studentsAge["bob"] = 31
    fmt.Println("Christy's age is", studentsAge["christy"])
}
```
运行上述代码时，你会看到以下输出：

```
Christy's age is 0
```

在很多情况下，访问映射中没有的项时 Go 不会返回错误，这是正常的。 但有时需要知道某个项是否存在。 在 Go 中，**映射的下标表示法可生成两个值**。 第一个是项的值。 第二个是指示键是否存在的布尔型标志。

```GO
package main

import "fmt"

func main() {
    studentsAge := make(map[string]int)
    studentsAge["john"] = 32
    studentsAge["bob"] = 31

    age, exist := studentsAge["christy"]
    if exist {
        fmt.Println("Christy's age is", age)
    } else {
        fmt.Println("Christy's age couldn't be found")
    }
}
```

运行上述代码时，你会看到以下输出：

```
Christy's age couldn't be found
```
使用第二个代码片段检查映射中的键在你访问之前是否存在。

### 删除项
若要从映射中删除项，请使用内置函数 `delete()`。 下例演示了如何从映射中删除项：

```GO
package main

import "fmt"

func main() {
    studentsAge := make(map[string]int)
    studentsAge["john"] = 32
    studentsAge["bob"] = 31
    delete(studentsAge, "john")
    fmt.Println(studentsAge)
}
```

运行代码时，你会获得以下输出：
```
map[bob:31]
```
正如上述所言，如果你尝试删除不存在的项，Go 不会执行 panic。 下面是该行为的示例：

```GO
package main

import "fmt"

func main() {
    studentsAge := make(map[string]int)
    studentsAge["john"] = 32
    studentsAge["bob"] = 31
    delete(studentsAge, "christy")
    fmt.Println(studentsAge)
}
```
运行代码时，你不会遇到错误，而且会看到以下输出：

```
map[bob:31 john:32]
```

### 映射中的循环
最后，让我们看看如何在映射中进行循环来以编程方式访问其所有的项。 为此，可使用基于`range`的循环，如下例所示：

```GO
package main

import (
    "fmt"
)

func main() {
    studentsAge := make(map[string]int)
    studentsAge["john"] = 32
    studentsAge["bob"] = 31
    for name, age := range studentsAge {
        fmt.Printf("%s\t%d\n", name, age)
    }
}
```
运行上述代码时，你会看到以下输出：

```
john    32
bob     31
```

请注意可如何将键和值信息存储在不同的变量中。 在本例中，我们将键保存在 name 变量中，将值保存在 age 变量中。 因此，range 会首先生成项的键，然后再生成该项的值。 可使用 `_` 变量忽略其中任何一个，如下例所示：

```GO
package main

import (
    "fmt"
)

func main() {
    studentsAge := make(map[string]int)
    studentsAge["john"] = 32
    studentsAge["bob"] = 31

    for _, age := range studentsAge {
        fmt.Printf("Ages %d\n", age)
    }
}
```
即使在本例中用这种方式打印年龄没有意义，但存在你无需知道项的键的情况。 或者，你可只使用**项的key**，如下例所示：

```GO
package main

import (
    "fmt"
)

func main() {
    studentsAge := make(map[string]int)
    studentsAge["john"] = 32
    studentsAge["bob"] = 31

    for name := range studentsAge {
        fmt.Printf("Names %s\n", name)
    }
}
```

## 使用结构struct

在 Go 中，可使用结构将可能构成记录的**不同字段**组合在一起。

Go 中的结构也是一种数据结构，它可包含**零个或多个任意类型**的字段，并将它们表示为单个实体

### 声明和初始化结构
若要声明结构，需要使用 `struct` 关键字
```GO
type Employee struct {
    ID        int
    FirstName string
    LastName  string
    Address   string
}
```

然后，可像操作其他类型一样使用新类型声明一个变量，如下所示：

```GO
var john Employee
```

如果要在声明变量的同时对其进行初始化，可按以下方式操作：
```GO
employee := Employee{1001, "John", "Doe", "Doe's Street"}
```

请注意，必须为结构中的每个字段指定一个值。 但这有时也可能会导致出现问题。 或者，可更具体地了解要在结构中初始化的字段：
```GO
employee := Employee{LastName: "Doe", FirstName: "John"}
```

如果未指定任何其他字段的值，也并不重要。 Go 将根据字段数据类型分配**默认值**。

最后，可使用 `&` 运算符生成指向结构的指针，如以下代码所示：
```GO
package main

import "fmt"

type Employee struct {
    ID        int
    FirstName string
    LastName  string
    Address   string
}

func main() {
    employee := Employee{LastName: "Doe", FirstName: "John"}
    fmt.Println(employee)
    employeeCopy := &employee
    employeeCopy.FirstName = "David"
    fmt.Println(employee)
}
```

如果我们有一个指向结构体的指针 `p`，那么可以通过 `(*p).X` 来访问其字段 `X`。不过这么写太啰嗦了，所以语言也允许我们使用隐式间接引用，直接写 `p.X` 就可以。

### 结构嵌入
通过 Go 中的结构，可将某`struct`嵌入到另一结构中。
```GO
type Person struct {
    ID        int
    FirstName string
    LastName  string
    Address   string
}

type Employee struct {
    Information Person
    ManagerID   int
}
```
但是，若要引用 Person 结构中的字段，你需要包含员工变量中的 Information 字段，如下例所示：

```GO
var employee Employee
employee.Information.FirstName = "John"
```

如果你要像我们这样重构代码，则会破坏我们的代码。 或者，你可只包含一个与你要嵌入的结构同名的新字段，如下例所示：
```GO
type Employee struct {
    Person
    ManagerID int
}
```

可使用以下代码进行演示：
```GO
package main

import "fmt"

type Person struct {
    ID        int
    FirstName string
    LastName  string
    Address   string
}

type Employee struct {
    Person
    ManagerID int
}

type Contractor struct {
    Person
    CompanyID int
}

func main() {
    employee := Employee{
        Person: Person{
            FirstName: "John",
        },
    }
    employee.LastName = "Doe"
    fmt.Println(employee.FirstName)
}
```

请注意如何在无需指定 Person 字段的情况下访问 Employee 结构中的 FirstName 字段，因为它会自动嵌入其所有字段。 但在你初始化结构时，必须明确要给哪个字段分配值。

### 用 JSON 编码和解码结构
最后，可使用结构来对 JSON 中的数据进行编码和解码。 Go 对 JSON 格式提供很好的支持，该格式已包含在标准库包中。
```GO
type Person struct {
    ID        int    
    FirstName string `json:"name"`
    LastName  string
    Address   string `json:"address,omitempty"`
}
```

然后，若要将结构编码为 JSON，请使用 `json.Marshal` 函数。 若要将 JSON 字符串解码为数据结构，请使用 `json.Unmarshal` 函数。 下例将所有内容组合在一起，将员工数组编码为 JSON，并将输出解码为新的变量：

```GO
package main

import (
    "encoding/json"
    "fmt"
)

type Person struct {
    ID        int
    FirstName string `json:"name"`
    LastName  string
    Address   string `json:"address,omitempty"`
}

type Employee struct {
    Person
    ManagerID int
}

type Contractor struct {
    Person
    CompanyID int
}

func main() {
    employees := []Employee{
        Employee{
            Person: Person{
                LastName: "Doe", FirstName: "John",
            },
        },
        Employee{
            Person: Person{
                LastName: "Campbell", FirstName: "David",
            },
        },
    }

    data, _ := json.Marshal(employees)
    fmt.Printf("%s\n", data)

    var decoded []Employee
    json.Unmarshal(data, &decoded)
    fmt.Printf("%v", decoded)
}
```
