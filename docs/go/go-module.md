## module

你可以打包代码，并在其他位置重复使用它。 包的源代码可以分布在多个 `.go` 文件中

### 包注释

每个包都应包含一段包注释，即放置在包子句前的一个块注释。对于包含多个文件的包， 包注释只需出现在其中的任一文件中即可。包注释应在整体上对该包进行介绍，并提供包的相关信息。 它将出现在 godoc 页面中的最上面，并为紧随其后的内容建立详细的文档。
```GO
/*
regexp 包为正则表达式实现了一个简单的库。

该库接受的正则表达式语法为：

    regexp:
        concatenation { '|' concatenation }
    concatenation:
        { closure }
    closure:
        term [ '*' | '+' | '?' ]
    term:
        '^'
        '$'
        '.'
        character
        '[' [ '^' ] character-ranges ']'
        '(' regexp ')'
*/
package regexp
```

### main包

在 Go 中，甚至最**直接的程序**都是包的一部分。 通常情况下，默认包是 `main 包`，即目前为止一直使用的包。 如果程序是 main 包的一部分，Go 会生成**二进制文件**。 运行该文件时，它将调用 `main()` 函数。

换句话说，当你使用 main 包时，程序将生成独立的**可执行文件**。 但当程序非是 main 包的一部分时，Go 不会生成二进制文件。 它生成包存档文件（具有 `.a` 扩展名的文件）。

### 权限控制

Go 不会提供 `public` 或 `private` 关键字，以指示是否可以从包的内外部调用变量或函数。 但 Go 须遵循以下两个简单规则：

- 如需将某些内容设为**专用**内容，请以**小写字母**开始。
- 如需将某些内容设为**公共**内容，请以**大写字母**开始。

### 模块module

可以将包放到模块中，Go模块通常包含可提供相关功能的包

按照约定，包名与导入路径的最后一个元素一致。例如，`math/rand`包中的源码均以 `package rand` 语句开始。

创建模块：
```
cd gocode
mkdir greetings
cd greetings
go mod init example.com/greetings
```

运行`go mod init`命令后，创建一个名为 `go.mod` 的新文件:
```GO
module example.com/greetings

go 1.19
```
创建`greetings.go`文件:
```GO
package greetings

import "fmt"

// Hello returns a greeting for the named person.
func Hello(name string) string {
    // Return a greeting that embeds the name in a message.
    message := fmt.Sprintf("Hi, %v. Welcome!", name)
    return message
}
```

最后，树目录现会如下列目录所示：
```
gocode/
  greetings/
    go.mod
    greetings.go
```

### 引用本地包
```GO
cd ..
mkdir hello
cd hello
go mod init example.com/hello
```
创建`hello.go`文件:
```GO
package main

import (
    "fmt"

    "example.com/greetings"
)

func main() {
    // Get a greeting message and print it.
    message := greetings.Hello("Gladys")
    fmt.Println(message)
}
```

树文件结构:
```
gocode/
  greetings/
    go.mod
    greetings.go
  hello/
    go.mod
    hello.go
```

由于你引用的是该模块的本地副本，因此你需要通知 Go 不要使用远程位置
```GO
go mod edit -replace example.com/greetings=../greetings
```
`go.mod`文件被修改为:
```GO
module example.com/hello

go 1.19

replace example.com/greetings => ../greetings
```

运行:
```GO
go mod tidy
```
同步依赖 `go.mod`文件被修改为:
```GO
module example.com/hello

go 1.19

replace example.com/greetings => ../greetings

require example.com/greetings v0.0.0-00010101000000-000000000000
```

### multi-module workspace

`workspace`内多modules可以一起工作

创建`workspace`:
```GO
cd gocode
go work init ./hello
```
多出go.work文件:
```GO
go 1.19

use ./hello
```

此时可以在`workspace`内执行
```GO
go run example.com/hello
```

将go.mod的
```GO
replace example.com/greetings => ../greetings

require example.com/greetings v0.0.0-00010101000000-000000000000
```
去掉 然后执行:
```GO
go work use ./greetings/
```
此时`hello`也能访问`greetings`

在`greetings`文件夹下新建`stringutil`文件夹 新建文件`toupper.go`：
```GO
package stringutil

import "unicode"

// ToUpper uppercases all the runes in its argument string.
func ToUpper(s string) string {
    r := []rune(s)
    for i := range r {
        r[i] = unicode.ToUpper(r[i])
    }
    return string(r)
}
```

在`hello`中引用:
```GO
package main

import (
	"fmt"

	"example.com/greetings/stringutil"
)

func main() {
	// Get a greeting message and print it.
	fmt.Println(stringutil.ToUpper("Hello"))
}
```
