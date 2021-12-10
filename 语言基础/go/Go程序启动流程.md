# Go语言启动流程

> 推荐阅读：
>
> [2.3 Go 程序启动引导](https://golang.design/under-the-hood/zh-cn/part1basic/ch02life/boot/)
>
> [Golang并发编程-GPM调度过程源码分析](https://juejin.cn/post/6976839612241018888#heading-3)
>
> [《Golang》深入Golang启动过程](http://www.pefish.club/2020/05/08/Golang/1005%E6%B7%B1%E5%85%A5Golang%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B/)
>
> [**从源码角度看 Golang 的调度**](https://github.com/0voice/Introduction-to-Golang/blob/main/%E6%96%87%E7%AB%A0/%E4%BB%8E%E6%BA%90%E7%A0%81%E8%A7%92%E5%BA%A6%E7%9C%8B%20Golang%20%E7%9A%84%E8%B0%83%E5%BA%A6.md)

## 前言

每次写 Go 程序我总是好奇他的启动流程，今天我们来扒一扒。

注：我用的电脑是 `win10`，所以很多地方并不是以 `linux` 为主。同时这是我自己的一个学习过程，可能会有错误，希望能够得到指导！

同时文章中的部分代码会经过处理的，会更注重于核心代码流程。

希望读者能够懂一点点的汇编语言。

## 汇编

Go 程序启动需要对自身运行时进行初始化，其真正的程序入口在 `runtime` 包里面。

不同平台的入口文件都不同， 以 `AMD64` 架构上的 `Linux` 和 `macOS` 以及 `win10` 为例，分别位于：`src/runtime/rt0_linux_amd64.s` 和 `src/runtime/rt0_darwin_amd64.s` 以及 `src/runtime/rt0_windows_amd64.s`。

这三个文件你都可以看到相类似的入口代码。

```assembly
# runtime/rt0_windows_amd64.s
#以windows 为例，linux 和macos 都是一致，只是名字的改变罢了。
TEXT _rt0_amd64_windows(SB),NOSPLIT,$-8
	JMP	_rt0_amd64(SB)
```

`JMP` 是无条件跳转，接着就跳转到了 `_rt0_amd64` 这个子程序。

> 这种做法符合直觉，在程序编译为机器码之后， 依赖特定 CPU 架构的指令集，而操作系统的差异则是直接反应在运行时进行不同的系统级操作上， 例如：系统调用。
>
> `rt0` 其实是 `runtime0` 的缩写，意为运行时的创生，随后所有创建的都是 `1` 为后缀。

操作系统通过入口参数的约定与应用程序进行沟通。程序刚刚启动时，栈指针 SP 的前两个值分别对应 `argc` 和 `argv`，分别存储参数的数量和具体的参数的值

```assembly
# runtime/asm_amd64.s
# _rt0_amd64 is common startup code for most amd64 systems when using
# internal linking. This is the entry point for the program from the
# kernel for an ordinary -buildmode=exe program. The stack holds the
# number of arguments and the C-style argv.
#_rt0_amd64 是使用内部链接时大多数 amd64 系统的常见启动代码。这是普通 -buildmode=exe 程序的内核程序的入口点。堆栈保存参数的数量和 C 风格的 argv。
TEXT _rt0_amd64(SB),NOSPLIT,$-8
	MOVQ	0(SP), DI	// argc
	LEAQ	8(SP), SI	// argv
	JMP	runtime·rt0_go(SB)
```

### rt0_go

接着继续跳转到 `rt0_go` 子程序里面。

我们来细细扒一扒这个里面的逻辑。

这前面一部分就是为了去确定程序入口参数和 `CPU` 处理器信息。

```assembly
# runtime/asm_amd64.s
TEXT runtime·rt0_go(SB),NOSPLIT|TOPFRAME,$0
	// 将参数向前复制到一个偶数栈上
	MOVQ	DI, AX		// argc
	MOVQ	SI, BX		// argv
	SUBQ	$(4*8+7), SP		// 2args 2auto
	ANDQ	$~15, SP
	MOVQ	AX, 16(SP)
	MOVQ	BX, 24(SP)

	#从给定的（操作系统）堆栈中创建 istack。 _cgo_init 可能会更新 stackguard。
	# 初始化 g0 执行栈
	MOVQ	$runtime·g0(SB), DI
	LEAQ	(-64*1024+104)(SP), BX
	MOVQ	BX, g_stackguard0(DI)
	MOVQ	BX, g_stackguard1(DI)
	MOVQ	BX, (g_stack+stack_lo)(DI)
	MOVQ	SP, (g_stack+stack_hi)(DI)

	// 确定 CPU 处理器的信息
	MOVL	$0, AX
	CPUID
	MOVL	AX, SI
	CMPL	AX, $0
	JE	nocpuinfo
		#弄清楚如何序列化 RDTSC。在英特尔处理器上，LFENCE 就足够了。 AMD 需要 MFENCE。不知道其余的，所以让我们做MFENCE。
	CMPL	BX, $0x756E6547  // "Genu"
	JNE	notintel
	CMPL	DX, $0x49656E69  // "ineI"
	JNE	notintel
	CMPL	CX, $0x6C65746E  // "ntel"
	JNE	notintel
	MOVB	$1, runtime·isIntel(SB)
	MOVB	$1, runtime·lfenceBeforeRdtsc(SB)
# 省略了一大段代码
```

一个影响运行时非常重要的操作便是本地线程存储 （Thread Local Storage, TLS）。

```assembly
# runtime/asm_amd64.s
TEXT runtime·rt0_go(SB),NOSPLIT|TOPFRAME,$0
# 省略了一大段代码

notintel:
#ifdef GOOS_darwin
	// 跳过 Darwin 上的 TLS 设置
	JMP ok
#endif
	LEAQ	runtime·m0+m_tls(SB), DI #// DI = m0.tls
	CALL	runtime·settls(SB) # 将 TLS 地址设置到 DI

	// // 使用它进行存储，确保能正常运行
	get_tls(BX)
	MOVQ	$0x123, g(BX)
	MOVQ	runtime·m0+m_tls(SB), AX 
	CMPQ	AX, $0x123 // 判断 TLS 是否设置成功
	JEQ 2(PC)  // 如果相等则向后跳转两条指令
	CALL	runtime·abort(SB) // 使用 INT 指令执行中断
```

创建全局变量 `g0` 和 `m0`，还需要将 `m0` 和 `g0` 通过指针进行相互关联。

```assembly
# runtime/asm_amd64.s
TEXT runtime·rt0_go(SB),NOSPLIT|TOPFRAME,$0
# 省略了一大段代码
	// 设置 per-goroutine 和 per-mach“寄存器”
	// 程序刚刚启动，此时位于主线程
	// 当前栈与资源保存在 g0
	// 该线程保存在 m0
	get_tls(BX)
	LEAQ	runtime·g0(SB), CX
	MOVQ	CX, g(BX)
	LEAQ	runtime·m0(SB), AX
	//m0 和 g0 通过指针进行相互关联。
	// save m->g0 = g0
	MOVQ	CX, m_g0(AX)
	// save m0 to g0->m
	MOVQ	AX, g_m(CX)
```

这里做一些校验和系统级的初始化工作，包括：运行时类型检查， 系统参数的获取以及影响内存管理和程序调度的相关常量的初始化。

```assembly
# runtime/asm_amd64.s
TEXT runtime·rt0_go(SB),NOSPLIT|TOPFRAME,$0
# 省略了一大段代码

	CLD				// convention is D is always left cleared
	//运行时类型检查
	CALL	runtime·check(SB)

	MOVL	16(SP), AX		// copy argc
	MOVL	AX, 0(SP)
	MOVQ	24(SP), AX		// copy argv
	MOVQ	AX, 8(SP)
	//系统参数的获取
	CALL	runtime·args(SB)
	//影响内存管理的相关常量的初始化。
	CALL	runtime·osinit(SB)
	//程序调度的相关常量的初始化
	CALL	runtime·schedinit(SB)
```

马上就要开始运行了！

```assembly
# runtime/asm_amd64.s
TEXT runtime·rt0_go(SB),NOSPLIT|TOPFRAME,$0
# 省略了一大段代码

	// 创建一个新的 goroutine 来启动程序
	MOVQ	$runtime·mainPC(SB), AX		// entry // entry mainPC方法（也就是runtime·main函数，是一个全局变量）压入AX寄存器
	PUSHQ	AX
	PUSHQ	$0			// arg size 压入第一个参数到栈
	CALL	runtime·newproc(SB) // 调用 newproc 函数创建一个新的g
	POPQ	AX
	POPQ	AX

	// 启动这个 M.mstart 
	CALL	runtime·mstart(SB)

	CALL	runtime·abort(SB)	// M.mstart 应该永不返回
	RET

	//防止 debugCallV2 的死代码消除，它旨在由调试器调用。
	MOVQ	$runtime·debugCallV2<ABIInternal>(SB), AX
	RET
```

编译器负责生成了 `main` 函数的入口地址，`runtime.mainPC` 在数据段中被定义为 `runtime.main` 保存主 `goroutine` 入口地址：

```assembly
# mainPC 是 runtime.main 的函数值，要传递给 newproc。对 runtime.main 的引用是通过 ABIInternal 进行的，因为 newproc 需要实际的函数（不是 ABI0 包装器）。
DATA	runtime·mainPC+0(SB)/8,$runtime·main<ABIInternal>(SB)
GLOBL	runtime·mainPC(SB),RODATA,$8
```

当 Go 程序的引导程序启动会调用下面核心函数完成校验与系统初始化：

* `check` ：运行时类型检查
* `args` ： 系统参数的获取
* `osinit` ：影响内存管理的相关常量的初始化
* `schedinit` ：程序调度与内存分配器、回收器的相关常量的初始化
* `newproc`：负责根据主 `goroutine` （即 `main`）入口地址创建可被运行时调度的执行单元 `G`。
* `mstart` ：开始启动调度器的调度循环。

根据分析，我们知道了，Go 程序既不是从 `main.main` 直接启动，也不是从 `runtime.main` 直接启动。 相反，其实际的入口位于 `runtime._rt0_amd64_*`。随后会转到 `runtime.rt0_go` 调用。

程序引导和初始化工作是整个运行时最关键的基础步骤之一。在 `schedinit` 这个函数的调用过程中， 还会完成整个程序运行时的初始化，包括调度器、执行栈、内存分配器、调度器、垃圾回收器等组件的初始化。 最后通过 `newproc` 和 `mstart` 调用进而开始由调度器转为执行主 `goroutine`。

启动流程图如下：

![image-20211206191645379](https://cdn.jsdelivr.net/gh/baici1/img-typora/20211206191645.png)

## 核心函数

我们在之前的分析里面了解到一些核心函数，现在我们来简单看看里面的逻辑，到底每个函数具体工作是什么？至于解析背后的原理，我们留到具体的章节去考虑。

`check` 函数，本质上是对编译器翻译工作的一个校验，再次检验类型的内存大小。

```go
//# runtime/runtime1.go
func check() {
    var (
		a     int8
		b     uint8
		c     int16
		d     uint16
        //省略
	)
    type x1t struct {
		x uint8
	}
	type y1t struct {
		x1 x1t
		y  uint8
	}
	var x1 x1t
	var y1 y1t
	// 校验 int8 类型 sizeof 是否为 1，下同
	if unsafe.Sizeof(a) != 1 {
		throw("bad a")
	}
    //省略
    
}
```

`args` 函数，将操作系统传递 `argc,argv` 两个参数赋值作为全局变量使用

```go
//# runtime/runtime1.go
var (
	argc int32
	argv **byte
)

func args(c int32, v **byte) {
	argc = c 
	argv = v
	sysargs(c, v)
}
```

![img](https://golang.design/under-the-hood/assets/proc-stack.png)

那么接下来调用系统特定的 `sysargs` 函数。

```go
//runtime/os_dragonfly.go
func sysargs(argc int32, argv **byte) {
    // 跳过 argv, envv 与第一个字符串为路径
	n := argc + 1

	//跳过 argv, envp 进入 auxv
	for argv_index(argv, n) != nil {
		n++
	}

	// skip NULL separator // 跳过 NULL 分隔符
	n++
	// 尝试读取 auxv
	auxv := (*[1 << 28]uintptr)(add(unsafe.Pointer(argv), uintptr(n)*sys.PtrSize))
	sysauxv(auxv[:])
}

func sysauxv(auxv []uintptr) {
    // 依次读取 auxv 键值对
	for i := 0; auxv[i] != _AT_NULL; i += 2 {
		tag, val := auxv[i], auxv[i+1]
		switch tag {
		case _AT_PAGESZ:
            // 读取内存页的大小
			physPageSize = val
		}
	}
}
```

在这里我已经懵了，已经涉及到了操作系统的底层那些内存页等等了。这里就不多去解释。我已经不懂了。😥

`osinit` 函数，会获取CPU核数，还会获取当前操作系统的页存大小。

```go
//runtime/os_dragonfly.go
func osinit() {
    // 获取CPU核数
	ncpu = getncpu()
	if physPageSize == 0 {
		physPageSize = getPageSize()
	}
}
```

`schedinit` 函数，名字上是调度器的一个初始化，其实内部实际上干的事情都是一些核心部分的初始化，例如：栈，内存，gc，线程等等。

这里的初始化也是有一定顺序规则的，至于为什么，可能是因为前面的函数为后面的函数提供一定的重要数据。

```go
// 引导的序列 is:
//	call osinit
//	call schedinit
//	make & queue new G //将new G加入到队列中
//	call runtime·mstart 
// The new G calls runtime·main. 
func schedinit() {
	lockInit(&sched.lock, lockRankSched)
    //省略 lockinit

	//获取 g 的一个对象
	_g_ := getg()

	sched.maxmcount = 10000 // 限制最大系统线程数量

	// The world starts stopped.  用于lock rank,
	worldStopped()

	moduledataverify()
	stackinit() // 初始化执行栈
	mallocinit() // 初始化内存分配器
	fastrandinit() // must run before mcommoninit // 随机数初始化，
	mcommoninit(_g_.m, -1) 	// 初始化当前系统线程 //预分配的 ID 可以作为“id”传递，或者通过传递 -1 来省略。
	cpuinit()       // must run before alginit // 初始化CPU信息
	alginit()       // maps must not be used before this call // 主要初始化哈希算法的值
	modulesinit()   // provides activeModules // activeModules数据初始化，主要是用于gc的数据,
	typelinksinit() // uses maps, activeModules // 主要初始化activeModules的typemap
	itabsinit()     // uses activeModules  // 初始化interface相关，

	sigsave(&_g_.m.sigmask) // 初始化m的signal mask
	initSigmask = _g_.m.sigmask

	goargs()  // 参数放到argslice变量中
	goenvs()  // 环境变量放到envs中
	parsedebugvars()  // 初始化一系列debug相关的变量
	gcinit()  // 垃圾回收器初始化
	//调度器加锁
	lock(&sched.lock)
	sched.lastpoll = uint64(nanotime())
    // 创建 P
	// 通过 CPU 核心数和 GOMAXPROCS 环境变量确定 P 的数量
	procs := ncpu // // procs设置成cpu个数
	if n, ok := atoi32(gogetenv("GOMAXPROCS")); ok && n > 0 {  // 如果GOMAXPROCS有设置，则覆盖procs的值
		procs = n
	}
    // 增加或减少p的实例个数(填procs个p到存放所有p的全局变量allp中)，多了就清理多的p，少了就新建p，但是并没有启动m，m启动后会从这里取p并挂钩上
	if procresize(procs) != nil {
		throw("unknown runnable goroutine during bootstrap")
	}
    //调度器解锁
	unlock(&sched.lock)
	//省略一大段代码
}
```

`newproc` 函数，当前 `M` 的 `P` 下创建了一个新的 `G`，其实也就是我们期待的 `runtime.main`，不会一开始就直接添加到运行队列中，而是放到 `P` 的本地队列，成为下一个运行的 `G`。

> 为什么这里一定要放到 runtime.runnext，不是运行队列中呢？

我的猜测当前是 `G0`,而且此时其实 `m` 的对应线程并没有创建出来，现在只是再初始化一些 `m` 的相关属性，所以不适合直接放入到运行队列中。

```go
func newproc(siz int32, fn *funcval) {
	argp := add(unsafe.Pointer(&fn), sys.PtrSize)
	gp := getg() // 获取当前goroutine的指针，
	pc := getcallerpc() // 获取伪寄存器PC的内容，函数也是由编译器填充
	systemstack(func() {
        //创建一个新的G
		newg := newproc1(fn, argp, siz, gp, pc) //关键函数
		//获取P的指针
		_p_ := getg().m.p.ptr()
        //将新创建的的 G，添加到 runtime.runnext 队列中如果运行队列满了，就添加到全局队列供其他P进行调度
		runqput(_p_, newg, true)
		//尝试再添加一个 P 来执行 G 的。当 G 变为可运行时调用（newproc，ready）。
		if mainStarted {
			wakep()
		}
	})
}
func newproc1(fn *funcval, argp unsafe.Pointer, narg int32, callergp *g, callerpc uintptr) *g {
    (...)
	_g_ := getg()
	_p_ := _g_.m.p.ptr()
	newg := gfget(_p_) //// 从p的dead g列表中获取一个g对象，没有的话就从全局g列表中抓取一批g对象放入p的的dead g列表中，再从中获取。g在运行结束后会重新放入dead g列表等待重复利用
	if newg == nil { // 一开始启动应该取不到
		newg = malg(_StackMin) // 新建一个g
		casgstatus(newg, _Gidle, _Gdead) // 设置g的状态从idle到dead
		allgadd(newg) // 使用 G-> 状态的 Gdead 发布，因此 GC 扫描程序不会查看未初始化的堆栈。
	}
    (...)

    (...)//关于newg的属性配置
	newg.startpc = fn.fn // 将mainPC方法(就是runtime·main方法)指定为这个协程的启动方法
	if _g_.m.curg != nil {
		newg.labels = _g_.m.curg.labels
	}
	if isSystemGoroutine(newg, false) {
		atomic.Xadd(&sched.ngsys, +1)
	}
	// Track initial transition?
	newg.trackingSeq = uint8(fastrand())
	if newg.trackingSeq%gTrackingPeriod == 0 { // 判断是不是系统协程（g启动函数包含runtime.*前缀的都是系统协程，除了runtime.main, runtime.handleAsyncEvent）
		newg.tracking = true
	}
	casgstatus(newg, _Gdead, _Grunnable)  // 设置g的状态从dead状态到runnable状态

	（...）
	releasem(_g_.m) // 放弃独占m

	return newg
}

```

mstart 函数，主要是启动 M，并且开启调度（我们下一次再讨论这个）。

```go
//mstart 是 new Ms 的入口点。它是用汇编编写的，使用 ABI0，标记为 TOPFRAME，并调用 mstart0。
func mstart()
func mstart0() {
	_g_ := getg()

	osStack := _g_.stack.lo == 0
	if osStack {
//从系统堆栈初始化堆栈边界。 Cgo 可能在 stack.hi 中保留了堆栈大小。 minit 可能会更新堆栈边界。注意：这些界限可能不是很准确。我们将 hi 设置为 &size，但它上面还有一些东西。 1024 应该可以弥补这一点，但有点武断。
		size := _g_.stack.hi
		if size == 0 {
			size = 8192 * sys.StackGuardMultiplier
		}
		_g_.stack.hi = uintptr(noescape(unsafe.Pointer(&size)))
		_g_.stack.lo = _g_.stack.hi - size + 1024
	}
	//初始化堆栈保护，以便我们可以开始调用常规
	// Go code.
	_g_.stackguard0 = _g_.stack.lo + _StackGuard
	// 这是 g0，所以我们也可以调用 go:systemstack 函数来检查 stackguard1。
	_g_.stackguard1 = _g_.stackguard0
	mstart1()

	// Exit this thread.
	if mStackIsSystemAllocated() {
		// Windows、Solaris、illumos、Darwin、AIX 和Plan 9 总是system-allocate stack，但是在mstart 之前放在_g_.stack 中，所以上面的逻辑还没有设置osStack。
		osStack = true
	}
	mexit(osStack)
}
func mstart1() {
	_g_ := getg()

	if _g_ != _g_.m.g0 { // 判断是不是g0
		throw("bad runtime·mstart")
	}
	_g_.sched.g = guintptr(unsafe.Pointer(_g_))
	_g_.sched.pc = getcallerpc()   // 保存pc、sp信息到g0
	_g_.sched.sp = getcallersp()

	asminit() // asm初始化
	minit()  // m初始化

	// Install signal handlers; after minit so that minit can
	// prepare the thread to be able to handle the signals.
	if _g_.m == &m0 {
		mstartm0()  // 启动m0的signal handler
	}

	if fn := _g_.m.mstartfn; fn != nil {
		fn()
	}

	if _g_.m != &m0 { // 如果不是m0
		acquirep(_g_.m.nextp.ptr())
		_g_.m.nextp = 0
	}
	schedule()   // 进入调度。这个函数会阻塞
}
```

## 总结流程

* 入口：rt0_windows_amd64.s 汇编函数
* 初始化 m0,g0
* check ：检查各个类型占用内存大小的正确性
* args ： 设置 `argc`、`argv`参数
* osinit ：操作系统相关的 `init`，比如页大小
* schedinit ：初始化所有 P，初始化其他细节
* newproc ：当前`m（m0）`的 `p` 下新建一个 `g`，指定为 `p` 的下一个运行的 `g`
* mstart ：m0启动，接着进入调度，这里阻塞
* abort：退出

## 进一步参考文章

> [MinGW-w64安装教程](https://www.jianshu.com/p/d66c2f2e3537)
>
> [Windows平台安装GDB调试器](http://c.biancheng.net/view/8296.html)
>
> [[go runtime] - go程序启动过程](https://juejin.cn/post/6942509882281033764)

## tips

> 自己整体流程过了一遍后，感觉还是有点点糊糊的。可能自己对操作系统的知识还是不够多，不够支撑自己理解整个过程，但是不用慌。慢慢来！加油团子！
>
> 后续会逐渐学习操作系统，然后补充相关的细节。
