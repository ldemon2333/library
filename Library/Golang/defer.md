#### defer 声明时会先计算确定参数的值
```go
func a() {
    i := 0
    defer fmt.Println(i) // 0
    i++
    return
}
```
在这个例子中，变量 i 在 `defer`被调用的时候就已经确定了，而不是在 `defer`执行的时候，所以上面的语句输出的是 0。


#### defer 可以修改有名返回值函数的返回值
如同官方所说：

> For instance, if the deferred function is a function literal and the surrounding function has named result parameters that are in scope within the literal, the deferred function may access and modify the result parameters before they are returned.

上面所说的是，如果一个被`defer`调用的函数是一个 function literal，也就是说是闭包或者匿名函数，并且调用 `defer`的函数时一个有名返回值(named result parameters)的函数，那么 defer 可以直接访问有名返回值并进行修改的。

例子如下：
```go
// f returns 42
func f() (result int) {
    defer func() {
        result *= 7
    }()
    return 6
}
```

但是需要注意的是，只能修改有名返回值(named result parameters)函数，匿名返回值函数是无法修改的，如下：

```go
// f returns 100
func f() int {
    i := 100
    defer func() {
        i++
    }()
    return i
}
```

因为匿名返回值函数是在`return`执行时被声明，因此在`defer`语句中只能访问有名返回值函数，而不能直接访问匿名返回值函数。

### defer 的类型
Go 在 1.13 版本 与 1.14 版本对 `defer` 进行了两次优化，使得 `defer` 的性能开销在大部分场景下都得到大幅降低。

#### 堆上分配
在 Go 1.13 之前所有 `defer` 都是在堆上分配，该机制在编译时：

1. 在 `defer` 语句的位置插入 `runtime.deferproc`，被执行时，`defer` 调用会保存为一个 `runtime._defer` 结构体，存入 Goroutine 的`_defer` 链表的最前面；
2. 在函数返回之前的位置插入`runtime.deferreturn`，被执行时，会从 Goroutine 的 `_defer` 链表中取出最前面的`runtime._defer` 并依次执行。

#### 栈上分配

Go 1.13 版本新加入 `deferprocStack` 实现了在栈上分配 `defer`，相比堆上分配，栈上分配在函数返回后 `_defer` 便得到释放，省去了内存分配时产生的性能开销，只需适当维护 `_defer` 的链表即可。按官方文档的说法，这样做提升了约 30% 左右的性能。

除了分配位置的不同，栈上分配和堆上分配并没有本质的不同。

值得注意的是，1.13 版本中并不是所有`defer`都能够在栈上分配。循环中的`defer`，无论是显示的`for`循环，还是`goto`形成的隐式循环，都只能使用堆上分配，即使循环一次也是只能使用堆上分配：

```go
func A1() {
    for i := 0; i < 1; i++ {
        defer println(i)
    }
}

$ GOOS=linux GOARCH=amd64 go tool compile -S main.go
        ...
        0x004e 00078 (main.go:5)        CALL    runtime.deferproc(SB)
        ...
        0x005a 00090 (main.go:5)        CALL    runtime.deferreturn(SB)
        0x005f 00095 (main.go:5)        MOVQ    32(SP), BP
        0x0064 00100 (main.go:5)        ADDQ    $40, SP
        0x0068 00104 (main.go:5)        RET
```

#### 开放编码

Go 1.14 版本加入了开发编码（open coded），该机制会`defer`调用直接插入函数返回之前，省去了运行时的 `deferproc` 或 `deferprocStack` 操作。，该优化可以将 `defer` 的调用开销从 1.13 版本的 ~35ns 降低至 ~6ns 左右。

不过需要满足一定的条件才能触发：

1. 没有禁用编译器优化，即没有设置 `-gcflags "-N"`；
2. 函数内 `defer` 的数量不超过 8 个，且`return`语句与`defer`语句个数的乘积不超过 15；
3. 函数的 `defer` 关键字不能在循环中执行；


# 堆上分配
下面我们看一下`runtime.deferproc`:

文件位置：src/runtime/panic.go

```go
func deferproc(siz int32, fn *funcval) {  
    if getg().m.curg != getg() { 
        throw("defer on system stack")
    }
    // 获取sp指针
    sp := getcallersp()
    // 获取fn函数后指针作为参数
    argp := uintptr(unsafe.Pointer(&fn)) + unsafe.Sizeof(fn)
    callerpc := getcallerpc()
    // 获取一个新的defer
    d := newdefer(siz)
    if d._panic != nil {
        throw("deferproc: d.panic != nil after newdefer")
    }
    // 将 defer 加入到链表中
    d.link = gp._defer
    gp._defer = d
    d.fn = fn
    d.pc = callerpc
    d.sp = sp
    // 进行参数拷贝
    switch siz {
    case 0: 
        //如果defered函数的参数只有指针大小则直接通过赋值来拷贝参数
    case sys.PtrSize:
        // 将 argp 所对应的值 写入到 deferArgs 返回的地址中
        *(*uintptr)(deferArgs(d)) = *(*uintptr)(unsafe.Pointer(argp))
    default:
        // 如果参数大小不是指针大小，那么进行数据拷贝
        memmove(deferArgs(d), unsafe.Pointer(argp), uintptr(siz))
    }

    return0() 
}
```
#### 一、`gp` 的本质与作用

1. **定义来源**
    - `gp` 是 Go 运行时内部的**全局指针**，全称为 `goroutine pointer`
    - 通过 `getg()` 获取当前 goroutine 的运行时表示（`g` 结构体指针）
    - **关键字段关联**：

```
type g struct { _defer *_defer // defer链表头 // 其他字段（栈、状态等） }
```
2. **在 `deferproc` 中的核心作用**
    - **维护 defer 链**：通过 `gp._defer` 管理当前 goroutine 的所有未执行 defer
    - **上下文绑定**：确保 defer 与创建它的 goroutine 严格绑定（避免并发安全问题）

#### 一、`gp` 的作用域特性

1. **非全局唯一，而是 goroutine 局部唯一**
    - 每个运行的 goroutine 都有自己独立的 `gp` 指针
    - 通过线程本地存储（TLS）实现隔离，不同 goroutine 的 `gp` 值不同
    - **类比理解**：类似线程局部存储（TLS），但粒度更细



并且我们知道，在堆上分配时，`defer`会以链表的形式存放在当前的 Goroutine 中，如果有 3个 `defer`分别被调用，那么最后调用的会在链表最前面：

对于 `newdefer`这个函数来总的来说就是从 P 的本地缓存池里获取，获取不到则从 sched 的全局缓存池里获取一半 `defer` 来填充 P 的本地资源池， 如果还是没有可用的缓存，直接从堆上分配新的 `defer` 和 `args` 。这里的内存分配和内存分配器分配的思路大致雷同，不再分析，感兴趣的可以自己看一下。

到这里我们基本上把 `defer`的函数的调用整个过程通过堆上分配展示给大家看了。从上面的分析中基本也回答了`defer` 可以修改有名返回值函数的返回值是如何做到的，答案其实就是在`defer`调用过程中传递的 `defer` 参数是一个返回值的指针，所以最后在 `defer`执行的时候会修改返回值。

#### 小结

下面通过一个图对比一下两者的在调用完 `runtime.deferreturn`栈帧的情况：

![stackframecompare](https://img.luozhiyun.com/20210501113013.png)

很明显可以看出有名返回值函数会在 16(SP) 的地方保存返回值的地址，而匿名返回值函数会在 16(SP) 的地方保存24(SP) 的地址。

通过上面的一连串的分析也顺带回答了几个问题：

1. defer 是如何传递参数的？
    
    我们在上面分析的时候会发现，在执行 `deferproc` 函数的时候会先将参数值进行复制到 `defer`内存地址值紧挨着的位置作为参数，如果是指针的传递会直接复制指针，值传递会直接复制值到`defer`参数的位置。

![[Pasted image 20250402105847.png]]
2. 多个 defer 语句是如何执行？
    
    在调用 `deferproc`函数注册 `defer`的时候会将新元素插在表头，执行的时候也是获取链表头依次执行。
    
    ![deferLink](https://img.luozhiyun.com/20210501113019.png)
    
3. defer、return、返回值的执行顺序是怎样的？
    
    对于这个问题，我们将上面例子中的输出的汇编拿过来研究一下就明白了：
    
    ```assembly
    "".f STEXT size=126 args=0x8 locals=0x20 
           ...
           0x004e 00078 (main.go:11)       MOVQ    $6, "".result+40(SP)        ;; 将常量6写入40(SP)，作为返回值
           0x0057 00087 (main.go:11)       XCHGL   AX, AX
           0x0058 00088 (main.go:11)       CALL    runtime.deferreturn(SB)     ;; 调用 runtime.deferreturn 函数
           0x005d 00093 (main.go:11)       MOVQ    24(SP), BP
           0x0062 00098 (main.go:11)       ADDQ    $32, SP
           0x0066 00102 (main.go:11)       RET
    ```
    
    从这段汇编中可以知道，对于
    
    1. 首先是最先设置返回值为常量6；
    2. 然后会调用 `runtime.deferreturn`执行 `defer`链表；
    3. 执行 RET 指令跳转到 caller 函数；

### 栈上分配
在开始的时候也讲到了，在 Go 的 1.13 版本之后加入了 `defer` 的栈上分配，所以和堆上分配有一个区别是在栈上创建 `defer` 的时候是通过 `deferprocStack`进行创建的。

Go 在编译的时候在 SSA 阶段会经过判断，如果是栈上分配，那么会需要直接在函数调用帧上使用编译器来初始化 `_defer` 记录，并作为参数传递给 `deferprocStack`。其他的执行过程和堆上分配并没有什么区别。

对于`deferprocStack`函数我们简单看一下：

文件位置：src/cmd/compile/internal/gc/ssa.go

```go
func deferprocStack(d *_defer) {
    gp := getg()
    if gp.m.curg != gp { 
        throw("defer on system stack")
    } 
    d.started = false
    d.heap = false  // 栈上分配的 _defer
    d.openDefer = false
    d.sp = getcallersp()
    d.pc = getcallerpc()
    d.framepc = 0
    d.varp = 0 
    *(*uintptr)(unsafe.Pointer(&d._panic)) = 0
    *(*uintptr)(unsafe.Pointer(&d.fd)) = 0
    // 将多个 _defer 记录通过链表进行串联
    *(*uintptr)(unsafe.Pointer(&d.link)) = uintptr(unsafe.Pointer(gp._defer))
    *(*uintptr)(unsafe.Pointer(&gp._defer)) = uintptr(unsafe.Pointer(d))

    return0() 
}
```

函数主要功能就是给 `_defer`结构体赋值，并返回。


### 开放编码

Go 语言在 1.14 中通过代码内联优化，使得函数末尾直接对`defer`函数进行调用， 做到几乎不需要额外的开销。在 SSA 的构建阶段 `buildssa`会根据检查是否满足条件，满足条件才会插入开放编码式，由于 SSA 的构建阶段的代码不太好理解，所以下面只给出基本原理，不涉及代码分析。

我们可以对堆上分配的例子进行汇编打印：


#### 二、触发堆分配的条件

1. **编译器逃逸分析决策**
    
    - 当defer出现在**循环体**中时（变量生命周期超出栈帧）

```
for i := 0; i < 10; i++ { defer fmt.Println(i) // 堆分配 }
```
- 当defer与**闭包**结合且捕获变量逃逸时

# 预计算参数
Go 语言中所有的函数调用都是传值的，虽然 `defer` 是关键字，但是也继承了这个特性。假设我们想要计算 `main` 函数运行的时间，可能会写出以下的代码：
```go
func main() {
	startedAt := time.Now()
	defer fmt.Println(time.Since(startedAt))

	time.Sleep(time.Second)
}

$ go run main.go
0s
```

然而上述代码的运行结果并不符合我们的预期，这个现象背后的原因是什么呢？经过分析，我们会发现调用 `defer` 关键字会立刻拷贝函数中引用的外部参数，所以 `time.Since(startedAt)` 的结果不是在 `main` 函数退出之前计算的，而是在 `defer` 关键字调用时计算的，最终导致上述代码输出 0s。

想要解决这个问题的方法非常简单，我们只需要向 `defer` 关键字传入匿名函数：
```go
func main() {
	startedAt := time.Now()
	defer func() { fmt.Println(time.Since(startedAt)) }()

	time.Sleep(time.Second)
}

$ go run main.go
1s
```

虽然调用 `defer` 关键字时也使用值传递，但是因为拷贝的是函数指针，所以 `time.Since(startedAt)` 会在 `main` 函数返回前调用并打印出符合预期的结果。

# 数据结构
在介绍 `defer` 函数的执行过程与实现原理之前，我们首先来了解一下 `defer` 关键字在 Go 语言源代码中对应的数据结构：
```go
type _defer struct {
	siz       int32
	started   bool
	openDefer bool
	sp        uintptr
	pc        uintptr
	fn        *funcval
	_panic    *_panic
	link      *_defer
}
```

[`runtime._defer`](https://draven.co/golang/tree/runtime._defer) 结构体是延迟调用链表上的一个元素，所有的结构体都会通过 `link` 字段串联成链表。
![[Pasted image 20250331230329.png]]
**图 5-10 延迟调用链表**

我们简单介绍一下 [`runtime._defer`](https://draven.co/golang/tree/runtime._defer) 结构体中的几个字段：

- `siz` 是参数和结果的内存大小；
- `sp` 和 `pc` 分别代表栈指针和调用方的程序计数器；
- `fn` 是 `defer` 关键字中传入的函数；
- `_panic` 是触发延迟调用的结构体，可能为空；
- `openDefer` 表示当前 `defer` 是否经过开放编码的优化；

除了上述的这些字段之外，[`runtime._defer`](https://draven.co/golang/tree/runtime._defer) 中还包含一些垃圾回收机制使用的字段，这里为了减少理解的成本就都省去了。

# 执行机制
中间代码生成阶段的 [`cmd/compile/internal/gc.state.stmt`](https://draven.co/golang/tree/cmd/compile/internal/gc.state.stmt) 会负责处理程序中的 `defer`，该函数会根据条件的不同，使用三种不同的机制处理该关键字：
```go
func (s *state) stmt(n *Node) {
	...
	switch n.Op {
	case ODEFER:
		if s.hasOpenDefers {
			s.openDeferRecord(n.Left) // 开放编码
		} else {
			d := callDefer // 堆分配
			if n.Esc == EscNever {
				d = callDeferStack // 栈分配
			}
			s.callResult(n.Left, d)
		}
	}
}
```
堆分配、栈分配和开放编码是处理 `defer` 关键字的三种方法，早期的 Go 语言会在堆上分配 [`runtime._defer`](https://draven.co/golang/tree/runtime._defer) 结构体，不过该实现的性能较差，Go 语言在 1.13 中引入栈上分配的结构体，减少了 30% 的额外开销[1](https://draven.co/golang/docs/part2-foundation/ch05-keyword/golang-defer/#fn:1)，并在 1.14 中引入了基于开放编码的 `defer`，使得该关键字的额外开销可以忽略不计[2](https://draven.co/golang/docs/part2-foundation/ch05-keyword/golang-defer/#fn:2)，我们在一节中会分别介绍三种不同类型 `defer` 的设计与实现原理。