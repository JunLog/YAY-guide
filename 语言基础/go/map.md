# map

## map概要

Go语言中map是散列表的引用。散列表可以用来存储键值对元素。

map类型是map[k]v，其中k是字典的键，v是字典中值对应的数据类型。

> 💡:k的类型必须是可以通过操作符==来进行比较的数据类型。v的类型则没有限制。

```go
var a map[string]string
```

map类型的变量本质上是个指针，这些键值对实际上是通过哈希表来存储的。

![图片](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20211029195933.webp)

> ❓：为什么要用哈希表？

能够存储键值对的数据结构有很多种，可以是数组，也可以是链表，但是能用不代表好用。如果每次查找一个键都要从第一个元素开始遍历，直到发现匹配的键或者遍历完所有元素，这样的时间复杂度就是O(n)。

散列表的集合，键值是唯一的，键对应的值可以通过键来获取，更新或者删除。这些操作基本上通过常量时间键的比较来完成，时间复杂度就是O(1)

## Hashmap

### 建表

哈希表通常会有一堆桶(`bucket`)来存储键值对。现在有一个键值对`{k,v}` ，那么怎么选桶？通过hash函数将k转换成hash值，通过hash从桶的编号进行选择，从而通过`array[hash[key]]`实现O(1)的查找效率。

这里有两种方式比较常用

1. 取模运算 `hash%m`（通过hash值与桶的个数进行取模得到编号）
2. 按位与运算 `  hash&(m-1)` (hash值与m-1进行与运算)

第二种更加高效，但是有一定的限制。

> ❗：m必须要是2的整数次幂这样才能保证与运算结果落在[0,m-1]，而不会出现有些桶注定不会被选中的情况。
>
> 当m是5，他的二进制数为`101` 那么m-1的二进制数为`100` ，无论什么数与m-1进行与运算，后面两位都是0，就会出现编号[1,3]的桶注定不会被选中的情况。
>
> ![image-20211028143533945](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20211028143534.png)

哈希函数的实现有很多，其目的都是让哈希结果能够尽可能的均匀分布在`0~length(array)`间。

一个完美的哈希函数就是将`key`和`index`一 一对应；时间复杂度为O(1)

一个极坏的哈希函数就是将所有的`key`都映射到一个`index`上。可能达到的时间复杂度为O(n)

### 哈希冲突

当我们再次存一个键值对`{k2,v2}`时候他的哈希值通过哈希函数得到桶的编号，桶里面已经有存储了一个键值对。这就是哈希冲突。在通常情况下，哈希函数输入的范围一定会远远大于输出的范围，所以在使用哈希表时一定会遇到冲突。**通常有两个思路来解决。**

> 第一种：开放地址法

假设此时需要存储的键值对`{k2,v2}`对应的桶编号1已经被占用了，那么就用下一个空闲桶编号3来存键值对。

当查找`k2`时候，根据哈希函数定位到编号为1的桶，但是通过比较发现key不对，那么就会继续检测下一个桶，直到匹配成功或者遇到空桶就代表key不存在。

![图片](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20211029200029.webp)

> 第二种：拉链法（go的map就是利用这种方式解决）

假设此时需要存储的键值对`{k2,v2}`对应的桶编号1已经被占用了，那么就在编号为1的桶后面链接一个桶存储键值对。此时的编号1的桶还要额外记录一个指针，指向连在一起桶。

当查找`k2`时候，根据哈希函数定位到编号为1的桶，但是通过比较发现key不对，就会通过指针去查询，通过比较key的值来锁定键值对。

![图片](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20211029200111.webp)

> ❗：如果哈希函数可以很均匀的把key映射到各个桶里，发生哈希冲突的几率就会降低。哈希冲突会影响哈希读写的效率，适当的对哈希表的扩容也是一个保障读写效率的手段之一。

### 扩容

关于扩容，我们要搞清楚两点：

* 何时扩容？
* 怎么扩容？

> 何时扩容？

哈希表中存储的键值对数量和桶数量的比值会作为判断哈希表是否需要扩容的依据，这个依据叫做**负载因子**（Load Factor）通常需要设定一个触发扩容的最大负载因子。

  `loadFactor = keyCount / bucketCount`

> 怎么扩容？

需要扩容时候，就会分配更多的桶作为新桶，此时需要把旧桶的值都迁移到新桶中，如果一次性把所有键值对挪到新的桶里，那么对于键值对数量很多情况，每次扩容占用的时间太长就会造成性能瞬时明显抖动，所以通常会选择**“渐进式扩容”**。

一个字段（oldbuckets ）记录旧桶的位置，一个字段（nevacuate ）记录旧桶迁移的进度。当需要扩容时候，会分配足够多的新桶，在每次哈希表读写操作，检测当当前处于迁移阶段，就迁移一部分键值对到新桶里，直到所有数据迁移成功。

![图片](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20211029200131.webp)

## Go中的map

map类型的变量本质上是一个hmap类型的指针：

```go

type hmap struct {
    count     int    //已经存储的键值对个数
    flags     uint8
    B         uint8  // 常规桶个数等于2^B,选择桶用的是与运算
    noverflow uint16 // 使用的溢出桶数量
    hash0     uint32 // hash seed
    buckets    unsafe.Pointer // 常规桶起始地址
    oldbuckets unsafe.Pointer // 扩容时保存原来常规桶的地址
    nevacuate  uintptr        // 渐进式扩容时记录下一个要被迁移的旧桶编号

    extra *mapextra //里面记录的都是溢出桶相关的信息：
}
```

### bmap

`map`使用的桶很有设计感，每个桶里可以存储8个键值对，并且为了内存使用更加紧凑，8个键放一起，8个值放一起。对应每个`key`只保存其哈希值的高8位（`tophash`）。而每个键值对的`tophash`、`key`和`value`的索引顺序一一对应。这就是`map`使用的桶的内存布局。

![图片](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20211029195952.webp)

既然每个桶里可以存储8个元素，那存满了怎么办？扩容的代价还是比较高的，所以为了减少扩容次数，这里引入了**溢出桶(overflow)**。溢出桶的内存布局与之前介绍的**常规桶**相同。如果哈希表要分配的桶的数目大于2^4，就会预分配2^(B-4)个溢出桶备用。这些常规桶和溢出桶在内存中是连续的，只是前2^B个用作常规桶，后面的用作溢出桶。

```go
//溢出桶相关的信息
type mapextra struct {
    overflow    *[]*bmap //把已经用到的溢出桶链起来
    oldoverflow *[]*bmap //渐进式扩容时，保存旧桶用到的溢出桶
    nextOverflow *bmap   //下一个尚未使用的溢出桶
}
```

![图片](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20211029200142.webp)

如果当前桶存满了以后，检查`hmap.extra.nextoverflow`，如果还有可用的溢出桶，就在这个桶后面链上这个溢出桶，然后继续往这个溢出桶里存。而`hmap.extra.nextoverflow`继续指向下一个空闲的溢出桶。

![图片](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20211029200150.webp)

如果我们把哈希表的桶存满了，那么现在再继续存储键值对，那么这个哈希表会创建溢出桶还是发生扩容呢？

### 扩容规则

> 翻倍扩容

go语言中`map`负载因子默认是`6.5`，当比值超过负载因子就会触发**翻倍扩容**。

` hmap.count / 2^hmap.B > 6.5`

分配新桶数目是旧桶的2倍，`hmap.oldbuckets`指向旧桶，`hmap.buckets`指向新桶。`hmap.nevacuate`为0，表示接下来要迁移编号为0的旧桶。

![图片](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20211029200157.webp)

例如：将旧桶数量记为m=4，新桶数量就是2m=8，如果一个哈希值选择0号旧桶：h&(m-1)=0，那么h的二进制最低两位一定为0(`xxxxxx00`)，对于新桶，如果第三位是1的话，就去编号为4的桶，如果为0的话就去编号为0的桶。每个旧桶的键值对都会分流到两个新桶中。

![图片](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20211029200203.webp)

编号为`hmap.nevacuate`的旧桶迁移结束后会增加这个编号值，直到所有旧桶迁移完毕，把`hmap.oldbuckets`置为nil，一次翻倍扩容结束。

> 等量扩容

如果没有超过设置的负载因子上限，但是使用的溢出桶较多，也会触发扩容，不过这一次是**等量扩容**。

**第一个问题：用多少溢出桶算多？**

1. 如果常规桶数目不大于`2^15`，那么使用的溢出桶数目超过常规桶就算是多了；
2. 如果常规桶数目大于`2^ 15`，那么使用溢出桶数目一旦超过`2^15`就算多了。

**第二个问题：为什么要等量扩容？**

在很多键值对被删除的情况下，桶的负载因子没有超过上限值，却偏偏使用了很多溢出桶。

![图片](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20211029200217.webp)

这种情况下，如果把这些键值对重新安排到等量的新桶中，虽然哈希值没变，常规桶数目没变，每个键值对还是会选择与旧桶一样的新桶编号，但是能够存储的更加紧凑，进而减少溢出桶的使用。

![图片](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20211029200224.webp)

## 常见问题

> 1. map每次遍历的顺序都是不同的。

为了防止开发人员依赖map的顺序，对于每一次map遍历（`range`），得到结果的顺序都是不同的，因为在初始化迭代器（`runtime.mapiterinit`）时都对`startBucket`和`startCell`进行了`random`操作。

```go
func mapiterinit(t *maptype, h *hmap, it *hiter) {

   //some code ...

   // decide where to start

   r := uintptr(fastrand()) //fastrand 是一个快速生成随机数的函数

   if h.B > 31-bucketCntBits {

      r += uintptr(fastrand()) << 31

   }

   it.startBucket = r & bucketMask(h.B) 

   //bucketMask返回bucket的个数-1，也就是全1，将r和bucketMask做与操作，获得开始的桶编号

   it.offset = uint8(r >> h.B & (bucketCnt - 1))

   //bucketCnt是常量8代表一个桶内的cell数，将r右移B位后与7做与运算，可以获得开始的cell编号

   //some code ...

}

```

> 2. k的类型必须是可以通过操作符==来进行比较的数据类型

实`Go`语言中每种类型都有对应的类型元数据，类型元数据都有一个相同的`Header`，就是`runtime._type`。而在`_type.alg`这里记录了该类型的两个函数：`hash`和`equal`。

```go
type typeAlg struct {
    hash  func(unsafe.Pointer, uintptr) uintptr
    equal func(unsafe.Pointer, unsafe.Pointer) bool
}
```

![图片](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20211029200230.webp)

slice、function value、map是不可比较的(但是这三种类型都可以和nil进行对比)

> 3. 不可寻址

正是因为扩容过程中会发生键值对迁移，键值对的地址也会发生改变，所以才说map的元素是**不可寻址**的，如果要取一个value的地址则**不能通过编译**。

> 4.map不支持并发读写，只能并发读

## 总结

Go语言中的map通过**与运算**找桶的编号，发生**哈希冲突**时候，采用**拉链法**解决，当**负载因子**超过`6.5`时候，发生**翻倍扩容**，如果溢出桶数量过多，那么采用**等量扩容**。采用**渐进式**迁移旧桶键值对。

## 使用注意

```go
func main(){
	find:=map[int]int{}
	find[1]=0
	idx:=0
    //在这里的短变量声明，不会对 idx 进行赋值处理，而是重新声明值。注意注意！
	if idx,ok:=find[1];!ok {
		println(111)
	}
	println(idx)
}
```



## 参考文章

 [【Golang】图解map](https://mp.weixin.qq.com/s/0sjLD4_12Ql0QeWsO_A5pQ) 

[【深度解析golang map】](https://juejin.cn/post/6954707500151078919#heading-4)

[[由浅到深，入门Go语言Map实现原理](https://segmentfault.com/a/1190000039101378)](https://segmentfault.com/a/1190000039101378)

[golang中map底层B值的计算逻辑](https://zhuanlan.zhihu.com/p/366472077)

