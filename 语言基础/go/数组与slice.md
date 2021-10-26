# 数组与Slice原理

## 数组

数组是一个具有固定长度且拥有0个或者多个相同数据类型元素的序列。

数组的每一个元素都是通过索引去访问的，索引从0到数组长度减一。Go内置的函数`len`可以返回数组中元素个数。

```go
package main

import "fmt"

func main(){
	var a [3]int
	b:=[3]int{1,2,3}
	c:=[...]int{1,2,3}
	d:=[...]int{9:-1}
	fmt.Println("a:",a)
	fmt.Println("b:",b)
	fmt.Println("c:",c)
	fmt.Println("d:",d)
}
```

![image-20211024192051643](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20211024192051.png)

默认情况下，声明一个数组`var a [3]int` ,元素的初始化值是元素类型的零值。（对于`int`,对应的零值就是0）

同时可以使用数组字面量根据一组值来初始化一个数组（`b:=[3]int{1,2,3}`）。

如果省略号`...`代替数组长度，那么数组长度由初始化数组的元素个数决定。（`c:=[...]int{1,2,3}` 长度为3）

**数组长度是数组类型的一部分。**

那么`[3]int`和`[4]int`是两个不同的数组类型。**数组长度必须是常量表达式。**

同时组值可以具有索引和索引对应的值`d:=[...]int{9:-1}`,定义了一个拥有10个元素的数组d，最后一个元素值为-1，其余都是0值。

当函数参数传入的是一个数组时候，传入的参数都会创建一个副本，然后赋值给对应的函数变量。Go中**把数组看作是值传递**，而在其他语言当中，数组是**隐式的引用传递。**

```go
func test1() {
	arrayA := [2]int{100, 200}
	var arrayB [2]int

	arrayB = arrayA

	fmt.Printf("arrayA : %p , %v\n", &arrayA, arrayA)
	fmt.Printf("arrayB : %p , %v\n", &arrayB, arrayB)

	testArray(arrayA)
}

func testArray(x [2]int) {
	fmt.Printf("func Array : %p , %v\n", &x, x)
}
```

![image-20211024194617478](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20211024194617.png)

可以看到，三个内存地址都不同，这也就验证了 Go 中数组赋值和函数传参都是值复制的。

> ❓：那么这会导致什么问题？

假想每次传参都用数组，那么每次数组都要被复制一遍。如果数组大小有 100万，在64位机器上就需要花费大约 800W 字节，即 8MB 内存。这样会消耗掉大量的内存。

> ❗：解决方案之一：

使用数组指针，同时也允许被调函数修改调用方数组中的元素。但是也有缺点：数组本身是不可变的，无法为数组添加和删除元素。

> 💡：由于数组长度不可变的特性等原因，除在特殊情况下，很少使用数组，一般使用`Slice`

## 切片

`slice` 表示一个拥有相同类型元素的可边长度的序列。

用切片解决上面那个问题，那么切片的优势也会表现出来。

**用切片传数组参数，既可以达到节约内存的目的，也可以达到合理处理好共享内存的问题。切片是引用传递，所以它们不需要使用额外的内存并且比使用数组更有效率。**

`Slice`由三个元素组成

* `data` :元素存哪里
* `len` :存了多少个元素
* `cap` :可以存多少个元素

```go
package main

import "fmt"

func main(){
	var egg []int
	//var egg []int=make([]int,2,5)
	//egg=append(egg,1)
	fmt.Printf("addres:%p,first item address:%p len:%v cap:%v data:%v",&egg,egg,len(egg),cap(egg),egg)
}

package main

import "fmt"

func main(){
	//var egg []int
	var egg []int=make([]int,2,5)
	//egg=append(egg,1)
	fmt.Printf("addres:%p,first item address:%p len:%v cap:%v data:%v",&egg,egg,len(egg),cap(egg),egg)
}
```

当定义一个`Slice` ,就会构造一个如下的一个结构

切片结构的地址为0xc000004078

* data的地址则为0x0 没有分配底层数组，这里就为nil

* len为0

* cap为0

![image-20211025220403035](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20211025220403.png)

如果通过make的方式去定义一个Slice，就会定义如下情况

![image-20211025220435472](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20211025220435.png)

当你使用make去定义这个变量，会分配三个结构，还会开辟一段内存作为他的底层数组。还会初始化为int类型的零值。

目前的Slice只存储了两个元素，此时的切片结构

* `data`应该指向开辟数组的首地址
* `len`为2
* `cap`为5

```go
package main

import "fmt"

func main(){
	//var egg []int
	var egg []int=make([]int,2,5)
	egg=append(egg,1)
	fmt.Printf("addres:%p,first item address:%p len:%v cap:%v data:%v",&egg,egg,len(egg),cap(egg),egg)
}
```

当我们添加一个元素时候，会将底层数组第三位改成3，len改成3

![image-20211026141230400](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20211026141237.png)

```go
package main

import "fmt"

func main(){
	//var egg []int
	var egg []int=make([]int,2,5)
	egg=append(egg,1)
	egg[0]=1
	fmt.Printf("addres:%p,first item address:%p len:%v cap:%v data:%v",&egg,egg,len(egg),cap(egg),egg)
}
```

当我们修改一个元素值时候，地址等不会发生改变，只有值发生了变化。

![image-20211026141521651](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20211026141521.png)

已经存储的是可以进行安全读写的。

> ❗：如果超出len的范围访问元素，属于越界访问，会发生panic
>
> ![image-20211026141749149](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20211026141749.png)

同时可以通过append去为未开辟底层数组的切片结构开辟一个底层数组

```go
package main

import "fmt"

func main(){
	//var egg []int
	var egg []int
	egg=append(egg,1)
	fmt.Printf("addres:%p,first item address:%p len:%v cap:%v data:%v",&egg,egg,len(egg),cap(egg),egg)
}
```

![image-20211026142501669](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20211026142501.png)

### 底层数组

前面所提到的：**数组是一个具有固定长度且拥有0个或者多个相同数据类型元素的序列。**

定义`int`类型的`slice`，那么底层数组对应的`int`类型

定义`string`类型的`slice`，那么底层数组对应的`string`类型

> 💡：切片结构的data不一定指向底层数组的首地址，指向开始元素的地址。

```go
func main(){
	egg:=[10]int{0,1,2,3,4,5,6,7,8,9}

	ans1:=egg[1:4]
	ans2:=egg[7:]

	fmt.Printf("addres:%p,len:%v cap:%v data:%v \n",&egg,len(egg),cap(egg),egg)
	fmt.Printf("addres:%p,first item address:%p,egg item address:%p len:%v cap:%v data:%v \n",&ans1,ans1,&egg[1],len(ans1),cap(ans1),ans1)
	fmt.Printf("addres:%p,first item address:%p,egg item address:%p len:%v cap:%v data:%v\n",&ans2,ans2,&egg[7],len(ans2),cap(ans2),ans2)
}
```

![image-20211026144659145](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20211026144659.png)

**不同slice与声明的数组可以共用一个底层数组**

`ans1`结构如下：

* `data`指向数组中索引为1的元素地址
* `len`为3
* `cap`为9

`ans1`的元素是`egg`索引1到4，左闭右开的区间，容量是从索引1到底层数组结束，所以为9

`ans2`结构如下：

* `data`指向数组中索引为7的元素地址
* `len`为3
* `cap`为3

`ans2`的元素是`egg`索引7到数组结束，容量是从索引7到底层数组结束，所以为3

 `ans1`的可读写范围是底层数组的索引1到3，如果想扩大读写范围，可以**利用append或者改变slice的范围**

> ❓：那么如果ans2使用append去添加元素，会发生什么？

```go
func main(){
	egg:=[10]int{0,1,2,3,4,5,6,7,8,9}

	ans1:=egg[1:4]
	ans2:=egg[7:]

	ans2=append(ans2,1)
	fmt.Printf("addres:%p,len:%v cap:%v data:%v \n",&egg,len(egg),cap(egg),egg)
	fmt.Printf("addres:%p,first item address:%p,egg item address:%p len:%v cap:%v data:%v \n",&ans1,ans1,&egg[1],len(ans1),cap(ans1),ans1)
	fmt.Printf("addres:%p,first item address:%p,egg item address:%p len:%v cap:%v data:%v\n",&ans2,ans2,&egg[7],len(ans2),cap(ans2),ans2)
}
```

![image-20211026150032973](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20211026150033.png)

因为之前ans1，ans2与egg共用一个底层数组，数组长度是不可改变的。

改变ans2的可读写范围

* 改变ans2的slice范围，但是如果值大于了10就会发生panic。
* 通过append，会发生重新开辟底层数组，将值拷贝然后添加值。

如上图所示。**当使用append为ans2添加元素，ans2会新开辟一个底层数组，将之前的数组元素进行拷贝然后修改底层数组的值。**

> ❓：我们只给ans2添加一个元素，容量从3变成了6呢？

### Slice扩容规则

> 1️⃣:STEP1 预估扩容后的容量

预估规则：

* 如果扩容前容量翻倍小于所需容量（oldCap*2<cap），那么新容量直接等于所需容量（newcap=cap）

* 当原slice的cap（oldCap<1024）小于1024时，新slice的cap变为原来的2倍；

* 原slice的cap大于1024（oldCap>1024）时，新slice变为原来的1.25倍

证明：

```go
//规则1
func main() {
	var ans []int
	ans=append(ans,1)
	fmt.Printf("cap:%d\n",cap(ans))
	ans=append(ans,1,2,3,4,5)
	fmt.Printf("cap:%d\n",cap(ans))
}
//规则2
func main() {
  slice := make([]int, 0)
  oldCap := cap(slice)
  for i := 0; i < 4096; i++ {
    slice = append(slice, i)
    newCap := cap(slice)
    if newCap != oldCap {
      fmt.Printf("oldCap = %-4d  after append %-4d  newCap = %-4d\n", oldCap, i, newCap)
      oldCap = newCap
    }
  }
}
```

![image-20211026153502610](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20211026153502.png)

![image-20211026152707948](https://cdn.jsdelivr.net/gh/baici1/image-host/newimg/20211026152707.png)

> 2️⃣:STEP2 计算需要的内存大小

预估容量*元素类型大小

> 3️⃣：STEP3 匹配合适的内存规格

例子：

newcap=5，定义是int类型

需要内存大小为5*8=40，会匹配到48的内存大小。

48的内存大小可以存6个元素。

## 参考文章

[【Golang源码系列】二：Slice实现原理分析](https://mp.weixin.qq.com/s/rbilTQKZ6WwlsQvyVA-qVA)

[深入解析 Go 中 Slice 底层实现](https://halfrost.com/go_slice/#toc-0)

