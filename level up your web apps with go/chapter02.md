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




