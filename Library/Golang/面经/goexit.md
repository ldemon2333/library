在 Go 的运行时包 (`runtime`) 中，`Goexit` 是一个用来立刻终止当前 goroutine 的低级函数。它的行为与普通的函数返回或 `panic` 有显著区别：

- **函数签名**
    
    ```go
    func Goexit()
    ```
    
    它没有参数，也没有返回值，而且**永远不会返回**到调用它的那行代码之后。
    
- **主要特性**
    
    1. **运行所有未执行的 `defer`**  
        在调用 `Goexit` 之后，当前 goroutine 中已经注册但尚未执行的所有 `defer` 都会**按 LIFO 顺序**依次执行。
        
    2. **终止 goroutine**  
        执行完所有 `defer` 后，goroutine 就彻底结束，不会继续执行后面的任何代码，也不会触发 panic 状态。
        
    3. **不影响其他 goroutine**  
        它只影响**当前** goroutine；其他 goroutine 继续按正常调度执行。
        
- **与其他方式的对比**
    
![[Pasted image 20250418222550.png]]

    
- **典型场景**
    
    - **测试框架**  
        在像 `testing` 这样的框架内部，当一个子测试结束后，需要立刻终止该子测试对应的 goroutine，但又要保证子测试注册的清理函数（`defer`）得到执行。这时就会调用 `runtime.Goexit`。
        
    - **内部运行时调度**  
        在 Go 的调度器（scheduler）实现中，也会在某些情况下用它来切换或回收 goroutine。
        
- **示例**
    
    ```go
    package main
    
    import (
        "fmt"
        "runtime"
    )
    
    func worker() {
        defer fmt.Println("defer in worker")  // 会被执行
        fmt.Println("worker start")
        runtime.Goexit()                     // 立即终止本 goroutine
        fmt.Println("this will NOT print")
    }
    
    func main() {
        go worker()
        // 给 worker 一点时间运行 defer
        select {}
    }
    ```
    
    输出：
    
    ```
    worker start
    defer in worker
    ```
    
- **底层实现要点**  
    在 Go 的源码中，`Goexit` 并不是一个普通的 Go 函数，而是由运行时的汇编代码实现的（在 `src/runtime/asm_*.s` 里），其核心逻辑大致是：
    
    1. 调用所有挂起的 `defer`（通过遍历 goroutine 的 defer 链表）。
        
    2. 将当前 goroutine 标记为“死”状态，并从调度队列中移除。
        
    3. 触发一次调度切换（`Gosched`），让其他可运行的 goroutine 获得 CPU。
        

综上，`runtime.Goexit` 是一个专门用于在保证 `defer` 执行的前提下，立刻终止当前 goroutine 的运行时原语。你在日常应用中几乎不用直接调用它，更多的是在实现并发框架或调度逻辑时被间接使用。

# runtime.Goexit() 是什么
内部实现
![[Pasted image 20250418222730.png]]
从代码上看，`runtime.Goexit()`会先执行一下`defer`里的方法，这里就解释了开头的代码里为什么**在 defer 里的打印 2**能正常输出。

![[Pasted image 20250418222802.png]]
所以简单总结一下，**只要执行 goexit 这个函数，当前协程就会退出，同时还能调度下一个可执行的协程出来跑。**

![[Pasted image 20250418224323.png]]
会发现，这个协程的堆栈底部是从`runtime.goexit()`里开始启动的。

如果大家平时有注意观察，会发现，**其实所有的堆栈底部，都是从这个函数开始的**。我们继续跟跟代码。

# goexit 是什么？
当程序执行结束的时候就都能跑到 `goexit` 里去，此时栈底的`goexit`，会在协程内的业务代码跑完后被执行到，从而实现协程退出，并调度下一个**可执行的 G**来运行。

那么问题又来了，栈底插入`goexit`这件事是谁做的，什么时候做的？

直接说答案，这个在`runtime/proc.go`里有个`newproc1`方法，只要是**创建协程**都会用到这个方法。里面有个地方是这么写的。
![[Pasted image 20250418224515.png]]


# main 函数也是个协程，栈底也是 goexit
main 函数栈底也是`goexit()`。

从 `asm_amd64.s`可以看到 Go 程序启动的流程，这里提到的 `runtime·mainPC` 其实就是 `runtime.main`.

![[Pasted image 20250418224715.png]]
通过`runtime·newproc`创建`runtime.main`协程，然后在`runtime.main`里会启动`main.main`函数，这个就是我们平时写的那个 main 函数了。

# os.Exit() 和 runtime.Goexit() 有什么区别

# 总结
- 通过 `runtime.Goexit()`可以做到提前结束协程，且结束前还能执行到 defer 的内容
- `runtime.Goexit()`其实是对 goexit0 的封装，只要执行 goexit0 这个函数，当前协程就会退出，同时还能调度下一个可执行的协程出来跑。
- 通过`newproc`可以创建出新的`goroutine`，它会在函数栈底部插入一个 goexit。
- `os.Exit()` 指的是整个**进程**退出；而`runtime.Goexit()`指的是**协程**退出。两者含义有区别。

