# GO泛型generic

泛型允许程序员在强类型程序设计语言中编写代码时使用一些以后才指定的类型，在实例化时作为参数指明这些类型

## 泛型概念

Go引入了非常多全新的概念

- 类型形参`Type parameter`
- 类型实参`Type argument`
- 类型形参列表`Type parameter list`
- 类型约束`Type constraint`
- 实例化`Instantiations`
- 泛型类型`Generic type`
- 泛型接收器`Generic receiver`
- 泛型函数G`eneric function`

定义一个泛型类型`Generic type`:
```GO
type Slice[T int|float32|float64 ] []T
```

- `T`就是上面介绍过的类型形参`Type parameter`，在定义`Slice`类型的时候`T`代表的具体类型并不确定，类似一个占位符
- `int|float32|float64`这部分被称为类型约束(`Type constraint`)，中间的 `|` 的意思是告诉编译器，类型形参`T`只可以接收`int`或`float32`或`float64`这三种类型的实参
- 中括号里的`T int|float32|float64`这一整串因为定义了所有的类型形参，所以我们称其为 类型形参列表(`type parameter list`) 例子只有一个
- 这里新定义的切片类型名称叫 `Slice[T]` 称之为 泛型类型(`Generic type`)

泛型类型不能直接拿来使用，必须传入类型实参(`Type argument`) 将其确定为具体的类型之后才可使用。而传入类型实参确定具体类型的操作被称为 实例化(`Instantiations`) ：
```GO
// 这里传入了类型实参int，泛型类型Slice[T]被实例化为具体的类型 Slice[int]
var a Slice[int] = []int{1, 2, 3}  
fmt.Printf("Type Name: %T",a)  //输出：Type Name: Slice[int]

// 传入类型实参float32, 将泛型类型Slice[T]实例化为具体的类型 Slice[string]
var b Slice[float32] = []float32{1.0, 2.0, 3.0} 
fmt.Printf("Type Name: %T",b)  //输出：Type Name: Slice[float32]

// ✗ 错误。因为变量a的类型为Slice[int]，b的类型为Slice[float32]，两者类型不同
a = b  

// ✗ 错误。string不在类型约束 int|float32|float64 中，不能用来实例化泛型类型
var c Slice[string] = []string{"Hello", "World"} 

// ✗ 错误。Slice[T]是泛型类型，不可直接使用必须实例化为具体的类型
var x Slice[T] = []int{1, 2, 3} 
```

定义一个**泛型map**:
```GO
// MyMap类型定义了两个类型形参 KEY 和 VALUE。分别为两个形参指定了不同的类型约束
// 这个泛型类型的名字叫： MyMap[KEY, VALUE]
type MyMap[KEY int | string, VALUE float32 | float64] map[KEY]VALUE  

// 用类型实参 string 和 flaot64 替换了类型形参 KEY 、 VALUE，泛型类型被实例化为具体的类型：MyMap[string, float64]
var a MyMap[string, float64] = map[string]float64{
    "jack_score": 9.6,
    "bob_score":  8.4,
}
```

**泛型结构体**:
```GO
// 一个泛型类型的结构体。可用 int 或 sring 类型实例化
type MyStruct[T int | string] struct {  
    Name string
    Data T
}
```

**泛型接口**:
```GO
// 一个泛型接口(关于泛型接口在后半部分会详细讲解）
type IPrintData[T int | float32 | string] interface {
    Print(data T)
}
```

**泛型chan**:
```GO
// 一个泛型通道，可用类型实参 int 或 string 实例化
type MyChan[T int | string] chan T
```

声明一个`General interfaces`:
```GO
type Number interface {
	int64 | float64
}

func main() {
	// Initialize a map for the integer values
	ints := map[string]int64{
		"first":  34,
		"second": 12,
	}

	// Initialize a map for the float values
	floats := map[string]float64{
		"first":  35.98,
		"second": 26.99,
	}

	fmt.Printf("Generic Sums with Constraint: %v and %v\n",
		SumNumbers[string, int64](ints),
		SumNumbers[string, float64](floats))

    // 可去掉ype arguments
    fmt.Printf("Generic Sums, type parameters inferred: %v and %v\n",
        SumIntsOrFloats(ints),
        SumIntsOrFloats(floats))
}

// SumNumbers sums the values of map m. It supports both integers
// and floats as map values.
func SumNumbers[K comparable, V Number](m map[K]V) V {
	var s V
	for _, v := range m {
		s += v
	}
	return s
}
```

类型形参是可以互相套用的，如下
```GO
type WowStruct[T int | float32, S []T] struct {
    Data     S
    MaxValue T
    MinValue T
}
```

## 几种错误

匿名函数不支持泛型:
```GO
// 错误，匿名函数不能自己定义类型实参
fnGeneric := func[T int | float32](a, b T) T {
        return a + b
} 

fmt.Println(fnGeneric(1, 2))
```
但是匿名函数可以使用别处定义好的类型实参，如：
```GO
func MyFunc[T int | float32 | float64](a, b T) {

    // 匿名函数可使用已经定义好的类型形参
    fn2 := func(i T, j T) T {
        return i*2 - j*2
    }

    fn2(a, b)
}
```

方法并不支持泛型:
```GO
type A struct {
}

// 不支持泛型方法
func (receiver A) Add[T int | float32 | float64](a T, b T) T {
    return a + b
}
```
但是因为`receiver`支持泛型， 所以如果想在方法中使用泛型的话，目前唯一的办法就是曲线救国，迂回地通过`receiver`使用类型形参：
```GO
type A[T int | float32 | float64] struct {
}

// 方法可以使用类型定义中的形参 T 
func (receiver A[T]) Add(a T, b T) T {
    return a + b
}

// 用法：
var a A[int]
a.Add(1, 2)

var aa A[float32]
aa.Add(1.0, 2.0)
```

定义泛型类型的时候，基础类型不能只有类型形参
```GO
// 错误，类型形参不能单独使用
type CommonType[T int|string|float32] T

//正确用法
type CommonType interface {
	int|string|float32
}
```

当类型约束的一些写法会被编译器误认为是表达式时会报错。如下：
```GO
//✗ 错误。T *int会被编译器误认为是表达式 T乘以int，而不是int指针
type NewType[T *int] []T
// 上面代码再编译器眼中：它认为你要定义一个存放切片的数组，数组长度由 T 乘以 int 计算得到
type NewType [T * int][]T 

//✗ 错误。和上面一样，这里不光*被会认为是乘号，| 还会被认为是按位或操作
type NewType2[T *int|*float64] []T 

//✗ 错误
type NewType2 [T (int)] []T
```

为了避免这种误解，解决办法就是给类型约束包上 interface{}  或加上逗号消除歧义（关于接口具体的用法会在后半篇提及）
```GO
type NewType[T interface{*int}] []T
type NewType2[T interface{*int|*float64}] []T

// 如果类型约束中只有一个类型，可以添加个逗号消除歧义
type NewType3[T *int,] []T

//✗ 错误。如果类型约束不止一个类型，加逗号是不行的
type NewType4[T *int|*float32,] []T
```

因为上面逗号的用法限制比较大，这里推荐统一用 `interface{}` 解决问题


## 特殊的泛型类型
这里讨论种比较特殊的泛型类型，如下：
```GO
type Wow[T int | string] int

var a Wow[int] = 123     // 编译正确
var b Wow[string] = 123  // 编译正确
var c Wow[string] = "hello" // 编译错误，因为"hello"不能赋值给底层类型int
```

这里虽然使用了类型形参，但因为类型定义是 `type Wow[T int|string] int` ，所以无论传入什么类型实参，实例化后的新类型的底层类型都是 `int` 。所以`int`类型的数字`123`可以赋值给变量`a`和`b`，但`string`类型的字符串 `“hello”` 不能赋值给c

这个例子没有什么具体意义，但是可以让我们理解泛型类型的实例化的机制

## 泛型类型的套娃

泛型和普通的类型一样，可以互相嵌套定义出更加复杂的新类型，如下：
```GO
// 先定义个泛型类型 Slice[T]
type Slice[T int|string|float32|float64] []T

// ✗ 错误。泛型类型Slice[T]的类型约束中不包含uint, uint8
type UintSlice[T uint|uint8] Slice[T]  

// ✓ 正确。基于泛型类型Slice[T]定义了新的泛型类型 FloatSlice[T] 。FloatSlice[T]只接受float32和float64两种类型
type FloatSlice[T float32|float64] Slice[T] 

// ✓ 正确。基于泛型类型Slice[T]定义的新泛型类型 IntAndStringSlice[T]
type IntAndStringSlice[T int|string] Slice[T]  
// ✓ 正确 基于IntAndStringSlice[T]套娃定义出的新泛型类型
type IntSlice[T int] IntAndStringSlice[T] 

// 在map中套一个泛型类型Slice[T]
type WowMap[T int|string] map[string]Slice[T]
// 在map中套Slice[T]的另一种写法
type WowMap2[T Slice[int] | Slice[string]] map[string]T
```

## 泛型用途

### 泛型receiver

可以给泛型类型添加方法
```GO
type MySlice[T int | float32] []T

func (s MySlice[T]) Sum() T {
    var sum T
    for _, value := range s {
        sum += value
    }
    return sum
}

var s MySlice[int] = []int{1, 2, 3, 4}
fmt.Println(s.Sum()) // 输出：10

var s2 MySlice[float32] = []float32{1.0, 2.0, 3.0, 4.0}
fmt.Println(s2.Sum()) // 输出：10.0
```

#### 泛型队列

```GO
// 这里类型约束使用了空接口，代表的意思是所有类型都可以用来实例化泛型类型 Queue[T] (关于接口在后半部分会详细介绍）
type Queue[T any] struct {
	elements []T
}

// 将数据放入队列尾部
func (q *Queue[T]) Put(value T) {
	q.elements = append(q.elements, value)
}

// 从队列头部取出并从头部删除对应数据
func (q *Queue[T]) Pop() (T, bool) {
	var value T
	if len(q.elements) == 0 {
		return value, true
	}

	value = q.elements[0]
	q.elements = q.elements[1:]
	return value, len(q.elements) == 0
}

// 队列大小
func (q Queue[T]) Size() int {
	return len(q.elements)
}

var q1 Queue[int]  // 可存放int类型数据的队列
q1.Put(1)
q1.Put(2)
q1.Put(3)
q1.Pop() // 1
q1.Pop() // 2
q1.Pop() // 3

var q2 Queue[string]  // 可存放string类型数据的队列
q2.Put("A")
q2.Put("B")
q2.Put("C")
q2.Pop() // "A"
q2.Pop() // "B"
q2.Pop() // "C"

var q3 Queue[struct{Name string}] 
var q4 Queue[[]int] // 可存放[]int切片的队列
var q5 Queue[chan int] // 可存放int通道的队列
var q6 Queue[io.Reader] // 可存放接口的队列
// ......
```

####  动态判断变量的类型

type switch和类型断言不能用:
```GO
func (q *Queue[T]) Put(value T) {
    value.(int) // 错误。泛型类型定义的变量不能使用类型断言

    // 错误。不允许使用type switch 来判断 value 的具体类型
    switch value.(type) {
    case int:
        // do something
    case string:
        // do something
    default:
        // do something
    }
    
    // ...
}
```

可使用反射:
```GO
func (receiver Queue[T]) Put(value T) {
    // Printf() 可输出变量value的类型(底层就是通过反射实现的)
    fmt.Printf("%T", value) 

    // 通过反射可以动态获得变量value的类型从而分情况处理
    v := reflect.ValueOf(value)

    switch v.Kind() {
    case reflect.Int:
        // do something
    case reflect.String:
        // do something
    }

    // ...
}
```

这看起来达到了我们的目的，可是当你写出上面这样的代码时候就出现了一个问题：

        你为了避免使用反射而选择了泛型，结果到头来又为了一些功能在在泛型中使用反射

当出现这种情况的时候你可能需要重新思考一下，自己的需求是不是真的需要用泛型（毕竟泛型机制本身就很复杂了，再加上反射的复杂度，增加的复杂度并不一定值得）

### 泛型函数
还有一个可以使用泛型的地方——`泛型函数`

要对不同的类型做同样的事情 需要两个函数 参数接收不同的类型：
```GO
func main() {
	// Initialize a map for the integer values
	ints := map[string]int64{
		"first":  34,
		"second": 12,
	}

	// Initialize a map for the float values
	floats := map[string]float64{
		"first":  35.98,
		"second": 26.99,
	}

	fmt.Printf("Non-Generic Sums: %v and %v\n",
		SumInts(ints),
		SumFloats(floats))
}

// SumInts adds together the values of m.
func SumInts(m map[string]int64) int64 {
	var s int64
	for _, v := range m {
		s += v
	}
	return s
}

// SumFloats adds together the values of m.
func SumFloats(m map[string]float64) float64 {
	var s float64
	for _, v := range m {
		s += v
	}
	return s
}
```

Go支持泛型函数处理不同类型的参数:
```GO
func main() {
	// Initialize a map for the integer values
	ints := map[string]int64{
		"first":  34,
		"second": 12,
	}

	// Initialize a map for the float values
	floats := map[string]float64{
		"first":  35.98,
		"second": 26.99,
	}

	fmt.Printf("Generic Sums: %v and %v\n",
		SumIntsOrFloats(ints),
		SumIntsOrFloats(floats))
}

// SumIntsOrFloats sums the values of map m. It supports both int64 and float64
// as types for map values.
func SumIntsOrFloats[K comparable, V int64 | float64](m map[K]V) V {
	var s V
	for _, v := range m {
		s += v
	}
	return s
}
```

函数的 `形参(parameter)` 只是类似占位符的东西并没有具体的值，只有我们调用函数传入`实参(argument)` 之后才有具体的值。

那么，如果我们将 形参 实参 这个概念推广一下，给变量的类型也引入和类似形参实参的概念的话，问题就迎刃而解：在这里我们将其称之为 `类型形参(type parameter)` 和 `类型实参(type argument)`


- 方括号内声明了两个`类型形参(type parameter)`数`K`和`V`
- comparable代表可用`==`或者`!=`进行比较的类型
- 使用`|`表示两种类型的union
- 传入`类型实参(type argument)`使用

## 组合使用

有时候使用泛型编程时，我们会书写长长的类型约束，如下：
// 一个可以容纳所有int,uint以及浮点类型的泛型切片
```GO
type Slice[T int | int8 | int16 | int32 | int64 | uint | uint8 | uint16 | uint32 | uint64 | float32 | float64] []T
```

理所当然，这种写法是我们无法忍受也难以维护的，而Go支持将类型约束单独拿出来定义到接口中，从而让代码更容易维护：
```GO
type IntUintFloat interface {
    int | int8 | int16 | int32 | int64 | uint | uint8 | uint16 | uint32 | uint64 | float32 | float64
}

type Slice[T IntUintFloat] []T
```

这段代码把类型约束给单独拿出来，写入了接口类型 IntUintFloat  当中。需要指定类型约束的时候直接使用接口 IntUintFloat 即可。

不过这样的代码依旧不好维护，而接口和接口、接口和普通类型之间也是可以通过 | 进行组合：
```GO
type Int interface {
    int | int8 | int16 | int32 | int64
}

type Uint interface {
    uint | uint8 | uint16 | uint32
}

type Float interface {
    float32 | float64
}

type Slice[T Int | Uint | Float] []T  // 使用 '|' 将多个接口类型组合
```
上面的代码中，我们分别定义了 Int, Uint, Float 三个接口类型，并最终在 Slice[T] 的类型约束中通过使用 | 将它们组合到一起。

同时，在接口里也能直接组合其他接口，所以还可以像下面这样：
```GO
type SliceElement interface {
    Int | Uint | Float | string // 组合了三个接口类型并额外增加了一个 string 类型
}

type Slice[T SliceElement] []T 
```

## ~ : 指定底层类型

上面定义的 `Slie[T]` 虽然可以达到目的，但是有一个缺点：
```GO
var s1 Slice[int] // 正确 

type MyInt int
var s2 Slice[MyInt] // ✗ 错误。MyInt类型底层类型是int但并不是int类型，不符合 Slice[T] 的类型约束
```

这里发生错误的原因是，泛型类型 `Slice[T]` 允许的是 `int` 作为类型实参，而不是  `MyInt` （虽然 MyInt 类型底层类型是 int ，但它依旧不是 int 类型）。

为了从根本上解决这个问题，Go新增了一个符号 `~`，在类型约束中使用类似 `~int` 这种写法的话，就代表着不光是 int ，所有以 `int` 为`底层类型`的类型也都可用于实例化。

使用 `~` 对代码进行改写之后如下：
```GO
type Int interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64
}

type Uint interface {
    ~uint | ~uint8 | ~uint16 | ~uint32
}
type Float interface {
    ~float32 | ~float64
}

type Slice[T Int | Uint | Float] []T 

var s Slice[int] // 正确

type MyInt int
var s2 Slice[MyInt]  // MyInt底层类型是int，所以可以用于实例化

type MyMyInt MyInt
var s3 Slice[MyMyInt]  // 正确。MyMyInt 虽然基于 MyInt ，但底层类型也是int，所以也能用于实例化

type MyFloat32 float32  // 正确
var s4 Slice[MyFloat32]
```

限制：使用 `~` 时有一定的限制：

- `~`后面的类型不能为接口
- `~`后面的类型必须为基本类型

```GO
type MyInt int

type _ interface {
    ~[]byte  // 正确
    ~MyInt   // 错误，~后的类型必须为基本类型
    ~error   // 错误，~后的类型不能为接口
}
```

## 从方法集(Method set)到类型集(Type set)

在Go1.18之前，Go官方对 接口`(interface)` 的定义是：接口是一个方法集(`method set`)

    An interface type specifies a method set called its interface

```GO
type ReadWriter interface {
    Read(p []byte) (n int, err error)
    Write(p []byte) (n int, err error)
}
```

我们如果换一个角度来重新思考上面这个接口的话，会发现接口的定义实际上还能这样理解：

我们可以把 `ReaderWriter` 接口看成代表了一个 类型的集合，所有实现了 `Read() Writer()` 这两个方法的类型都在接口代表的类型集合当中

通过换个角度看待接口，在我们眼中接口的定义就从 `方法集(method set)` 变为了 `类型集(type set)`。而Go1.18开始就是依据这一点将接口的定义正式更改为了 `类型集(Type set)`

    An interface type defines a type set (一个接口类型定义了一个类型集)

你或许会觉得，这不就是改了下概念上的定义实际上没什么用吗？是的，如果接口功能没变化的话确实如此。但是还记得下面这种用接口来简化类型约束的写法吗：
```GO
type Float interface {
    ~float32 | ~float64
}

type Slice[T Float] []T 
```

这就体现出了为什么要更改接口的定义了。用 `类型集` 的概念重新理解上面的代码的话就是：

接口类型 `Float` 代表了一个 `类型集合`， 所有以 `float32`  或 `float64` 为底层类型的类型，都在这一类型集之中

而 `type Slice[T Float] []T`  中， `类型约束` 的真正意思是：

    类型约束 指定了类型形参可接受的类型集合，只有属于这个集合中的类型才能替换形参用于实例化

如：
```GO
var s Slice[int]      // int 属于类型集 Float ，所以int可以作为类型实参
var s Slice[chan int] // chan int 类型不在类型集 Float 中，所以错误
```

## 接口实现(implement)定义的变化
既然接口定义发生了变化，那么从Go1.18开始 接口实现`(implement)` 的定义自然也发生了变化：

当满足以下条件时，我们可以说 类型 T 实现了接口 I ( type T implements interface I)：

- T 不是接口时：类型 T 是接口 I 代表的`类型集`中的一个成员 (T is an element of the type set of I)
- T 是接口时： T 接口`代表的类型集`是 I 代表的`类型集的子集`(Type set of T is a subset of the type set of I)


## 类型的并集
并集我们已经很熟悉了，之前一直使用的 `|` 符号就是求类型的`并集( union )`
```GO
type Uint interface {  // 类型集 Uint 是 ~uint 和 ~uint8 等类型的并集
    ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64
}
```

## 类型的交集
接口可以不止书写一行，如果一个接口有多行类型定义，那么取它们之间的 交集
```GO
type AllInt interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64 | ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint32
}

type Uint interface {
    ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64
}

type A interface { // 接口A代表的类型集是 AllInt 和 Uint 的交集
    AllInt
    Uint
}

type B interface { // 接口B代表的类型集是 AllInt 和 ~int 的交集
    AllInt
    ~int
}
```

上面这个例子中

接口 A 代表的是 AllInt 与 Uint 的 交集，即 `~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64`

接口 B 代表的则是 AllInt 和 ~int 的交集，即 `~int`

除了上面的交集，下面也是一种交集：
```GO
type C interface {
    ~int
    int
}
```

很显然，`~int` 和 `int` 的交集只有`int`一种类型，所以接口C代表的类型集中只有`int`一种类型

## 空集
当多个类型的交集如下面 Bad 这样为空的时候， Bad 这个接口代表的类型集为一个空集：
```GO
type Bad interface {
    int
    float32 
} // 类型 int 和 float32 没有相交的类型，所以接口 Bad 代表的类型集为空
```

没有任何一种类型属于空集。虽然 Bad 这样的写法是可以编译的，但实际上并没有什么意义

## 空接口和 any
上面说了空集，接下来说一个特殊的类型集——`空接口 interface{}` 。因为，Go1.18开始接口的定义发生了改变，所以 interface{} 的定义也发生了一些变更：

空接口代表了所有类型的集合

所以，对于Go1.18之后的空接口应该这样理解：

虽然空接口内没有写入任何的类型，但它代表的是`所有类型的集合`，而**非**一个 空集

类型约束中指定 `空接口` 的意思是指定了一个包含所有类型的类型集，并不是类型约束限定了只能使用 空接口 来做类型形参

```GO
// 空接口代表所有类型的集合。写入类型约束意味着所有类型都可拿来做类型实参
type Slice[T interface{}] []T

var s1 Slice[int]    // 正确
var s2 Slice[map[string]string]  // 正确
var s3 Slice[chan int]  // 正确
var s4 Slice[interface{}]  // 正确
```

因为空接口是一个包含了所有类型的类型集，所以我们经常会用到它。于是，Go1.18开始提供了一个和空接口 interface{} 等价的新关键词 `any` ，用来使代码更简单：
```GO
type Slice[T any] []T // 代码等价于 type Slice[T interface{}] []T
```

实际上 any 的定义就位于Go语言的 builtin.go 文件中（参考如下）， any 实际上就是 interaface{} 的别名(alias)，两者完全等价
```GO
// any is an alias for interface{} and is equivalent to interface{} in all ways.
type any = interface{}
```

所以从 Go 1.18 开始，所有可以用到空接口的地方其实都可以直接替换为any，如：
```GO
var s []any // 等价于 var s []interface{}
var m map[string]any // 等价于 var m map[string]interface{}

func MyPrint(value any){
    fmt.Println(value)
}
```

如果你高兴的话，项目迁移到 Go1.18 之后可以使用下面这行命令直接把整个项目中的空接口全都替换成 any。当然因为并不强制，所以到底是用 interface{} 还是 any 全看自己喜好
```
gofmt -w -r 'interface{} -> any' ./...
```

## comparable(可比较) 和 可排序(ordered)
对于一些数据类型，我们需要在类型约束中限制只接受能 != 和 == 对比的类型，如map：

```GO
// 错误。因为 map 中键的类型必须是可进行 != 和 == 比较的类型
type MyMap[KEY any, VALUE any] map[KEY]VALUE
```

所以Go直接`内置`了一个叫 `comparable` 的接口，它代表了所有可用 `!=` 以及 `==` 对比的类型：
```GO
type MyMap[KEY comparable, VALUE any] map[KEY]VALUE // 正确
```
comparable 比较容易引起误解的一点是很多人容易把他与可排序搞混淆。可比较指的是 可以执行 `!= ==` 操作的类型，并**没确保这个类型可以执行大小比较**`>,<,<=,>=`。如下：
```GO
type OhMyStruct struct {
    a int
}

var a, b OhMyStruct

a == b // 正确。结构体可使用 == 进行比较
a != b // 正确

a > b // 错误。结构体不可比大小
```

而可进行大小比较的类型被称为 `Orderd` 。目前Go语言并没有像  `comparable` 这样直接内置对应的关键词，所以想要的话需要自己来定义相关接口，比如我们可以参考Go官方包`golang.org/x/exp/constraints` 如何定义：
```GO
// Ordered 代表所有可比大小排序的类型
type Ordered interface {
    Integer | Float | ~string
}

type Integer interface {
    Signed | Unsigned
}

type Signed interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64
}

type Unsigned interface {
    ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 | ~uintptr
}

type Float interface {
    ~float32 | ~float64
}
```

这里虽然可以直接使用官方包 [golang.org/x/exp/constraints](http://golang.org/x/exp/constraints) ，但因为这个包属于实验性质的 x 包，今后可能会发生非常大变动，所以并不推荐直接使用

## 接口两种类型
我们接下来再观察一个例子，这个例子是阐述接口是类型集最好的例子：
```GO
type ReadWriter interface {
    ~string | ~[]rune

    Read(p []byte) (n int, err error)
    Write(p []byte) (n int, err error)
}
```

最开始看到这一例子你一定有点懵不太理解它代表的意思，但是没关系，我们用类型集的概念就能比较轻松理解这个接口的意思：

接口类型 `ReadWriter` 代表了一个`类型集合`，所有以 `string` 或 `[]rune` 为底层类型，并且实现了 `Read() Write()` 这两个方法的类型都在 `ReadWriter` 代表的类型集当中

如下面代码中，`StringReadWriter` 存在于接口 `ReadWriter` 代表的类型集中，而 `BytesReadWriter` 因为底层类型是 `[]byte`（既不是`string`也是不`[]rune`） ，所以它不属于 `ReadWriter` 代表的类型集
```GO
// 类型 StringReadWriter 实现了接口 Readwriter
type StringReadWriter string 

func (s StringReadWriter) Read(p []byte) (n int, err error) {
    // ...
}

func (s StringReadWriter) Write(p []byte) (n int, err error) {
 // ...
}

//  类型BytesReadWriter 没有实现接口 Readwriter
type BytesReadWriter []byte 

func (s BytesReadWriter) Read(p []byte) (n int, err error) {
 ...
}

func (s BytesReadWriter) Write(p []byte) (n int, err error) {
 ...
}
```

你一定会说，啊等等，这接口也变得太复杂了把，那我定义一个 `ReadWriter` 类型的接口变量，然后接口变量赋值的时候不光要考虑到方法的实现，还必须考虑到具体底层类型？心智负担也太大了吧。是的，为了解决这个问题也为了保持Go语言的兼容性，Go1.18开始将接口分为了两种类型

- 基本接口(Basic interface)
- 一般接口(General interface)

### 基本接口(Basic interface)
接口定义中如果**只有方法**的话，那么这种接口被称为基本接口(Basic interface)。这种接口就是Go1.18之前的接口，用法也基本和Go1.18之前保持一致。基本接口大致可以用于如下几个地方：

最常用的，定义接口变量并赋值
```GO
type MyError interface { // 接口中只有方法，所以是基本接口
    Error() string
}

// 用法和 Go1.18之前保持一致
var err MyError = fmt.Errorf("hello world")
```

基本接口因为也代表了一个类型集，所以也可用在类型约束中
```GO
// io.Reader 和 io.Writer 都是基本接口，也可以用在类型约束中
type MySlice[T io.Reader | io.Writer]  []Slice
```

## 一般接口(General interface)
如果接口内不光只有方法，还有类型的话，这种接口被称为 `一般接口(General interface)` ，如下例子都是一般接口：
```GO
type Uint interface { // 接口 Uint 中有类型，所以是一般接口
    ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64
}

type ReadWriter interface {  // ReadWriter 接口既有方法也有类型，所以是一般接口
    ~string | ~[]rune

    Read(p []byte) (n int, err error)
    Write(p []byte) (n int, err error)
}
```

一般接口类型不能用来定义变量，只能用于泛型的`类型约束`中。所以以下的用法是错误的：
```GO
type Uint interface {
    ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64
}

var uintInf Uint // 错误。Uint是一般接口，只能用于类型约束，不得用于变量定义
```

这一限制保证了一般接口的使用被限定在了泛型之中，不会影响到Go1.18之前的代码，同时也极大减少了书写代码时的心智负担

## 泛型接口
所有类型的定义中都可以使用类型形参，所以接口定义自然也可以使用类型形参，观察下面这两个例子：
```GO
type DataProcessor[T any] interface {
    Process(oriData T) (newData T)
    Save(data T) error
}

type DataProcessor2[T any] interface {
    int | ~struct{ Data interface{} }

    Process(data T) (newData T)
    Save(data T) error
}
```

因为引入了类型形参，所以这两个接口是泛型类型。而泛型类型要使用的话必须传入类型实参实例化才有意义。所以我们来尝试实例化一下这两个接口。因为 T 的类型约束是 any，所以可以随便挑一个类型来当实参(比如string)：
```GO
DataProcessor[string]

// 实例化之后的接口定义相当于如下所示：
type DataProcessor[string] interface {
    Process(oriData string) (newData string)
    Save(data string) error
}
```

经过实例化之后就好理解了， DataProcessor[string] 因为只有方法，所以它实际上就是个 `基本接口(Basic interface)`，这个接口包含两个能处理string类型的方法。像下面这样实现了这两个能处理string类型的方法就算实现了这个接口：
```GO
type CSVProcessor struct {
}

// 注意，方法中 oriData 等的类型是 string
func (c CSVProcessor) Process(oriData string) (newData string) {
    ....
}

func (c CSVProcessor) Save(oriData string) error {
    ...
}

// CSVProcessor实现了接口 DataProcessor[string] ，所以可赋值
var processor DataProcessor[string] = CSVProcessor{}  
processor.Process("name,age\nbob,12\njack,30")
processor.Save("name,age\nbob,13\njack,31")

// 错误。CSVProcessor没有实现接口 DataProcessor[int]
var processor2 DataProcessor[int] = CSVProcessor{}
```

再用同样的方法实例化 `DataProcessor2[T]` ：
```GO
DataProcessor2[string]

// 实例化后的接口定义可视为
type DataProcessor2[T string] interface {
    int | ~struct{ Data interface{} }

    Process(data string) (newData string)
    Save(data string) error
}
```

`DataProcessor2[string]` 因为带有类型并集所以它是 `一般接口(General interface)`，所以实例化之后的这个接口代表的意思是：

只有实现了 `Process(string) string` 和 `Save(string) error` 这两个方法，并且以 `int` 或  `struct{ Data interface{} }` 为底层类型的类型才算实现了这个接口

`一般接口(General interface)` 不能用于变量定义只能用于类型约束，所以接口 `DataProcessor2[string]` 只是定义了一个用于类型约束的类型集

```GO
// XMLProcessor 虽然实现了接口 DataProcessor2[string] 的两个方法，但是因为它的底层类型是 []byte，所以依旧是未实现 DataProcessor2[string]
type XMLProcessor []byte

func (c XMLProcessor) Process(oriData string) (newData string) {

}

func (c XMLProcessor) Save(oriData string) error {

}

// JsonProcessor 实现了接口 DataProcessor2[string] 的两个方法，同时底层类型是 struct{ Data interface{} }。所以实现了接口 DataProcessor2[string]
type JsonProcessor struct {
    Data interface{}
}

func (c JsonProcessor) Process(oriData string) (newData string) {

}

func (c JsonProcessor) Save(oriData string) error {

}

// 错误。DataProcessor2[string]是一般接口不能用于创建变量
var processor DataProcessor2[string]

// 正确，实例化之后的 DataProcessor2[string] 可用于泛型的类型约束
type ProcessorList[T DataProcessor2[string]] []T

// 正确，接口可以并入其他接口
type StringProcessor interface {
    DataProcessor2[string]

    PrintString()
}

// 错误，带方法的一般接口不能作为类型并集的成员(参考6.5 接口定义的种种限制规则
type StringProcessor interface {
    DataProcessor2[string] | DataProcessor2[[]byte]

    PrintString()
}
```

## 接口定义的种种限制规则
Go1.18从开始，在定义类型集(接口)的时候增加了非常多十分琐碎的限制规则，其中很多规则都在之前的内容中介绍过了，但剩下还有一些规则因为找不到好的地方介绍，所以在这里统一介绍下：

用 `|` 连接多个类型的时候，类型之间不能有相交的部分(即必须是不交集):
```GO
type MyInt int

// 错误，MyInt的底层类型是int,和 ~int 有相交的部分
type _ interface {
    ~int | MyInt
}
```

但是相交的类型中是接口的话，则不受这一限制：
```GO
type MyInt int

type _ interface {
    ~int | interface{ MyInt }  // 正确
}

type _ interface {
    interface{ ~int } | MyInt // 也正确
}

type _ interface {
    interface{ ~int } | interface{ MyInt }  // 也正确
}
```

类型的并集中不能有类型形参
```GO
type MyInf[T ~int | ~string] interface {
    ~float32 | T  // 错误。T是类型形参
}

type MyInf2[T ~int | ~string] interface {
    T  // 错误
}
```


接口不能直接或间接地并入自己
```GO
type Bad interface {
    Bad // 错误，接口不能直接并入自己
}

type Bad2 interface {
    Bad1
}
type Bad1 interface {
    Bad2 // 错误，接口Bad1通过Bad2间接并入了自己
}

type Bad3 interface {
    ~int | ~string | Bad3 // 错误，通过类型的并集并入了自己
}
```

接口的并集成员个数大于一的时候不能直接或间接并入 comparable 接口
```GO
type OK interface {
    comparable // 正确。只有一个类型的时候可以使用 comparable
}

type Bad1 interface {
    []int | comparable // 错误，类型并集不能直接并入 comparable 接口
}

type CmpInf interface {
    comparable
}
type Bad2 interface {
    chan int | CmpInf  // 错误，类型并集通过 CmpInf 间接并入了comparable
}
type Bad3 interface {
    chan int | interface{comparable}  // 理所当然，这样也是不行的
}
```

带方法的接口(无论是基本接口还是一般接口)，都不能写入接口的并集中：
```GO
type _ interface {
    ~int | ~string | error // 错误，error是带方法的接口(一般接口) 不能写入并集中
}

type DataProcessor[T any] interface {
    ~string | ~[]byte

    Process(data T) (newData T)
    Save(data T) error
}

// 错误，实例化之后的 DataProcessor[string] 是带方法的一般接口，不能写入类型并集
type _ interface {
    ~int | ~string | DataProcessor[string] 
}

type Bad[T any] interface {
    ~int | ~string | DataProcessor[T]  // 也不行
}
```