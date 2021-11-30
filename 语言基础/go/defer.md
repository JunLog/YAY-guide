# defer

> 推荐阅读：[【Golang】脱胎换骨的defer](https://mp.weixin.qq.com/s/gaC2gmFhJezH-9-uxpz07w)

## 如何延迟，因何倒序

当你开发时候，我们都会去用 `defer` 去关闭一个打开的文件，释放 `Mysql/Redis` 连接，或者解锁一个 `Mutex`。`Go` 语言的 `defer` 是一个很方便的机制，能够把某些函数调用推迟到当前函数返回前才实际执行。而且`Go` 语言在设计上保证，即使发生 `panic` ，所有的 `defer` 调用也能够被执行。不过多个 `defer` 函数是按照定义顺序倒序执行的。

```go
func f1()  {
	defer A()
	fmt.Println(1)
}
func A()  {
	fmt.Println(2)
}
func main()  {
	fmt.Println(0)
	f1()
}
/*
0
1
2
*/
```

我们先来看看为什么 `defer` 会延时执行。

```go
func f1() {
    defer A()
   	fmt.Println(1)
}
```

像上面的代码，在 `Go1.12` 编译后的伪指令是这样的：

```go
 func f1() {
    r := runtime.deferproc(0, A) // 经过recover返回时r为1，否则为0
    if r > 0 {
        goto ret
    }
    fmt.Println(1)
    runtime.deferreturn()//调用
    return
ret:
    runtime.deferreturn()
}
```

> 如何延迟

我们可以发现与 `defer` 指令相关的有两个部分。

第一部分是 `deferproc`，他负责保存要执行的函数信息，我们称之为defer**“注册”**。

```go
func deferproc(siz int32, fn *funcval)
```

`deferproc` 有两个参数，第一个参数是被注册的 `defer` 函数的参数加返回值占多少字节。这里没有参数与返回值所以为 `0`。第二个参数是一个 `runtime.funcval` 结构体的指针，指向函数入口。

第二部分是 `deferreturn`，它被编译器被插入到了函数返回前进行调用。负责执行已经注册的 `defer` 函数。这也是为什么 `defer` 函数能够延迟执行的原因。先注册后调用。

> 因何倒序

`defer` 注册信息会保存到 `defer` 链表。主函数运行会自动启一个 `goroutine`，在运行时都对应一个 `runtime.g` 结构体，其中有一个 `_defer` 字段，保存的就是 `defer` 链表的头指针。

`deferproc` 新注册的 `defer` 函数信息会添加到链表头部。`deferreturn` 执行时候也从链表头开始，所以`defer` 函数才会表现为倒序执行。如下图所示：

![图片](https://mmbiz.qpic.cn/mmbiz_png/ibjI8pEWI9L6iaibdGLYroCCtK18JQJIyoLvsljiadJa49Uwcn8u4yeePTibmr50ZTlM0PZUyN1TsTiaxWCGVmv0rvNw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



## defer 信息

defer 连接的是 _defer 结构体。

```go
type _defer struct {
    siz       int32 //deferproc第一个参数传入，就是defer函数参数加返回值的总大小。
    started   bool //标识defer函数是否已经开始执行；
    sp        uintptr // sp at time of defer 就是注册defer函数的函数栈指针；
    pc        uintptr //是deferproc函数返回后要继续执行的指令地址；
    fn        *funcval //由deferproc的第二个参数传入，也就是被注册的defer函数；
    _panic    *_panic // panic that is running defer 是触发defer函数执行的panic指针，正常流程执行defer时它就是nil；
    link      *_defer //自然是链到之前注册的那个_defer结构体。
 }
//参数和返回值这段空间会直接分配在_defer结构体后面，用于在注册时保存给defer函数传入的参数，并在执行时直接拷贝到defer函数的调用者栈上。
```

## 传参机制

```go
func A1(a int) {
	fmt.Println(a)//1
}
func A() {
	a, b := 1, 2
	defer A1(a)

	a = a + b
	fmt.Println(a, b) //3,2
}
func main()  {
	A()
}
```

我们从函数 `A` 开始分析。

在 `A` 的函数栈中，局部变量存储了 `a=1,b=2`。开始注册 `defer` 函数，`deferproc` 函数会输入两个参数

* 所以参数空间里面第一个参数是 `A1`的参数加返回值共占多少字节的变量，这里 `A1` 没有返回值，有参数，`64` 位下的一个 `int` 类型参数占用 `8` 个字节。
* 第二个参数是函数 `A1`，前面我们介绍过，没有捕获列表的 `Function Value`，在编译阶段会做出优化，就是在只读数据段分配一个共用的 `funcval` 结构体。所以函数 `A1` 的指令入口地址为 `addr1`。在只读数据段分配的指向 `A1` 指令入口的 `funcval` 结构体地址为 `addr2`，所以 `deferproc` 函数第二个参数就是 `addr2`。

![图片](https://mmbiz.qpic.cn/mmbiz_png/ibjI8pEWI9L6iaibdGLYroCCtK18JQJIyoLTKDcibLx5NbWVBibPHlYMDElF9lpib4TzYQ63q6HU4eiadTyr7861JmvGQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

那么 A1 函数的参数 a 放哪儿呢？

上图空缺的地方就是放参数与返回值的。编译器会在 `deferproc` 函数的两个参数后面开辟一段空间，用于存储 `defer` 函数 `A1` 的返回值和参数。同时一段空间也会通过值拷贝到 `_defer` 结构体后面。所以这里空缺的部分填写 `a=1`。

![图片](https://mmbiz.qpic.cn/mmbiz_png/ibjI8pEWI9L6iaibdGLYroCCtK18JQJIyoLnVGiajuhV5yQpjLzS83xrCkmHK70d9D97DchK4JGoltSibrht2Jz8I3Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

当 `deferproc` 函数执行时候，需要堆分配一段空间，用于存放 `_defer` 结构体以及 `defer` 函数传递的参数与返回值。现在看看这个堆空间的构成。

```go
type _defer struct {
    siz       int32 //deferproc第一个参数传入，就是defer函数参数加返回值的总大小。 //8
    started   bool //标识defer函数是否已经开始执行； //false
    sp        uintptr // sp at time of defer 就是注册defer函数的函数栈指针； //调用栈 A 的栈指针
    pc        uintptr //是deferproc函数返回后要继续执行的指令地址； //deferproc函数的返回地址return addr；
    fn        *funcval //由deferproc的第二个参数传入，也就是被注册的defer函数；//A1
    _panic    *_panic // panic that is running defer 是触发defer函数执行的panic指针，正常流程执行defer时它就是nil； //nil
    link      *_defer //自然是链到之前注册的那个_defer结构体。 //nil
 }
//在结构体后面还有 8 个字节用于保存传递给 A1 的参数 a
```

然后这个 `_defer` 结构体就会被添加到 defer 链表头，此时 `deferproc` 注册结束。

> *频繁的堆分配势必影响性能，所以Go语言会预分配不同规格的deferpool，执行时从空闲_defer中取一个出来用。没有空闲的或者没有大小合适的，再进行堆分配。用完以后，再放回空闲_defer池。这样可以避免频繁的堆分配与回收。*

`defer` 函数的注册已经完成，那么就要开始执行 `code` 相关的代码了。执行 `a = a + b` 这里的局部变量 `a=3`，随后就会输出 `a=3,b=2`。

就要到 `deferreturn` 执行 `defer` 链表了，从当前的 `goroutine` 找到链表头的 `_defer` 结构体，通过 `_defer.fn` 找到 `defer` 函数的 `funcval` 结构体，进而拿到 `A1` 函数的入口地址，接下来就是调用 `A1`  函数了。

`_defer` 结构体后面的参数与返回值会拷贝到 `A1`的函数调用栈中，`A1` 开始执行，就会把之前的参数 a=1，输出来。整个过程就在这里结束了。`defer` 函数的参数会在注册时候拷贝到堆上，执行时候再拷贝到函数调用栈上。

## defer + 闭包

如果此时 `deferproc` 注册是有捕获列表的 `Function Value`，又会发生什么呢？

```go
func A() {
	a, b := 1, 2
	defer func(b int) {
		a = a+b
		fmt.Println(a, b)//5,2
	}(b)
	a = a + b
	fmt.Println(a, b)//3,2
}
func main()  {
	A()
}
```

这个例子中，`defer` 函数传递了局部变量 `b` 的参数，还捕获了外层函数的局部变量 `a`，形成了闭包。我们来看看这到底会发生什么？

这里捕获变量 `a` 在闭包中被修改过，所以这里函数调用栈的局部变量 `a` 会改为堆分配，只会存储他的地址。接下来就会创建闭包对象，在堆上分配一个 `funcval` 结构体，`funcval.fn` 指向闭包函数入口 `addr1`。接下来是堆上的 _defer 结构体的构成。

```go
type _defer struct {
    siz       int32 //deferproc第一个参数传入，就是defer函数参数加返回值的总大小。 //8
    started   bool //标识defer函数是否已经开始执行； //false
    sp        uintptr // sp at time of defer 就是注册defer函数的函数栈指针； //调用栈 A 的栈指针
    pc        uintptr //是deferproc函数返回后要继续执行的指令地址； //deferproc函数的返回地址return addr；
    fn        *funcval //由deferproc的第二个参数传入，也就是被注册的defer函数；//闭包函数的入口
    _panic    *_panic // panic that is running defer 是触发defer函数执行的panic指针，正常流程执行defer时它就是nil； //nil
    link      *_defer //自然是链到之前注册的那个_defer结构体。 //nil
 }
//在结构体后面还有 8 个字节用于保存传递给 闭包函数 的参数 b
```

`_defer` 结构体会被添加到 defer 链表头，`deferproc` 注册结束，开始执行 `code` 的代码。

执行 `a = a + b`，局部变量 `a=3,b=2` ，然后输出。

接下来是 `deferreturn` 执行注册的 `defer` 函数，同时将参数 `b` 拷贝到执行函数的栈上。闭包函数通过寄存器存储的 `funcval` 地址加上偏移找到捕获变量 `a`。此时 `a=3`，所以执行 `defer` 函数后，捕获变量 `a=5，`，参数 `b=2`。

## 奇怪的 defer 函数

```go
func B(a int) int {
	a++
	return a
}
func A(a int) {
	a++
	fmt.Println(a)//3
}
func main() {
	a := 1
	defer A(B(a))
	a++
	fmt.Println(a)//2
}
```

不知道你是否已经猜到答案了？

我们来分析一波：我们需要明确一点，在 `main` 函数里面注册的是函数 `A`，所以 `B(a)` 是作为参数，同时执行 `B(a)` 拿到参数值，拷贝到堆上。所以函数 `B` 在注册时候就执行了，返回值是 `2`。然后再去执行 `defer` 函数 `A` 就会输出 `a=3`。

## defer 嵌套

我们之前眼光聚焦于 `defer` 的细节，现在我们抛开细节，看看当发生 `defer` 嵌套时候，`defer` 函数又会怎么样注册与执行。

```go
func A(){
    //......
    defer A1()
    //......
    defer A2()
    //......
}
func A2(){
    //......
    defer B1()
    defer B2()
    //......
}
func A1(){
    //......
}
//所有defer函数都正常执行....
```

函数 `A` 分别注册了两个 `defer` 函数 `A1和A2`。因为此阶段是注册阶段，所以并不会去执行函数。

![图片](https://mmbiz.qpic.cn/mmbiz_png/ibjI8pEWI9L6iaibdGLYroCCtK18JQJIyoLiaIufmKESlvBcXok7QzHA5o9Omz0HQ93od3eOa7bibn3f5fvKtLXLKpA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

执行 `deferreturn`，会先判断 `defer` 链表上的 `defer` 是不是函数 `A` 注册的。判断的方法时通过判断 `_defer` 结构体记录的 `sp` 是否等于 `A` 的栈指针。`A2` 是函数 `A` 注册的，保存 `defer` 函数调用的相关信息，然后这一项从 `defer` 链表中移除，当 `A2` 执行时候，又会注册两个 `defer` 函数 `B1和B2`。

![图片](https://mmbiz.qpic.cn/mmbiz_png/ibjI8pEWI9L6iaibdGLYroCCtK18JQJIyoLGzXxICrqXdV3JZ6lzWUx5bRBXtr8zAzeY7pbZjXTBBIMuhSUibFpMoA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



`A2` 函数结束前会执行 `defer` 链表，同时也会去判断是否是自己注册的链表。`B2` 执行，然后 `B1` 执行，其实此时 `A2` 不知道自己注册的链表是否已经执行完了。那么什么时候会判断 `A2` 结束了。直到出现 下一个 `_defer.sp` 不等于 `A2` 的栈指针即遇到不属于自己注册的 `defer` 函数代表 `A2` 注册的 `defer` 执行完了。

这里又判断到 `A1` 是函数 `A` 注册的 `defer` 函数，又会回到 `A` 的`defer` 执行流程。`A1` 执行完后，链表为空，此时函数 `A` 结束。

> *理解了defer注册与执行的逻辑，再配合之前介绍过的Function Value、函数调用栈等内容，就很容易理解上面几个例子，也就不用刷那些重复的defer面试题了*

## 似乎 defer1.12 有点慢

因为 `defer` 真的很方便，所以大家都已经习惯了随手使用它。但是与一般的函数调用比起来，`defer1.12` 的实现方式会在调用时造成较大的额外开销，尤其是在锁释放这种场景。因此经常被一些库设计者所诟病，甚至有些项目的注释中写明了不用 `defer` 能节省多少多少纳秒。

`defer1.12` 的性能问题主要缘于两个方面：

1. `_defer` 结构体堆分配，即使有预分配的 `deferpool`，也需要去堆上获取与释放。而且 `defer` 函数的参数还要**在注册时从栈拷贝到堆，执行时又要从堆拷贝到栈。**这样左右拷贝，又怎会不慢呢？
2.  `defer` 信息保存到链表，而**链表操作比较慢。**

但是，`defer` 作为一个关键的语言特性，怎能如此受人诟病？所以 `GO` 语言在 1.13 和1.14 中做出了不同的优化。

## defer1.13

我们来看看 `defer1.13` 到底升级了啥？

`Go1.13` 中 `defer` 性能的优化点，主要集中在减少 `defer` 结构体堆分配。

```go
func A1(a int) {
	fmt.Println(a)//1
}
func A() {
	a, b := 1, 2
	defer A1(a)

	a = a + b
	fmt.Println(a, b) //3,2
}
func main()  {
	A()
}
```

`defer1.13` 编译后的伪指令是这样的：

```go
func A() {
    var d struct {
        runtime._defer
        i int
    }
    d.siz = 0
    d.fn = A1
    d.i = 10
    r := runtime.deferprocStack(&d._defer)
    if r > 0 {
        goto ret
    }
    // code to do something
    a = a + b
	fmt.Println(a, b) //3,2
    
    runtime.deferreturn()
    return
ret:
    runtime.deferreturn()
}
```

你会发现好像和 `defer 1.12` 做了很大的改变。

| 结构                           | defer1.12                                                    | defer1.13                                                    |
| ------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| _defer 的位置                  | 当执行 deferproc 函数时候会在堆上开辟一段空间专门存储 _defer 结构体 | 会在函数调用栈直接分配空间存储 defer 结构体相关字段。        |
| 参数                           | 会存储在 _defer 结构体后面的空间，使用时候需要从堆上拷贝到函数调用栈 | 参数则存储在函数的局部变量空间。使用时候从栈上拷贝变量到参数空间 |
| _defer 结构体添加到 defer 链表 | 当堆上存储 _defer 结构体后，就会将结构体分配添加到链表头上。 | runtime.deferprocStack 则会把栈上分配的_defer 结构体注册到 defer 链表 |

从上面伪指令，你会发现多了一个结构体 d，它由两部分组成分别是 `runtime._defer` 结构体，传给 `defer` 函数 `A1` 的参数。

值得注意的是，1.13 版本并不是所有的 defer 都能够在栈上进行分配空间。循环中的 `defer`，无论是显示的 `for` 循环，还是 `goto` 形成的隐式循环，都只能使用 `1.12` 版本中的处理方式在**堆上分配**。即使只执行一次的 `for` 循环也是一样。

```go
//显示循环
for i:=0; i< n; i++{
    defer B(i)
}
......

//隐式循环
again:
    defer B()
    if i<n {
        n++
        goto again
    }
```

所以为了区分 `_defer` 结构体存储的位置，在 `defer1.13` 中，`runtime._defer` 结构体增加了一个字段 **heap**，用于标识是否为堆分配。

```go
type _defer struct {
    siz       int32 //deferproc第一个参数传入，就是defer函数参数加返回值的总大小。
    started   bool //标识defer函数是否已经开始执行；
    heap      bool       //标识是否为堆分配
    sp        uintptr // sp at time of defer 就是注册defer函数的函数栈指针；
    pc        uintptr //是deferproc函数返回后要继续执行的指令地址；
    fn        *funcval //由deferproc的第二个参数传入，也就是被注册的defer函数；
    _panic    *_panic // panic that is running defer 是触发defer函数执行的panic指针，正常流程执行defer时它就是nil；
    link      *_defer //自然是链到之前注册的那个_defer结构体。
 }
//参数和返回值这段空间会直接分配在_defer结构体后面，用于在注册时保存给defer函数传入的参数，并在执行时直接拷贝到defer函数的调用者栈上。
```

`defer` 函数执行在 `1.13` 中没有变化，依旧通过 `deferreturn` 实现，这一次的 `_defer` 结构体后面的参数和返回值空间，不是从堆拷贝到栈上，而是从栈上的局部变量空间拷贝到参数空间，`defer` 函数通过相对寻址找到参数。

`1.13` 版本的 `defer` 主要通过减少了 `_defer` 结构体的堆分配，达到了性能优化在 `30%` 左右。好像只解决了第一个问题，仍然在使用 `defer` 链表。在 `1.14` 版本又会又怎么样的优化呢？

## defer1.14

我们举一个例子看看到底做了怎么样的优化呢？

```go
func A(i int) {
    defer A1(i, 2*i)
    if(i > 1){
        defer A2("Hello", "eggo")
    }
    // code to do something
    return
}
func A1(a,b int){
    //......
}
func A2(m,n string){
    //......
}
```

定义的伪指令会是怎么样的。(以 `defer` 函数 `A1` 为例)

```go
func A(i int){
    var a, b int = i, 2*i
    //......
        
    A1（a, b）
    return
    //......
}
```



| 结构                           | defer1.12                                                    | defer1.13                                                    | defer 1.14                                    |
| ------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | --------------------------------------------- |
| _defer 的位置                  | 当执行 deferproc 函数时候会在堆上开辟一段空间专门存储 _defer 结构体 | 会在函数调用栈直接分配空间存储 defer 结构体相关字段。        | 没有_defer 结构体了                           |
| 参数                           | 会存储在 _defer 结构体后面的空间，使用时候需要从堆上拷贝到函数调用栈 | 参数则存储在函数的局部变量空间。使用时候从栈上拷贝变量到参数空间 | 会提前定义并分配到函数调用栈局部变量上        |
| _defer 结构体添加到 defer 链表 | 当堆上存储 _defer 结构体后，就会将结构体分配添加到链表头上。 | runtime.deferprocStack 则会把栈上分配的_defer 结构体注册到 defer 链表 | 没有_defer 链表了                             |
| defer 函数执行                 | 利用 defer 链表找到 _defer 结构体然后找到函数入口地址 ，进行调用。 | 与 defer 1.12 一致                                           | 会将执行函数置于函数 A 前，当作普通函数调用。 |

通过 `defer1.14` 这样的方式有以下几个优点：

* 不用构建 `_defer` 结构体
* 用不到 `defer` 链表

但是好像这样的逻辑怎么用到 `defer` 函数 `A2` 呢？怎么判断我应不应该调用这个函数呢？

`defer1.14` 通过增加一个标识变量 `df` 来解决这个问题。`df` 变量每一位对应当前函数的一个 `defer` 函数是否执行。

函数 `A1` 需要被执行，那么 df|=1 方式将 `df` 第一位置为1，然后函数结束前通过判断 `df` 的第一位是否为 1，来决定是否执行。这样的逻辑对于函数 `A2` 也适用。

```go
func A(i int){
    var df byte
    //A1的参数
    var a, b int = i, 2*i
    df |= 1

    //A2的参数
    var m,n string = "Hello", "eggo"
    if i > 1 {
        df |= 2
    }
    //code to do something
        
    //判断A2是否要调用
    if df&2 > 0 {
        df = df&^2
        A2(m, n)
    }
    //判断A1是否要调用
    if df&1 > 0 {
        df = df&^1
        A1(a, b)
    }
    return
    //省略部分与recover相关的逻辑
}
```

`Go1.14` 把 `defer` 函数在当前函数内展开并直接调用，这种方式被称为 open coded defer。这种方式不仅不用创建 `_defer` 结构体，也脱离了 `defer` 链表的束缚。不过这种方式依然不适用于循环中的 `defer`，所以 `1.12` 版本 `defer` 的处理方式是一直保留的。

## defer 提升后带来的后果

我们一直讨论的是程序正常执行 defer 的处理逻辑，那么如果程序不正常执行了呢，发生了 `panic` 或者使用了 `untime.Goexit` 函数，当前的正常程序反而会不执行，而是去执行 `defer` 链表，可是在 `1.14` 版本里面并没有注册链表啊。

`defer1.14` 版本的 `_defer` 结构体又增加了几个字段，使那些没有注册到链表的 `defer` 函数通过栈扫描来注册到链表里。

```go

type _defer struct {
    siz       int32
    started   bool
    heap      bool
    openDefer bool           //1
    sp        uintptr
    pc        uintptr
    fn        *funcval
    _panic    *_panic
    link      *_defer 
    fd        unsafe.Pointer //2
    varp      uintptr        //3
    framepc   uintptr        //4
}
```

可是这样会导致 `defer` 确实变快了，但是 `panic` 却变慢了。

我们后面看看 `panic` 又会有怎么样的优化呢？
