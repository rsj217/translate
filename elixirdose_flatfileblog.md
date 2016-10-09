# Cowboy 教程 2：创建一个基于文件的扁平化博客


> 前言：学习Elixir的时候，发现了一篇文章，使用Cowboy框架做一个简单的blog。看完之后受益匪浅，于是利用翻译的方式，重新阅读一遍。原文有两个小错误，已经修改。其次就是原文使用的 `markdwon`库，现在已经不可用了。为此更换了另外一个库`earmark`。Elixir作为函数式语言，递归能力很强大，在处理首页的时候，尤其体现了它的魔力。




如果你一直阅读ElixirDose的博文，那么肯定知道之前我们已经创建了一个十分简单的框架（[web framework](http://elixirdose.com/post/lets-build-a-web-framework)）。本篇文章，我们将会介绍一个响应markdown文件的服务器例子。我们的markdwon服务器要比之前行将就木博客程序更深入一点。上一篇文章确实有点过时，因此我们一起从头开始学习[cowboy](https://github.com/ninenines/cowboy), 一个小巧，快速，可扩展的Erlang写的HTTP服务器。

对于将要实现的博客程序，让我们先讨论更多关于它的细节吧。博客程序可以从文件夹中读取markdown文件。任何没有数据库渠道的博客都很快很酷炫。写文章的时候，只需要用编辑器写markdown文件接口，然后放入到文件夹中，随后就能发布文章了，快速且简单。我们还得支持更换主题，一遍有一天可以重新设计博客。

创建一个博客程序，首先需要知道下面的topics：

* 使用Cowboy处理静态文件（图片，css样式表，javascript）
* 将markdown文件转换成html文件
* 根据浏览器url动态的匹配markdown博客文件
* 渲染`EEx`模板，字符串操作，修改主题样式。

实现本教程的项目，需要如下条件：


* Elixir 1.0.0 或更高版本
* Cowboy 1.0.1 版本,
* 使用 ExDoc 作为 markdown工具

开始吧！

## 创建 Elixir 项目

使用`mix`工具开启elixir项目之旅。

```elixir
$> mix new dds_blog --sup
```
博客程序名`dds_blog`， 其为`Drop Dead Simple Blogging`的缩写。`--sup`参数表示让`mix`创建一个[OTP](http://learnyousomeerlang.com/what-is-otp) supervisor 程序和在 `DdsBlog`模块中创建[OTP](http://learnyousomeerlang.com/what-is-otp)应用回调函数。


别忘了进入刚才新建的dds_blog文件夹：

```
$> cd dds_blog
```

好了，是时候召唤cowboy出场啦。


## 添加 Cowboy


为了实现我们的博客，需要一个后端服务器处理http请求。[Cowboy](https://github.com/ninenines/cowboy)非常合适，并且足够[awesome](http://elixirdose.com/post/lets-build-a-web-framework)。

打开`mix.exs`文件，添加Cowboy为项目的依赖。

```elixir
defp deps do
    {:cowboy, "1.0.0"}
end
```

然后在`mix.exs`文件中的application中指定`:cowboy`：

```elxiir
def application do
    [applications: [:logger, :cowboy],
    mod: {DdsBlog, []}]
end
```

运行 `mix deps.get` 获取安装依赖
```elixir
$ mix deps.get
Running dependency resolution
Unlocked:   cowboy
Dependency resolution completed successfully
  ranch: v1.0.0
  cowlib: v1.0.1
  cowboy: v1.0.0

[..]
```

如您所见，除了Cowboy，我们还下载了cowboy本身的依赖`cowlib` 和 `ranch`。


## 创建 Cowboy 路由

创建一个函数用于定义项目的路由规则：

```elixir
def run do
  routes = [
    {"/", DdsBlog.Handler, []}
  ]

  dispatch = :cowboy_router.compile([{:_, routes}])

  opts = [port: 8000]
  env = [dispatch: dispatch]

  {:ok, _pid} = :cowboy.start_http(:http, 100, opts, [env: env])
end
```

项目应用启动的时候，可以自动用run函数。这个设置也是可选的，如果不这样，你就得手动调用一遍启动程序。其他的代码大概如下：

```elixir
defmodule DdsBlog do
  use Application

  def start(_type, _args) do
    import Supervisor.Spec, warn: false

    children = [
      worker(__MODULE__, [], function: :run)
    ]

    opts = [strategy: :one_for_one, name: DdsBlog.Supervisor]
    Supervisor.start_link(children, opts)
  end

  def run do
    routes = [
      {"/", DdsBlog.Handler, []}
    ]

    dispatch = :cowboy_router.compile([{:_, routes}])

    opts = [port: 8000]
    env = [dispatch: dispatch]

    {:ok, _pid} = :cowboy.start_http(:http, 100, opts, [env: env])
  end
end
```

如果启动程序，访问`http://localhost:8000/`就能看的响应结果啦。


## 处理请求

上面的代码可知，路由`"/"`绑定了处理模块`DdsBlog.Handler`。接下来可以在模块写代码，用于响应请求。创建一个`lib/dds_blog/handler.ex`文件，并添加下面的代码：

```elixir
defmodule DdsBlog.Handler do
  def init({:tcp, :http}, req, opts) do
    headers = [{"content-type", "text/plain"}]
    body = "Hello program!"
    {:ok, resp} = :cowboy_req.reply(200, headers, body, req)
    {:ok, resp, opts}
  end

  def handle(req, state) do
    {:ok, req, state}
  end

  def terminate(_reason, _req, _state) do
    :ok
  end
end
```

`init`函数做了很多工作。首先它告诉cowboy是什么类型的连接，我们将要处理基于TCP的HTTP。然后使用`:cowboy_req.reply` 函数设置了200状态码和http的headers列表，body内容和请求本身。

目前暂且不用处理`handle` 和 `terminate`函数，把他们当成样板来对待吧。

## 首次运行

现在开始第一次运行应用程序：

    $> iex -S mix
    Erlang/OTP 17 [erts-6.2] [source] [64-bit] [smp:4:4] [async-threads:10] [hipe] [kernel-poll:false] [dtrace]

    ==> ranch (compile)
    Compiled src/ranch_transport.erl
    Compiled src/ranch_sup.erl
    Compiled src/ranch_tcp.erl
    Compiled src/ranch_ssl.erl
    Compiled src/ranch_protocol.erl
    Compiled src/ranch_listener_sup.erl
    Compiled src/ranch_app.erl
    Compiled src/ranch_acceptors_sup.erl
    Compiled src/ranch_acceptor.erl
    Compiled src/ranch.erl
    Compiled src/ranch_server.erl
    Compiled src/ranch_conns_sup.erl
    ==> cowlib (compile)
    Compiled src/cow_qs.erl
    Compiled src/cow_spdy.erl
    Compiled src/cow_multipart.erl
    Compiled src/cow_http_te.erl
    Compiled src/cow_http_hd.erl
    Compiled src/cow_date.erl
    Compiled src/cow_http.erl
    Compiled src/cow_cookie.erl
    Compiled src/cow_mimetypes.erl
    ==> cowboy (compile)
    Compiled src/cowboy_sub_protocol.erl
    Compiled src/cowboy_middleware.erl
    Compiled src/cowboy_websocket_handler.erl
    Compiled src/cowboy_sup.erl
    Compiled src/cowboy_static.erl
    Compiled src/cowboy_spdy.erl
    Compiled src/cowboy_router.erl
    Compiled src/cowboy_websocket.erl
    Compiled src/cowboy_protocol.erl
    Compiled src/cowboy_loop_handler.erl
    Compiled src/cowboy_http_handler.erl
    Compiled src/cowboy_rest.erl
    Compiled src/cowboy_handler.erl
    Compiled src/cowboy_clock.erl
    Compiled src/cowboy_bstr.erl
    Compiled src/cowboy_app.erl
    Compiled src/cowboy_http.erl
    Compiled src/cowboy.erl
    Compiled src/cowboy_req.erl
    Compiled lib/dds_blog/handler.ex
    Compiled lib/dds_blog.ex
    Generated dds_blog.app

    Interactive Elixir (1.0.1) - press Ctrl+C to exit (type h() ENTER for help)
    iex(1)>


如果没有把`run`函数作为supervisor 树中的`worker`的子函数，那么就得在命令行运行项目。除此之外，一切顺利，打开浏览器访问`http://localhost:8000/`将会看到，全世界最美妙的编程单词 :)

![Hello world](http://i.imgur.com/J1jgiBh.png)


## 静态文件处理

给Cowboy增加静态文件处理功能：图片，css样式表，js脚本等。每当请求`http://localhost:8000/static/`的时候，cowboy将会从接下来将要创建的静态文件夹中提取文件响应。现在，先打开`lib/dds_blog.ex` 增加一个路由吧：


```elixir
def run do
  routes = [
    {"/:something", DdsBlog.Handler, []},
    {"/static/[...]", :cowboy_static, {:priv_dir, :dds_blog, "static_files"}}
  ]

  dispatch = :cowboy_router.compile([{:_, routes}])

  opts = [port: 8000]
  env = [dispatch: dispatch]

  {:ok, _pid} = :cowboy.start_http(:http, 100, opts, [env: env])
end
```

别忘了新建`priv/static_files`文件夹用于存放静态文件

```elixir
    $> mkdir -p priv/static_files
```

在`static_files`文件夹中放一些文件，然后运行`iex -S mix`启动程序，浏览器访问，将会看到响应图片啦：

![lily](http://i.imgur.com/ijQEq98.png)

很简单，对不对？

## 处理 markdown文件

有趣的才刚刚开始。思路是：访问`http://localhost:8000/some-markdown-file`，程序会寻找 `priv/contents/` 文件夹内名为`some-markdown-file.md` 的文件，然后将该文件的内容转成html格式，最后再把html内容响应给浏览器。

修改路由内容：

```elixir
def run do
  routes = [
    {"/:filename", DdsBlog.Handler, []},
    {"/static/[...]", :cowboy_static, {:priv_dir, :dds_blog, "static_files"}}
  ]

  dispatch = :cowboy_router.compile([{:_, routes}])

  opts = [port: 8000]
  env = [dispatch: dispatch]

  {:ok, _pid} = :cowboy.start_http(:http, 100, opts, [env: env])
end
```

目前的路由设置将会接受任何url。然后修改`dds_blog/handler.ex`中的处理函数，在Cowboy中，url写可以和字符串[绑定(binding)](http://ninenines.eu/docs/en/cowboy/1.0/manual/cowboy_req/index.html#bindings)。最开始我以为是[查询字符串(query strings)](http://ninenines.eu/docs/en/cowboy/1.0/manual/cowboy_req/index.html#qs_vals)，可是我错了。查询字符串的写法是这样的 `http://localhost:8000/?query=yes`。可是我们想要的写法是这样的`http://localhost:8000/some-file`。因此我们使用url[绑定](http://ninenines.eu/docs/en/cowboy/1.0/manual/cowboy_req/index.html#bindings).


打开`lib/dds_blog/handler.ex` ，通过请求的url，可以知道响应哪一个文件。

```elixir
def init({:tcp, :http}, req, opts) do
  {:ok, req, opts}
end

def handle(req, state) do
  {method, req} = :cowboy_req.method(req)
  {param, req} = :cowboy_req.binding(:filename, req)
  IO.inspect param
  {:ok, req} = get_file(method, param, req)
  {:ok, req, state}
end

def get_file("GET", :undefined, req) do
  headers = [{"content-type", "text/plain"}]
  body = "Ooops. Article not exists!"
  {:ok, resp} = :cowboy_req.reply(200, headers, body, req)
end

def get_file("GET", param, req) do
  headers = [{"content-type", "text/html"}]
  {:ok, file} = File.read "priv/contents/filename.md"
  body = file
  {:ok, resp} = :cowboy_req.reply(200, headers, body, req)
end
```

在`priv/contenst`文件夹中放一些markdown文件。如果文件夹不存在，就创建一个啦。首先使用`:cowboy.bindings`从url获取文件名，然后把其当成参数传入`get_file`函数中。`get_file`函数有两个类型签名，一个函数有参数，另外一个函数没有参数（使用`:undefined`作为参数）。如果无法从url获取文件名参数，则其值为 :undefined。正常用于响应文件找不到的请求，否则这回使用文件名参数调用`get_file`。

当接受参数之后，读取`priv/contents`文件夹的文件。此时，设置的文件是`filename.md`。我们将会看到如下的输出（纯markdown）

![Markdown raw](http://i.imgur.com/NHq0fHP.png)

响应的内容是markdown的原始内容。因此我们需要将markdown转会成html，我们需要一个库/包来帮我们转换。增加`earmark`包到`mix.exs`文件。（译者注：原文使用的库是 markdown，实际试验，最新版的markdown也无法编译，使用earmark代替了）


```elixir
    defp deps do
      [
        {:cowboy, "1.0.0"},
        { :earmark, "0.2.1" }
      ]
    end
```

获取依赖：

```elixir
$> mix deps.get
```

安装好依赖之后，可以使用`Earmark`模块的`to_html`函数把markdown转换成html。然后编辑`lib/dds_blog/handler.ex` 的`get_file` 函数。

```elixir
def get_file("GET", :undefined, req) do
  headers = [{"content-type", "text/plain"}]
  body = "Ooops. Article not exists!"
  {:ok, resp} = :cowboy_req.reply(200, headers, body, req)
end

def get_file("GET", param, req) do
  headers = [{"content-type", "text/html"}]
  {:ok, file} = File.read "priv/contents/filename.md"
  body = Earmark.to_html file
  {:ok, resp} = :cowboy_req.reply(200, headers, body, req)
end
```

重启`iex -S mix` ，刷新页面就能看到html格式化输出了。

![Imgur](http://i.imgur.com/hZwprG6.png)

非常酷吧。稍微休息一下，有点精疲力尽啦...

![meme](http://i.imgur.com/b6H4wKI.jpg)

回到工作中。我们需要针对`param`变量来载入 `priv/contents/`对于的文件，而不是之前写死的`filename`。使用字符串连接的特性即可：

```elixir
    def get_file("GET", :undefined, req) do
      headers = [{"content-type", "text/plain"}]
      body = "Ooops. Article not exists!"
      {:ok, resp} = :cowboy_req.reply(200, headers, body, req)
    end

    def get_file("GET", param, req) do
      headers = [{"content-type", "text/html"}]
      {:ok, file} = File.read "priv/contents/" <> param <> ".md"
      body = Earmark.to_html file
      {:ok, resp} = :cowboy_req.reply(200, headers, body, req)
    end
```

在`priv/contents/`文件夹内多增加几个markdown文件，然后重启服务。访问`http://localhost:8000/basicfp` 将返回`basicfp.md` 文件的内容。

![basicfp](http://i.imgur.com/K5gT0Y7.png)

很cool吧？

现在我们需要一个首页索引。索引可以看到所有文章内容的的文摘列表。为此我们需要迭代`priv/contents` 的文件，获取所有markdown内容并把文摘输出到索引页。

```elixir
    def get_file("GET", :undefined, req) do
      headers = [{"content-type", "text/html"}]
      file_lists = File.ls! "priv/contents/"
      content = print_articles file_lists, ""
      {:ok, resp} = :cowboy_req.reply(200, headers, content, req)
    end

    def get_file("GET", param, req) do
      headers = [{"content-type", "text/html"}]
      {:ok, file} = File.read "priv/contents/" <> param <> ".md"
      body = Earmark.to_html file
      {:ok, resp} = :cowboy_req.reply(200, headers, body, req)
    end


    def print_articles [h|t], index_contents do
      {:ok, article} = File.read "priv/contents/" <> h
      sliced = String.slice article, 0, 1000
      marked = Earmark.to_html sliced
      filename = String.slice(h, 0, String.length(h) - 3)
      more = "<a class='button' href='#{filename}'>More</a><hr />"
      print_articles t, index_contents <> marked <> more
    end

    def print_articles [], index_contents do
      index_contents
    end
```

路由如果没有任何匹配的参数，将会输出首页。我们将会使用函数式编程强大的递归将`priv/contents` 文件迭代。看看两个`print_articles` 函数。首先通过`File.ls` 获取一个文件列表，然后做完参数传入其中一个 `print_articles` 函数中。`print_articles`将会递归调用，知道所有文件返回的值为空。

重启服务。访问`http://localhost:8000/` ，将会看到如下结果：

![truncated](http://i.imgur.com/Y29KOT1.png)

点击  `More`，将会跳转到详细页面

> 如你所见，markdown文件并没有缩短，如果你知道如何使markdown缩短，请告诉我。

## 模板和主题

当你访问我们的页面的时候，无论首页还是详细页，都会立马返回markdwon转换的html页。我们需要小心的对待输出的markdwon。为此，更好的办法是使用模板，添加样式，让页面更加漂亮。

将会用到的样式有[Skeleton, a responsive css boilerplate](http://getskeleton.com/). 你也可以自由的使用别的css框架。下载css文件和图片（或者一些js文件），然后拷贝到静态文件夹目录`priv/static_files。


```elixir
priv/static_files/
├── css
│   ├── normalize.css
│   └── skeleton.css
└── images
    └── favicon.png
```

同时创建`priv/themes`文件夹，用于存放模板文件。新建 `index.html.eex`模板文件。注意其扩展名是`eex`，这是Elixir默认提供的`EEx` 模板引擎文件。

```elixir
priv/themes/
└── index.html.eex
```

编辑 `index.html.eex` 文件，其内容为： 

```elixir
<!DOCTYPE html>
<html lang="en">
<head>

  <!-- Basic Page Needs
  –––––––––––––––––––––––––––––––––––––––––––––––––– -->
  <meta charset="utf-8">
  <title><%= title %></title>
  <meta name="description" content="">
  <meta name="author" content="">

  <!-- Mobile Specific Metas
  –––––––––––––––––––––––––––––––––––––––––––––––––– -->
  <meta name="viewport" content="width=device-width, initial-scale=1">

  <!-- FONT
  –––––––––––––––––––––––––––––––––––––––––––––––––– -->
  <link href="//fonts.googleapis.com/css?family=Raleway:400,300,600" rel="stylesheet" type="text/css">

  <!-- CSS
  –––––––––––––––––––––––––––––––––––––––––––––––––– -->
  <link rel="stylesheet" href="/static/css/normalize.css">
  <link rel="stylesheet" href="/static/css/skeleton.css">

  <!-- Favicon
  –––––––––––––––––––––––––––––––––––––––––––––––––– -->
  <link rel="icon" type="image/png" href="static/images/favicon.png">

</head>
<body>

  <!-- Primary Page Layout
  –––––––––––––––––––––––––––––––––––––––––––––––––– -->
  <div class="container">
    <div class="row">
      <%= content %>
    </div>
  </div>

<!-- End Document
  –––––––––––––––––––––––––––––––––––––––––––––––––– -->
</body>
</html>
```

使用`<%= content %>`绑定模板标签，即将代码中的变量与模板的标签替换。编译`eex`为html并返回。我们还增加了`<%= title %>` 。

```elixir
def get_file("GET", :undefined, req) do
  headers = [{"content-type", "text/html"}]
  file_lists = File.ls! "priv/contents/"
  content = print_articles file_lists, ""
  title = "Welcome to DDS Blog"
  body = EEx.eval_file "priv/themes/index.html.eex", [content: content, title: title]
  {:ok, resp} = :cowboy_req.reply(200, headers, body, req)
end

def get_file("GET", param, req) do
  headers = [{"content-type", "text/html"}]
  {:ok, file} = File.read "priv/contents/" <> param <> ".md"
  content = Earmark.to_html file
  title = String.capitalize(param)
  body = EEx.eval_file "priv/themes/index.html.eex", [content: content, title: title]
  {:ok, resp} = :cowboy_req.reply(200, headers, body, req)
end
```

重启服务，刷新浏览器。

![Beauty](http://i.imgur.com/qlSIW6w.png)

现在点击more按钮，将会看到漂亮的详细页。我们将要完成啦，不过为了更好看，还需要增加header和foorter。

添加一些样式吧 `button button-primary` 和 `a` 

```elixir
def print_articles [h|t], index_contents do
  {:ok, article} = File.read "priv/contents/" <> h
  sliced = String.slice article, 0, 1000
  marked =Earmark.to_html sliced
  filename = String.slice(h, 0, String.length(h) - 3)
  more = "<a class='button button-primary' href='#{filename}'>More</a><hr />"
  print_articles t, index_contents <> marked <> more
end
```

还有一点需要注意。可以把html字符串用eex模板代替

```elixir
def print_articles [h|t], index_contents do
  {:ok, article} = File.read "priv/contents/" <> h
  sliced = String.slice article, 0, 1000
  marked = Earmark.to_html sliced
  filename = String.slice(h, 0, String.length(h) - 3)
  more = EEx.eval_file "priv/themes/more_button.html.eex", [filename: filename]
  print_articles t, index_contents <> marked <> more
end
```

创建`priv/themes/more_button.html.eex` 模板，

```elixir
<a class='button button-primary' href='<%= filename %>'>More</a><hr />
```

刷新浏览器

## 总结

我们使用`Cowboy` and `earmark`两个包将文件组合起来，创建了一个blog程序。还有什么事比这个更cool么？我懂，有一些代码可能有点业余。不过我们完成了我们的任务，这才是最重要的。对吧？以后随时可以优化这些不足之处。

[完整代码](https://github.com/rizafahmi/dds-blog). 你可以给我提issues或者提 pull request。再会啦~
## 参考

* http://learnyousomeerlang.com/what-is-otp
* https://github.com/ninenines/cowboy
* http://elixir-lang.org/docs/stable/elixir/
