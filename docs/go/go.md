# GO

### go env 常见的环境变量

展示变量

#### GOROOT
GOROOT 表示 Go 语言的安装目录

#### GOPATH
GOPATH 用于指定我们的工作空间(workspace)，用来存放源代码、测试文件、库静态文件、可执行文件。注意，GOPATH 的值不能和 GOROOT 相同。

#### GOBIN
GOBIN 表示程序编译后二进制命令的安装目录。

当使用 go install 命令编译和打包应用程序时，该命令会将编译后二进制程序打包 GOBIN 目录，一般我们将 GOBIN 设置为 GOPATH/bin 目录。

#### GOPROXY
下载源代码时将会通过这个环境变量设置的代理地址

#### GOPRIVATE
go get 通过代理服务拉取私有仓库（企业内部 module 或托管站点上的 private 库），而代理服务不可能访问到私有仓库，会出现了 404 错误。Go1.13 版本提供了一个方便的解决方案：GOPRIVATE 环境变量。

### 设置源

```go
go env -w GOPROXY=https://goproxy.cn,direct
```

### go get
下载包

这个命令可以动态获取远程代码包，目前支持的有 BitBucket、GitHub、Google Code 和 Launchpad

```GO
go get github.com/davyxu/cellnet
```
地址格式: 网站域名/作者或机构/项目名

包通过`go get`会下载到自己设定的**GOPATH**的位置


