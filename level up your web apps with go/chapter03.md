# HTTP

发送和接受`HTTP`请求是每一个`web`应用程序的核心工作。Go的标准库提供了很多包让我们轻松的驾驭http请求和响应。本章中，将会涉及响应请求，使用中间件（middleware）路由请求，创建`HTML`模板和响应`JSON`字串，所有这些功能都只依赖Go的标准库完成。

## 响应请求

Go创建web服务器相当简单。在多数其他语言，创建一个web服务器通常还需要一个额外的服务器软件，例如`PHP`需要`Apache`。Go的标准库提供了`http`包，我们无效依赖第三方包，仅使用`http`就能创建web服务。

有多种方式创建GOhttp服务，最简单的方式就是指定一个`url`模式，并注册一个**处理函数**，这个函数的签名是`func(w http.ResponeWriter, r *http.Request)`。一旦url匹配了，就会调用注册的函数处理请求。接下来可以调用`http.ListenAndServer`启动服务监听：

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

















### 返回错误
### Handler接口
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



