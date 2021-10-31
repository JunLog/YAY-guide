# gin到底是怎么启动的？

## 代码结构

```xml
|-- binding                     将请求的数据对象化并校验
|-- examples                    各种列子
|-- json                        提供了另外一种json实现
|-- render                      响应

|-- gin.go                      gin引擎所在
|-- gin_test.go
|-- routes_test.go
|-- context.go                  上下文，将各种功能聚焦到上下文（装饰器模式）
|-- context_test.go
|-- response_writer.go          响应的数据输出
|-- response_writer_test.go
|-- errors.go                   错误处理
|-- errors_test.go
|-- tree.go                     路由的具体实现
|-- tree_test.go
|-- routergroup.go
|-- routergroup_test.go
|-- auth.go                     一个基本的HTTP鉴权的中间件
|-- auth_test.go
|-- logger.go                   一个日志中间件
|-- logger_test.go
|-- recovery.go                 一个崩溃处理插件
|-- recovery_test.go

|-- mode.go                     应用模式
|-- mode_test.go
|-- utils.go                    杂七杂八
|-- utils_test.go
```

## 参考文档

> gin中文文档：https://gin-gonic.com/zh-cn/docs/

> gin框架分析文章参考：
>
> https://segmentfault.com/a/1190000022414015
>
> https://zhuanlan.zhihu.com/p/372097558
>
> https://github.com/hhstore/blog/issues/131
>
> https://segmentfault.com/a/1190000022466684

> 思维导图：https://www.processon.com/view/link/5f4b70c2079129356ec5cb70#map

> 七天实现GEE ：https://geektutu.com/post/gee.html

## 前言

> 因为自己今年是大三的，可能阅读gin框架更多的目的是了解整体框架是怎么样启动，了解我使用的函数都是怎么样获取数据等等。更多的基于在一个web项目中的那些函数的原理，也扩展一些没有使用过的函数。更好的去应对面试！！！希望能拿到offer！！！🐱‍🏍🐱‍🏍🐱‍🏍

## gin到底是怎么启动的？

```go
package main

import "github.com/gin-gonic/gin"

func main() {
	r := gin.Default()
	r.GET("/ping", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "pong",
		})
	})
	r.Run() // 监听并在 0.0.0.0:8080 上启动服务
}
```

这是官方给的最简单的示例！！！

### gin.Default()

我们进入`Default()` 函数看看，源码如下：

```go
// Default returns an Engine instance with the Logger and Recovery middleware already attached.
//Deafault会返回一个Engine的实例，其中包含了logger和recovery的中间件
func Default() *Engine {
	debugPrintWARNINGDefault()//进行debug，告知一些信息。
	engine := New()//// New returns 一个新的空白的engine，没有任何中间件
	engine.Use(Logger(), Recovery())//这里会给engine带上Logger和recovery的handlerfunc
	return engine
}
```

这个函数作用就是会构造一个默认的engine实例 ，其中会带着Logger和recovery的中间件。

我们发现一个空白的engine实例是通过`New()`所构造的。

### New()

```go
//New returns 一个新的空白的engine，没有任何中间件
func New() *Engine {
	debugPrintWARNINGNew()//在控制台打印相关的信息。可以忽略
	engine := &Engine{//engine实例的默认配置情况
		RouterGroup: RouterGroup{
			Handlers: nil,
			basePath: "/",
			root:     true,
		},
		FuncMap:                template.FuncMap{},
		RedirectTrailingSlash:  true,
		RedirectFixedPath:      false,
		HandleMethodNotAllowed: false,
		ForwardedByClientIP:    true,
		RemoteIPHeaders:        []string{"X-Forwarded-For", "X-Real-IP"},
		TrustedProxies:         []string{"0.0.0.0/0"},
		AppEngine:              defaultAppEngine,
		UseRawPath:             false,
		RemoveExtraSlash:       false,
		UnescapePathValues:     true,
		MaxMultipartMemory:     defaultMultipartMemory,
		trees:                  make(methodTrees, 0, 9),
		delims:                 render.Delims{Left: "{{", Right: "}}"},
		secureJSONPrefix:       "while(1);",
	}
    //这里好像和router有关，看不懂，先放着
	engine.RouterGroup.engine = engine
    //根据名字来看是创建了一个池子，看不懂先放着
	engine.pool.New = func() interface{} {
		return engine.allocateContext()
	}
	return engine
}
```

原来这就是空白的engine实例默认的一个配置情况

#### engine

看看engine的结构是怎么样的

```go
type Engine struct {
    //路由组
	RouterGroup

// 如果当前路由无法匹配，则启用自动重定向
// 存在（不存在）尾部斜杠的路径处理程序。
// 例如，如果 /foo/ 被请求，但路由只存在于 /foo，则
// 客户端被重定向到 /foo 带有 HTTP 状态代码 301 的 GET 请求
// 和 307 用于所有其他请求方法。
	RedirectTrailingSlash bool

// 如果启用，路由器会尝试修复当前的请求路径，如果没有
// 为它注册了句柄。
// 第一个多余的路径元素，如 ../ 或 // 被删除。
// 之后路由器对清理过的路径进行不区分大小写的查找。
// 如果可以找到此路由的句柄，则路由器进行重定向
// 使用状态代码 301 的 GET 请求和 307 的更正路径
// 所有其他请求方法。
// 例如 /FOO 和 /..//Foo 可以重定向到 /foo。
// RedirectTrailingSlash 与此选项无关。
	RedirectFixedPath bool

    //上面两个字段，我觉得主要作用就是路由匹配不成功的一些处理，提高容错率。
    
    
// 如果启用，路由器检查是否允许其他方法用于
// 当前路由，如果当前请求无法路由。
// 如果是这种情况，则使用“不允许的方法”来回答请求
// 和 HTTP 状态码 405。
// 如果不允许其他方法，则将请求委托给 NotFound
// 处理程序。
	HandleMethodNotAllowed bool
    
    //检查当前路由是否能够请求成功，

// 如果启用，将从请求的标头中解析客户端 IP
// 匹配存储在 `(*gin.Engine).RemoteIPHeaders` 中的那些。 如果没有获取到IP 它回退到从中获取的IP
// `(*gin.Context).Request.RemoteAddr`。
	ForwardedByClientIP bool

// 用于获取客户端 IP 时的标头列表
// `(*gin.Engine).ForwardedByClientIP` 是 `true` 并且
// `(*gin.Context).Request.RemoteAddr` 至少与 `(*gin.Engine).TrustedProxies` 的网络来源之一匹配。
	RemoteIPHeaders []string

	// 当 `(*gin.Engine).ForwardedByClientIP` 为 `true` 时，信任包含备用客户端 IP 的请求标头的网络源列表（IPv4 地址、IPv4 CIDR、IPv6 地址或 IPv6 CIDR）。
	TrustedProxies []string

	//如果启用，它将信任一些以“X-AppEngine ...”开头的标头，以便更好地与该 PaaS 集成。
	AppEngine bool

	//如果启用，将使用 url.RawPath 查找参数。
	UseRawPath bool

	// 如果为 true，路径值将不被转义。
// 如果 UseRawPath 为 false（默认情况下），则 UnescapePathValues 实际上为 true，
// 作为 url.Path 将被使用，它已经是未转义的。
	UnescapePathValues bool

	// 提供给 http.Request 的 ParseMultipartForm 方法调用的“maxMemory”参数的值。
	MaxMultipartMemory int64

	//RemoveExtraSlash 即使带有额外的斜杠，也可以从 URL 解析参数。
	RemoveExtraSlash bool

    //上面这些标记，就是近几年的gin的发展，有兴趣的话，可以继续了解，但是我们的目的不再这里。
    
	delims           render.Delims
	secureJSONPrefix string
	HTMLRender       render.HTMLRender
	FuncMap          template.FuncMap
	allNoRoute       HandlersChain
	allNoMethod      HandlersChain
	noRoute          HandlersChain
	noMethod         HandlersChain
	pool             sync.Pool
	trees            methodTrees
	maxParams        uint16
	trustedCIDRs     []*net.IPNet
}
```

我们了解到了engine大致的结构如此，后面我们一个一个讲解重要的engine的字段。

> 现在我们已经知道了，Default函数的作用是返回engine实例，你后续的操作都是在这个实例上进行的。

## GET()

我们来到了`GET()`函数，可以大致的了解他到底是怎么样注册路由的。

进入他的函数里面发现

```go
//GET 是 router.Handle("GET", path, handle) 的快捷方式。
func (group *RouterGroup) GET(relativePath string, handlers ...HandlerFunc) IRoutes {
	return group.handle(http.MethodGet, relativePath, handlers)
}
```

他的这个方法的承载着是`*RouterGroup` 

既然我们利用**返回的engine实例**可以调用此方法说明，`engine`应该有`RouterGroup`这个字段。我们返回去看，确实有这个字段

![image-20210920212757079](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20210920212804.png)

`http.MethodGet`是`gin`对请求方式的一个封装的情况

```go
// Unless otherwise noted, these are defined in RFC 7231 section 4.3.
const (
	MethodGet     = "GET"
	MethodHead    = "HEAD"
	MethodPost    = "POST"
	MethodPut     = "PUT"
	MethodPatch   = "PATCH" // RFC 5789
	MethodDelete  = "DELETE"
	MethodConnect = "CONNECT"
	MethodOptions = "OPTIONS"
	MethodTrace   = "TRACE"
)
```

这是常用的解耦方式，不能随便出现字符串，都以变量去使用

我们看到一段注释`GET 是 router.Handle("GET", path, handle) 的快捷方式。`

其实我们demo注册的GET路由**等价于**

```go
r.Handle("GET", "/hello",  func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "pong",
		})
	})
```

那么重点就不在此处了，应该在`Handle` 

 Handle实现函数如下

```go
func (group *RouterGroup) Handle(httpMethod, relativePath string, handlers ...HandlerFunc) IRoutes {
	if matches, err := regexp.MatchString("^[A-Z]+$", httpMethod); !matches || err != nil {
        //对请求的方式的匹配情况，只能匹配大写字母的字符串
		panic("http method " + httpMethod + " is not valid")
	}
	return group.handle(httpMethod, relativePath, handlers)
}
```

Handle函数上的一段注释告诉你，其实你可以自定义请求方式的。但请求的方式是一个**只包含大写字母的字符串**。

发现重点依旧不在此处，接着发现到无论是`r.GET`还是`r.Handle`都返回`group.handle` 

我们来看看`group.handle`函数如下

```go
func (group *RouterGroup) handle(httpMethod, relativePath string, handlers HandlersChain) IRoutes {
    //将当前路由组的base path与现在注册的路径结合
	absolutePath := group.calculateAbsolutePath(relativePath)//==base+relative
    //将路由组handlerFunc加到注册函数handlerFunc之前
	handlers = group.combineHandlers(handlers)
    //注册路由
	group.engine.addRoute(httpMethod, absolutePath, handlers)
    //返回路由的所有信息
	return group.returnObj()
}
```

我们接下来简单看看gin是如何注册路由的？

```go
func (engine *Engine) addRoute(method, path string, handlers HandlersChain) {
	assert1(path[0] == '/', "path must begin with '/'")
	assert1(method != "", "HTTP method can not be empty")
	assert1(len(handlers) > 0, "there must be at least one handler")
	//对三个参数的判断的处理
	debugPrintRoute(method, path, handlers)
	//打印相关信息
    
    //根据请求方式去构建了一个method树 ，都放在一样的请求方式的路由
	root := engine.trees.get(method)
    //如果没有，那么建立一个新的节点
	if root == nil {
		root = new(node)
		root.fullPath = "/"
		engine.trees = append(engine.trees, methodTree{method: method, root: root})
	}
    //注册路由，添加节点
	root.addRoute(path, handlers)

	// Update maxParams
	if paramsCount := countParams(path); paramsCount > engine.maxParams {
		engine.maxParams = paramsCount
	}
}
```

我们在10处与18处打个断点看看，怎么去增添的路由

![image-20210921153735148](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20210921153735.png)

他的handlers是三个，看具体的名字发现是Logger和recovery中间件和自定义的handlerfunc

验证了`handlers = group.combineHandlers(handlers)` 我们的猜想

当我们把if语句执行完后，看到engine他的tree建立一个空节点

![image-20210921153944032](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20210921153944.png)

当我们进入`root.addRoute` 函数里面

他建立一个节点n，里面放着路径和handlerFunc

![image-20210921154235487](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20210921154235.png)

我们再回到之前的engine看到

![image-20210921154321537](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20210921154321.png)

我们注册的/ping作为GET路由的根节点了

同时此时的root也发生了改变

![image-20210921154505390](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20210921154505.png)

整体的过程就是

* 当我注册一个路由时候，我会先查看路由树是否由这个请求方式
* 如果没有，则建立一个节点，然后添加我注册路由的路径和中间件及函数
* 当GET函数只有一个时候，他作为GET树的根节点

>  我们现在了解到，当你注册路由时候，他会根据你的路由请求方式去建立一个树，当只有一个路由时候，让他作为这个树的根节点。我们后面会详细讲解gin的路由树



## Run()

最后一步了，这是他的启动函数。

```go
// Run 将路由器附加到 http.Server 并开始侦听和服务 HTTP 请求。
// 它是 http.ListenAndServe(addr, router) 的快捷方式
// 注意：除非发生错误，否则此方法将无限期地阻塞调用 goroutine。
func (engine *Engine) Run(addr ...string) (err error) {
	defer func() { debugPrintError(err) }()

	trustedCIDRs, err := engine.prepareTrustedCIDRs()//获取ip地址
	if err != nil {
		return err
	}
	engine.trustedCIDRs = trustedCIDRs
    address := resolveAddress(addr)//获取端口号，默认端口号是:8080
	debugPrint("Listening and serving HTTP on %s\n", address)
	err = http.ListenAndServe(address, engine)//建立web服务器
	return
}
```

其实你发现除去一些打印代码以及获取相关信息代码后，和你之前用原生的http包去建立的web服务器是一样的。

```go
package main

import (
    "log"
    "net/http"
)

func main() {
    http.HandleFunc("/", indexHandler)

    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

只不过他把run进行了封装，最重要的还是最后一段代码

```go
err = http.ListenAndServe(address, engine)//建立web服务器
```

咦？😆😆😆突然发现和原生好像还是有点不同的。

> 原生的-> 他建立请求后面一个参数写的是`nil`

> gin的Run-> 他建立请求后面却写了engine，之前我们说的engine实例。

**为什么需要写这个呢？🙄🙄🙄**

我们进入`http.ListenAndServe`看下

```go
// ListenAndServe 监听 TCP 网络地址 addr 然后调用
// 与处理程序一起服务以处理传入连接的请求。
// 接受的连接被配置为启用 TCP 保持连接。
//
// 处理程序通常为 nil，在这种情况下使用 DefaultServeMux。
//
// ListenAndServe 总是返回一个非零错误。
func ListenAndServe(addr string, handler Handler) error {
	server := &Server{Addr: addr, Handler: handler}
	return server.ListenAndServe()
}
```

其实这里没有特别的地方，关键点应该在`Handler`

根据注释，我们猜测所有的请求经过Handler，然后处理请求。

我们进入Handler看看，他是怎么定义的

```go
// 处理程序响应 HTTP 请求。
//
// ServeHTTP 应该将回复标头和数据写入 ResponseWriter
// 然后返回。返回请求完成的信号；它
// 使用 ResponseWriter 或从
// Request.Body 在完成之后或同时完成
// 服务 HTTP 调用。
//
// 取决于 HTTP 客户端软件、HTTP 协议版本，以及
// 客户端和 Go 服务器之间的任何中介，它可能不会
// 可以在写入后从 Request.Body 中读取
// 响应写入器。谨慎的处理程序应该阅读 Request.Body
// 首先，然后回复。
//
// 除了读取主体外，处理程序不应修改
// 提供的请求。
//
// 如果 ServeHTTP 崩溃，服务器（ServeHTTP 的调用者）假设
// 恐慌的影响与活动请求隔离。
// 它恢复恐慌，将堆栈跟踪记录到服务器错误日志中，
// 并关闭网络连接或发送 HTTP/2
// RST_STREAM，取决于 HTTP 协议。中止处理程序
// 客户端看到一个中断的响应，但服务器没有记录
// 一个错误，带有值 ErrAbortHandler 的恐慌。 
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```

根据注释，看来Handler是一个接口，**只要传入任何实现了ServeHTTP接口的实例，所有HTTP的请求，就都交给了该实例去处理。**

engine实现的ServeHTTP接口

```go
// ServeHTTP conforms to the http.Handler interface.
func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    //从池子中建立一个空白的Context对象
	c := engine.pool.Get().(*Context)
	c.writermem.reset(w)
	c.Request = req
	c.reset()

	engine.handleHTTPRequest(c)
	//释放刚才建立的c
	engine.pool.Put(c)
}
```

我们打个断点，发起个请求，看看这里是怎么处理HTTP请求的。

先建立一个空白的Context对象

他会将`http.Request`和`http.ResponseWrite`都会写入到c里面。

![image-20210921162439830](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20210921162439.png)

然后交到了gin具体的处理HTTP函数里面了。

经过处理后c的信息发生了改变

![image-20210921162806155](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20210921162806.png)

估计已经将handlerfunc函数的返回信息给到了请求响应报文里面了。

然后会释放这个c，之后http包会返回执行完的结果。也就是`ResponsWriter`里面的内容

## 总结

根据官方给的demo，我们了解到他的基本运行情况。

* `gin.Default()` 建立默认的engine实例（带有logger，recovery的中间件）
* `r.GET`注册路由，gin会建立一个路由树出来，方便查找。
* `r.Run() ` 建立web服务器，监听HTTP请求，由于engine实现了ServeHTTP接口，所以所有的请求都会交到engine去处理

>  这样看来engine是gin框架的核心,同时Context对象是gin框架的重心所在。
