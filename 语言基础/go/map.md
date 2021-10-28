# map

## map概要

Go语言中map是散列表的引用。散列表可以用来存储键值对元素。

map类型是map[k]v，其中k是字典的键，v是字典中值对应的数据类型。

> 💡:k的类型必须是可以通过操作符==来进行比较的数据类型。v的类型则没有限制。

```go
var a map[string]string
```

map类型的变量本质上是个指针，这些键值对实际上是通过哈希表来存储的。

![图片](https://mmbiz.qpic.cn/mmbiz_png/ibjI8pEWI9L4iaD5VQHquw0dxSYrziavjtia18CzzSekKVu6VSXzjE2icFug0ibPPhbOJ1FntSSibOzemFmpb236TibxMQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

> ❓：为什么要用哈希表？

能够存储键值对的数据结构有很多种，可以是数组，也可以是链表，但是能用不代表好用。如果每次查找一个键都要从第一个元素开始遍历，直到发现匹配的键或者遍历完所有元素，这样的时间复杂度就是O(n)。

散列表的集合，键值是唯一的，键对应的值可以通过键来获取，更新或者删除。这些操作基本上通过常量时间键的比较来完成，时间复杂度就是O(1)

## Hashmap

### 建表





## 参考文章

 [【Golang】图解map](https://mp.weixin.qq.com/s/0sjLD4_12Ql0QeWsO_A5pQ) 

