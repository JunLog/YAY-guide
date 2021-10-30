# 请求输入与输出

请求的输入与输出都是比较常见的。怎么样使输入与输出变得更加优雅？这是一个问题。

GF基本上已经处理好，这些事情！！！让这些杂乱的事情变得更加优雅！！

## 请求输入

GF 框架获取参数的方法非常丰富。有好多好多！！！🙄

1. `Get*`: 常用方法，简化参数获取，`GetRequest*`的别名。
2. `GetQuery*`: 获取`GET`方式传递过来的参数，包括`Query String`及`Body`参数解析。
3. `GetForm*`: 获取表单方式传递过来的参数，表单方式提交的参数`Content-Type`往往为`application/x-www-form-urlencoded`, `application/form-data`, `multipart/form-data`, `multipart/mixed`等等。
4. `GetRequest*`: 获取客户端提交的参数，不区分提交方式。
5. `Get*Struct`: 将指定类型的请求参数绑定到指定的`struct`对象上，注意给定的参数为对象指针。绝大部分场景中往往使用`Parse`方法将请求数据转换为请求对象，具体详见后续章节。
6. `GetBody/GetBodyString`: 获取客户端提交的原始数据，该数据是客户端写入到`body`中的原始数据，与`HTTP Method`无关，例如客户端提交`JSON/XML`数据格式时可以通过该方法获取原始的提交数据。
7. `GetJson`: 自动将原始请求信息解析为`gjson.Json`对象指针返回，`gjson.Json`对象具体在 [gjson (数据动态编解码)](https://www.bookstack.cn/read/goframe-1.16-zh/9370c51aaadff3fb.md) 章节中介绍。
8. `Exit*`: 用于请求流程退出控制，

**日常写项目，基本上很多都用不到，这些是给确定的参数输入，而懒人使用的则不是这些！！**

基本上请求输入，都会涉及到对象的转换，而不是一个一个参数的获取。

>  GF框架推荐使用`Parse`方法来实现`struct`的转换，该方法是一个便捷方法，内部会自动进行转换及数据校验

```go
func (c *Controller) Get(r *ghttp.Request) {
	var user *model.U
	if err:=r.Parse(&user);err!=nil{
		fmt.Println(err.Error())
		response.JsonExit(r, 400, gerror.Current(err).Error())
	}
	response.JsonExit(r,200,"成功",g.Map{
		"data":user.Id,
	})
	r.Response.Write("GET")
}
```

**`Parse`方法能够满足`query`,`formdata`,`body`,`json`,`router`等多个提交类型，都能进行转换。**

参数映射的规则比较灵活。

1. `struct`中需要匹配的属性必须为**`公开属性`**(首字母大写)。
2. 参数名称会自动按照 **`不区分大小写`** 且 **忽略`-/_/空格`符号** 的形式与`struct`属性进行匹配。
3. 如果匹配成功，那么将键值赋值给属性，如果无法匹配，那么忽略该键值。

总结来说：

> `Parse`方法能够让我们更便捷，更加灵活的获得请求输入的参数。懒人的必备！！！😁

### 请求校验

因为`Parse`方法里面会自动去进行数据校验，我们需要做的就是添加tag标签`v` 

具体规则：[数据校验规则](https://goframe.org/pages/viewpage.action?pageId=1114367)

推荐以下两篇文章：

[定义错误](https://goframe.org/pages/viewpage.action?pageId=1114282)

[自定义规则](https://goframe.org/pages/viewpage.action?pageId=1114384)

可以让你得数据校验更加人性化以及智能化。

回到正题。

定义得`model`对象，添加了tag-`v`后，`Parse`会自动得进行数据校验，根据校验规则会返回你自定义的错误。

例子：

```go
//model
type U struct{
    Id    uint `json:"id" p:"id" v:"required|between:6,30#请输入数字|数字大小为:min到:max位"`
}

//api
func (c *Controller) Post(r *ghttp.Request) {
	var user *model.U
	if err:=r.Parse(&user);err!=nil{
		response.JsonExit(r, 400, err.Error())
	}
	response.JsonExit(r,200,"成功",g.Map{
		"data":user.Id,
	})
	r.Response.Write("POST")
}
```

当不进行任何输入，发起请求，就会导致错误全部会输出出来。对用户体验不是很友好。

![image-20211005163257335](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20211005163257.png)

有两种处理方式

* 可以通过`err.(gvalid.Error)`断言的方式判断错误是否为校验错误，然后返回第一条错误信息
* 可以使用`gerror.Current`获取第一条报错信息，这里会自动判断，返回第一条错误信息

### 参数优先级

请求参数的获取也有对应得优先级

优先级规则：`Router < Query < Body < Form < Custom`

例如，`Query`和`Form`中都提交了同样名称的参数`id`，参数值分别为`1`和`2`，那么`Get("id")`/`GetForm("id")`将会返回`2`，而`GetQuery("id")`将会返回`1`。

### 默认值

GF框架支持通过`struct tag`的方式为输入对象的属性绑定默认值。默认值的`struct tag`名称为`d`(也可以使用`default`)。

```go
type ContentServiceGetListReq struct {
    Type       string                                    // 内容模型
    CategoryId uint   `p:"cate"`                         // 栏目ID
    Page       int    `d:"1"  v:"min:0#分页号码错误"`      // 分页号码
    Size       int    `d:"10" v:"max:50#分页数量最大50条"` // 分页数量，最大50
    Sort       int    // 排序类型(0:最新, 默认。1:活跃, 2:热度)
}
```

## 数据返回

1. `Write*`方法用于数据的输出，可为任意的数据格式，内部通过断言对参数做自动分析。
2. `Write*Exit`方法用于数据输出后退出当前服务方法，可用于替代`return`返回方法。
3. `WriteJson*`/`WriteXml`方法用于特定数据格式的输出，这是为开发者提供的简便方法。
4. `WriteTpl*`方法用于模板输出，解析并输出模板文件，也可以直接解析并输出给定的模板内容。
5. `ParseTpl*`方法用于模板解析，解析模板文件或者模板内容，返回解析后的内容。

对于前后端分离的项目是前三个。

1. `WriteJson*` 方法用于返回`JSON`数据格式，参数为任意类型，可以为`string`、`map`、`struct`等等。返回的`Content-Type`为`application/json`。

例子

```go
 r.Response.WriteJson(g.Map{
                "id":   1,
                "name": "john",
            })
```

2. `Exit`: 仅退出当前执行的逻辑方法，不退出后续的请求流程，可用于替代`return`。

忠告：

> 以上的所有都是我自己根据以往的项目经验，然后对文档进行总结的，对于很少使用（例如：模板）就只是简单看来下，我个人比较热衷于前后端分离的项目，所以希望了解到完整的，可以去官网看相关的文档。

