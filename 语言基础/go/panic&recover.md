# panic & recover

> 推荐阅读：[【Golang】图解panic & recover](https://mp.weixin.qq.com/s/vcJ6TsnknaCoYhH6XZnNMw)

## 前言

之前在 `defer` 的解析知道，当前执行的 `goroutine` 持有一个 `defer` 链表的头指针。其实他也有一个 `panic` 头指针。

![图片](https://mmbiz.qpic.cn/mmbiz_png/ibjI8pEWI9L6DI134xDP4GZnDlEvP9ibyRU6TwxQoNiaPRmJbdef2OfTLyiamP217QKkbUzaqju5yd4mJbT5WnlBRA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

接下来看看 `panic` 是怎么处理的。

## panic

```go
func A(){
	defer A1()
	defer A2()
	panic("panicA")
	fmt.Println("这里不会被执行")
}
func A2(){
	fmt.Println("A2正常结束")
}
func A1(){
	fmt.Println("A1正常结束")
}

func main()  {
	A()
}
```

函数 `A` 注册了两个 `defer` 函数 `A1` 和 `A2`，然后发生了 `panic`。发生 `panic` 后，它会立刻停止执行当前函数剩余代码，而是进入了 `panic` 处理逻辑函数。

首先会在 `panic` 链表头处增加一项 `panicA`，是现在的执行的 `panicA`，然后在当前的 `goroutine` 递归执行注册的 `defer` 函数。这里有点不一样的地方。

```go
//defer 1.12
type _defer struct { 
    siz       int32
    started   bool    // panic执行defer时会把它标记为true
    sp        uintptr 
    pc        uintptr
    fn        *funcval
    _panic    *_panic // 记录触发defer执行的_panic指针
    link      *_defer
}
```

`panic` 执行到一个 `defer` 会先将他的 `_defer` 结构体的字段 `started` 设为 `true`，代表他已经开始执行了，并将字段 `_panic` 指向当前执行的 `panic`，表示当前执行的 `defer` 函数由这个 `panic` 函数触发的。

处理完字段赋值，函数 `A2` 就会正常执行与结束，然后就会移除这一项，继续执行下一个 `defer`。

之所以要等到 `defer` 函数正常返回以后再移除对应的 `defer` 链表项，主要是为了应对 `defer` 函数没有正常结束的情况。

## defer 非正常结束

我们再来看个例子：

```go
func A(){
    defer A1()
    panic("panicA")//打印panic 信息
}   
func A1(){
    fmt.Println("A1再次panic")
    panic("panicA1")//打印panic信息
}
```

在函数 A 中先注册 `defer` 函数 `A1`，然后执行到 `panic`，`panic` 链表就会增加一项 `panicA`。之后就会执行 `defer` 链表的函数。`A1` 的 `_defer` 结构体会将 `started` 置为 `true` 和 `_panic`  指向当前执行的 `panic`。

![图片](https://mmbiz.qpic.cn/mmbiz_png/ibjI8pEWI9L6DI134xDP4GZnDlEvP9ibyRtdPd2MX0V8aRqWoGic7dncC5icSCYcZq8SMB4tleRhNnI5dIdKvF9Tcw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

A1 开始执行，再次发生了 `panic`，当前函数立刻停止执行剩下的代码，在 `panic` 链表插入了一个新的 `_panic`，记为 `panicA1`。

现在要开始去执行 `defer` 链表的函数了，但是发现 `defer` 链表的函数 `A1`，触发执行的不是当前 `panicA1`，是之前的 `panicA`。

这里会根据 A1 的字段 `_panic` 找到之前触发执行的 `panicA` 函数，将它标记为终止。

```go
type _panic struct {
    argp      unsafe.Pointer //用来存储panic正在执行的defer函数的参数空间地址;
    arg       interface{} //是panic函数自己的参数;
    link      *_panic //自然是链到上一个_panic结构体；
    recovered bool //标识这个panic是否被恢复；
    aborted   bool //标识这个panic是否被终止。
}
```

`panicA` 的结构体字段 `aborted` 置为 `true`。并移除 `defer` 链表的 `A1` 函数。

此时链表已经为空了，那么对于当前执行的 `panicA1` 函数，没有需要去递归执行的 `defer` 函数了，就要开始打印信息了。

`panic` 打印信息**从链表尾开始**，也就是根据链表项插入顺序逐一输出。在这里就会先输出 `panicA` 的信息然后输出 `panicA1` 的信息，程序结束。

关键点：

* `panic` 执行 `defer` 函数方式是：先标记，执行结束后移除，如果触发指针不匹配则终止之前工作的 `panic`。
* `panic` 打印异常信息：顺序从链表尾输出即根据注册顺序来输出信息，所有在 `panic` 链表的都会被输出。

## recover

`recover` 可以中止 `panic` 造成的程序崩溃，但是它只能在 `defer` 中发挥左右，在其他作用域中调用不会起作用。

```go
func A(){
	defer A1()
	defer A2()
	panic("panicA")
	fmt.Println("这里不会被执行")
}
func A2(){
	p := recover()
	fmt.Println(p) //这里会正常执行输出“panicA”
}
func A1(){
	fmt.Println("A1正常结束")
}

func main()  {
	A()
}
```

加了 `recover` 好像就可以正常执行程序了，那么这个流程又会是怎么样的呢？

函数 `A` 会在当前执行的 `goroutine` 注册 `defer` 函数 `A1` 和 `A2`。然后在 `panic` 链表增加一项 `panicA`。接着就会递归执行 `defer` 链表的函数，先执行 `A2`。

![图片](https://mmbiz.qpic.cn/mmbiz_png/ibjI8pEWI9L6DI134xDP4GZnDlEvP9ibyRDWX7xAPQ7IHRxtcwFicueWdDibICLNicNO6ibMwFYerFtCCOmYjibOMCO7A/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

当执行 `A2` 函数时候，会发生 `recover`，其实 `recover` 函数的做的事情很少，就是将当前执行的 `panic` 字段 `recovered` 置为 `true`。`A2` 函数接着继续执行，直到结束。

每次 `defer` 函数执行完后，在 `panic` 处理流程都会去检查当前执行的 `panic` 是否被恢复了，如果被恢复了，那么就会移除。

`A2` 执行完后，发现当前执行的 `panic` 已经被恢复了，那么把它从 `panic` 链表中移除，同时 `A2` 执行完后，也会从 `defer` 函数移除。但是在 `A2` 移除前，会要保存 `_defer` 结构体的 `sp` 和 `pc` 两个字段的值。

**为什么要保存这两个字段啊？有什么用吗？当前的 panic 函数还没有结束哦！**

先说明这两个字段的作用：

* `sp`：函数 `A` 的栈指针。
* `pc`：调用 `deferproc` 函数的返回地址。

![图片](https://mmbiz.qpic.cn/mmbiz_png/ibjI8pEWI9L6DI134xDP4GZnDlEvP9ibyRO7c2rZ7OLeVZ3yiaJLK1upl2w42abQpqbnWcyYBFIa7qqOXMur21XeA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



根据这段伪指令，我们直到利用 `sp` 字段能够回到函数 `A` 的栈帧，利用 `pc` 字段可以通过返回地址的值 `r` 通过判断回到 `deferreturn` 这里然后继续执行 `defer` 链表的函数。

**注：函数 `A` 这里的 `deferreturn` 只负责执行函数 `A` 中注册的 `defer` 函数，也是通过 `sp` 字段去判断的。**

兜兜转转又回到了 `defer` 链表，下一个执行的函数是 `A1`，是函数 `A` 注册的。执行函数 `A1`，结束后 `defer` 链表为空，函数 `A` 结束了。

关键点：

* `recover` 做的事情很少，就是将当前的 `panic` 结构体字段 `recovered` 置为 `true` ，代表此 `panic` 被恢复了。
* `panic` 恢复后，会被移除然后通过 `sp` 和 `pc` 字段通过判断恢复到 `defer` 链表，执行的函数都是同一个函数注册的 `defer` 函数
* 在发生 `recover` 的函数正常返回以后，才会检测当前 `panic` 是否被恢复，然后才会删除被恢复的 `panic`。

## recoder 非正常结束

如果 `recover` 不是正常返回结果，中间又有一个 `panic`，那么这过程又会发生什么呢？

```go
func A(){
	defer A1()
	defer A2()
	panic("panicA")
	fmt.Println("这里不会被执行")//输出异常信息
}
func A2(){
	p := recover()
	fmt.Println(p) //这里会正常执行输出“panicA”
	panic("panicA2") //输出异常信息
}
func A1(){
	fmt.Println("A1正常结束")
}

func main()  {
	A()
}
```

之前的过程省略，直接来到关键处。

当 `panicA` 被恢复时候，`A2` 函数会继续接着执行（注：只有正常返回才会移除 `panicA`），再次发生了 `panic`，在 `panic` 链表增加了一项 `panicA2`。现在他是当前执行的 `panic` 函数了。

在执行 `panicA2` 流程会触发 `defer` 链表的执行，发现 `defer` 函数 `A2` 已经被执行了，触发者是之前的 `panicA`。那么就会终止 `panicA`，`A2` 也会从 `defer` 链表中移除。

![图片](https://mmbiz.qpic.cn/mmbiz_png/ibjI8pEWI9L6DI134xDP4GZnDlEvP9ibyRvk1lEWnXhWcda1uQDwm8hgGoA1lT6waLzR4SN0yeQFSU7NqicibYuJZw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

咦！好像 `panicA` 没有被移除，即使他现在被恢复了。它会被移除吗？接着往下看。

`panicA2` 函数会继续去执行 `defer` 链表接下来的函数 `A1`。`A1` 的 `_defer`结构体中 `_panic` 会指向 `panicA2`。

`A1` 结束了，那么就要开始输出异常信息了。

输出异常信息，会将 `panic` 链表的所有项都会输出出来，不同的是 `panicA` 会被打上 `recover` 标记。

```bash
panic: panicA [recovered]
	panic: panicA2
```

## recover 限制

recover 函数只能在 defer 函数中直接调用，也不能间接调用，如果不满足这个要求，那么 recover 不会有任何效果。

![图片](https://mmbiz.qpic.cn/mmbiz_png/ibjI8pEWI9L6DI134xDP4GZnDlEvP9ibyRlYHtt9ZWnUSczRsmxMtR7uaA3aFn0LZEaCQUVIbyvLuYEMXuJL4GBQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## 后续

我们知道在 `1.14` 版本后 `defer` 的执行改成了 `open coded defer` 的方式，既然 `panic` 需要调用 `defer` 链表，所以那些没有注册到 `defer` 链表会通过栈扫描注册到 `defer` 链表中。剩余的过程是很繁琐的，都是为了迎合 `open coded defer`。但是 `panic` 和 `recover` 的总体设计思路不变。
