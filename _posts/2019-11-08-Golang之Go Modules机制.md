---
layout: post
title:  "Golang之Go Modules机制"
date:   2019-11-08 13:57:12 +0800--
categories: [Golang]
tags: [Golang]  
---

在PHP或者JAVA等语言的项目中，都有很多成熟的版本依赖包管理工具，比如Gradle，Composer等，非常方便，都快被惯坏了呢~   

但是在之前的Golang版本中，并没有官方提供的版本依赖包管理工具，大家都是通过<code>go get</code>获取最新版的依赖，获取的依赖都会挤在GOPATH目录下，非常混乱，而且我们并不知道最新版的依赖包，是否会和之前的项目代码有兼容问题。  

之后官方出了一个 vendor 机制，将项目依赖的包都放在该目录中，但这也并没有很好地管理依赖的版本。  

再之后官方出了一个准官方版本管理工具 go dep，这也算是 go modules 的前身了吧。  

随着 Go1.11 的发布，Golang 给我们带来了 modules 全新特性，这是 Golang 新的一套依赖管理系统，用以替代GOPATH机制。

在这片文章发布前1个月，Go1.13更新了，go modules已经被定为默认开发机制，而且官方预告将在1.14版本中，确认最终版本，现在比较推荐大家使用go modules来做依赖管理了。

> A module is a collection of related Go packages. Modules are the unit of source code interchange and versioning. The go command has direct support for working with modules, including recording and resolving dependencies on other modules. Modules replace the old GOPATH-based approach to specifying which source files are used in a given build.  
>
>官方定义：模块是相关的Go软件包的集合。模块是源代码交换和版本控制的单元。go命令直接支持使用模块，包括记录和解决对其他模块的依赖关系。模块取代了旧的基于GOPATH的方法来指定在给定版本中使用哪些源文件。

## GO111MODULE
如果项目目录中没有go.mod文件，默认情况下是未开启 mudules 的，我们需要在项目目录下手动执行以下命令：
```bash
$ export GO111MODULE=on
```
`GO111MODULE` 有三个值：`off`, `on`和`auto`（默认值）。
- `GO111MODULE=off`，go命令行将不会支持module功能，寻找依赖包的方式将会沿用旧版本那种通过vendor目录或者GOPATH模式来查找。
- `GO111MODULE=on`，go命令行会使用modules，而一点也不会去GOPATH目录下查找。
- `GO111MODULE=auto`，默认值，go命令行将会根据当前目录来决定是否启用module功能。这种情况下可以分为两种情形：
  - 当前目录在`GOPATH/src`之外且该目录包含go.mod文件
  - 当前文件在包含go.mod文件的目录下面。

## go mod命令
golang 提供了 go mod 命令来管理包。  
go mod 有以下命令：

命令|说明
download|下载依赖包
edit|编辑go.mod
graph|打印模块依赖图
init|在当前目录初始化mod
tidy|拉取缺少的模块，移除不用的模块
vendor|将依赖复制到vendor下
verify|验证依赖是否正确
why|解释为什么需要依赖
  
## 创建一个自己的modules
### 1、新建项目
```bash
$ mkdir testmod
$ cd testmod

$ echo 'package testmod
		import "fmt"
	func SayHello(name string) string {
   return fmt.Sprintf("Hello, %s", name)
}' >> testmod.go
```

### 2、初始化modules 
> {github_name}替换成你自己的github账号

```bash
$ go mod init github.com/{github_name}/testmod
go: creating new go.mod: module github.com/{github_name}/testmod
```
初始化成功之后，在项目中创建一个 go.mod 文件，内容如下：
```bash
module github.com/{github_name}/testmod
```
这时此项目已经成为一个module

### 3、推送到github仓库
> 当然你也可以推送到gitlab，或者其他自建git服务

```bash
$ git init
$ git add *
$ git commit -am "First commit"
$ git push -u origin master
```
### 4、版本规则
go modules 是一个版本化依赖管理系统，版本需要遵循一些规则，比如版本号需要遵循以下格式：
```bash
vX.Y.Z-pre.0.yyyymmddhhmmss-abcdefabcdef
vX.0.0-yyyymmddhhmmss-abcdefabcdef
vX.Y.(Z+1)-0.yyyymmddhhmmss-abcdefabcdef
vX.Y.Z
```
vX.Y.Z 是我们仓库打的标签版本，也就是 go modules 是根据仓库标签来确定版本号的，因此我们发布版本时，需要给我们的仓库打上一个标签版本号。

也就是版本号 + 时间戳 +hash，我们自己指定版本时只需要制定版本号即可，没有版本 tag 的则需要找到对应 commit 的时间和 hash 值。

还有一个重要的规则是，版本 0 和 1，最好需要有不同的依赖路径，如：v1.0.0 和 v2.0.0 是有不同的依赖路径，下面会详细介绍一下这个版本规则。

### 5、发布版本
了解了 go modules 的版本规则后，现在我们发布一下该项目的版本：
```bash
$ git tag v1.0.0
$ git push --tags
```
这时我们最好还需要创建一条 v1 分支，以便我们在其它分支写代码不会影响到 v1.0.0 版本：
```bash
$ git checkout -b v1
$ git push -u origin v1
```

### 6、迭代小版本
我把「Hello」改成「你好」，我们把这个修改在 v1 分支中进行：
```bash
$ cd gomodules
$ echo 'package testmod
		import "fmt"
	func SayHello(name string) string {
   return fmt.Sprintf("你好, %s", name)
}' >> testmod.go
```
```bash
$ git commit -m "update testmod" testmod.go
$ git tag v1.0.1
$ git push --tags origin v1
```
现在我们的 项目已经升级到 v1.0.1 版本了，我们可以有多种方式获取这个版本依赖，go1.11 中，go get 拥有了很多新特性，我们可以直接通过以下命令获取 v1.01 版本依赖：
```bash
$ go get github.com/{github_name}/testmod@v1.0.1
```
也可以通过 go mod 命令：
```bash
$ go mod edit -require="github.com/{github_name}/testmod@v1.0.1"
$ go mod tidy
```
> go mod tidy 对版本进行更新，它会自动清理掉不需要的依赖项，同时可以将依赖项更新到当前版本。

### 7、更新大版本
上面版本规则说了，版本 0 和 1，即大版本更新，最好需要有不同的依赖路径，如：v1.0.0 和 v2.0.0 是有不同的依赖路径，那么用 go modules 怎么实现呢，我们可以通过修改 go.mod 文件第一行添加新路径：
```bash
$ cd testmod
$ echo 'module github.com/{github_name}/testmod/v2' >> go.mod
```
然后我们修改 testmod 函数 Hi()：
```bash
$ cd testmod

$ echo 'package testmod
import (
	"fmt"
)
func SayHello(name, str string) string {
	return fmt.Sprintf("你好, %s, %s", name, str)
}' >> testmod.go
```
这时，SayHello() 方法将不兼容 v1 版本，我们需要新建一个 v2.0.0 版本，还是老样子，我们最好在 v2.0.0 版本新建一条 v2 分分支，将 v2.0.0 版本的代码写到这条分支中（这只是一个规范，实际上你将代码也写到任何分支中都行，Go并没有这个规范）：
```bash
$ git add *
$ git checkout -b v2
$ git commit testmod.go -m "v2.0.0"
$ git tag v2.0.0
$ git push --tags origin v2
```
这时候 testmod 的版本已经更新到 v2.0.0 版本了，该版本并不兼容以前的版本，但是目前项目依然只能获取到 v1.0.1 的依赖版本，因为我们 testmod 的 module 路径已经加上 v2 了，因此并不会出现冲突的问题，那么如果我们需要使用 v2.0.0 版本呢？我们只需要在项目中更改一下 import 路径：
```golang
package main

import (
    "fmt"
  	"github.com/{github_name}/testmod/v2"
)
func main() {
    fmt.Println(testmod.SayHello("蔡徐坤", "鸡你太美"))
}
```
执行
```bash
go mod tidy
```
这时我们把 testmod 依赖版本号更新到了 v2.0.0 版本了，虽然是此时的 import 路径是以 “v2” 结尾，但是 Go 很人性化，我们依然可以使用 testmod 来使用。


## 使用modules
现在我们把刚刚做好的 module，拿过来用，新建一个 go modules 项目：
```golang
package main

import (
    "fmt"
    "github.com/{github_name}/testmod"
)

func main() {
    fmt.Println(testmod.SayHello("蔡徐坤"))
}
```
初始化go modules，会生成一个go.mod文件
```bash
$ go mod init
```
这时执行 go build 等命令就会下载依赖包，并把依赖信息添加到 go.mod 文件中，同时把依赖版本哈希信息存到 go.sum 文件中：
```bash
$ go build

go: finding github.com/{github_name}/testmod v1.0.0
go: downloading github.com/{github_name}/testmod v1.0.0
```
现在go.mod文件内容如下：
```bash
module gomodules
require github.com/{github_name}/testmod v1.0.0
```
go.sum文件如下：
```bash
github.com/{github_name}/testmod v1.0.0 h1:fGa15gBXoqkG0BVkQGP9i5Pg2nt8nayFpHFf+GLiX6A=
github.com/{github_name}/testmod v1.0.0/go.mod h1:LGpYEmOLZhLQC3JW88STU2Ja3rsfoGZmsidsHJhDNGU=
```

## 参考与出处
[Using Go Modules](https://blog.golang.org/using-go-modules)  
[Go Modules详解](https://objcoding.com/2018/09/13/go-modules/)
