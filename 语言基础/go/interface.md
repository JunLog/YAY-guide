# 接口 interface

> 推荐阅读：[【Golang】图解Interface](https://mp.weixin.qq.com/s?__biz=Mzg5NjIwNzIxNQ==&mid=2247484072&idx=2&sn=0363c7102943888e0f390f3f5a9ae662&chksm=c005d2a8f7725bbe068f418f72bd9f8ecc3aec3daf11ba7e7ad80d202e6ad90ceec459c5c7e8&scene=21#wechat_redirect)
>
> [深入研究 Go interface 底层实现](https://halfrost.com/go_interface/)

## 空接口 interface{}

空接口类型可以接受任意类型的数据。干的事情不多，记录数据的位置和数据类型即可。空接口类型如下：

```go
type eface struct {
    _type *_type //指向接口的动态类型元数据
    data  unsafe.Pointer //指向接口的动态值。
}
```

举个例子🌰：

```go
var e interface{} //这里定了一个空接口类型

//在 e 没有赋值前
type eface struct {
    _type *_type //指向接口的动态类型元数据 ->nil
    data  unsafe.Pointer //指向接口的动态值。 ->nil
}
```

```go
f, _ := os.Open("eggo.txt")
e = f
```

如果将 `*os.File` 类型的变量 `f` 赋值给 `e`。来看看变量 `e` 的结构。

```go
type eface struct {
    _type *_type //指向接口的动态类型元数据 ->*os.File 的类型元数据
    data  unsafe.Pointer //指向接口的动态值。 ->f
}
```

**注：类型元数据这里可以找到类型关联的方法元数据列表。**

![图片](https://mmbiz.qpic.cn/mmbiz_png/ibjI8pEWI9L53Y12T80IUoDOdNgcCOb0BBUWtBay6WaLES8uevBQbeM0Amxibne8wIk27uAFpiciawgyn6Y5FpRhFw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## 非空接口

非空接口就是有方法列表的接口类型。如果一个变量想赋值一个非空接口类型，那么其类型必须实现该接口的要求的所有方法才行。非空接口类型如下：

```go
type iface struct {
    tab   *itab
    data  unsafe.Pointer //接口的动态值
}
//itab
type itab struct {
    inter  *interfacetype //接口的类型元数据
    _type  *_type //指向接口的动态类型元数据
    hash   uint32 //itab._type中拷贝来的，类型哈希值，用于快速判断类型是否相等时使用，后续会有介绍
    _      [4]byte
    fun    [1]uintptr //动态类型实现的接口要求方法地址
}
//interfacetype
type interfacetype struct {
    typ      _type
    pkgpath  name
    mhdr     []imethod //接口要求的方法列表
}   
```

举个例子🌰：

我们声明了一个 `io.ReadWriter` 的接口类型变量 `rw`。被赋值前，结构如下：

```go
type iface struct {
    tab   *itab  //->nil
    data  unsafe.Pointer //接口的动态值 ->nil
}
```

```go
f, _ := os.Open("eggo.txt")
rw = f
```

我们将 `*os.File` 类型变量 `f`。赋值给 `rw`。

那么 `rw` 的动态值是 `f`，动态类型是 `*os.File`。`itab.fun` 这个数组记录的是 `*os.File` 这个类型实现的 `io.ReadWriter` 接口的方法。

![图片](https://mmbiz.qpic.cn/mmbiz_png/ibjI8pEWI9L53Y12T80IUoDOdNgcCOb0BbQ8syrnoVjFbicOmEkIpPJBEOucLzmWEWPCEry6vSEeWrXxULZKs4cA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

注：`rw` 能使用的方法只是接口类型已经注册的。

总结：

> 接口，你可以看作是一个工具（枪），你往里面装怎么样的子弹，他就会打出什么样子弹。

## itab缓存

一个非空接口类型 `interfacetype` 和一个动态类型 `_type` 就可以确定一个 `itab`了。剩余的字段要么来自 `_type` 要么是动态类型的方法。似乎这个 `itab` 是可以复用的。那么对于两个个非空接口类型定义的变量，赋值后只要动态类型不变，变得只是动态值 `data`。

实际上 `Go` 语言会把用到的 `itab` 结构体缓存起来，并且以 **<接口类型，动态类型>** 组合为 key，以 `*itab` 为 `value`，构成一个哈希表，用于存储与查询以及复用 `itab` 信息

![图片](https://mmbiz.qpic.cn/mmbiz_png/ibjI8pEWI9L75jtH0rYodSfoAGwwpIyARicXkrbAZP3pmf1LVqkylMEGMkskmd1CYBmkE1PEzt1eTEW9nOBicVbaA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

这个哈希表可与我们使用的 `map` 底层的哈希表不同，结构设计更为简便。

```go
type itabTableType struct {
    size    uintptr             // length of entries array. Always a power of 2. // entries 数组的长度。总是 2 的幂。
    count   uintptr             // current number of filled entries.  //当前填充的entries数目
    entries [itabInitSize]*itab // really [size] large //实际的大小
}
```

* 当需要一个 `itab` 会先去 `itabTable` 里面查找，计算哈希值时会用到接口类型(`itab.inter`)和动态类型(`itab._type`)的类型哈希值
* 如果能查询到对应的 `itab` 指针就直接拿来用，如果没有就直接创建一个新的 `itab`。

