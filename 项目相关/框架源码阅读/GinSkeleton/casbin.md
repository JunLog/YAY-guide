# casbin

> 推荐学习：[casbin开发文档](https://casbin.org/docs/zh-CN/overview)
>
> [Go 每日一库之 casbin](https://darjun.github.io/2020/06/12/godailylib/casbin/)
>
> [casbin视频](https://www.bilibili.com/video/BV1qz4y167XP?spm_id_from=333.999.0.0)
>
> [在 Go 语言中使用 casbin 实现基于角色的 HTTP 权限控制](https://studygolang.com/articles/12323)

## 什么是casbin

Casbin是一个强大的、高效的开源访问控制框架，其权限管理机制支持多种访问控制模型。

说起访问控制，之前所用的只是 jwt 方式，一个简单的鉴权，但是遇到了复杂场景，那么这个方式就不太合适了。

说到鉴权，先问三个问题：

* 如何鉴别一个人身份？
* 如何允许一个人的行为？
* 如何将身份与行为进行关联？

## ACL模型

ACL 模型至少会包含以下四个部分：

> [request_definition]：请求定义

```ini
[request_definition]
r = sub, obj, act
```

`sub, obj, act` 表示经典三元组: 访问实体 (Subject)，访问资源 (Object) 和访问方法 (Action)。

通俗话讲：sub 指是谁来访问，obj 指你要访问什么，act 指你打算怎么访问

这里规定：你需要按照这个定义去发起一个访问。

> [policy_definition]：策略定义

```ini
[policy_definition]
p = sub, obj, act，(eft)
```

这里也是经典三元组`sub, obj, act` 。当然啦，你也可以定义其他名称。

eft 是影响，你可以去制定影响，但是这里的值只能是 allow 和 deny 。

这里规定：可以通过哪些部分进行限制

> [matchers]：规则

```ini
[matchers]
m = r.sub == p.sub && r.obj == p.obj && r.act == p.act
```

通过两部分（`request_definition，policy_definition`）去制定访问的规则。

您可以使用算术（如 `+、-、*、/`）和逻辑运算符（如 `&&、||、!`）。 

> [policy_effect]：策略的影响

这里的策略影响只能有以下这五个，不允许有其他的。

| Policy effect                                                | 意义                        |                             示例                             |
| :----------------------------------------------------------- | --------------------------- | :----------------------------------------------------------: |
| some(where (p.eft == allow))                                 | 至少一个allow               | [ACL, RBAC, etc.](https://casbin.org/docs/en/supported-models#examples) |
| !some(where (p.eft == deny))                                 | 不允许有一个deny            | [Deny-override](https://casbin.org/docs/en/supported-models#examples) |
| some(where (p.eft == allow)) && !some(where (p.eft == deny)) | 至少一个allow但是不能有deny | [Allow-and-deny](https://casbin.org/docs/en/supported-models#examples) |
| priority(p.eft) \|\| deny                                    | 优先事项                    | [Priority](https://casbin.org/docs/en/supported-models#examples) |
| subjectPriority(p.eft)                                       | 基于角色的优先级            | [主题优先级](https://casbin.org/docs/en/supported-models#examples) |

以上是一个模型的部分。

我们要根据这个模型去指定一个访问控制的规则。

你可以打开 Casbin 的编译器，进行模拟使用。

![image-20211118163554237](https://cdn.jsdelivr.net/gh/baici1/img-typora/20211118163554.png)

简述一下流程：

* 以`alice, data1, read`去发起请求，那么在模型中就会进行对标 `sub=alice`，`obj=data1`，`act=read`
* 策略这边会根据你所写的进行对标`sub=alice`，`obj=data1`，`act=read`
* 利用请求和策略中拿到的值，进行规则的匹配，就会有一个`eft` ，也就是结果（allow 或者 deny）。
* `eft`，就会进入 `policy_effect`，再进行一次验证。
* 最后得出结果

> 注意点

①如果你希望去控制一个策略的`eft`，你需要在`policy_definition` ，否则会默认，你没有这个值。即使你的策略后面加了，也是没有用的。

![image-20211118165314217](https://cdn.jsdelivr.net/gh/baici1/img-typora/20211118165314.png)

因为他的 eft 是通过匹配生成的，而不是你匹配上就给 eft 赋值。

②如果你想使用`!some(where (p.eft == deny))`这样的匹配规则，那么你需要给`policy_definition`加上 `eft` 字段，否则在这里相当于没有规则。

![image-20211118165832331](https://cdn.jsdelivr.net/gh/baici1/img-typora/20211118165832.png)

你发起的一个请求，会与所有的策略进行匹配，如果没有匹配成功，不会将 eft 去赋值一个默认的 allow 或者一个定义 deny。最后结果这次的请求的 eft 没有一个deny （可能是一个空值）。这也对应了他的要求：匹配完成后不存在一个 deny。所以返回结果就是 true。

你可以利用 Casbin 去验证自己的所有的猜想。

这种模型，就好像是一对一的比较，对于一些复杂场景，例如：只要是学生就可以吃饭，我总不能把所有学生的名字都这样一个一个写上去吧，好像就有点玩不来了。我们来看下一个模型。

## RBAC

这个模型又比 ACL 高级在哪里呢？

> [role_definition]：身份定义

```ini
[role_definition]
g = _, _
```

第一个下划线代表名字（具体的请求实体），第二个下划线代表处于什么身份（角色集）

我们回到刚才提到的问题：只要是学生就可以吃饭。

加了一个 role_definition 就能解决了吗？

根据这个问题进行拆分：角色集：学生，资源：饭，行为：吃。

只要当前请求实体身份是学生，想要的资源是饭，做出的行为是吃，就可以通过这次请求。

![image-20211118171726304](https://cdn.jsdelivr.net/gh/baici1/img-typora/20211118171726.png)

在这个模型里面，你可以通过身份达到对权限控制，同时也可以通过精准匹配达到权限控制。

`g(r.sub, p.sub)`他干了什么？进行身份的匹配，如果匹配的上，那么后续的策略匹配，我会带着这个身份去进行匹配，如果匹配不上，那么我只能去进行精准匹配了。

在某些特定的场景，好像这个也玩不来。例如：我希望学生中启明星工作室的能够拿学习资料。这里就有两个身份了，并且这两个身份启明星工作室是学生的子集。

## 扩展RBAC

他比 RBAC 多了一个域。在我看来就是又再次在身份下面又划分出一个域。就类似于有的学生还有启明星工作室这个身份。

```ini
[request_definition]
r = sub, dom, obj, act

[policy_definition]
p = sub, dom, obj, act

[role_definition]
g = _, _, _

[policy_effect]
e = some(where (p.eft == allow))

[matchers]
m = g(r.sub, p.sub, r.dom) && r.dom == p.dom && r.obj == p.obj && r.act == p.act
```

之前是基于一个全局的概念去进行的权限控制，现在可以在全局下再分多个域出来，基于这个域进行权限控制。

> 注意点

![image-20211118182637014](https://cdn.jsdelivr.net/gh/baici1/img-typora/20211118182637.png)

`g(r.sub, p.sub, r.dom)` 感觉好像只是做了一个身份验证，还需要其他的限制才能更好的控制域。

**没有看源码！只是自己的猜测。后面看源码后再做进一步的解释。**

## ABAC

`RBAC`模型对于实现比较规则的、相对静态的权限管理非常有用。但是对于特殊的、动态的需求，`RBAC`就显得有点力不从心了。例如，我们在不同的时间段对数据`data`实现不同的权限控制。正常工作时间`9:00-18:00`所有人都可以读写`data`，其他时间只有数据所有者能读写。这种需求我们可以很方便地使用`ABAC`（attribute base access list）模型完成

![image-20211118183224493](https://cdn.jsdelivr.net/gh/baici1/img-typora/20211118183224.png)

## 其他控制模型

控制模型远不止点，还有很多。

| 访问控制模型       | Model 文件                                                   | Policy 文件                                                  |
| ------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| ACL                | [basic_model.conf](https://github.com/casbin/casbin/blob/master/examples/basic_model.conf) | [basic_policy.csv](https://github.com/casbin/casbin/blob/master/examples/basic_policy.csv) |
| 具有超级用户的ACL  | [basic_with_root_model.conf](https://github.com/casbin/casbin/blob/master/examples/basic_with_root_model.conf) | [basic_policy.csv](https://github.com/casbin/casbin/blob/master/examples/basic_policy.csv) |
| 没有用户的ACL      | [basic_without_users_model.conf](https://github.com/casbin/casbin/blob/master/examples/basic_without_users_model.conf) | [basic_without_users_policy.csv](https://github.com/casbin/casbin/blob/master/examples/basic_without_users_policy.csv) |
| 没有资源的ACL      | [basic_without_resources_model.conf](https://github.com/casbin/casbin/blob/master/examples/basic_without_resources_model.conf) | [basic_without_resources_policy.csv](https://github.com/casbin/casbin/blob/master/examples/basic_without_resources_policy.csv) |
| RBAC               | [rbac_model.conf](https://github.com/casbin/casbin/blob/master/examples/rbac_model.conf) | [rbac_policy.csv](https://github.com/casbin/casbin/blob/master/examples/rbac_policy.csv) |
| 支持资源角色的RBAC | [rbac_with_resource_roles_model.conf](https://github.com/casbin/casbin/blob/master/examples/rbac_with_resource_roles_model.conf) | [rbac_with_resource_roles_policy.csv](https://github.com/casbin/casbin/blob/master/examples/rbac_with_resource_roles_policy.csv) |
| 支持域/租户的RBAC  | [rbac_with_domains_model.conf](https://github.com/casbin/casbin/blob/master/examples/rbac_with_domains_model.conf) | [rbac_with_domains_policy.csv](https://github.com/casbin/casbin/blob/master/examples/rbac_with_domains_policy.csv) |
| ABAC               | [abac_model.conf](https://github.com/casbin/casbin/blob/master/examples/abac_model.conf) | 无                                                           |
| RESTful            | [keymatch_model.conf](https://github.com/casbin/casbin/blob/master/examples/keymatch_model.conf) | [keymatch_policy.csv](https://github.com/casbin/casbin/blob/master/examples/keymatch_policy.csv) |
| 拒绝优先           | [rbac_with_not_deny_model.conf](https://github.com/casbin/casbin/blob/master/examples/rbac_with_not_deny_model.conf) | [rbac_with_deny_policy.csv](https://github.com/casbin/casbin/blob/master/examples/rbac_with_deny_policy.csv) |
| 同意与拒绝         | [rbac_with_deny_model.conf](https://github.com/casbin/casbin/blob/master/examples/rbac_with_deny_model.conf) | [rbac_with_deny_policy.csv](https://github.com/casbin/casbin/blob/master/examples/rbac_with_deny_policy.csv) |
| 优先级             | [priority_model.conf](https://github.com/casbin/casbin/blob/master/examples/priority_model.conf) | [priority_policy.csv](https://github.com/casbin/casbin/blob/master/examples/priority_policy.csv) |
| 明确优先级         | [priority_model_explicit](https://github.com/casbin/casbin/blob/master/examples/priority_model_explicit.conf) | [priority_policy_explicit.csv](https://github.com/casbin/casbin/blob/master/examples/priority_policy_explicit.csv) |
| 主体优先级         | [subject_priority_model.conf](https://github.com/casbin/casbin/blob/master/examples/subject_priority_model.conf) | [subject_priority_policyl.csv](https://github.com/casbin/casbin/blob/master/examples/subject_priority_policy.csv) |

