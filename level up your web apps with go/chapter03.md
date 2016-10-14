# HTTP

发送和接受`HTTP`请求是每一个`web`应用程序的核心工作。Go的标准库提供了很多包让我们轻松的驾驭http请求和响应。本章中，将会涉及响应请求，使用中间件（middleware）路由请求，创建`HTML`模板和响应`JSON`字串，所有这些功能都只依赖Go的标准库完成。

## 响应请求

Go创建web服务器相当简单。在多数其他语言，创建一个web服务器通常还需要一个额外的服务器软件，例如`PHP`需要`Apache`。Go的标准库提供了`http`包，我们无效依赖第三方包，仅使用`http`就能创建web服务。

有多种方式创建GOhttp服务，最简单的方式就是指定一个`url`模式（一下简称模式或URL模式），并注册一个**处理函数**，这个函数的签名是`func(w http.ResponeWriter, r *http.Request)`。一旦url匹配了模式，就会调用注册的函数处理请求。接下来可以调用`http.ListenAndServer`启动服务监听：

```
package main
import (    "fmt"	  "net/http"
)
func main() {    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {			fmt.Fprintf(w, "Hello Gopher")	  })    http.ListenAndServe(":3000", nil)}
```

这个示例中，当请求访问端口3000的任何路径时，注册的函数就会执行。函数中使用了`fmt.Fprintf`往`response writer`对象写入字串`“Hello Gpher”`来返回给客户端的响应。

> 路径模式匹配
> 
> 模式`/`将匹配任何路径的请求。关于url路径的模式匹配需要一点解释，后面的章节再详细阐述。


运行上述代码，使用浏览器访问`http://127.0.0.1:3000`，将会看到如下的显示：

![http hello world](./images/3.1.png)

### 一探究竟

处理函数（handler function）的两个参数提供了给任何请求响应所需要的全部功能。

第一个参数是`http.ResponseWriter`的实例，我们将返回给浏览器的信息写入这个对象中。响应主要由页面的主体（body）组成，不过也可以提供对其他信息其他例如响应头（headers）和状态码（status code）的访问。

> 所谓浏览器
> 
> 本书中发送任何方法的请求所用的“浏览器”不仅仅是浏览器软件，还可以是命令行工具例如`cURL`
> 如果安装了`cURL`， 可以使用命令`curl -i 127.0.0.1:3000`替代浏览器软件。

第二个参数是`http.Request`的实例，它涵盖我们期望看到的所有请求的信息，包括主体数据（data body），查询字串（query string）和请求头（headers）。

### 更多细节

现在的例子响应的信息并不多。实际上，真正的HTTP（raw HTTP） 响应也很小：

```
HTTP/1.1 200 OKDate: Mon, 01 Jun 2015 08:03:30 GMTContent-Length: 12Content-Type: text/plain; charset=utf-8
Hello Gopher
```

第一行的含义是告知接受一个处理成功的HTTP响应（OK HTTP）。然后是少许`header`信息，响应的日期`date`和`content`内容。注意`Content-Type`的值是`text/plain`。Go会读取响应主体的前512个字节并嗅探出响应的类型。例子中并没有明细的类型，因此返回了基本的text类型。除了`header`之外，剩下的内容为响应的主体。

要提供更多信息，我们需要操作处理函数中的`ResponseWriter`对象。通过一个命名贴切的`Header`方法返回一个`Header`结构的图来修改响应头`header`。可以通过其`Add`方法给响应头追加内容：

```
w.Header().Add("Server", "Go Server")
```

写入一些`HTML`标签，将会触发go嗅探出返回的`content-type`为`text/html`类型：

```
fmt.Fprintf(w, `<html>    <body>        Hello Gopher    </body></html>`)
```
再次运行服务器代码，将会收到新的`Server`响应头和`Content-type`：

```
HTTP/1.1 200 OKServer: Go ServerDate: Mon, 01 Jun 2015 09:12:32 GMTContent-Length: 57Content-Type: text/html; charset=utf-8<html>    <body>        Hello Gopher    </body></html
```

### 深入URL模式匹配

因为所有的请求是 `http.ListenAndServe`所监听的端口接受处理。因此我们需要一种将不同资源的请求路由到我们代码的不同部分的方法。幸运的是Go提供了`http.ServeMux`结构原生支持这样的方式。`ServeMux`是一个请求多路复用器，它将请求的`URL`与模式中中`url`进行匹配，并执行最匹配模式的处理函数。

`http`包提供了一个默认的`DefaultServeMux`实例，并且`http.HandleFunc`也是`DefaultServeMux`方法`(*ServeMux) HandleFunc`的包装。

当你创建`ServeMux`的时候，你可以为不同的URL模式注册不同处理函数。模式（Pattern）不必完全匹配路径（Path）。 有两种类型的模式：路径（path）和子树（subtree）。路径的结尾没有斜杠`/`，匹配严格的路径。子树的结尾包含斜杠`/`，并且匹配所有以子树开头的url。因此，在下一个例子中，请求`/articles/latest`将会返回`“Hello from /articles/”`，因为url`/articles/latest`与模式中的`/articles/`子树匹配。可是访问`/users/latest`将会返回一个404错误，因为模式`/users`缺少尾部的`/`，需要完全匹配（译者注：下面注释部分为源代码，可是是有问题的用法，参考上下文和go源代码，正确的用法为非注释的地方）：

```
/*
func main() {

	mux := http.NewServeMux()
	mux.HandleFunc("/articles/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "Hello from /articles/")
	})
	mux.HandleFunc("/users", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "Hello from /users")
	})
	http.ListenAndServe(":3000", mux)
}
*/

func main() {

	mux := http.NewServeMux()
	mux.HandleFunc("/articles/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "Hello from /articles/")
	})
	mux.HandleFunc("/users", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "Hello from /users")
	})
	http.ListenAndServe(":3000", mux)
}

```

模式的长度也很重要。模式越长的优先级越高。例如，模式`/articles/latest/`要比`/articles/`优先级高。因为有了顺序优先级，所以与添加路由的顺序没有区别。添加在`/articles/latest/`的路由规则在`/articles/`的处理程序之前，实际访问的时候也和预期绝对一致：

```
func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/articles/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "Hello from /articles/")
	})
	mux.HandleFunc("/articles/latest/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "Hello from /articles/latest/")
	})
	mux.HandleFunc("/users", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "Hello from /users")
	})
	http.ListenAndServe(":3000", mux)
}

```
> 主机名匹配
> 
> 路由匹配的时候也可以从主机名开始，并且只有匹配的主机名才算真正的匹配。主机名匹配的模式拥有最高的优先级。

### 返回错误

有时候事与愿违，程序并不会正常工作，此时就需要返回一个错误，至少需要返回与`200 ok`不一样的响应。Web开发中状态码至关重要；如果没有状态码，访问不存在的资源的时候，或者网页的移动了，又或者遇了问题的时候希望能够回退。

常见的几个状态码：

■ 301 Moved Permanently― 重定向页面■ 404 Not Found― 访问的资源不存在■ 500 Internal Server Error―服务器内部错误

`http`包提供另一个`Error`函数用于返回错误，它需要两个参数，一个是`ResponseWriter`，另一个是整型的状态码。例如，500的错误返回如下：

```
http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {    http.Error(w, "Something has gone wrong", 500)})
```

每个状态代码都有语义，所以你应该努力使用最合适的。如果有疑问，请记住，你并不是第一个吃螃蟹的人，在你试图找到合适的状态码之前，可能有无数的人已经尝试做了。Google是你的好朋友。完整的状态代码的请查看维基百科文章：[List of HTTP status codes](https://en.wikipedia.org/wiki/List_of_HTTP_status_codes)

> 状态码常量
> 
> Go的http包提供了很多HTTP状态码的常量。你可以直接使用这些常量而不是自己声明整数的code。`http.StatusBadRequest`就比 400 更容易理解。
> 
> 几个常见的的`Helper函数`：
> `http.NotFound` 可能是最普遍错误码是 `404 Not Found`。`http.NotFound`函数的参数为`ResponseWriter` 和`Request`类型的实例。（`http.NotFound(w, r)`）
> 
> `http.Redirect` 它的参数除了是`ResponseWriter`和`Request`的实例之外还有两个参数，一个是字串类型的参数，表示需要重定向的`url`；另外一个是整型的数字状态码。重定向的状态码有好几个。有多种类型的重定向状态代码。 最常见的是301用于永久移动的资源，302用于临时移动的资源。（` http.Redirect(w, r, "http://google.com", http.StatusMovedPermanently)`）

### Handler接口

还有另一种方式注册函数来处理`HTTP`请求。任何实现接口`http.Handler`的类型都可以和模式字串一起传递到`http.Handle`函数。 正如我们之前使用`HandleFunc`函数看到的，这是使用默认ServeMux实例的快捷方式。

`http.Handler`接口只有一个函数，这个函数的签名和`http.HandleFunc`函数一致。

```
type Handler interface {    ServeHTTP(ResponseWriter, *Request)}
```
我们可以给任何类型实现`ServeHTTP`方法。如果我们想创建一个handler函数用来处理返回服务器当前的时间的响应。例如定义一个类型`UptimeHandler`并实现`Handler`接口的`ServeHTTP`方法：

```
package mainimport (    "fmt""net/http""time" )type UptimeHandler struct {    Started time.Time}func NewUptimeHandler() UptimeHandler {
   return UptimeHandler{ Started: time.Now() }}func (h UptimeHandler) ServeHTTP(w http.ResponseWriter, req *http.Request) {    fmt.Fprintf(        w,        fmt.Sprintf("Current Uptime: %s", time.Since(h.Started)),    )}func main() {    http.Handle("/", NewUptimeHandler())    http.ListenAndServe(":3000", nil)}
```

使用浏览器访问`http://127.0.0.1:3000`，就能看见服务器返回的当前时间。`time.Since`函数返回一个`time.Duration`结构对象，表示自服务器启动以来所经过的时间。`time.Duration`类型被很好地转换为人类可读的字串; 例如，`"Current Uptime: 4m20.843867792s"`。

### 中间件

## HTML 模板
### 访问模板数据
### 模板中的if else 条件判断
### 循环
### Multiple 模板
### 管道过滤器（Piplines）
### 模板变量

## 渲染 JSON
### 序列化（Marshling）
### 序列化结构体
### 自定义JSON字段名
### 内嵌类型
### 反序列化（Unmarshling）
### 未知的JSON结构处理
## 总结it



