# Go语言启动流程

> 推荐阅读：
>
> [2.3 Go 程序启动引导](https://golang.design/under-the-hood/zh-cn/part1basic/ch02life/boot/)
>
> [Golang并发编程-GPM调度过程源码分析](https://juejin.cn/post/6976839612241018888#heading-3)

## 前言

每次写 Go 程序我总是好奇他的启动流程，今天我们来扒一扒。

注：我用的电脑是 `win10`，所以很多地方并不是以 `linux` 为主。同时这是我自己的一个学习过程，可能会有错误，希望能够得到指导！

同时文章中的部分代码会经过处理的，会更注重于核心代码流程。

希望读者能够懂一点点的汇编语言。

## 初始化

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
	MOVQ	$runtime·mainPC(SB), AX		// entry
	PUSHQ	AX
	PUSHQ	$0			// arg size
	CALL	runtime·newproc(SB)
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

### 核心函数

我们在之前的分析里面了解到一些核心函数，现在我们来简单看看里面的逻辑，到底每个函数具体工作是什么？至于解析背后的原理，我们留到具体的章节去考虑。

`check` 函数：

## 进一步参考文章

> [MinGW-w64安装教程](https://www.jianshu.com/p/d66c2f2e3537)
>
> [Windows平台安装GDB调试器](http://c.biancheng.net/view/8296.html)
>
> [[go runtime] - go程序启动过程](https://juejin.cn/post/6942509882281033764)
