## 包package

你可以打包代码，并在其他位置重复使用它。 包的源代码可以分布在多个 `.go` 文件中

### main 包

在 Go 中，甚至最**直接的程序**都是包的一部分。 通常情况下，默认包是 `main 包`，即目前为止一直使用的包。 如果程序是 main 包的一部分，Go 会生成**二进制文件**。 运行该文件时，它将调用 `main()` 函数。

换句话说，当你使用 main 包时，程序将生成独立的**可执行文件**。 但当程序非是 main 包的一部分时，Go 不会生成二进制文件。 它生成包存档文件（具有 `.a` 扩展名的文件）。

### 创建包

```
src/
  calculator/
    sum.go
```
代码
```GO
package calculator

var logMessage = "[LOG]"

// Version of the calculator
var Version = "1.0"

func internalSum(number int) int {
    return number - 1
}

// Sum two integer numbers
func Sum(number1, number2 int) int {
    return number1 + number2
}
```

Go 不会提供 `public` 或 `private` 关键字，以指示是否可以从包的内外部调用变量或函数。 但 Go 须遵循以下两个简单规则：

- 如需将某些内容设为**专用**内容，请以**小写字母**开始。
- 如需将某些内容设为**公共**内容，请以**大写字母**开始。

### 模块module

可以将包放到模块中，Go 模块通常包含可提供相关功能的包

按照约定，包名与导入路径的最后一个元素一致。例如，"math/rand" 包中的源码均以 package rand 语句开始。

创建模块：
```
go mod init github.com/myuser/calculator
```

运行此命令后，`github.com/myuser/calculator` 就会变成模块的名称。 在其他程序中，你将使用该名称进行引用。 命令还会创建一个名为 `go.mod` 的新文件。 最后，树目录现会如下列目录所示：
```
src/
  calculator/
    go.mod
    sum.go
```

### 引用本地包
树文件结构
```
src/
  calculator/
    go.mod
    sum.go
  helloworld/
    main.go
```
我们会将此代码用于 $GOPATH/src/helloworld/main.go 文件：
```GO
package main

import (
  "fmt"
  "github.com/myuser/calculator"
)

func main() {
    total := calculator.Sum(3, 5)
    fmt.Println(total)
    fmt.Println("Version: ", calculator.Version)
}
```
运行以下命令:
```
go mod init helloworld
```
树目录会如下所示：
```
src/
  calculator/
    go.mod
    sum.go
  helloworld/
    go.mod
    main.go
```

由于你引用的是该模块的本地副本，因此你需要通知 Go 不要使用远程位置.你需要手动修改 go.mod 文件，使其包含引用
```
module helloworld

go 1.14

require github.com/myuser/calculator v0.0.0

replace github.com/myuser/calculator => ../calculator
```
