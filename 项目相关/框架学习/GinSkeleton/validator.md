# validator

> 推荐学习：[validator库参数校验若干实用技巧](https://www.liwenzhou.com/posts/Go/validator_usages/)
>
> 官方文档：[validator](https://godoc.org/github.com/go-playground/validator#hdr-Baked_In_Validators_and_Tags)

这里主要介绍validator以注册功能为例从注册到使用的全过程。

## 注册验证器

首先你需要在`http\validator\common\register_validator` 这里注册你的验证器，也就是将它以键值对的形式进行存储到容器里面。当然了，这里需要进行初始化。

```go
	key = consts.ValidatorPrefix + "UsersRegister"
	containers.Set(key, users.Register{})
```

## 编写验证器

这里注释写的很清楚了。

```go
// 给出一些最常用的验证规则：
//required  必填；
//len=11 长度=11；
//min=3  如果是数字，验证的是数据范围，最小值为3，如果是文本，验证的是最小长度为3，
//max=6 如果是数字，验证的是数字最大值为6，如果是文本，验证的是最大长度为6
// mail 验证邮箱
//gt=3  对于文本就是长度>=3
//lt=6  对于文本就是长度<=6

type Register struct {
	BaseField
	// 表单参数验证结构体支持匿名结构体嵌套、以及匿名结构体与普通字段组合
	Phone  string `form:"phone" json:"phone"`     // 手机号， 非必填
	CardNo string `form:"card_no" json:"card_no"` //身份证号码，非必填
}

// 特别注意: 表单参数验证器结构体的函数，绝对不能绑定在指针上
// 我们这部分代码项目启动后会加载到容器，如果绑定在指针，一次请求之后，会造成容器中的代码段被污染

func (r Register) CheckParams(context *gin.Context) {
	//1.先按照验证器提供的基本语法，基本可以校验90%以上的不合格参数
	if err := context.ShouldBind(&r); err != nil {
		errs := gin.H{
			"tips": "UserRegister参数校验失败，参数不符合规定，user_name 长度(>=1)、pass长度[6,20]、不允许注册",
			"err":  err.Error(),
		}
		response.ErrorParam(context, errs)
		return
	}
	//2.继续验证具有中国特色的参数，例如 身份证号码等，基本语法校验了长度18位，然后可以自行编写正则表达式等更进一步验证每一部分组成
	// r.CardNo  获取值继续校验，这里省略.....

	//  该函数主要是将本结构体的字段（成员）按照 consts.ValidatorPrefix+ json标签对应的 键 => 值 形式绑定在上下文，便于下一步（控制器）可以直接通过 context.Get(键) 获取相关值
	extraAddBindDataContext := data_transfer.DataAddContext(r, consts.ValidatorPrefix, context)
	if extraAddBindDataContext == nil {
		response.ErrorSystem(context, "UserRegister表单验证器json化失败", "")
	} else {
		// 验证完成，调用控制器,并将验证器成员(字段)递给控制器，保持上下文数据一致性
		(&web.Users{}).Register(extraAddBindDataContext)
	}

}

```

> 复杂参数的解决

如果这样一个一个进行获取在大量参数上会很复杂的。

其实很简单就是你单独设置一个就可了。

## 从工厂中取出验证器

```go
func Create(key string) func(context *gin.Context) {
	//通过键key取出验证器对象。
	if value := container.CreateContainersFactory().Get(key); value != nil {
		//通过断言拿到对象
		if val, isOk := value.(interf.ValidatorInterface); isOk {
			return val.CheckParams
		}
	}
	variable.ZapLog.Error(my_errors.ErrorsValidatorNotExists + ", 验证器模块：" + key)
	return nil
}
```

然后就会执行对象的`CheckParams`接口（验证器）。

## 控制器

```go
(&web.Users{}).Register(extraAddBindDataContext)
```

接下来就是你编写控制器日常的操作了。

获取参数。

```go
	userName := context.GetString(consts.ValidatorPrefix + "user_name")
	pass := context.GetString(consts.ValidatorPrefix + "pass")
	userIp := context.ClientIP()
```

剩余就常规处理了。

以上整体的流程如下：

* 创建一个对象，实现`CheckParams`接口
* 项目初始化，注册到容器里面
* 当你请求来的时候，从工厂中取出验证器（切入验证器）将参数添加到上下文中，供全局使用。
* 然后进入控制器，之后就是`service`和`model`

**我只能说优雅！！！！**

