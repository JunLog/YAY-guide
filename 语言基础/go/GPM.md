# GPM

> 推荐学习：
>
> [【Golang】一个Hello World程序的执行](https://mp.weixin.qq.com/s?__biz=Mzg5NjIwNzIxNQ==&mid=2247484382&idx=1&sn=cbf22d781d90a2991c6b41d9f592c164&chksm=c005d3def7725ac8bf7555b5fb3533ed33808e84f4ec05926ead12cdf28ee2469c7d0dec8225&scene=21#wechat_redirect)
>
> [Golang并发编程-GPM调度过程源码分析](https://juejin.cn/post/6976839612241018888#heading-3)
>
> [Golang并发编程-GPM协程调度模型原理及结构分析](https://juejin.cn/post/6976538466863546382#heading-0)
>
> [[典藏版]Golang调度器GPM原理与调度全分析](https://zhuanlan.zhihu.com/p/323271088)

## hello world！

我们一直写 `Go` 程序，你是否考虑过，一个 `hello word！`他是如何运行起来的。我们来深挖一波！！

```go
package main

import "fmt"

func main()  {
	fmt.Println("hello world!")
}
```

这段程序编译后成为为一个可执行文件，执行时候可执行文件被加载到内存中。相信大家在学校里面学过编译原理，一段简单的汇编代码会分成多个段：数据段，代码段，堆栈段，子程序等等。

> 数据段

对于数据段，有很多重要的全局变量，我们一起来看看。

* 主协程：g0 与 主线程 m0

协程对应的数据结构是 `runtime.g`，工作线程对应的数据结构是 `runtime.m`。``

 `g0` 就是主协程对应的 `g`，与其他协程不同，他的协程栈实际上是在主线程栈上分配的。

`m0` 是主线程对应的 `m`。

`g0` 持有 `m0` 的指针，`m0` 也记录着 `g0` 的指针。一开始 `m0` 上执行的协程正是 `g0`。`m0` 与 `g0` 就联系起来了。

* allgs，allm，allp

`allgs` 记录着所有的 `g`。

`allm` 记录着所有的 `m`。

`allp` 记录着所有的 `p`。

* sched

最初 `Go` 语言的调度模型里面只有 `M` 和 `G`，所以会有一个 `G run queue` 里面都是待执行的 `G` ，每个 M 来到队列获取一个 `G` 时候都会加锁。多个 `M` 会分担多个 `G` 的任务，途中会因为前面的 M 频繁加锁和解锁而发生等待，影响程序的并发性能。

为了解决这个问题，又引入了一个 `P`，`P` 对应的数据结构是 `runtime.p`。他有一个 `runq [256]guintptr` ，通过把一个 `P` 关联到一个 `M`，这个 `M` 就可以从 `P` 的本地 `runq` 这里获取待执行的 `G`，不用每次都与其他的 `M` 在队列中争抢任务了，性能也会提升。

全局 `runq` 保存在 全局变量 `sched` 中，`sched` 代表是调度器，对应的数据结构是 `runtime.schedt`。这里记录着所有空闲的 `M`，空闲的 `P`，以及许多与调度相关的内容。

如果一个 `P` 的本地 `runq` 已满，那么等待执行的 `G` 会被放到这个全局 `runq` 中。`M` 执行 `G` 过程如下：

1. `M` 会先去执行对应的 `P` 本地 `runq` 中待执行的  `G`，
2. 如果没有的话，再到调度器这里全局 `runq` 领取任务。
3. 如果也没有了，就会从别的 `P` 那里分担一部分的 `G` 过来执行。

![image-20211204194241881](https://cdn.jsdelivr.net/gh/baici1/img-typora/20211204194241.png)

> 代码段

对于这段程序编译成的代码段，程序入口并不是 `main.main`，不同平台下程序开始进入入口不同。简单的流程如下：

* 进入后再进行一系列检查与初始化等准备工作后，
* 当 `main.goroutine`  创建后会被加入到当前 `P` 的本地队列中，
* 然后通过 `mstart` 函数开启调度循环，队列中只有 `main.goroutine` 正在等待执行，所以 `g0` 会切换成 `main.goroutine`，
* 执行入口就是 `runtime.main`，他会做一些准备工作，监控线程，包初始化等，
* 然后就要调用 `main.main` 了，终于输出了 `hello world！`。

![image-20211204193223832](https://cdn.jsdelivr.net/gh/baici1/img-typora/20211204193223.png)

在这里只是简单的用一个例子去介绍了一些相关的代码与变量，后续会进行详细解答。

## 详解 GPM

我们在前面总说 `GPM` ，那么他们分别代表什么意思呢？

* `G` 代表 `Goroutine`，`Golang` 中的协程，通过 `Goroutine` 封装的代码片段将以协程方式并发执行，是GPM调度器调度的基本单位。
* `P` 代表 `Processor`，`GPM` 调度器中关联内核级线程与协程的中间调度器，帮助线程去执行协程的任务。
* `M` 代表 `Machine`，是内核线程的封装，`Goroutine`的执行提供了底层线程能力支持。

> 白话解释：`M` 需要去执行 `G` 的任务，为了更高并发执行性能，我们引入 `P`，来起到帮助作用。

### M

我们来看看 M 的结构：

```go
type m struct {
   g0      *g  //g 结构体指针，主协程对应的 g0  
   mstartfn      func() //函数类型，对应着当前内核线程需要执行的 Goroutine 函数片段
   curg          *g      //g 结构体指针，对应着当前该 M 相关联(要执行)的 G。
   p             puintptr  //地址类型，对应着当前该 M 关联的 P。
   nextp         puintptr //地址类型，标识下一个可能与该 M 存在关联的 P。
   oldp          puintptr  //地址类型，记录上一个与该 M 关联的 P。
   lockedg       guintptr //地址类型，标识当前正在锁定该 M 的G，通过 LockOSThread 进行 G 和 M 的锁定，一旦 G 和 M 锁定后，该 G 只可由该 M 执行。
   spinning      bool //布尔类型，表示当前是否正在自旋，自旋则代表当前 M 正在寻找可执行的 G。
   incgo         bool //布尔类型，表示当前是否正在执行 cgo 调用。
   ncgo          int32 //int32类型，表示当前正在执行的 cgo 调用数目。
   // 忽略
 }
```





## 详解调度过程

在程序初始化过程中，会进行调度器初始化，会按照 `GOMAXPROCS` 这个环境变量决定创建多个 `P`。保存在全局变量 `allp` 中，同时把第一个 `P` 与吗 `m0` 关联起来。

