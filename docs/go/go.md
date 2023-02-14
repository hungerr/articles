# GO配置

## PATH

安装目录`Go\bin`加入环境变量PATH

## go env 常见的环境变量

展示变量

### GOROOT
GOROOT 表示 Go 语言的安装目录

### GOPATH
GOPATH 用于指定我们的工作空间(workspace)，用来存放源代码、测试文件、库静态文件、可执行文件。注意，GOPATH 的值不能和 GOROOT 相同。

### GOBIN
GOBIN 表示程序编译后二进制命令的安装目录。

当使用 go install 命令编译和打包应用程序时，该命令会将编译后二进制程序打包 GOBIN 目录，一般我们将 GOBIN 设置为 GOPATH/bin 目录。

### GOPROXY
下载源代码时将会通过这个环境变量设置的代理地址

```go
go env -w GOPROXY=https://goproxy.cn,direct
```

### GOPRIVATE
go get 通过代理服务拉取私有仓库（企业内部 module 或托管站点上的 private 库），而代理服务不可能访问到私有仓库，会出现了 404 错误。Go1.13 版本提供了一个方便的解决方案：GOPRIVATE 环境变量。

## GO MODULE

在以前，Go 语言的的包依赖管理一直都被大家所诟病，Go官方也在一直在努力为开发者提供更方便易用的包管理方案，从最初的 `GOPATH` 到 `GO VENDOR`，再到最新的 `GO Modules`

### GOPATH

可以将其理解为工作目录，在这个工作目录下，通常有如下的目录结构

- `bin`：存放编译后生成的二进制可执行文件
- `pkg`：存放编译后生成的 .a 文件
- `src`：存放项目的源代码，可以是你自己写的代码，也可以是你 `go get` 下载的包

在这个模式下，使用 `go install` 时，生成的可执行文件会放在 `$GOPATH/bin` 下

如果你安装的是一个库，则会生成 `.a` 文件到 `$GOPATH/pkg` 下对应的平台目录中（由 GOOS 和 GOARCH 组合而成），生成 `.a` 为后缀的文件。

以下几点是你使用 `GOPATH` 一定会碰到的问题：

- 你无法在你的项目中，使用指定版本的包，因为不同版本的包的导入方法也都一样

- 其他人运行你的开发的程序时，无法保证他下载的包版本是你所期望的版本，当对方使用了其他版本，有可能导致程序无法正常运行

- 在本地，一个包只能保留一个版本，意味着你在本地开发的所有项目，都得用同一个版本的包，这几乎是不可能的。

### GO VENDOR

为了解决 `GOPATH` 方案下不同项目下无法使用多个版本库的问题，`Go v1.5` 开始支持 `vendor` 。

以前使用 `GOPATH` 的时候，所有的项目都共享一个 `GOPATH`，需要导入依赖的时候，都来这里找，正所谓一山不容二虎，在 `GOPATH` 模式下只能有一个版本的第三方库。

解决的思路就是，在每个项目下都创建一个 `vendor` 目录，每个项目所需的依赖都只会下载到自己`vendor`目录下，项目之间的依赖包互不影响。在编译时，v1.5 的 Go 在你设置了开启 `GO15VENDOREXPERIMENT=1` （注：这个变量在 v1.6 版本默认为1，但是在 v1.7 后，已去掉该环境变量，默认开启 `vendor` 特性，无需你手动设置）后，会提升 `vendor` 目录的依赖包搜索路径的优先级（相较于 GOPATH）。

其搜索包的优先级顺序，由高到低是这样的

- 当前包下的 `vendor` 目录

- 向上级目录查找，直到找到 `src` 下的 `vendor` 目录

- 在 `GOROOT` 目录下查找

- 在 `GOPATH` 下面查找依赖包


虽然这个方案解决了一些问题，但是解决得并不完美。

如果多个项目用到了同一个包的同一个版本，这个包会存在于该机器上的不同目录下，不仅对磁盘空间是一种浪费，而且没法对第三方包进行集中式的管理（分散在各个角落）。

并且如果要分享开源你的项目，你需要将你的所有的依赖包悉数上传，别人使用的时候，除了你的项目源码外，还有所有的依赖包全部下载下来，才能保证别人使用的时候，不会因为版本问题导致项目不能如你预期那样正常运行。

### go mod

go modules 在 `v1.11` 版本正式推出，在最新发布的 `v1.14` 版本中，官方正式发话，称其已经足够成熟，可以应用于生产上。

从 `v1.11` 开始，`go env` 多了个环境变量：` GO111MODULE` ，这里的 `111`，其实就是 `v1.11` 的象征标志， go 里好像很喜欢这样的命名方式，比如当初 vendor 出现的时候，也多了个 `GO15VENDOREXPERIMENT`环境变量，其中 15，表示的vendor 是在 `v1.5` 时才诞生的。

`GO111MODULE` 是一个开关，通过它可以开启或关闭 `go mod` 模式。

它有三个可选值：`off`、`on`、`auto`，默认值是`auto`。

- `GO111MODULE=off` 禁用模块支持，编译时会从`GOPATH`和`vendor`文件夹中查找包。
- `GO111MODULE=on` 启用模块支持，编译时会忽略`GOPATH`和`vendor`文件夹，只根据 go.mod下载依赖。
- `GO111MODULE=auto`，当项目在`$GOPATH/src`外且项目根目录有`go.mod`文件时，自动开启模块支持。

开启
```bash
$ go env -w GO111MODULE="on"
```

初始化：
```BASH
mkdir hello
cd hello
go mod init example/hello
```
会生成`go.mod`文件

文件内容

```go
module example.com/hello

go 1.16

replace example.com/greetings => ../greetings

require example.com/greetings v0.0.0-00010101000000-000000000000

exclude example.com/banana v1.2.4
```

- `exclude`： 忽略指定版本的依赖包
- `replace`：由于在国内访问golang.org/x的各个包都需要翻墙，你可以在go.mod中使用replace替换成github上对应的库。

`go.sum` 文件:

每一行都是由 `模块路径`，`模块版本`，`哈希检验值`

```
<module> <version>/go.mod <hash>
```

有的包有两行

```
<module> <version> <hash>
<module> <version>/go.mod <hash>
```
那些有两行的包，区别就在于 `hash` 值不一行，一个是 `h1:hash`，一个是 `go.mod h1:hash`

而 `h1:hash` 和 `go.mod h1:hash`两者，要不就是同时存在，要不就是只存在 `go.mod h1:hash`。那什么情况下会不存在 `h1:hash` 呢，就是当 Go 认为肯定用不到某个模块版本的时候就会省略它的`h1 hash`，就会出现不存在 `h1 hash`，只存在 `go.mod h1:hash` 的情况

go mod 命令:

- `go mod init`：初始化 生成`go.mod`文件，后可接参数指定 `module` 名
- `go mod download`：手动触发下载依赖包到本地cache（默认为`$GOPATH/pkg/mod`目录）
- `go mod graph`： 打印项目的模块依赖结构
- `go mod tidy` ：添加缺少的包，且删除无用的包
- `go mod verify` ：校验模块是否被篡改过
- `go mod why`： 查看为什么需要依赖
- `go mod vendor` ：导出项目所有依赖到vendor下
- `go mod edit` ：编辑`go.mod`文件，接 `-fmt` 参数格式化 `go.mod `文件，接 `-require=golang.org/x/text` 添加依赖，接 `-droprequire=golang.org/x/text` 删除依赖，详情可参考 `go help mod edit`
- `go list -m -json all`：以 `json` 的方式打印依赖详情

有两种方法给项目添加依赖（写进 go.mod）呢？

- 你只要在项目中有 `import`，然后 `go build` 就会 `go module` 就会自动下载并添加。
- 自己手工使用 `go get` 下载安装后，会自动写入 `go.mod`

### go version
查看版本

### go build
生成可执行文件
编译后生成的有默认名生成的文件在当前目录或在指定目录下
```
go build main.go
```

### go run
执行
```
go run main.go
```

## go get
下载包

这个命令可以动态获取远程代码包，目前支持的有 BitBucket、GitHub、Google Code 和 Launchpad

```GO
go get github.com/davyxu/cellnet
```
地址格式: 网站域名/作者或机构/项目名

下载包 并且更新`go.mod`  and downloads source code into the
module cache

To add a dependency for a package or upgrade it to its latest version:

        go get example.com/pkg

To upgrade or downgrade a package to a specific version:

        go get example.com/pkg@v1.2.3

To remove a dependency on a module and downgrade modules that require it:

        go get example.com/mod@none

See https://golang.org/ref/mod#go-get for details.

'go get' accepts the following flags.

The `-t` flag instructs get to consider modules needed to build `tests` of
packages specified on the command line.

The `-u` flag instructs get to `update` modules providing dependencies
of packages named on the command line to use newer minor or patch
releases when available.

The `-u=patch` flag (not -u patch) also instructs get to update dependencies,
but changes the default to select patch releases.

When the `-t` and `-u` flags are used together, get will update
test dependencies as well.

The `-x` flag prints commands as they are executed. This is useful for
debugging version control commands when a module is downloaded directly
from a repository.

## go install

用于构建和安装二进制文件

In earlier versions of Go, `go get` was used to build and install packages.
Now, `go get` is dedicated to adjusting dependencies in `go.mod`. 

`go install`may be used to build and install commands instead.

可执行文件一般下载于`GOBIN`下，一般默认值是`$GOPATH/bin`，如果`GOPATH`未设置值则是`$HOME/go/bin`

Executables in `$GOROOT` are installed in `$GOROOT/bin` or `$GOTOOLDIR` instead of `$GOBIN`

When a version is specified, `go install` runs in `module-aware` mode and `ignores` the `go.mod` file in the current directory. For example:

        go install example.com/pkg@v1.2.3
        go install example.com/pkg@latest

If the arguments don't have version suffixes, `go install` may run in
`module-aware` mode or `GOPATH` mode, depending on the `GO111MODULE` environment
variable and the presence of a `go.mod` file. See 'go help modules' for details.

If `module-aware` mode is `enabled`, "go install" runs in the context of the `main`
module.

When `module-aware` mode is `disabled`, other packages are installed in the
directory `$GOPATH/pkg/$GOOS_$GOARCH`. When `module-aware` mode is `enabled`,
other packages are built and cached but not installed.

See 'go help install' or https://golang.org/ref/mod#go-install for details.

```bash
$ mkdir hello
$ cd hello
$ go mod init example/user/hello
$ go install example/user/hello
```
以上命令会在`GOBIN`创建可执行文件

以下命令效果是一样的:
```bash
$ go install example/user/hello
$ go install .
$ go install
```