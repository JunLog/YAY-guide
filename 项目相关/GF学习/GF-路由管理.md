# GF-路由管理

## 路由注册

```go
package main
import (
    "github.com/gogf/gf/frame/g"
    "github.com/gogf/gf/net/ghttp"
)
func main() {
    s := g.Server()
    s.BindHandler("/", func(r *ghttp.Request) {
        r.Response.Write("哈喽世界！")
    })
    s.Run()
}
```

使用`BindHandler`方法进行路由注册的方式叫做`回调函数注册` 。如果你用过`Gin` 框架的话，方式都是差不多的。

### 函数注册

> 这种方式的路由注册不限制给定的回调函数是一个对象方法还是包方法，它仅仅需要一个函数的内存地址指针即可，使用比较灵活。

涉及到的函数仅仅有一个

```go
func (s *Server) BindHandler(pattern string, handler HandlerFunc) error
```

例1：包方法注册

```go
package main
import (
    "github.com/gogf/gf/container/gtype"
    "github.com/gogf/gf/frame/g"
    "github.com/gogf/gf/net/ghttp"
)
var (
    total = gtype.NewInt()
)
func Total(r *ghttp.Request) {
    r.Response.Write("total:", total.Add(1))
}
func main() {
    s := g.Server()
    s.BindHandler("/total", Total)
    s.SetPort(8199)
    s.Run()
}
```

此方法注册与gin框架注册路由方式一致。

`total`变量使用了`gtype.Int`并发安全类型。

例2：对象方法注册

```go
package main
import (
    "github.com/gogf/gf/container/gtype"
    "github.com/gogf/gf/frame/g"
    "github.com/gogf/gf/net/ghttp"
)
type Controller struct {
    total *gtype.Int
}
func (c *Controller) Total(r *ghttp.Request) {
    r.Response.Write("total:", c.total.Add(1))
}
func main() {
    s := g.Server()
    c := &Controller{
        total: gtype.NewInt(),
    }
    s.BindHandler("/total", c.Total)
    s.SetPort(8199)
    s.Run()
}
```

以上两种方式注册路由，与gin框架一致，都是利用`BindHandler` 函数完成，一个参数写路径，一个参数填写回调函数。

### 对象注册

> 使用同一个实例化的`struct`对象进行路由注册，多个请求都将会由该对象进行管理，请求与请求之间也可能会通过该对象的属性共享变量。大多数的`go web`库也仅提供这种方式，大部分场景下也推荐使用这种方式进行注册。

涉及到的函数有三个

```go
func (s *Server) BindObject(pattern string, obj interface{}, methods ...string) error
func (s *Server) BindObjectMethod(pattern string, obj interface{}, method string) error
func (s *Server) BindObjectRest(pattern string, obj interface{}) error
```

例子：

```go
package api

import "github.com/gogf/gf/net/ghttp"

type Controller struct {}

func (c *Controller) Index(r *ghttp.Request) {
	r.Response.Write("index")
}
//router
s.BindObject("/controller",new(api.Controller))
```

#### 默认路由方法

制器中的`Index`方法是一个特殊的方法，例如，当注册的路由规则为`/user`时，HTTP请求到`/user`时，将会自动映射到控制器的`Index`方法。也就是说，访问地址`/user`和`/user/index`将会达到相同的执行效果。如果你没有写Index路由的话，那么就不会有两个路由的注册。

#### 路由内置变量

当使用`BindObject`方法进行对象注册时，在路由规则中可以使用两个内置的变量：`{.struct}`和`{.method}`，前者表示当前**对象名称**，后者表示当前注册的**方法名**。

```go
package main
import (
    "github.com/gogf/gf/frame/g"
    "github.com/gogf/gf/net/ghttp"
)
type Order struct{}
func (o *Order) List(r *ghttp.Request) {
    r.Response.Write("list")
}
func main() {
    s := g.Server()
    o := new(Order)
    s.BindObject("/{.struct}-{.method}", o)
    s.SetPort(8199)
    s.Run()
}
```

```shell
  SERVER  | DOMAIN  | ADDRESS | METHOD |    ROUTE    |      HANDLER       | MIDDLEWARE
|---------|---------|---------|--------|-------------|--------------------|------------|
  default | default | :8199   | ALL    | /order-list | main.(*Order).List |
|---------|---------|---------|--------|-------------|--------------------|------------|
```

#### 命名风格

我们可以通过`g.Server().SetNameToUriType`方法来设置对象方法名称的路由生成方式。支持的方式目前有`4`种，对应`4`个常量定义

```go
URI_TYPE_DEFAULT  = 0 // （默认）全部转为小写，单词以'-'连接符号连接
URI_TYPE_FULLNAME = 1 // 不处理名称，以原有名称构建成URI
URI_TYPE_ALLLOWER = 2 // 仅转为小写，单词间不使用连接符号
URI_TYPE_CAMEL    = 3 // 采用驼峰命名方式
```

#### 可选方法注册

我们可以通过`BindObject`传递**第三个非必需参数**替换实现，参数支持传入**多个**方法名称，多个名称以英文`,`号分隔（**方法名称参数区分大小写**）。

#### 绑定路由方法

我们可以通过`BindObjectMethod`方法绑定指定的路由到指定的方法执行（**方法名称参数区分大小写**）。

简单讲就是单独路径去注册路由对象的某一个方法。

最大的用处就是给对象注册动态路由。

```go
 s.BindObjectMethod("/show", c, "Show")
```

#### `RESTful`对象注册

* `POST`方式将会映射到控制器的`Post`方法中(公开方法，首字母大写)，`DELETE`方式将会映射到控制器的`Delete`方法中，以此类推。
* **其他非HTTP Method命名的方法，即使是定义的包公开方法，将不会自动注册，对于应用端不可见**。
* 当然，如果控制器并未定义对应HTTP Method的方法，该Method请求下将会返回 `HTTP Status 404`。

```go
package main
import (
    "github.com/gogf/gf/frame/g"
    "github.com/gogf/gf/net/ghttp"
)
type Controller struct{}
// RESTFul - GET
func (c *Controller) Get(r *ghttp.Request) {
    r.Response.Write("GET")
}
// RESTFul - POST
func (c *Controller) Post(r *ghttp.Request) {
    r.Response.Write("POST")
}
// RESTFul - DELETE
func (c *Controller) Delete(r *ghttp.Request) {
    r.Response.Write("DELETE")
}
// 该方法无法映射，将会无法访问到
func (c *Controller) Hello(r *ghttp.Request) {
    r.Response.Write("Hello")
}
func main() {
    s := g.Server()
    c := new(Controller)
    s.BindObjectRest("/object", c)//安装Restful风格注册方法
    s.SetPort(8199)
    s.Run()
}
```

路由表

```shell
  SERVER  | DOMAIN  | ADDRESS | METHOD |  ROUTE  |          HANDLER          | MIDDLEWARE
|---------|---------|---------|--------|---------|---------------------------|------------|
  default | default | :8199   | DELETE | /object | main.(*Controller).Delete |
|---------|---------|---------|--------|---------|---------------------------|------------|
  default | default | :8199   | GET    | /object | main.(*Controller).Get    |
|---------|---------|---------|--------|---------|---------------------------|------------|
  default | default | :8199   | POST   | /object | main.(*Controller).Post   |
|---------|---------|---------|--------|---------|---------------------------|------------|
```

#### 构造方法与析构方法

这两个方法类似于对象的构造函数和析构函数。

`Init`

```go
// “构造函数”对象方法 
func (c Controller) Init(r ghttp.Request) {

}
```

`Shut`

```go
// “析构函数”对象方法 
func (c Controller) Shut(r ghttp.Request) {

}
```

### 分组路由

常用的注册路由的函数

```go
// 创建分组路由
func (s *Server) Group(prefix string, groups ...func(g *RouterGroup)) *RouterGroup
func (d *Domain) Group(prefix string, groups ...func(g *RouterGroup)) *RouterGroup
// 注册Method路由
func (g *RouterGroup) ALL(pattern string, object interface{}, params...interface{})
func (g *RouterGroup) GET(pattern string, object interface{}, params...interface{})
func (g *RouterGroup) PUT(pattern string, object interface{}, params...interface{})
func (g *RouterGroup) POST(pattern string, object interface{}, params...interface{})
func (g *RouterGroup) DELETE(pattern string, object interface{}, params...interface{})
func (g *RouterGroup) PATCH(pattern string, object interface{}, params...interface{})
func (g *RouterGroup) HEAD(pattern string, object interface{}, params...interface{})
func (g *RouterGroup) CONNECT(pattern string, object interface{}, params...interface{})
func (g *RouterGroup) OPTIONS(pattern string, object interface{}, params...interface{})
func (g *RouterGroup) TRACE(pattern string, object interface{}, params...interface{})
// 中间件绑定
func (g *RouterGroup) Middleware(handlers ...HandlerFunc) *RouterGroup
// REST路由
func (g *RouterGroup) REST(pattern string, object interface{})
// 批量注册
func (g *RouterGroup) ALLMap(m map[string]interface{})
```

1. `Group`方法用户创建一个分组路由对象，并且支持在指定域名对象上创建。
2. 以`HTTP Method`命名的方法用以绑定指定的`HTTP Method`路由；其中`ALL`方法用于注册所有的`HTTP Method`到指定的函数/对象/控制器上；`REST`方法用户注册`RESTful`风格的路由，需给定一个执行对象或者控制器对象。（具体使用与对象注册`RESTFUL`路由一致）
3. `Middleware`方法用于绑定一个或多个中间件到当前分组的路由上，具体详见中间件章节。
4. `Bind`方法用于批量路由注册，每一个路由注册项为`Slice`类型的参数，且参数数量应该`>=3`个，具体使用请见后续示例。

#### 层级路由注册。

例子：

```go
package main
import (
    "net/http"
    "github.com/gogf/gf/frame/g"
    "github.com/gogf/gf/net/ghttp"
)
func main() {
    s := g.Server()
    s.Use(MiddlewareLog)
    s.Group("/api.v2", func(group *ghttp.RouterGroup) {
        group.GET("/test", func(r *ghttp.Request) {
            r.Response.Write("test")
        })
        group.Group("/order", func(group *ghttp.RouterGroup) {
            group.GET("/list", func(r *ghttp.Request) {
                r.Response.Write("list")
            })
            group.PUT("/update", func(r *ghttp.Request) {
                r.Response.Write("update")
            })
        })
        group.Group("/user", func(group *ghttp.RouterGroup) {
            group.GET("/info", func(r *ghttp.Request) {
                r.Response.Write("info")
            })
            group.POST("/edit", func(r *ghttp.Request) {
                r.Response.Write("edit")
            })
            group.DELETE("/drop", func(r *ghttp.Request) {
                r.Response.Write("drop")
            })
        })
    })
    s.SetPort(8199)
    s.Run()
}
```

使用层级方式注册路由，让整个路由管理更加方便，尤其是你项目都是使用的是函数注册方式。

至于批量注册，可以作为了解，挺有意思的注册方式。

**批量注册只能通过`group *ghttp.RouterGroup`完成注册，如果使用`*RouterGroup`反而可能会注册不成功**

```go
s := g.Server()
// 前台系统自定义错误页面
s.BindStatusHandler(401, func(r *ghttp.Request) {
    if !gstr.HasPrefix(r.URL.Path, "/admin") {
        service.View.Render401(r)
    }
})
s.BindStatusHandler(403, func(r *ghttp.Request) {
    if !gstr.HasPrefix(r.URL.Path, "/admin") {
        service.View.Render403(r)
    }
})
s.BindStatusHandler(404, func(r *ghttp.Request) {
    if !gstr.HasPrefix(r.URL.Path, "/admin") {
        service.View.Render404(r)
    }
})
s.BindStatusHandler(500, func(r *ghttp.Request) {
    if !gstr.HasPrefix(r.URL.Path, "/admin") {
        service.View.Render500(r)
    }
})
// 前台系统路由注册
s.Group("/", func(group *ghttp.RouterGroup) {
    group.Middleware(service.Middleware.Ctx)
    group.ALLMap(g.Map{
        "/":            api.Index,          // 首页
        "/login":       api.Login,          // 登录
        "/register":    api.Register,       // 注册
        "/category":    api.Category,       // 栏目
        "/topic":       api.Topic,          // 主题
        "/topic/:id":   api.Topic.Detail,   // 主题 - 详情
        "/ask":         api.Ask,            // 问答
        "/ask/:id":     api.Ask.Detail,     // 问答 - 详情
        "/article":     api.Article,        // 文章
        "/article/:id": api.Article.Detail, // 文章 - 详情
        "/reply":       api.Reply,          // 回复
        "/search":      api.Search,         // 搜索
        "/captcha":     api.Captcha,        // 验证码
        "/user/:id":    api.User.Index,     // 用户 - 主页
    })
    // 权限控制路由
    group.Group("/", func(group *ghttp.RouterGroup) {
        group.Middleware(service.Middleware.Auth)
        group.ALLMap(g.Map{
            "/user":     api.User,     // 用户
            "/content":  api.Content,  // 内容
            "/interact": api.Interact, // 交互
            "/file":     api.File,     // 文件
        })
    })
})
```

### 通过域名注册路由

需要获取`Domain`对象：

```go
func (s *Server) Domain(domains string) *Domain
```

然后剩余的操作与上诉的注册方式一致。`Server`的路由注册对应方法一致，只不过在`Domain`对象的底层会自动将方法绑定到指定的域名列表中，只有**对应的域名**才能提供访问。

## 中间件

### 中间件类型

前置中间件

```go
func Middleware(r *ghttp.Request) {
    // 中间件处理逻辑
    r.Middleware.Next()
}
```

后置中间件

```go
func Middleware(r *ghttp.Request) {
    r.Middleware.Next()
    // 中间件处理逻辑
}
```

### 中间件注册

全局注册

```go
func (s *Server) BindMiddlewareDefault(handlers ...HandlerFunc)
// BindMiddlewareDefault 别名
func (s *Server) Use(handlers ...HandlerFunc)
//指定路径进行注册中间件
func (s *Server) BindMiddleware(pattern string, handlers ...HandlerFunc)
```

分组路由注册

```go
func (g *RouterGroup) Middleware(handlers ...HandlerFunc) *RouterGroup
```

执行优先级

简单概括：路由深度越深，会执行离得近的中间件，同一路由（组）下，按照中间件的注册优先顺序执行

## 路由规则

```go
func (s *Server) BindHandler(pattern string, handler HandlerFunc) error
```

以函数注册举例，patter的参数的规则是：`[HTTPMethod:]路由规则[@域名]`

* `HTTPMethod` 的参数：`GET,PUT,POST,DELETE,PATCH,HEAD,CONNECT,OPTIONS,TRACE` 如果不写，默认`ALL`
* `@域名`为非必要参数
* 中间填路由路径

### 动态路由

获取参数的对应值方式

```go
func (r *Request) GetRouterValue(key string, def ...interface{}) interface{}
func (r *Request) GetRouterVar(key string, def ...interface{}) *gvar.Var
func (r *Request) GetRouterString(key string, def ...interface{}) string
//ghttp.Request.Get*
```

* 命名匹配规则

> `/user/:user`

* 模糊匹配规则

> `/src/*path`

* 字段匹配规则

> `/order/list/{page}.php`

### 优先级控制

优先级控制按照深度优先策略，最主要的几点因素：

1. **层级越深的规则优先级越高**；
2. **同一层级下，精准匹配优先级高于模糊匹配**；
3. **同一层级下，模糊匹配优先级：字段匹配 > 命名匹配 > 模糊匹配**；

我们来看示例（左边的规则优先级比右边高）：

```
/:name                   >            /*any
/user/name               >            /user/:action
/:name/info              >            /:name/:action
/:name/:action           >            /:name/*action
/:name/{action}          >            /:name/:action
/src/path/del            >            /src/path
/src/path/del            >            /src/path/:action
/src/path/*any           >            /src/path
```

