Go 提供了多种方式来实现同步，包括 **goroutines** 和 **channel**，以及 **sync** 包中的同步原语（如 `Mutex` 和 `WaitGroup`）。以下是一些常见的同步机制示例。

### 示例 1: 使用 `sync.Mutex` 进行互斥锁同步

`sync.Mutex` 可以用来保护共享资源，确保在同一时刻只有一个 goroutine 可以访问该资源。

```go
package main

import (
	"fmt"
	"sync"
)

var counter int
var mu sync.Mutex

// 增加计数
func increment() {
	mu.Lock()         // 锁住共享资源
	defer mu.Unlock() // 确保在函数结束时解锁
	counter++
}

func main() {
	var wg sync.WaitGroup

	// 启动 10 个 goroutines，增加计数
	for i := 0; i < 10; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			increment()
		}()
	}

	wg.Wait() // 等待所有 goroutines 完成

	// 打印最终计数
	fmt.Println("Final counter:", counter)
}
```

#### 解释：

- `mu.Lock()` 和 `mu.Unlock()` 用于在访问共享资源（`counter`）时加锁和解锁，确保只有一个 goroutine 可以在同一时刻访问该资源。
- `sync.WaitGroup` 用于等待所有 goroutines 执行完成。

### 示例 2: 使用 `sync.WaitGroup` 等待多个 goroutine 完成

`sync.WaitGroup` 用于等待一组 goroutines 完成。

```go
package main

import (
	"fmt"
	"sync"
)

func printMessage(msg string, wg *sync.WaitGroup) {
	defer wg.Done() // 在函数返回时通知 WaitGroup
	fmt.Println(msg)
}

func main() {
	var wg sync.WaitGroup

	messages := []string{"Hello", "Go", "Concurrency", "with", "WaitGroup"}

	// 启动多个 goroutines
	for _, msg := range messages {
		wg.Add(1) // 增加 WaitGroup 计数
		go printMessage(msg, &wg)
	}

	wg.Wait() // 等待所有 goroutines 完成
	fmt.Println("All messages printed.")
}
```

#### 解释：

- `wg.Add(1)` 表示增加一个计数，表示有一个新的 goroutine 启动。
- `defer wg.Done()` 在每个 goroutine 完成时减少计数。
- `wg.Wait()` 阻塞主 goroutine，直到所有计数器值为零，表示所有 goroutine 完成。

### 示例 3: 使用 Channel 进行同步

Go 的 Channel 可以用于 goroutines 之间的同步。Channel 也可以用来传递数据。

```go
package main

import (
	"fmt"
)

func worker(id int, ch chan string) {
	// 模拟工作
	fmt.Printf("Worker %d is working...\n", id)
	ch <- fmt.Sprintf("Worker %d finished", id) // 将完成消息发送到 channel
}

func main() {
	ch := make(chan string)

	// 启动多个 goroutines
	for i := 1; i <= 5; i++ {
		go worker(i, ch)
	}

	// 接收所有 goroutines 的结果
	for i := 1; i <= 5; i++ {
		msg := <-ch
		fmt.Println(msg)
	}

	fmt.Println("All workers finished.")
}
```

#### 解释：

- 每个 `worker` goroutine 在完成工作后将结果发送到 `ch`（一个 `channel`）。
- 主 goroutine 等待从 `ch` 接收到所有工作完成的消息，从而同步所有 goroutines 的执行。

### 示例 4: 使用 `sync.Once` 进行单次执行

`sync.Once` 确保一个操作只会执行一次。常用于初始化操作。

```go
package main

import (
	"fmt"
	"sync"
)

var once sync.Once

func initialize() {
	fmt.Println("Initializing...")
}

func main() {
	// 启动多个 goroutines
	for i := 0; i < 5; i++ {
		go func() {
			once.Do(initialize) // 确保 initialize 只执行一次
		}()
	}

	// 等待 goroutines 完成
	fmt.Scanln() // 阻塞主线程
}
```

#### 解释：

- `once.Do(initialize)` 确保 `initialize` 函数在所有 goroutines 中只执行一次，即使调用多次。
- `sync.Once` 可以用来保证一个初始化操作只执行一次，适用于多线程/多 goroutine 的并发场景。

---

### 总结：

- `sync.Mutex` 用于互斥锁，防止多个 goroutine 同时访问共享资源。
- `sync.WaitGroup` 用于等待多个 goroutine 完成工作。
- `channel` 用于在 goroutines 之间传递数据或进行同步。
- `sync.Once` 确保某个操作只执行一次，适用于初始化等场景。

这些同步机制是 Go 并发编程的基本工具，可以帮助我们在多 goroutine 的场景中保证数据一致性和正确性。