# GO 创建 RESTful JSON API 实战

本文不仅涉及如何使用`Go`创建`RESTful JSON API`，同时讨论好的`RESTful`设计。如果你曾经使用过不遵守好的设计规则的`API`，那么你也会因此写出设计不好的代码。幸好，本文之后，你将会对好的`API`行为有更好的认识。

### JSON API

`API`方式通信方式中，`JSON`问世之前是`XML`。毫无疑问，人们使用了`XML`和`JSON`之后，`JSON`成为了赢家。我不打算深入的介绍`JSON API`的概念，更多细节可以访问[jsonapi.org](http://jsonapi.org/)。

### 基本的Web Server

`RESTful`服务首先也是一个最基础的`web`服务。下面是一个基本的`web`服务，用于将任何请求的`url`简单的响应输出：

```
package main

import (
    "fmt"
    "html"
    "log"
    "net/http"
)

func main() {
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Hello, %q", html.EscapeString(r.URL.Path))
    })

    log.Fatal(http.ListenAndServe(":8080", nil))

}
```

运行这个例子，将会开启一个监听8080端口的服务器，然后可以访问`http://localhost:8080`

### 增加路由

虽然Go标准库（standard）提供了路由（router），我还是发现很多人对如何使用它很迷困惑。我曾经在自己的项目中使用过一下第三方的router。主要是来自[Gorilla Web Toolkit](www.gorillatoolkit.org)工具包的[mux router](http://www.gorillatoolkit.org/pkg/mux)库

另外一个流行的`router`是`Julien Schmidt`写的[httprouter](https://github.com/julienschmidt/httprouter)。

```
package main

import (
    "fmt"
    "html"
    "log"
    "net/http"

    "github.com/gorilla/mux"
)

func main() {

    router := mux.NewRouter().StrictSlash(true)
    router.HandleFunc("/", Index)
    log.Fatal(http.ListenAndServe(":8080", router))
}

func Index(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hello, %q", html.EscapeString(r.URL.Path))
```

运行这个例子之前，你需要在当前目录下执行下面命令（安装第三方库`mux`）

```
go get
```

该命令将解析`github.com/gorilla/mux`代码，从`GORILLA MUX`包的`GitHub`版本库下载。

上述的例子创建了一个基本的`router`，即添加了路由模式`/`与`Index`函数处理器的绑定，当请求的路径（endpoint）匹配的时候函数被调用执行。你也会注意到了，请求`http://localhost:8080/foo`也一样匹配路由模式。如果没有定义路由，那么只有访问`http://localhost:8080`才会有响应。


### 更多的路由

现在我们绑定了一个路由函数，是时候创建更多的路由规则啦。

假设我们将要创建一个基本的`Todo`应用。

```
package main

import (
    "fmt"
    "log"
    "net/http"

    "github.com/gorilla/mux"
)

func main() {

    router := mux.NewRouter().StrictSlash(true)
    router.HandleFunc("/", Index)
    router.HandleFunc("/todos", TodoIndex)
    router.HandleFunc("/todos/{todoId}", TodoShow)

    log.Fatal(http.ListenAndServe(":8080", router))
}

func Index(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintln(w, "Welcome!")
}

func TodoIndex(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintln(w, "Todo Index!")
}

func TodoShow(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    todoId := vars["todoId"]
    fmt.Fprintln(w, "Todo show:", todoId)
}

```

我们已经多增加了两个路由规则。

Todo 列表路由： http://localhost:8080/todos

Todo 展示路由：	http://localhost:8080/todos/{todoId}

这是设计RESTful的开始。

请注意最后一个路由`http://localhost:8080/todos/{todoId}`，我们在路径中增加了一个变量`todoId`。

这将允许我们传递一个`id`参数到路由中，以便响应与之管理的`todo`记录。

### 数据模型

现在我们有了路由规则，下面将要创建一个基础的`Todo`数据模型，以便我们可以读写数据。在`Go`中，结构体（struct）是实现数据模型最典型的方式。许多其他语言使用类（Class）实现。

```
package main

import "time"

type Todo struct {
    Name      string
    Completed bool
    Due       time.Time
}

type Todos []Todo
```

注意最后一行代码，我们创建了另外一个称之为`Todos`的类型，它是一个`Todo`类型元素的切片（slice）即有序集合。很快你就会发现它很有用。

### 返回JSON

我们有了基本的数据模型，现在可以在TodoIndex函数中mock静态数据模拟一个真实的响应：

```
func TodoIndex(w http.ResponseWriter, r *http.Request) {
    todos := Todos{
        Todo{Name: "Write presentation"},
        Todo{Name: "Host meetup"},
    }

    json.NewEncoder(w).Encode(todos)
}
```

上面的例子中，我们创建了一个包含`Todo`的`Todos`切片返回给客户端。请求`thttp://localhost:8080/todos`将会得到如下的响应：

```
[
    {
        "Name": "Write presentation",
        "Completed": false,
        "Due": "0001-01-01T00:00:00Z"
    },
    {
        "Name": "Host meetup",
        "Completed": false,
        "Due": "0001-01-01T00:00:00Z"
    }
]
``` 

### 更好的模型

对于一个经验丰富的开发者，你肯定注意到一个问题。虽然无伤大雅，可是使用驼峰式作为的key不符合JSON的习惯。修改如下：

```
package main

import "time"

type Todo struct {
    Name      string    `json:"name"`
    Completed bool      `json:"completed"`
    Due       time.Time `json:"due"`
}

type Todos []Todo
```

使用结构体标签（tag），就能精确的控制任何结构体被`JSON`序列化后的`key`。

### 拆分组织文件

接下来，因为我们的项目有太多的代码都在一个文件里，因此我们我们需要重构啦。

我们将会创建下面几个文件，并把代码移到相应的文件里：

* main.go
* handlers.go
* routes.go
* todo.go


Handler.go

```
package main

import (
    "encoding/json"
    "fmt"
    "net/http"

    "github.com/gorilla/mux"
)

func Index(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintln(w, "Welcome!")
}

func TodoIndex(w http.ResponseWriter, r *http.Request) {
    todos := Todos{
        Todo{Name: "Write presentation"},
        Todo{Name: "Host meetup"},
    }

    if err := json.NewEncoder(w).Encode(todos); err != nil {
        panic(err)
    }
}

func TodoShow(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    todoId := vars["todoId"]
    fmt.Fprintln(w, "Todo show:", todoId)
}

```

Routes.go

```
package main

import (
    "net/http"

    "github.com/gorilla/mux"
)

type Route struct {
    Name        string
    Method      string
    Pattern     string
    HandlerFunc http.HandlerFunc
}

type Routes []Route

func NewRouter() *mux.Router {

    router := mux.NewRouter().StrictSlash(true)
    for _, route := range routes {
        router.
            Methods(route.Method).
            Path(route.Pattern).
            Name(route.Name).
            Handler(route.HandlerFunc)
    }

    return router
}

var routes = Routes{
    Route{
        "Index",
        "GET",
        "/",
        Index,
    },
    Route{
        "TodoIndex",
        "GET",
        "/todos",
        TodoIndex,
    },
    Route{
        "TodoShow",
        "GET",
        "/todos/{todoId}",
        TodoShow,
    },
}
```

Todo.go

```
package main

import "time"

type Todo struct {
    Name      string    `json:"name"`
    Completed bool      `json:"completed"`
    Due       time.Time `json:"due"`
}

type Todos []Todo

```

Main.go

```
package main

import (
    "log"
    "net/http"
)

func main() {

    router := NewRouter()

    log.Fatal(http.ListenAndServe(":8080", router))
}
```

### 更好的路由

在部分重构中，我们创建了更多功能的路由文件。新文件可以充分利用结构来包含更多路由的细节信息。尤其是可以指定特殊的请求方法，例如 `GET`, `POST`, `DELETE`等等。

像大多数现代的webserver一样 我们增加日志（log）功能。在Go中，标准库并没有`logging`包或功能函数，因此我们需要自己创建。

首先创建一个`logger.go`的文件，然后写入下面的代码：

```
package main

import (
    "log"
    "net/http"
    "time"
)

func Logger(inner http.Handler, name string) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()

        inner.ServeHTTP(w, r)

        log.Printf(
            "%s\t%s\t%s\t%s",
            r.Method,
            r.RequestURI,
            name,
            time.Since(start),
        )
    })
}
```

这是标准的Go风格。我们将有效的把`handler`函数传递到该函数中，是得被包裹的`handler`函数可以有打印日志和访问时间的功能。

### 使用 Logger 装饰器

为了使用日志装饰器，当创建路由的时候，我们需要在当前的路由器中进行一次包裹，更新`NewRouter`函数如下：

```
func NewRouter() *mux.Router {

    router := mux.NewRouter().StrictSlash(true)
    for _, route := range routes {
        var handler http.Handler

        handler = route.HandlerFunc
        handler = Logger(handler, route.Name)

        router.
            Methods(route.Method).
            Path(route.Pattern).
            Name(route.Name).
            Handler(handler)
    }

    return router
}
```

现在当你访问`http://localhost:8080/todos`的时候将会看见命令行中如下的输出日志：

```
2014/11/19 12:41:39 GET /todos  TodoIndex       148.324us
```
	
### 路由器看起很疯狂，重构吧

路由的文件变得很大了，我们需要再次拆分：

* router.go
* routes.go

Routes.go - redux
```
package main

import "net/http"

type Route struct {
    Name        string
    Method      string
    Pattern     string
    HandlerFunc http.HandlerFunc
}

type Routes []Route

var routes = Routes{
    Route{
        "Index",
        "GET",
        "/",
        Index,
    },
    Route{
        "TodoIndex",
        "GET",
        "/todos",
        TodoIndex,
    },
    Route{
        "TodoShow",
        "GET",
        "/todos/{todoId}",
        TodoShow,
    },
}
```
	
Router.go

```
package main

import (
    "net/http"

    "github.com/gorilla/mux"
)

func NewRouter() *mux.Router {
    router := mux.NewRouter().StrictSlash(true)
    for _, route := range routes {
        var handler http.Handler
        handler = route.HandlerFunc
        handler = Logger(handler, route.Name)

        router.
            Methods(route.Method).
            Path(route.Pattern).
            Name(route.Name).
            Handler(handler)

    }
    return router
}
```
	
### 构造一下响应数据

现在我们有了一个好的框架蓝图，事实婚重写我们的`handler`处理函数啦。我们需要一些数据响应。首先修改`TodoIndex`函数，只要增加两行代码即可：

```
func TodoIndex(w http.ResponseWriter, r *http.Request) {
    todos := Todos{
        Todo{Name: "Write presentation"},
        Todo{Name: "Host meetup"},
    }

    w.Header().Set("Content-Type", "application/json; charset=UTF-8")
    w.WriteHeader(http.StatusOK)
    if err := json.NewEncoder(w).Encode(todos); err != nil {
        panic(err)
    }
}
```

有两件事将会发生。首先，我们将`Content type`在响应返回，并告知了客户端所期望的`json`格式。其次，我们明确的设置你了状态码（status code）。

`Go`的`net/http`包将尝试猜测输出的`content type` 类型（当然这并不一定总是正确），因此只要我们知道type类型，最好就是显式设置。


### 等等，数据库在哪儿呢

如果我们打算创建一个`RESTful API` 服务，很明细我们需要某个地方用于读取和存储我们的数据。可是，这超出了本篇文章的范畴，因此我们将会实现一个简单（非线程安全）的`mock`数据库。

创建一个`repo.go`的文件增加如下内容：

```
package main

import "fmt"

var currentId int

var todos Todos

// 初始化一些数据
func init() {
    RepoCreateTodo(Todo{Name: "Write presentation"})
    RepoCreateTodo(Todo{Name: "Host meetup"})
}

func RepoFindTodo(id int) Todo {
    for _, t := range todos {
        if t.Id == id {
            return t
        }
    }
    // 如果没有找到则返回空的Todo
    return Todo{}
}

func RepoCreateTodo(t Todo) Todo {
    currentId += 1
    t.Id = currentId
    todos = append(todos, t)
    return t
}

func RepoDestroyTodo(id int) error {
    for i, t := range todos {
        if t.Id == id {
            todos = append(todos[:i], todos[i+1:]...)
            return nil
        }
    }
    return fmt.Errorf("Could not find Todo with id of %d to delete", id)
}
```

### 给todo模型增加ID字段

现在我们有了一个`mock`数据库，我们将会使用`id`来更新`todo`模型结构。

```
package main

import "time"

type Todo struct {
    Id        int       `json:"id"`
    Name      string    `json:"name"`
    Completed bool      `json:"completed"`
    Due       time.Time `json:"due"`
}

type Todos []Todo
```

### 更新TodoIndex函数

因为有了初始化的数据，我们现在需要在`TodoIndex`函数中读取数据库的内容，修改函数如下：

```
func TodoIndex(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json; charset=UTF-8")
    w.WriteHeader(http.StatusOK)
    if err := json.NewEncoder(w).Encode(todos); err != nil {
        panic(err)
    }
}
```

### 解析 JSON

目前为止，我们值输出了`JSON`，现在需要读取并存储一些`JSON`数据了。

增加下面的路由代码到`routes.go`文件中：

```
Route{
    "TodoCreate",
    "POST",
    "/todos",
    TodoCreate,
},
```

### Create 函数

在`handler`文件中增加创建todo的函数：

```
func TodoCreate(w http.ResponseWriter, r *http.Request) {
    var todo Todo
    body, err := ioutil.ReadAll(io.LimitReader(r.Body, 1048576))
    if err != nil {
        panic(err)
    }
    if err := r.Body.Close(); err != nil {
        panic(err)
    }
    if err := json.Unmarshal(body, &todo); err != nil {
        w.Header().Set("Content-Type", "application/json; charset=UTF-8")
        w.WriteHeader(422) 
        if err := json.NewEncoder(w).Encode(err); err != nil {
            panic(err)
        }
    }

    t := RepoCreateTodo(todo)
    w.Header().Set("Content-Type", "application/json; charset=UTF-8")
    w.WriteHeader(http.StatusCreated)
    if err := json.NewEncoder(w).Encode(t); err != nil {
        panic(err)
    }
}
```
首先我们读取请求的body。请注意`io.LimitReader`限制了读取的大小。这样做得好处是可以保护对服务器的恶意攻击。想像一些，如果有人不怀好意的发送500GB的json数据。


读取完`body`的数据之后，然后反序列化Todo结构体。我们将要处理失败的情况，不仅返回明确的状态码422，还需要把错误编码成`JSON`返回。这不仅让客户端知道发生了错误，还可以针对特殊的错误做相应的处理。

最后，一切正常的话，将会返回201的状态码，它代表这一个实体被成功创建。同时，我们也把创建成功之后的实体信息返回给客户端，其中包含一个`id`字段，用于客户端做下一步操作。


### 提交一些JSON


现在，可以创建一些数据。通过如下命令使用`curl`：

```
curl -H "Content-Type: application/json" -d '{"name":"New Todo"}' http://localhost:8080/todos
```

访问 http://localhost/todos 将会看见下面的输出：

```
    {
        "id": 1,
        "name": "Write presentation",
        "completed": false,
        "due": "0001-01-01T00:00:00Z"
    },
    {
        "id": 2,
        "name": "Host meetup",
        "completed": false,
        "due": "0001-01-01T00:00:00Z"
    },
    {
        "id": 3,
        "name": "New Todo",
        "completed": false,
        "due": "0001-01-01T00:00:00Z"
    }
]
```

### 我们没有做的

虽然我们有一个伟大的开始，可是还有很多事情要做。 我们没有解决：

* 版本控制 -- 如果我们修改API，返回的结构不兼容以前的版本。我们最好在接口路由中添加`/v1/`前缀

* 认证 -- 除非是免费/开放的API，否则我们需要认证。我推荐学习`JSON web token`机制

* eTags -- 如果构建的一些api需要分急，你需要实现`eTags` 


###  还剩什么？

与所有项目一样，开始都微不足道，但可以迅速失控。 如果我要达到一个新的水平，并且服务可以在生产环境使用，还有一些额外的事情要做：

* 更多的重构
* 把上面的文件分隔在不同的包里面。例如创建一些JSON的`helpers`包， `decorators`包，`handlers`包等等
* 测试。不要忘记测试。我们并没有任何测试代码。对于生成环境，测试是必不可少的。



### 我能获得源码吗？

是的，下面是源码仓库地址：


[https://github.com/corylanou/tns-restful-json-api](https://github.com/corylanou/tns-restful-json-api)


### 总结


对于我而言，最重要的事情就是构建一个健壮负责的`API`。严格的返回目标状态码，包含`headers`等让你的`API`适用范围更广。我希望本篇文章能够启发你不久之后创建属于自己的API。


原文地址：[http://thenewstack.io/make-a-restful-json-api-go/](http://thenewstack.io/make-a-restful-json-api-go/)


