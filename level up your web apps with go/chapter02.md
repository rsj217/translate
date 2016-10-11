# 探究Go类型

上一章介绍了`Go`基础类型和语法。本章会在此基础上对部分类型更深入的介绍，以及如何自定义类型和接口。此外还会涉及如何在软件中使用`Go`第三方代码包。

## 自定义类型

Go不能给内建的类型添加自定义方法，但是可以基于内建类型定义自己的类型。一个经典的例子就是定义一个`ByteSize`类型，它基于`float64`类型，提供了自定义的打印格式化方法，即根据结构大小的`bytes`显示不同的输出，例如_2KB_或者_3.14GB_：


```
const (    KB = 1024    MB = KB * 1024    GB = MB * 1024    TB = GB * 1024    PB = TB * 1024)

type ByteSize float64func (b ByteSize) String() string {    switch {    case b >= PB:        return "Very Big"    case b >= TB:        return fmt.Sprintf("%.2fTB", b/TB)    case b >= GB:        return fmt.Sprintf("%.2fGB", b/GB)    case b >= MB:        return fmt.Sprintf("%.2fMB", b/MB)    case b >= KB:        return fmt.Sprintf("%.2fKB", b/KB)}    return fmt.Sprintf("%dB", b)}func main(){    fmt.Println(ByteSize(2048)) // 输出: 2.00KB    fmt.Println(ByteSize(3292528.64)) // 输出: 3.14MB}
```

首先，定义了一些常量用于表示KB,MB的不同大小的值。常量和变量类似，不同在于常量的值不能改变。

其次，创建一个基于`float64`的新类型`ByteSize`，同时给它定义一个`String`方法，用于将它的值的大小依据常量的定义进行格式化输出。条件判断的过程使用了`switch`语句，`case`语句中第一个匹配的从句将会执行代码逻辑，即输出精确到两位小数并且追加一个后缀的结果。

用多个`if else`语句也可以实现`switch`语句的功能，但后者更简洁。

`main`函数创建了一些`ByteSize`实例，并通过`fmt.Println`打印输出。你可能注意到我们并没有显示的调用`String`方法，原因是如果类型定义了`String`方法，`fmt`包会自动调用并输出结果。看起来有点像魔法，下一节再披露其中神秘之处。


> 优雅的`Iota`
> 
> 在定义`KB`，`GB`这些常量的时候是不是代码有点冗余？幸运的，Go提供了`Iota`语法用于定义递增的常量。它会从`0`开始计数。例如，定义一个包含5个数的常量：
> ```
const(    one int = iota + 1    two.
.
.)
```

> 我们只需要给第一个常量类型指定`iota`表达式，后面的常量将会自动计算递增iota的值，直到遇到下一个iota才重置为0。下面是使用更加简洁的表达式来定义bytes常量：

> ```
const(    KB ByteSize = 1<<(10*(iota+1))    MB    GB    TB    PB)
```
> 这里我们使用左移操作符`<<`创建常量的大小。理解这个运算符可能很棘手，其实很简单：将左边的数字乘以2的X次方，其中X是右边的数字。第一个表达式的值等于`<<`左边的`1`乘以2`的`10`次方（右边等于10*(0+1)）,即`KB`等于`1024`。第二个常量的值为 `1*2` 的20次方（`10 *(1+1)`），得到`1048576`。








## 接口（Interface）
### 错误处理
## 内嵌类型
## defer指令
## 三方包
## 可选的语法规则和惯用风格
### 可选的语法规则
#### 初始化匿名字段的结构
#### 变量初始化空值
#### 尾部的逗号

### 代码风格
#### 变量命名
#### 常量（Constants）
#### 构造（Constructors）
## 总结




