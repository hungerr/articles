# Go类型转换与类型推导

## 类型转换

### 显式强制转换

在 Go 中隐式**强制转换**不起作用。 
```GO
var a int = 1
var b float64 = 1.0
fmt.Println(a + b) // invalid operation: a + b (mismatched types int and float64)
```

需要**显式强制转换**。 Go 提供了将一种数据类型转换为另一种数据类型的一些本机方法。 例如，一种方法是对每个类型使用**内置函数**，如下所示：

```GO
var integer16 int16 = 127
var integer32 int32 = 32767
fmt.Println(int32(integer16) + integer32)
```

内置数字类型之间可以相互转换：
```GO
i := 42
f := float64(i)
u := uint(f)
```

不兼容类型是不可以转换的:
```go
a := 1.1
b := int(a) // 可行 a float64
c := int(1.1) // cannot convert 1.1 (untyped float constant) to int
d := int(1.0) // 可行
```
### 常量的隐式类型转换

未命名常量只在编译期间存在，不会存储在内存中；而命名常量存在于内存静态区，不允许修改。考虑如下代码:

```GO
const k = 5
```
`5`就是未命名常量，而`k`即为命名常量，当编译后`k`的值为`5`，而等号右边的`5`不再存在。


**常量不允许取址**
```GO
const k = 5
addr := &k // invalid operation: cannot take address of k (untyped int constant 5)
```

兼容的类型可以进行**隐式转换**。例如
```GO
const c int = 123
var c int = 123.0
const c int = 123.1 // cannot use 123.1 (untyped float constant) as int value in constant declaration (truncated)

const c float64 = 123.0
const c float64 = 123
```

**运算中的隐式转换**:

- 除位运算、未定义常量外，运算符两边的操作数类型必须相同
- 如果运算符两边是不同类型的未定义常量(`untyped constant`)，则隐式转换的优先级为: `int` < `rune` < `float` < `Imag`
- 已定义类型的常量都不会发生类型转换。换言之，编译器不允许对变量标识符引用的值进行强制类型转换。即无关优先级

```GO
const c = 1 / 2    // 1和2类型相同，无隐式转换发生
const c = 1 / 2.0  // 整数优先转换为浮点数1.0， c的结果为0.5(float64)

const a int = 1
const c = a * 1.0 // 可行 untyped float constant 1.0转为int
const c = a * 1.1 // *左边的a是已定义类型的常量，因此1.1将被转换为int，但浮点数1.1与整形不兼容，无法进行转换，因此编译器会报错
//  (untyped float constant) truncated to int 
```

### 数字和字符串转换
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

## 类型推导
在声明一个变量而不指定其类型时（即使用不带类型的 := 语法或 var = 表达式语法），变量的类型由右值推导得出。

golang类型推断可以省略类型，像写动态语言代码一样，让编程变得更加简单，同时也保留了静态类型的安全性。 类型推断往往也伴随着类型的隐式转换，二者均与golang的编译器有关

当右值声明了类型时，新变量的类型与其相同：
```GO
var i int
j := i // j 也是一个 int
```
不过当右边包含未指明类型的数值常量时，新变量的类型就可能是 int, float64 或 complex128 了，这取决于常量的精度：
```GO
i := 42           // int
f := 3.142        // float64
g := 0.867 + 0.5i // complex128
```

## 数值常量

数值常量是高精度的值。

一个未指定类型的常量由上下文来决定其类型。
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

## 总结

- 常量不允许取址。
- 运算符两边的操作数类型必须相同。
- 如果运算符两边是不同类型的未定义常量(untyped constant)，则会发生隐式转换，且转换的优先级为:整数(int) <符文数(rune)<浮点数(float)<复数(Imag)。
- 如果运算符的某一边是已定义类型常量(变量标识符)，则该已定义类型的常量任何时候都不会发生类型转换。因为编译器不允许对变量标识符引用的值进行强制类型转换。
- :=会显式的触发类型推断，其只能作用于函数或方法体内。
- 不指定类型的var变量声明，也会触发类型推断，可声明于局部也可声明在全局。
- 指定类型的var变量声明，不会触发类型推断(因为类型已经显式指定了)，但有可能发生类型隐式转换。

## 举例

```GO
a := 123 // 显式的触发类型推断 构建为一个整型节点
var a = 123 // 左边的a未显式指定其类型，因此仍然会触发类型推断
var a int = 123.0 // 左边显式定义了a的类型为int, 因此在类型检查阶段，右边的123.0会发生隐式类型转换，因为类型兼容，会转换为整型123。因此对于显式指定类型的表达式不会发生类型推断。

// eg.1
a := 1.1 // 显式的触发类型推断 构建为一个float64
b := 1 + a // 1转换为float64

// eg.2
a := 1 // 显式的触发类型推断 构建为一个int
b := 1.1 + a // 1.1 (untyped float constant) truncated to int

// eg.3
a1 := 1
a2 := 1.1
b := a1 + a2 // invalid operation: a1 + a2 (mismatched types int and float64)

// eg.4
const b = 3 * 0.333 // const b untyped float = 0.999

// eg.5
const a int = 1.0
const b = a * 0.333 // 0.333 (untyped float constant) truncated to int

// eg.6
const a = 1.0 / 3
b := &a // invalid operation: cannot take address of a (untyped float constant 0.333333)
```