# captcha

> 文档：[官方](https://github.com/dchest/captcha)

这个第三方的库挺有意思的。他具备很多优点：

* 可自定义验证码长度
* 生成的图片不需要额外内存存储，通过 `path` 生成即可
* 使用本地 `cache`，可 `reload` 验证码
* 通过hash生成随机数作为验证码 `ID`

## 快速使用

这个第三方库，他已经将很多东西进行了封装，你只需要按照你的需求进行调用。

### 生成验证码 ID

```go
type Captcha struct {
	Id      string `json:"id,omitempty"`      //验证码ID
	ImgUrl  string `json:"img_url,omitempty"` //验证码图像地址
	Refresh string `json:"refresh,omitempty"` //重新获取
	Verify  string `json:"verify,omitempty"`  //验证
}

//GenerateId
/* @Description: 生成验证码Id
 * @receiver c
 * @param ctx
 */
func (c *Captcha) GenerateId(ctx *gin.Context) {
	//获取验证码验证数字长度
	var length = variable.ConfigYml.GetInt("Captcha.Length")
	//自定义验证码数字长度
	captchaId := captcha.NewLen(length)
	//提供相关信息
	c.Id = captchaId
	c.ImgUrl = captchaId + ".png"
	c.Refresh = c.ImgUrl + "?reload=1"
	c.Verify = captchaId + "/这里替换为正确的验证码进行验证"
	response.Success(ctx, "验证码信息", c)
}
```

`ID` 可以生成相对应的验证码图片，然后以键的形式去校验验证码。这个还是很重要的。

### 获取图片

```go
//GetImg
/* @Description: 生成验证码的图片
 * @receiver c
 * @param ctx
 */
func (c *Captcha) GetImg(ctx *gin.Context) {
	//通过param获取验证码Id
	captchaId := ctx.Param(captchaIdKey)
	//根据路径的最后一个斜杠， 将路径划分为目录部分和文件名部分。 例 http://localhost:20201/captcha/  JUpvESBqAcPxvSVz5Cy9.png
	_, file := path.Split(ctx.Request.URL.Path)
	//获取扩展名 .png
	ext := path.Ext(file)
	//获取路径上的Id
	id := file[:len(file)-len(ext)]
	//验证参数是否获取正常
	if ext == "" || captchaId == "" {
		response.Fail(ctx, http.StatusBadRequest, consts.CaptchaGetParamsInvalidCode, consts.CaptchaGetParamsInvalidMsg)
		return
	}
	//当获取到 Reload 就重新加载一个
	if ctx.Query(Reload) != "" {
		//重载为给定的验证码生成并记住新的数字。
		captcha.Reload(id)
	}
	//设置http协议
	ctx.Header("Cache-Control", "no-cache, no-store, must-revalidate")
	ctx.Header("Expires", "0")
	var vBytes bytes.Buffer
	if ext == ".png" {
		ctx.Header("Content-Type", "image/png")
		//写入图片的二进制信息
		_ = captcha.WriteImage(&vBytes, id, Width, Height)
		//读取文件内容并输出的方法
		http.ServeContent(ctx.Writer, ctx.Request, id+ext, time.Time{}, bytes.NewReader(vBytes.Bytes()))
	}
}
```

`Captcha` 他通过验证码 ID 去生成验证码图像的二进制，你只需要将二进制信息放到 `http` 协议中，然后进行显示就可，这个操作不需要你利用额外的空间存储图片。

### 校验验证码

```go
//CheckCode
/* @Description: 校验验证码传来的数字
 * @receiver c
 * @param ctx
 */
func (c *Captcha) CheckCode(ctx *gin.Context) {
	//获取验证码ID 与验证码的值
	captchaId := ctx.Param(captchaIdKey)
	value := ctx.Param(captchaValueKey)
	//去存储器中进行验证
	if captcha.VerifyString(captchaId, value) {
		response.Success(ctx, consts.CaptchaCheckParamsOk)
	} else {
		response.Fail(ctx, http.StatusBadRequest, consts.CaptchaCheckParamsInvalidCode, consts.CaptchaCheckParamsInvalidMsg)
	}
}
```

根据请求路径获取的字段值，`Captcha`他会将 `captchaId` 与 `value` 放到本地的存储器中进行查询，如果可以查到就返回 `true`。

### 创建路由

然后你就可以直接创建验证码的路由。

```go
	// 创建一个验证码路由
	verifyCode := router.Group("captcha")
	{
		// 验证码业务，该业务无需专门校验参数，所以可以直接调用控制器
		verifyCode.GET("/", (&captcha.Captcha{}).GenerateId)                          //  获取验证码ID
		verifyCode.GET("/:captcha_id", (&captcha.Captcha{}).GetImg)                   // 获取图像地址
		verifyCode.GET("/:captcha_id/:captcha_value", (&captcha.Captcha{}).CheckCode) // 校验验证码
	}
```

这个整体流程其实不难。主要是明白，他这个流程是怎么样的！

## 扩展

### 更改存储器

他设置了两个常量 `CollectNum` 和 `Expiration` 。

* `CollectNum` 代表创建验证码的数量会去触发垃圾回收。（一次清理过期验证码的数量）
* `Expiration` 代表验证码的过期时间

``` go
func init() {
	//对存储器进行自定义设置
	//设置一次清理过期验证码的数量
	collectNum := variable.ConfigYml.GetInt("Captcha.CollectNum")
	//过期时间
	expiration := variable.ConfigYml.GetDuration("Captcha.Expiration")
	// 返回一个新的标准内存存储器
	s := captcha.NewMemoryStore(collectNum, expiration)
	//设置一个新的存储器
	captcha.SetCustomStore(s)
}
```

### 创建音频验证码

```go
func (c *Captcha) GetAudio(ctx *gin.Context) {
	//通过param获取验证码Id
	captchaId := ctx.Param(captchaIdKey)
	//根据路径的最后一个斜杠， 将路径划分为目录部分和文件名部分。 例 http://localhost:20201/captcha/  JUpvESBqAcPxvSVz5Cy9.wav
	_, file := path.Split(ctx.Request.URL.Path)
	//获取扩展名 .wav
	ext := path.Ext(file)
	//获取路径上的Id
	id := file[:len(file)-len(ext)]
	//验证参数是否获取正常
	if ext == "" || captchaId == "" {
		response.Fail(ctx, http.StatusBadRequest, consts.CaptchaGetParamsInvalidCode, consts.CaptchaGetParamsInvalidMsg)
		return
	}
	ctx.Header("Cache-Control", "no-cache, no-store, must-revalidate")
	ctx.Header("Pragma", "no-cache")
	ctx.Header("Expires", "0")
	//当获取到 Reload 就重新加载一个
	if ctx.Query(Reload) != "" {
		//重载为给定的验证码生成并记住新的数字。
		captcha.Reload(id)
	}
	var vBytes bytes.Buffer
	if ext == ".wav" {
		ctx.Header("Content-Type", "audio/wav")
		//写入audio的二进制信息
		_ = captcha.WriteAudio(&vBytes, id, Lang)
		//读取文件内容并输出的方法
		http.ServeContent(ctx.Writer, ctx.Request, id+ext, time.Time{}, bytes.NewReader(vBytes.Bytes()))
	}
}

```



## 参考文章

> [captcha验证码总结](https://forster.site/2017-04-20-%E9%AA%8C%E8%AF%81%E7%A0%81%E6%80%BB%E7%BB%93.html)
>
>  [Golang Http 验证码示例](https://segmentfault.com/a/1190000023703468)