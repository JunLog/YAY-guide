# yang-bot

# 安装

先跟着[基础教程](https://docs.go-cqhttp.org/guide/quick_start.html#%E5%9F%BA%E7%A1%80%E6%95%99%E7%A8%8B)，将机器人框架装入服务器中。

**一定要选择最新的版本进行安装**

然后后面会要求改配置文件。

> 注：需要改两个地方
>
> *  uin: 1665085209 # QQ账号
> *    \# 服务端监听地址 host: 0.0.0.0
>
> 其余的可不动！！！！！

然后进行测试！

* 你可以给你的机器人发送消息，你就会看到服务器上会接受到消息

![image-20210917200144268](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20210917200144.png)

* 调用发送私信的api接口

`http://填服务器地址:端口/send_private_msg?user_id=qq号&message=ffeecoishp`

如果调用成功，你也会在服务器上看到消息！

![image-20210917200323329](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20210917200323.png)

如果上面测试全部成功，那么恭喜你，你的机器人安装成功，可以开始编写程序了！！！



后续要做的事情有

* 获取心跳包，所有参数，反序列化
* 获取发送的信息，将其反序列化
* 如需发送信息的话，调用api