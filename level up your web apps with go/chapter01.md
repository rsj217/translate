# 欢迎新的Gopher

本章的目的是为了让您从宏观方面了解Go的基础知识。首先在我们学习Go的**工具链**、**语法**和**类型**系统之前，我们将起始于在不同操作系统平台下安装Go，并写下第一份Go代码。让我们开始吧。

## 安装

写Go代码的第一步就是着手安装Go语言，你可以访问[https://golang.org/doc/install](https://golang.org/doc/install)。Go的开发者为Windows，Mac OS和Linunx都提供了安装包。当然，如果您的系统并提供安装程序，您也可以使用go的源码安装。

一旦安装完毕go之后，在命令行下输入`go version`，终端将会打印所安装的go的版本。如果发生错误，重启命令行重新输入，否则可能需要像go的[官网](https://golang.org/)报告错误并寻求解决办法。

> Go的别名golang
> Go 的官网是 [golang.org]（https://golang.org/），语言的全面其实是“golang”。当您使用“Go”关键字搜索的时候找不到解决问题有效的办法，可以尝试搜索关键字“Golang”。他们两者本质上是等价的


### Go工作区（Workspace）

通过本书，你将会感知到Go是一门强制选择配置的语言，你必须按照Go的方式构建工作区和组织代码。其他语言你可以根据自己下喜好任意组织代码，Go则希望你使用Go的方式，将你的所有项目源码存放一个特殊路径`GOPATH`的文件夹中。这样做的好处是你可以轻松的是用go工具下载和管理第三方（third-party）包源码，如果不这样做，除了标准库之外，你将无法使用任何第三方包。

这很可能与你熟悉的工作方式不同，请尽量尝试改变吧。起初我也很反感，后来尝试学习并接受Go的方式（Go way），如今已经习惯这样的代码管理方式。

`GOPATH`路径中可以有多个子文件夹，最重要的一个莫过于`src`文件夹。src文件夹存放你写的go代码和go的第三方包源码。现在创建一个文件命名为`src`。当你开始写go代码的时候，请将这些代码保存在src文件夹中。为了告诉Go哪一个文件夹为`GOPATH`路径，我们需要设置一个同名的环境变量（environment variables）。如果你对环境变量不熟，下面将会逐一接受在在Windows，Mac OS, Linux不同系统中的配置。

#### Mac OS 和 Linux

每当命令行窗口启动的时候，都会加载一个`bash profile`的文件，
在Mac OS 和 Linux中设置环境变量，只需要往这个bash中添加即可：

```
echo "export GOPATH=~/Gocode" >> ~/.bash_profilesource ~/.bash_profile
```
上述的代码将会指定你的`GOPATH`环境变量为在你的家（home）目录下的`Gocode`文件夹。然后重新载bash_profile文件以激活配置。

> 不同shell
> 如果你不使用bash，上述的代码可能会失效。比如安装了别的shell例如zsh，你可以google查询zsh的配置方式。如果是你自己写的shell，想必你也指定如何设置啦


#### Windows

Windows8中设置环境变量，只需要在**桌面**左下角右键，选择**系统**->**高级系统设置**->，然后选择**高级**选项卡（tab），然后是**环境变量**按钮。如下图1.1

Windows7, Windows Vista, 右键桌面**计算机**图标或者开始菜单。选择**属性**然后选择**高级系统设置**，然后是**高级**选项卡，最后是**环境变量**按钮。

![windows环境变量菜单](/Users/ghost/Documents/translate/level up your web apps with go/images/1.1.png)

一旦打开了环境变量设置窗口，选择在**系统变量**中选择**新建**，然后创建一个新的变量命名为`GOPATH`，变量的值为你的go代码工作区的路径，如下图，我设置`C:\Gocode`为`GOPATH`。

![创建新的环境变量](/Users/ghost/Documents/translate/level up your web apps with go/images/1.2.png)

### 第一份Go代码

本章将会介绍一些基本的go语法，以及使用go写代码的一些常用工具。该介绍只涉及一些基础的知识，而不是全面的go语言教程。随着我们逐步贯穿此书，会逐渐介绍所遇到的概念知识。

任何一门编程的书都会起始于“Hello World”例子，因此让我们跟随巨人的脚步，开启go编程第一步。创建一个文件---`helloworld.go`，然后键入如下代码：

helloworld.go

```
package mainimport "fmt"func main() {    fmt.Println("Hello World")}
```

然后使用命令行打开hellowworld.go所在的目录，运行代码：

```
go run helloworld.go
```

运行成功就会在命令行打印输出`Hello World`。麻雀虽小，五脏俱全，这个简单的程序宝贝了go程序的基本结构。代码开头，我们声明了代码所属的包（package）名。包是我们组织代码的基础结构，`main`包则是特殊的包，它可以直接执行。如果想要写一个包在别的Go程序中使用，必须指定一个唯一的main package。后面的章节将会介绍如何写我们自己的包。

开头的下一行，我们导入（import）了程序所使用的包。因为这是一个简单程序，我们只应引用了一个包：`fmt`。`fmt`包来自Go标准库，后者随着go语言安装的时候一起安装。它提供了基本的格式化输出打印操作。`fmt`是标准库的一部分，是一个基础的导入语句。随后，我们将会见识到go如何通过源码url导入第三方包的导入语句；例如：“import github.com/tools/godep”。

最后，我们定义了一个`main`函数，函数中使用了`fmt`包的`Println`函数。main函数是go程序的入口点，当程序启动的时候会执行main函数。`Println`将会打印传入变量的值，你可以传入任何变量。

至此，你可能注意到了go语法的一些字符。首先，括号和花括号和类C的语言很像。不过，你会发现每一句代码结束时并没有分号，每一句结束的下一行为新的语句的开始。最后你会发现代码编译和运行的时间都很快。这是go团队早期的决策之一，让go牺牲一些性能一遍确保提升代码编译速度，这让go在开发程序的时候更有乐趣。

## Go 工具

我们已经见识了第一个go工具（tools）的例子。我们使用go命令行工具运行`.go`文件。`go run`将会编译并运行代码。如果想要创建可执行文件取代每次run文件，可是使用命令`go build`。该命令将会编译当前文件夹下的所有文件，并创建一个以当前文件夹命名的可行性的二进制文件。例如，`chapter1`文件下创建了`helloworld.go`文件，运行`go build`，将会得到一个名为`chapter1`的可行性文件。进一步可以使用`go install`命令，该命令会把编译生成的二进制文件移动到`$GOPATH/bin`文件夹内。如果该文件夹在系统的`PATH`环境变量内，你就可以在任何路径下执行运行这个编译后的go程序。

> PATH 环境变量
> PATH 环境变量是系统寻找可行性文件的路径。当你在命令行输入某个命令的时候，系统会在PATH目录下寻找命令，并执行所找到的第一个命令，如果找不到就会报错。在windows中，你也可以设置PATH包含GOPATH。在MacOS 和Linxu中可以使用早先介方式添加： `export PATH=$PATH:$GOPATH/bin`追加到你的bash profile文件中。

## 基本类型（Types）










