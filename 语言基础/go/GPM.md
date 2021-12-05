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
>
> [Golang并发调度的GMP模型](https://juejin.cn/post/6886321367604527112#heading-0)

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

### P

```go
const (
   _Pidle = iota //当前p尚未与任何m关联，处于空闲状态 ->0
   _Prunning //当前p已经和m关联，并且正在运行g代码 ->1
   _Psyscall  //当前p正在执行系统调用 ->2
   _Pgcstop //当前p需要停止调度，一般在GC前或者刚被创建时 ->3
   _Pdead //当前p已死亡，不会再被调度 ->4
)

type p struct {
   status      uint32  //表示当前P的状态，为上述五个状态之一
   schedtick   uint32  //调度计数器，每被调度一次则自增1
   syscalltick uint32  //系统调用计数器，每进行一次系统调用则自增1
   m           muintptr //即将要关联的m，M的nextp字段对应着该P
   runqhead uint32 //可运行G队列头，标识目前正在运行的G
   runqtail uint32  //可运行G队列尾
   runq     [256]guintptr //可运行的G队列，默认容量为256个G
   runnext guintptr //下一个将要运行的G
   gFree struct { //空闲G列表，存储着状态为Gdead的G，当其数目过多时，将会被转移到调度器全局G列表，用于被其他P再次使用（相当于一个G缓存池）
      gList
      n int32
   }
}

```

`P` 的生命周期：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a8b1bab3f170480aac6592b2aa8afa57~tplv-k3u1fbpfcp-watermark.awebp)

### G

```go
const (
   _Gidle = iota //当前 G 刚被分配，还未初始化 ->0
   _Grunnable //正在可运行队列等待运行 ->1
   _Grunning  //正在运行中，执行G函数 ->2
   _Gsyscall  //正在执行系统调用 ->3
   _Gwaiting  //正在被阻塞，一般是该G正在执行网络I/O操作，或正在执行time.Timer、time.Sleep ->4
   _Gmoribund_unused //_Gmoribund_unused is currently unused, but hardcoded in gdb ->5
   _Gdead  //已经使用完正在闲置，放入空闲G列表中，可被再次使用（和P不同，P处于Pdead状态则无法被再次调度） ->6
  _Genqueue_unused //_ Genqueue _ unused 当前未使用。 ->7
   _Gcopystack //表示当前 G 的栈正在被移动，可能是因为栈的收缩或扩容 ->8
   _Gscan         = 0x1000 //表明当前正在进行GC扫描，由于在GC扫描的过程中肯定会处于某个前置状态， 
   _Gscanrunnable = _Gscan + _Grunnable //代表当前 G 正等待运行，同时栈正被 GC 扫描  // 0x1001
   _Gscanrunning  = _Gscan + _Grunning //表示正处于 Grunning状态，同时栈在被 GC 扫描 // 0x1002
   _Gscansyscall  = _Gscan + _Gsyscall //表示正处于 Gwaiting状态，同时栈在被 GC 扫描 // 0x1003
   _Gscanwaiting  = _Gscan + _Gwaiting //表示正处于 Gsyscall状态，同时栈在被 GC 扫描 // 0x1004
    _Gscanpreempted = _Gscan + _Gpreempted // 0x1009
)

type g struct {
   stack       stack   // offset known to runtime/cgo //当前G所被分配的栈内存空间，由lo及hi两个内存指针组成
   stackguard0 uintptr // offset known to liblink g0的最大栈内存地址，当超过了这个数值则需要进行栈扩张
   stackguard1 uintptr //普通用户G的最大栈内存地址，当超过了这个数值则需要进行栈扩张
   m              *m      // current m; offset known to arm liblink 当前关联该G实例的M实例
   sched          gobuf  //记录G上下文环境，用于上下文切换
   atomicstatus   uint32 //G的状态值，表示上述几个状态
   waitreason     waitReason // if status==Gwaiting 处于Gwaiting的原因
   preempt        bool       // preemption signal, duplicates stackguard0 = st 当前G是否可抢占
   startpc        uintptr         // pc of goroutine function 当前G所绑定的函数内存地址
}

```

`G` 生命周期：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c6c550b00bcd41ea9a3ff4222e87e40e~tplv-k3u1fbpfcp-watermark.awebp)

### sched (调度器)

```go
type schedt struct {
   // 全局唯一id
   goidgen  uint64
   // 记录的最后一次从i/o中查询G的时间
   lastpoll uint64
   // 互斥锁 
   lock mutex
   // M的空闲链表，通过m.schedlink组成一个M空闲链表
   midle        muintptr
   // 正处于自旋状态的M数量
   nmidle       int32
   // 已经被锁定且正在自旋的M数量
   nmidlelocked int32
   // 下一个M的id，或者是目前已存在的M数量
   mnext        int64
   // M数量的最大值
   maxmcount    int32
   // 已被释放掉的M数量
   nmfreed      int64
   // 系统所开启的协程数量（非用户协程）
   ngsys uint32
   // 空闲P列表
   pidle      puintptr
   // 空闲的P数量
   npidle     uint32
   // 全局的G队列
   // 根据runqhead可以获取队列头的G及g.schedlink形成G链表
   runqhead guintptr
   runqtail guintptr
   // 全局G队列大小
   runqsize int32
   // 等待释放的M列表
   freem *m
   // 是否需要暂停调度（通常因为GC带来的STW）
   gcwaiting  uint32
   // 需要停止但是仍为停止的P数量
   stopwait   int32
   // 实现stopwait事件通知
   stopnote   note
   // 停止调度期间是否进行系统监控任务
   sysmonwait uint32
   // 实现sysmonwait事件通知
   sysmonnote note
}

```

### 核心容器

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c64fb28175494bb0b2419bd1fe0d760e~tplv-k3u1fbpfcp-watermark.awebp)

## 详解调度过程

## 系统初始化

Go 程序的引导程序启动进行系统初始化，核心步骤：

1. 

