# GM 模型
##  Go 1.0 源码
![[Pasted image 20250424172453.png]]
- 调用 `schedlock` 方法来获取全局锁。
    
- 获取全局锁成功后，将当前 Goroutine 状态从 Running 状态修改为 Runnable 状态。
    
- 调用 `gput` 方法来保存当前 Goroutine 的运行状态等信息，以便于后续的使用。
    
- 调用 `nextgandunlock` 方法来寻找下一个可运行 Goroutine，并且释放全局锁给其他调度使用。
    
- 获取到下一个待运行的 Goroutine 后，将其运行状态修改为 Running。
    
- 调用 `runtime·gogo` 方法，将刚刚所获取到的下一个待执行的 Goroutine 运行起来，进入下一轮调度。

当前（代指 Go1.0 的 GM 模型）的 Goroutine 调度器限制了用 Go 编写的并发程序的可扩展性，尤其是高吞吐量服务器和并行计算程序。

实现有如下的问题：

- 存在单一的全局 mutex（Sched.Lock）和集中状态管理：
    

- mutex 需要保护所有与 goroutine 相关的操作（创建、完成、重排等），导致锁竞争严重。
    

- Goroutine 传递的问题：
    

- goroutine（G）交接（G.nextg）：工作者线程（M's）之间会经常交接可运行的 goroutine。
    
- 上述可能会导致延迟增加和额外的开销。每个 M 必须能够执行任何可运行的 G，特别是刚刚创建 G 的 M。
    

- 每个 M 都需要做内存缓存（M.mcache）：
    

- 会导致资源消耗过大（每个 mcache 可以吸纳到 2M 的内存缓存和其他缓存），数据局部性差。
    

- 频繁的线程阻塞/解阻塞：
    

- 在存在 syscalls 的情况下，线程经常被阻塞和解阻塞。这增加了很多额外的性能开销。

# 单机的 goroutine 数量控制在多少比较合适？
Go scheduler 的主要功能是针对在处理器上运行的 OS 线程分发可运行的 Goroutine，而我们一提到调度器，就离不开三个经常被提到的缩写，分别是：

- G：Goroutine，实际上我们每次调用 `go func` 就是生成了一个 G。
    
- P：Processor，处理器，一般 P 的数量就是处理器的核数，可以通过 `GOMAXPROCS` 进行修改。
    
- M：Machine，系统线程。
    

这三者交互实际来源于 Go 的 M: N 调度模型。也就是 M 必须与 P 进行绑定，然后不断地在 M 上循环寻找可运行的 G 来执行相应的任务。

`GOMAXPROCS` 是 Go 语言运行时的一个参数，用来**设置当前程序可同时使用的操作系统线程数（准确说是 CPU 核心数）**，从而控制**并行执行的最大线程数量**。

---

### 🧠 它到底控制什么？

`GOMAXPROCS` 控制的是**并行度**（parallelism），不是并发（concurrency）。

Go 的调度器会用 `GOMAXPROCS` 个操作系统线程来并行运行 goroutine。比如：

- 你有 1000 个 goroutine；
    
- 如果 `GOMAXPROCS=1`，Go 调度器只会在一个内核线程上切换运行它们（是并发，但不是并行）；
    
- 如果 `GOMAXPROCS=4`，最多有 4 个 goroutine 可以**同时**运行在 4 个 CPU 核心上（即并行执行）。
    

---

### 🧪 设置方法

#### ✅ 方式 1：通过代码设置（推荐）

```go
import "runtime"

runtime.GOMAXPROCS(n) // n 为要使用的 CPU 数量
```

> 返回值是之前的 GOMAXPROCS 值

#### ✅ 方式 2：通过环境变量设置

```bash
GOMAXPROCS=4 ./your_program
```

---

### ⚙️ 默认值

自 Go 1.5 起，`GOMAXPROCS` 默认设置为机器的 **CPU 核心数**（即 `runtime.NumCPU()`），不再是 1。

你可以查看当前值：

```go
fmt.Println(runtime.GOMAXPROCS(0)) // 0 表示仅获取当前值，不修改
```

---

### ✅ 应用场景

- **高并发服务器**（如 HTTP 服务）：根据 CPU 核数合理设置 `GOMAXPROCS` 可以提升吞吐量；
    
- **CPU 密集型任务**：例如压缩/加密/图像处理，合适的 GOMAXPROCS 能让性能更好；
    
- **限资源执行**：在资源受限环境（如容器）中，可以手动限制 GOMAXPROCS 避免过度占用 CPU。
    
## 调度流程
![[Pasted image 20250424182544.png]]
1. 当我们执行 `go func()` 时，实际上就是创建一个全新的 Goroutine，我们称它为 G。
    
2. 新创建的 G 会被放入 P 的本地队列（Local Queue）或全局队列（Global Queue）中，准备下一步的动作。需要注意的一点，这里的 P 指的是创建 G 的 P。
    
3. 唤醒或创建 M 以便执行 G。
    
4. 不断地进行事件循环
    
5. 寻找在可用状态下的 G 进行执行任务
    
6. 清除后，重新进入事件循环

在描述中有提到全局和本地这两类队列，其实在功能上来讲都是用于存放正在等待运行的 G，但是不同点在于，本地队列有数量限制，不允许超过 256 个。

并且在新建 G 时，会优先选择 P 的本地队列，如果本地队列满了，则将 P 的本地队列的一半的 G 移动到全局队列。

这可以理解为调度资源的共享和再平衡。

## 窃取行为
其实当 P 执行 G 完毕后，它也会 “干活”，它会将其从本地队列中弹出 G，同时会检查当前本地队列是否为空，如果为空会随机的从其他 P 的本地队列中尝试窃取一半可运行的 G 到自己的名下。

## M 的限制
第一，要知道**在协程的执行中，真正干活的是 GPM 中的哪一个**？

那势必是 M（系统线程） 了，因为 G 是用户态上的东西，最终执行都是得映射，对应到 M 这一个系统线程上去运行。

那么 M 有没有限制呢？

答案是：有的。在 Go 语言中，**M 的默认数量限制是 10000**，如果超出则会报错：

通常只有在 Goroutine 出现阻塞操作的情况下，才会遇到这种情况。这可能也预示着你的程序有问题。

若确切是需要那么多，还可以通过 `debug.SetMaxThreads` 方法进行设置。

## G 的限制
第二，那 G 呢，Goroutine 的创建数量是否有限制？

答案是：没有。但**理论上会受内存的影响**，假设一个 Goroutine 创建需要 4k（via @GoWKH）：

- 4k * 80,000 = 320,000k ≈ 0.3G内存
    
- 4k * 1,000,000 = 4,000,000k ≈ 4G内存
    

以此就可以相对计算出来一台单机在通俗情况下，所能够创建 Goroutine 的大概数量级别。

注：Goroutine 创建所需申请的 2-4k 是需要连续的内存块。

## 何为之合理

在介绍完 GMP 各自的限制后，我们回到一个重点，就是 “Goroutine 数量怎么预算，才叫合理？”。

“合理” 这个词，是需要看具体场景来定义的，可结合上述对 GPM 的学习和了解。得出：

- M：有限制，默认数量限制是 10000，可调整。
    
- G：没限制，但受内存影响。
    
- P：受本机的核数影响，可大可小，不影响 G 的数量创建。
    

Goroutine 数量在 MG 的可控限额以下，多个把个、几十个，少几个其实没有什么影响，就可以称其为 “合理”。

“goroutine 泄露”是 Go 语言中一个常见但容易被忽视的问题，指的是某些 goroutine 在程序中被创建后，因为某些原因 **永远无法退出**，从而一直占用资源，最终可能导致程序崩溃或性能显著下降。

---

### 🔍 goroutine 泄露的常见原因：

#### 1. **channel 阻塞**

```go
ch := make(chan int)
go func() {
    ch <- 1 // 如果没有人从 ch 中读取，这里会阻塞
}()
```

如果没人从 `ch` 中读取，这个 goroutine 永远卡在发送操作上。

---

#### 2. **缺失的退出信号 / 上下文控制**

```go
func worker() {
    for {
        // 做点什么
    }
}
```

这种写法没有任何终止条件，会让 goroutine 永远运行。

正确做法应加入 `context.Context` 或其他终止信号：

```go
func worker(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            return
        default:
            // 做点什么
        }
    }
}
```

---

#### 3. **从不关闭的 channel 被 select 阻塞**

```go
ch := make(chan int)
go func() {
    select {
    case <-ch: // 如果 ch 永远不关闭，这里会永远阻塞
    }
}()
```

---

#### 4. **资源未清理 / 管道未消费完**

```go
func readLines(reader io.Reader) <-chan string {
    out := make(chan string)
    go func() {
        defer close(out)
        scanner := bufio.NewScanner(reader)
        for scanner.Scan() {
            out <- scanner.Text()
        }
    }()
    return out
}

func main() {
    lines := readLines(os.Stdin)
    // 没有消费 lines，会导致 goroutine 卡住
}
```

---

### 🔎 如何检测 goroutine 泄露

#### 1. 使用 Go 提供的 runtime API 查看 goroutine 数：

```go
fmt.Println("Number of goroutines:", runtime.NumGoroutine())
```

#### 2. 使用 pprof 分析 goroutine 堆栈：

```bash
go run -memprofile mem.prof ...
go tool pprof -http=:8080 ./app mem.prof
```

---

### 🛠️ 解决方案 / 建议：

- 所有的 goroutine **都要有退出路径**。
    
- 使用 `context.WithCancel` / `context.WithTimeout` 管理生命周期。
    
- 所有的 channel 都应该由“写的人”关闭（如果没有多写者）。
    
- 避免泄露的方式是将所有 goroutine 的启动和结束控制在你的预期内。
    

# 为什么 P 的逻辑不直接加在 M 上
主要还是因为`M`其实是**内核**线程，内核只知道自己在跑线程，而`golang`的运行时（包括调度，垃圾回收等）其实都是**用户空间**里的逻辑。操作系统内核哪里还知道，也不需要知道用户空间的 golang 应用原来还有那么多花花肠子。这一切逻辑交给应用层自己去做就好，毕竟改内核线程的逻辑也不合适啊。

