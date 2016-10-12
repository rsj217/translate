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

`Go`通过**接口**提供了强大的“数据类型共享”功能。接口是一种包含定义了函数签名集合的类型，任何其他类型，只要实现了接口定义的函数（类型方法与接口定义的函数签名一致）就等于实现了该接口。代码是最好的解释，请看下面的例子：

```
type Fruit interface {    String() string}type Apple struct {    Variety string}func (a Apple) String() string {    return fmt.Sprintf("A %s apple", a.Variety)}type Orange struct {    Size string}func (o Orange) String() string {    return fmt.Sprintf("A %s orange", o.Size)}func PrintFruit(fruit Fruit) {    fmt.Println("I have this fruit:", fruit.String())}func main() {    apple := Apple{"Golden Delicious"}    orange := Orange{"large"}    PrintFruit(apple)    PrintFruit(orange)}

I have this fruit: A Golden Delicious appleI have this fruit: A large orange

```

代码定义了一个`Fruit`接口，`Fruit`只有一个`String() string`方法需要实现。这个方法没有参数，返回一个字串。因此任何类型只要实现一个`String`方法并返回一个字串，就等同于实现了`Fruit`接口。然后我们创建了两个结构`Apple`和`Orange`，它们有不同的字段，但是都正确的实现了`String`方法，因此它们都实现了`Fruit`接口。最后，定义一个函数`PrintFruit`，接受一个`Fruit`实例作为参数，函数内打印这个实例，即输出接口的`String`方法被调用返回的值。

`main`函数中，分别实例化`Apple`类型和`Orange`，并把它们作为`PrintFruit`的参数传递。请注意，`apple`和`orange`都是它们各自的类型的实例，只有当成参数传递的时候，才是`Fruit`类型。

接口的功能在于，调用代码不受底层类型或实现的影响，只要它匹配方法签名。这很强大，尤其是在写代码包进行复用的时候。

让我们来看一个新的假设情况：假设您销售提供产品目录的Web应用程序。


首先，您的客户只有几个产品要销售，所以您只需将信息存储在文件中，并在应用程序运行时加载它。 然后你有一个客户有大量的产品，所以是时候移动产品到某种数据库。 如果要使用相同的代码库来处理产品，但抽象出存储和检索产品的位置的详细信息，则可以为产品目录定义一个接口，然后对文件目录和数据库目录使用不同的实现。 我们将跳过实现，但是我们可以看看这个方法的一般结构。 首先，我们将创建一个接口来定义我们为产品目录处理的方法：

```
type ProductCatalogue interface {    All() ([]Product, error)    Find(string) (Product, error)}
```

这使我们能够处理获得所有的产品和获得一个特定的产品一个小的电子商务网站的一般要求。 可能需要更多的方法，但我会保持这个例子最小。 接下来，我们将定义文件存储和某些数据库存储的接口的实际实现：

```
type FileProductCatalogue struct {    ...  // 文件结构一些字段，比如文件位置之类的
    }
func (c FileProductCatalogue) All() ([]Product, error) {    ... // 实现All接口}

func (c FileProductCatalogue) Find(id string) (Product, error) {    ... // 实现Find接口}
type DBProductCatalogue struct {    ... // 数据库的一些字段，比如数据库连接}
func (c DBProductCatalogue) All() ([]Product, error) {    ... // 实现All接口}
func (c DBProductCatalogue) Find(id string) (Product, error) {    ... // 实现Find接口}

```

现在我们有实现了同一接口的不同实例。一个从文件加载，另一个从数据库加载。 当我们创建一个变量时，我们可以使用接口`ProductCatalogue`类型，而不是它们的具体实现：（译者注：变量可以是接口的类型，而不用专门指定是`FileProductCatalogue`类型还是`DBProductCatalogue`类型）


```
func main() {    var myCatalogue ProductCatalogue    myCatalogue = DBProductCatalogue{}    ...}
```

或者可以创建一个初始化函数，它接受任何`ProductCatalogue`接口的类型的实例：

```
func LoadProducts(catalogue ProductCatalogue) {    products, err := catalogue.All()    ...}
```
> 多接口（Multiple Interfaces）
> 
> 类型不限于实现单个接口。 例如，我们在第一个例子中看到的`Fruit`接口与`fmt`包中的`Stringer`接口完全相同。 如果你将一个`Fruit`实例传递给`fmt.Println`，你将看到`String`方法的结果。（Fruit接口声明了String方法， 实现了`Fruit`接口则有`String`方法）
> `Go`语言中最基本的接口是**空接口**interface{}，它没有方法，因此每个类型都实现了它。
> 类型对于所需要实现的接口方法没有限制（类型可以实现多个接口的方法），重要的是类型必须实现接口所要求的方法。（一旦打算实现某个接口，就必须实现这个接口的所有方法）。

### 错误处理


Go中的错误处理是一种面对面的情况。许多语言会不强迫你考虑错误处理的情况下允许编程。让你可以捕获错误，但是如果你不想或者不理会，那么你的代码将遇到可笑的失败，并且很难跟踪。

在Go中哲学是检查错误无处不在，因此不断地考虑如何处理它们和它们所表示的意思。 虽然有些人可能会认为这是一个糟糕的想法---处理错误确实导致更多的代码。但是它能促进产生健壮的代码，把错误处理成看出功能的核心部分，而不是事后再补救。

还记得先前函数多返回值的情况么？大多数情况下返回多个值的函数都会把错误当成最后一个值返回。这意味着接受函数调用返回值的时候，需要声明一个变量接受错误（或者提供一个`_`以忽略返回的错误）并检查。下面的代码很普遍：

```
someValue, err := doSomething()if err != nil {    return err // Handle the error}
```

每次调用返回错误结果的函数后，都需要检查是否发生错误，如果发生错误，则进行处理。通常情况下，这意味着将它传入一个需要不停处理错误的函数链中（译者注：即处理错误的函数又有可能返回错误，然后继续处理这个错误，像一个链式一样）。虽然这会给你的代码添加额外的行，同时也会促使你思考代码无法继续执行，在可能发生的故障的时候到底将会发生什么？许多代码依赖于外部资源（如网络连接和磁盘可用性）的可用性的世界中，设计失败比以往任何时候都更重要，Go使用它的错误处理做到了由表及里。

那么所谓的error到底是什么呢？error是一个关于接口很好的例子。error本质上是一个普通的Go接口，并且任一类型都可以实现其`Error`方法：

```
type error interface {    Error() string}
```

这可以提示我们创建自己的错误类型的方式。 既可以扩展现有的类型，又可以创建一个带有额外数据的结构体。例如，如果我们想要一个表示无效的HTTP状态代码的错误类型，我们可以定义基于整型自定义错误类型：

```
type ErrInvalidStatusCode intfunc (code ErrInvalidStatusCode) Error() string {    return fmt.Sprintf("Expected 200 ok, but got %d", code)}
```

如果遇到了一个未验证的状态码（译者注：http状态码 200 表示正确处理请求），我们就能把这个状态码用`ErrInvalidStatusCode`包装。

```
if myStatusCode != 200 {    return ErrInvalidStatusCode(myStatusCode)}
```

## 内嵌类型


**结构**类型允许嵌入其他类型。 这允许一个级别的对象“继承”，虽然它不像经典面向对象的继承。直接写内嵌类型而不写字段名是添加内嵌匿名类型，类型访问匿名类型的字段和方法与访问自身的一样：


```
type User struct {    Name string}

func (u User) IsAdmin() bool { return false }func (u User) DisplayName() string {    return u.Name}
type Admin struct {    User}
func (a Admin) IsAdmin() bool { return true }func (a Admin) DisplayName() string {    return "[Admin] " + a.User.DisplayName()}
func main() {    u := User{"Normal User"}    fmt.Println(u.Name)          // 正常 User    fmt.Println(u.DisplayName()) // 正常 User    fmt.Println(u.IsAdmin())     // false    a := Admin{User{"Admin User"}}    fmt.Println(a.Name)          // Admin User    fmt.Println(a.User.Name)     // Admin User    fmt.Println(a.DisplayName()) // [Admin] Admin User    fmt.Println(a.IsAdmin())     // true}
```

我们可以看到，`User`类型很简单，但是`Admin`类型有一个内嵌的`User`。 `Admin`类型的实例可以访问嵌入类型`User`的字段和方法，就像访问它们是自己的一样。当我们在开始时打`Name`和`User.Name`时，它们的输出是完全一样的。


有趣的部分在于当覆盖内嵌类型的方法时。在`DisplayName`方法可以看到，在调用内嵌类型的同名方面之前，增加了一个当前类型的的前缀字符串`[Admin]`。

内嵌类型在实现接口的时候很有用。如果一个类型的内嵌类型实现了某个接口，那么就等价于该接口也自动的实现了那个接口。这是因为嵌入类型的所有方法都可用于嵌入它的类型，并且方法这些方法恰恰接口定义的时候所需要实现的方法。

> 内嵌类型与子类的区别 
> 
> 多数面向对象语言中的子类化意味着创建一个从另一个类继承的类，继承所有的方法和属性。主要区别是在这些情况下，父类和子类共享其属性和方法; 即访问从子到父或者父到子两种方式。 然而，在Go中，内嵌式类型没有意识到它嵌入的类型，所以前者不能访问后者的字段或方法（父不能访问子）。

## defer指令

本章将会介绍最后一个语言特性**defer**指令。此命令可以让我们在当前函数返回后再运行一个函数。defer定义的所谓延迟函数，可以让我们确保，函数无论什么时候返回，都会执行这个延迟函数，这有利于将函数运行完毕的收尾功能代码集中组织。


一个简明的例子就是在函数中关闭文件。使用`defer`定义一个函数用于关闭所打开的文件，此时不用担心因为别的原因过早的返回函数时主动去关闭文件。

```

func CopyFile(dstName, srcName string) error {    src, err := os.Open(srcName)    if err != nil {return err }    defer src.Close()    dst, err := os.Open(dstName)    if err != nil {return err }    defer dst.Close()    _, err := io.Copy(dst, src)return err }
```

在这个例子中，我们打开两个不同的文件，以便我们可以从一个复制到另一个。通过在打开`src`文件之后调用`defer src.Close()`，我们可以避免在函数结束之前，或者在打开`dst`文件时返回错误的时候提醒自己记住调用它。这减少了我们必须编写的代码量，并使代码更稳定。如果函数改变，比如需要添加一段代码逻辑并且会返回，此时则不需在新增的代码后面修改原有的关闭逻辑。

## 三方包

到目前为止，我们只在示例中使用了标准库; 它们作为Go安装的一部分代码。 然而，在现实生活中，您将同时使用标准库和第三方代码。Go的第三方生态系统使其运行起来非常容易，事实上只需要一个`go get`命令工具即可使用`Internet`上托管第三方软件包。下面将会介绍如何使用第三方包。在后面的章节中，还将研究如何创建和分发我们自己的包。

> 源码管理软件
> 
> Go can download third-party libraries stored in several varieties of source control software. If you attempt to download a package stored in source control software not on your system, you’ll see a message along the lines of “go: missing Git com- mand. See http://golang.org/s/gogetcmd.” The vast majority of packages are stored in Git, so I suggest you install it if you're yet to do so. Visit http://golang.org/s/gogetcmd now and follow the link to install it. Go also supports Mercurial, Subversion, and Bazaar.

Go可以下载几个源代码控制软件中托管的第三方库。如果您尝试下载存储在源代码控制软件而不是系统上的软件包，您将看到一条消息，显示`“go：missing Git command”`。 请参见[http://golang.org/s/gogetcmd](http://golang.org/s/gogetcmd)。“绝大多数软件包都存储在Git中，因此建议您安装它，如果你还没有这样做。立即访问[http://golang.org/s/gogetcmd](http://golang.org/s/gogetcmd)，并按照链接安装。`Go`也支持`Mercurial`，`Subversion`和`Bazaar`。


为了实践三方包的使用，我们将会写一个简单的程序将`markdwon`文本转换成`HTML`文件。如果不熟悉`markdwon`也没关系。它只是写一个格式化文本的方式，由于不用使用`HTML`标记就能写出漂亮的标记文本而广受欢迎。

我们将会使用**Russ Ross**所开发的库`Black Friday`。这个库托管在`Github`上`github.com/russross/blackfriday`。使用`go get`命令下载安装：

```go get github.com/russross/blackfriday
```

该命令将会把`blackfriday`包安装在`$GOPATH`的源码文件夹下，即`$GOPATH/src/git- hub.com/russross/blackfriday`。命令执行一切正常将不会有任何输出。重复输入将会很快返回，因为go检查到了已经下载了文件。为了强制更新下载，需要给命令添加一个`-u`的参数标记。
第三方安装的位置在 现在可以轻松的通过第三方的安装位置`github.com/russross/blackfriday`导入代码使用：

```
package mainimport (    "fmt"    "github.com/russross/blackfriday")func main() {    markdown := []byte(`# This is a header* and* this* is*a* list    `)    html := blackfriday.MarkdownBasic(markdown)    fmt.Println(string(html))}
```

运行上述代码，可以看到输出：

```
<h1>This is a header</h1><ul><li>and</li><li>this</li><li>is</li><li>a</li><li>list</li></ul>
```

这里有几点值得注意。首先是我们使用反引号（``` ` ```）创建一个跨越多行的字符串，我们还将它转换为一个字节数组，用圆括号括起来，前面加一个类型`[]byte('一些字串')` 。我们将在后面的章节讨论字节数组，但是暂时只是将它们看作是表示字符串的另一种方式。一旦我们准备好了`markdown`文本，可以创建一个新的变量html。这也是一个字节数组，所以当我们打印出来，我们首先将它转换回字串`string(html)`。

你会注意到我们将库称为`blackfriday`，而不是完整路径。包名称不是由`URL`中定义的，而是由库的代码中的`packege xyz` 指令定义的; 一般惯例是路径的最后部分的名称和所导入的包的名称匹配。(即 `github.com/russross/xyz`和`package xyz`中的`xyz`一致，`xyz`是**文件夹名**，而不是**文件名**)。


> Go 获取库
> 
> 值得注意的是，`go get`命令不只是下载您指定的库，它还会下载您正在下载的库所依赖的其他的库。 因此，即使您只需要一个库，在`$GOPATH`中看到多个项目也是很常见的。你也可以使用这个优势：如果我们忘记运行`go get github.com/russross/blackfriday`，我们的代码将无法编译。但是我们可以在目录代码中直接运行`go get`而不带任何参数，Go会根据源码中import的关系自动下载所有的，Go会下载所有的依赖关系。 如果你想尝试这个，你可以删除`$GOPATH/src/github.com/russross/blackfriday`目录，并运行`go get`从上面的示例代码的目录。

## 约定语法规则和惯用风格

Go对于你应该如何编写代码是相当有见地的; 事实上，它有一些非常坚定的规则。一些格式化由编译器强制执行，而一些是约定。

### 约定语法规则

下面介绍一些语法风格，部分是日常编程所使用的或者别的库里所遇见的。

#### 初始化匿名字段的结构

实例化结构的时候可以不用指定字段的名字。只需要安装结构多定义的顺序书写即可，也可以不按照顺序，但是必须把字段名写全。否则，将会出现编译错误。使用这种方法，甚至可以把值都写在一行上：

```
type Human struct {		Name string		Age int 
}
// 正确的做法me := Human{"Mal Curtis", 29}

// 错误，29多种的位置是string，int值将非法，同理”Mal Curtis“ 也不是intme = Human{29, "Mal Curtis"}
// 错误，缺少指定的字段值，无法得知是第一个还是第二个参数
me = Human{"Mal Curtis"}

// 正确，指定了Name，Age则取零值
me = Human{Name:"Mal Curtis"}
// 错误，字段名没有写全me = Human{"Mal Curtis", Age: 29}

```


#### 变量初始化空值

到目前为止，我们创建变量都是使用`:=`符号声明并初始化。实际上你也可以使用变量名跟岁类型关键字（变量名 类型）的语法进行声明变量，然后再使用所声明的类型值进行赋值：

```
var myString string // myString 变量的空值是空字串（`""`）myString = "Hello!" // 直接赋值，无需`:=`符号声明并赋值
```

使用上述的方式的时候，在声明变量的时候，变量将自动获得一个所属类型的**空值**（empty value）。例如，数字类型的空值是`0`，字串的空值是`""`空串，指针类型的空值是`nil`，切片和图的这些应用类型也是`nil`

```
var myMap map[string]string // myMap 的空值是 nilmyMap["Test"] = "Hi" // panic: 赋值错误，不能把给nil值的图添加键值对
```

你需要先初始化并赋值给变量，然后才能添加键值对：

```
myMap = map[string]string{}
```


#### 尾部的逗号

关于替代语法的最后一点是，你可以经常把序列的元素使用逗号分隔为多行，但每行必须有一个尾随逗号。一些语言或语法不允许尾随逗号，这使得重新排列列表的顺序会很烦人（因为要经常删掉最后一项的尾逗号）。然而，Go不仅允许尾随逗号，还不可或缺，否则代码将无法编译。记住，这仅适用于在多行上书写时，单行的时候必须去掉尾逗号。我们在初始化结构和图时已经见识到了，切片等序列也同样适用：

```
mySlice := []string{"one", "two"} // 好的风格mySlice := []string{    "one",    "two",} // 另外一种写法
mySlice := []string{"one", "two",} // 编译错误，必须去掉尾逗号
myMap := map[string]string{"one": "1", "two": "2"} // 正确的做法myMap := map[string]string{    "one": "1",    "two": "2",} // 合法的写法myMap := map[string]string{    "one": "1",    "two": "2"} // 编译错误
```

这也适用于函数的定义和调用。 如果定义参数过多，可以在多行上列出参数和返回值。规则是按照惯例，每一行结束都需要一个逗号，除非没有下一行了：（译者注：多行的项中，最后一项的后面如果没有`)`圆括号或者`}`花括号，都需要尾逗号）

```
// 通常的函数func JoinTwoStrings(stringOne, stringTwo string) string {    return stringOne + stringTwo}

// 参数分行，行结束的时候没有圆括号或花括号func JoinTwoStrings(stringOne,    stringTwo string,) string {    return stringOne + stringTwo}myString := JoinTwoStrings("Hello", "World")
myString := JoinTwoStrings(	"Hello",	"World", 
)

```

### 代码风格

虽然不一定要遵循这些约定，但是这样做是有意义的，因为它们被标准库和大多数Go开发人员使用。这些约定中的一些是有争议的，如使用`tab`缩进的。除此之外，其他规则则相当平常，例如选择驼峰（camel-casing）是命名变量。

下面介绍一些常见的约定，这并不详尽，但是我相信你会在日常编程中使用。

#### 变量命名

使用小写字母开头的驼峰式命名变量（译者注：Go的首字母大小写与访问私有性有关，建议使用驼峰式，而不是下划线的方式，下面的例子是以组为单位）：

```
// 好风格的一组
myVar := "something" 
variable := "something" 

// 坏风格的一组MyVar := "something" my_var := "something" 
```

当使用单词缩写时，根据初始字符的情况，坚持一致的情况：

```
// 好风格一组
id := "123" myID := "123"

// 坏风格的一组Id := "123" myId := "123" 
```

#### 常量（Constants）

别的语言中常量通常是全是大写的字母，可是Go却不是这样的约定，Go依然使用与变量一致的驼峰式命名：

```
const retryTimeout = 60 // 好的风格const RETRY_TIMEOUT = 60 // 坏的风格const Retry_TIME_OUT = 60 // 糟糕的风格
```

#### 构造（Constructors）

要注意的最后一个方面是初始化类型的新实例的方式。GO不像许多语言一样具有**构造函数**（constructor）的概念，惯用的做法是用`new`或`New`开始（取决于是否要导出）后跟类型的函数。这个函数通常会采用结构使用的时候所需要的字段为参数。例如，对于前文涉及的`Movie`类型，我们可能有一个`NewMovie(title string，year int) Movie`函数，因为我们只需要`title`和`year`字段。其他字段值可以以后再追加：

```
type Movie struct {    Actors      []string    Rating      float32    ReleaseYear int    Title       string}func NewMovie(title string, year int) Movie {    return Movie{        Title: title,        ReleaseYear: year,
        Actors: []string{},    }}
```

注意我们如何为`Actor`字段创建一个空字符串数组？这些新函数提供了类型所需字段初始化空值的机会，并且通常的做法是在`new`函数中初始化空数组或图。

## 总结

如果你已经做了这么远，你几乎涵盖了如何编写Go程序的整个基本原理！在下一章中，我们将看看一些Web开发相关代码。我们还将介绍如何处理HTTP请求和响应，如何处理模板，以及如何读写JSON。

