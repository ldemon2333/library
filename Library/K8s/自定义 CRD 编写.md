本文主要分析k8s中各个核心组件经常使用到的`Informer`机制(即List-Watch)。该部分的代码主要位于`client-go`这个第三方包中。

此部分的逻辑主要位于`/vendor/k8s.io/client-go/tools/cache`包中，代码目录结构如下：

k8s 的其他组件都是通过 client-go 的 informer 机制与 API Server 进行通信的。
![[Pasted image 20250526162607.png]]

![[Pasted image 20250526162413.png]]

# client-go 组件
- `Reflector`：reflector用来watch特定的k8s API资源。具体的实现是通过`ListAndWatch`的方法，watch可以是k8s内建的资源或者是自定义的资源。当reflector通过watch API接收到有关新资源实例存在的通知时，它使用相应的列表API获取新创建的对象，并将其放入watchHandler函数内的Delta Fifo队列中。
    
- `Informer`：informer从Delta Fifo队列中弹出对象。执行此操作的功能是processLoop。base controller的作用是保存对象以供以后检索，并调用我们的控制器将对象传递给它。
    
- `Indexer`：索引器提供对象的索引功能。典型的索引用例是基于对象标签创建索引。 Indexer可以根据多个索引函数维护索引。Indexer使用线程安全的数据存储来存储对象及其键。 在Store中定义了一个名为`MetaNamespaceKeyFunc`的默认函数，该函数生成对象的键作为该对象的`<namespace> / <name>`组合。

在 ListAndWatch 机制下，一旦 API Server 端有新的 Network 实例被创建、删除或者更新，Reflector 都会收到 “事件通知”。这是，该事件及其对应的 API 对象这个组合，就称为增量（delta），放入 delta FIFO queue 队列中。

Informer 会不断地从这个 delta FIFO 里读取 Pop 增量。每拿到一个增量，Informer 就会判断这个增量里的事件类型，然后创建或者更新本地对象的缓存（store）。

如果事件类型是 Added，那么 Informer 就会通过一个叫作 Indexer 的库把这个增量里的 API 对象保存在本地缓存中，并为它创建索引。如果增量的事件是 Deleted，那么 Informer 就会从本地缓存中删除这个对象。这个本地缓存的工作是 Informer 的首要职责。

Informer 的第二个职责，是根据这些事件的类型触发事先注册好的 ResourceEventHandler。
==它维护了一个本地缓存和一组注册的回调==

![[Pasted image 20250526174656.png]]

首先通过 kubernetes.NewForConfig 创建 clientSet 对象，Informer 需要通过 ClientSet 与 k8s API Server 进行交互。另外，创建 stopCh 对象，该对象用于在程序进程退出之前通知 Informer 提前退出。

informers.NewSharedInformerFactory 函数实例化了 SharedInformer 对象，它接收两个参数。第1个参数clientset是用于与Kubernetes API Server交互的客户端，第2个参数time.Minute用于设置多久进行一次resync（重新同步），resync会周期性地执行List操作，将所有的资源存放在Informer Store中，如果该参数为0，则禁用resync功能。

#### 1. **工厂模式（Factory Pattern）**

- `NewSharedInformerFactory` 就是一个**工厂函数**，它隐藏了创建各种具体 informer 的细节。
    
- 它统一对外提供类似 `factory.Core().V1().Pods()` 这样的接口，来获取某个资源的 informer。
    
- 开发者不需要知道 PodInformer 是如何构造的。
    

#### 2. **单例 + 共享内存缓存模式**

- Informer 工厂背后的 informer 是**共享的**。
    
- 如果你请求多次 `factory.Core().V1().Pods()`，你拿到的是同一个 informer，而不是每次新建一个。
    
- 节省资源、避免重复 watch 请求。

在Informers Example代码示例中，通过sharedInformers.Core().V1().Pods().Informer可以得到具体Pod资源的informer对象。通过informer.AddEventHandler函数可以为Pod资源添加资源事件回调方法，支持3种资源事件回调方法，分别介绍如下。
- AddFunc：当创建Pod资源对象时触发的事件回调方法。
- UpdateFunc：当更新Pod资源对象时触发的事件回调方法。
- DeleteFunc：当删除Pod资源对象时触发的事件回调方法。

在正常的情况下，Kubernetes的其他组件在使用Informer机制时触发资源事件回调方法，将资源对象推送到WorkQueue或其他队列中，在Informers Example代码示例中，我们直接输出触发的资源事件。最后通过informer.Run函数运行当前的Informer，内部为Pod资源类型创建Informer。

通过Informer机制可以很容易地监控我们所关心的资源事件，例如，当监控Kubernetes Pod资源时，如果Pod资源发生了Added（资源添加）事件、Updated（资源更新）事件、Deleted（资源删除）事件，就通知client-go，告知Kubernetes资源事件变更了并且需要进行相应的处理。

# 资源 Informer
每一个Kubernetes资源上都实现了Informer机制。每一个Informer上都会实现Informer和Lister方法，例如PodInformer，代码示例如下：

vendor/k8s.io/client-go/informers/core/v1/pod.go

![[Pasted image 20250526192906.png]]

调用不同资源的Informer，代码示例如下：

```text
informer := shardInformer.Core().V1().Pods().Informer()
nodeinformer := shardInformer.Node().V1beta1().RuntimeClasses().Informer()
```
定义不同资源的Informer，允许监控不同资源的资源事件，例如，监听Node资源对象，当Kubernetes集群中有新的节点（Node）加入时，client-go能够及时收到资源对象的变更信息。

# Shared Informer 共享机制
Informer也被称为Shared Informer，它是可以共享使用的。在用client-go编写代码程序时，若同一资源的Informer被实例化了多次，每个Informer使用一个Reflector，那么会运行过多相同的ListAndWatch，太多重复的序列化和反序列化操作会导致Kubernetes API Server负载过重。

Shared Informer可以使同一类资源Informer共享一个Reflector，这样可以节约很多资源。通过map数据结构实现共享的Informer机制。Shared Informer定义了一个map数据结构，用于存放所有Informer的字段，代码示例如下：

vendor/k8s.io/client-go/informers/factory.go
![[Pasted image 20250526193343.png]]

![[Pasted image 20250526193645.png]]

informers字段中存储了资源类型和对应于SharedIndexInformer的映射关系。InformerFor函数添加了不同资源的Informer，在添加过程中如果已经存在同类型的资源Informer，则返回当前Informer，不再继续添加。

最后通过Shared Informer的Start方法使f.informers中的每个informer通过goroutine持久运行。


`informer` 和 `lister` 是 Kubernetes 中控制器编程模型里的两个关键接口，它们分别处理资源的**事件监听**和**本地缓存读取**，是构建高效控制器的核心部分。

---

## 一、先看它们的典型调用方式

```go
podInformer := factory.Core().V1().Pods()
informer := podInformer.Informer()  // 1. 事件监听
lister := podInformer.Lister()      // 2. 本地缓存读取
```

---

## 二、两者的核心区别和职责

|方法|作用|本质|用途|
|---|---|---|---|
|`Informer()`|监听资源事件（Add/Update/Delete）|注册事件回调并维护本地缓存|异步响应资源变化|
|`Lister()`|从本地缓存中查询资源|查询已经缓存的对象，非事件|快速访问已有资源，避免重复 API 调用|

---

## 三、`Informer()` 详解

### 作用：

- 启动一个 `SharedIndexInformer`，监听某种 Kubernetes 资源的增删改事件。
    
- 在本地维护一份缓存副本（基于 List + Watch）
    
- 注册事件回调函数（通过 `AddEventHandler`）
    

### 方法签名：

```go
func (f *podInformer) Informer() cache.SharedIndexInformer
```

### 用法示例：

```go
informer := podInformer.Informer()
informer.AddEventHandler(cache.ResourceEventHandlerFuncs{
    AddFunc: func(obj interface{}) {
        pod := obj.(*v1.Pod)
        fmt.Println("Pod added:", pod.Name)
    },
})
```

你注册的这些事件处理函数会在资源发生变化时自动触发。

---

## 四、`Lister()` 详解

### 作用：

- 提供一个读取缓存中数据的接口。
    
- **不需要发起 API 请求**，速度快，避免对 kube-apiserver 的压力。
    
- 一般用于你的业务逻辑中：比如 controller 的 `Reconcile` 函数内。
    

### 方法签名：

```go
func (f *podInformer) Lister() listersv1.PodLister
```

`PodLister` 接口有很多查询方法，例如：

```go
pods, err := lister.Pods("default").List(labels.Everything())
pod, err := lister.Pods("default").Get("mypod")
```

注意：这里的数据是 informer 缓存中的**快照**，不是最新的状态，但一致性是可接受的。

---

## 五、配合使用：一个控制器的典型流程

```go
1. podInformer.Informer().AddEventHandler(...)   // 监听事件
2. controller <- workqueue <- enqueue object key // 入队
3. controller 的 Reconcile(key) 执行业务逻辑
4. 在 Reconcile 中通过 podInformer.Lister() 查询对象状态
```

---

## 六、图解（逻辑关系）

```
        API Server
            ↑
       [List + Watch]
            ↓
   SharedIndexInformer ←─── Informer()
          ↓
      本地缓存 ←──────────── Lister()
          ↓
    调用你的处理逻辑
```

---

## 七、总结

|接口|类型|用途|是否访问 API Server|
|---|---|---|---|
|`Informer()`|`cache.SharedIndexInformer`|监听资源事件|是（List+Watch）|
|`Lister()`|`PodLister` 等|查询 informer 缓存|否，本地|



# Workqueue
这份 Go 代码是一个非常经典的 Kubernetes **控制器 (Controller)** 的示例实现，使用了 `client-go` 库。这个控制器会监听 Pod 资源的变化，并对这些变化进行简单的处理（打印日志）。它展示了 Kubernetes 控制器模式的核心组成部分：**Informer、Workqueue 和业务逻辑**。

---
### **导入必要的库**

```go
import (
	"context" // 用于上下文管理，控制 Goroutine 生命周期
	"errors"  // 错误处理
	"flag"    // 命令行参数解析
	"fmt"     // 格式化输入输出
	"time"    // 时间操作

	"k8s.io/klog/v2" // Kubernetes 风格的日志库

	v1 "k8s.io/api/core/v1" // Kubernetes 核心 API 组的 v1 版本 (如 Pod, Service)
	meta_v1 "k8s.io/apimachinery/pkg/apis/meta/v1" // Kubernetes API 的元数据类型 (如 ObjectMeta)
	"k8s.io/apimachinery/pkg/fields" // 字段选择器
	"k8s.io/apimachinery/pkg/util/runtime" // 运行时工具，用于处理 panic 和错误
	"k8s.io/apimachinery/pkg/util/wait" // 等待工具，用于 Goroutine 循环
	"k8s.io/client-go/kubernetes" // Kubernetes Clientset，用于与 API Server 通信
	"k8s.io/client-go/tools/cache" // Informer 和 Indexer 相关的工具
	"k8s.io/client-go/tools/clientcmd" // Kubeconfig 文件加载工具
	"k8s.io/client-go/util/workqueue" // 工作队列，用于异步处理事件
)
```

- 导入了构建 Kubernetes 控制器所需的所有核心 `client-go` 包。
- `klog/v2` 是 Kubernetes 推荐的日志库。

---

### **`Controller` 结构体**
``` go
// Controller demonstrates how to implement a controller with client-go.
type Controller struct {
	indexer  cache.Indexer // 本地缓存，存储 Pod 对象
	queue    workqueue.TypedRateLimitingInterface[string] // 工作队列，存储待处理的 Pod Key
	informer cache.Controller // Informer 的控制器部分，负责 List/Watch 和事件分发
}
```

- 这是控制器的核心结构体，包含了控制器所需的三个主要组件：
    - **`indexer cache.Indexer`**: 这是 Informer 的本地缓存。控制器通过这个缓存来获取其关心的 Kubernetes 对象（这里是 Pod），而无需每次都向 API Server 发送请求。它还支持通过字段进行索引，加速查找。
    - **`queue workqueue.TypedRateLimitingInterface[string]`**: 这是一个带有速率限制的工作队列。Informer 发现的事件（Pod 的添加、更新、删除）不会立即处理，而是将相应 Pod 的唯一标识符（`namespace/name`，即 `key`）添加到这个队列中。工作队列能够处理事件的去重、重试和并发处理。
    - **`informer cache.Controller`**: 这是 `client-go` Informer 的核心控制器接口。它负责执行 List-Watch 循环，填充 `indexer` 缓存，并将收到的事件（通过 `ResourceEventHandlerFuncs`）传递给注册的处理器。

---

### **`NewController` 函数**
```go
// NewController creates a new Controller.
func NewController(queue workqueue.TypedRateLimitingInterface[string], indexer cache.Indexer, informer cache.Controller) *Controller {
	return &Controller{
		informer: informer,
		indexer:  indexer,
		queue:    queue,
	}
}
```

- 这是一个简单的工厂函数，用于创建 `Controller` 结构体的新实例。它接收 `queue`、`indexer` 和 `informer` 作为参数，并在控制器启动时使用它们。

---

### **`processNextItem` 方法**
``` go
func (c *Controller) processNextItem() bool {
	// 等待工作队列中有新项
	key, quit := c.queue.Get()
	if quit { // 如果队列关闭，则退出
		return false
	}
	// 告知队列我们已处理完此 key。这会解锁 key，允许其他 worker 并行处理。
	// 这允许安全的并行处理，因为具有相同 key 的两个 Pod 永远不会并行处理。
	defer c.queue.Done(key) // 确保 item 处理完成后调用 Done()

	// 调用包含业务逻辑的方法
	err := c.syncToStdout(key)
	// 处理业务逻辑执行过程中出现的错误
	c.handleErr(err, key)
	return true // 返回 true 表示继续处理下一个 item
}
```

- 这是控制器工作循环中的一个步。它从工作队列中获取一个 `key`（通常是 `namespace/name` 格式的资源标识符）。
- `c.queue.Get()`: 阻塞式地从队列中取出下一个待处理的 `key`。如果队列被关闭，`quit` 会是 `true`。
- `defer c.queue.Done(key)`: **非常重要**。这会延迟执行，直到 `processNextItem` 返回。它告诉队列，当前 `key` 已经处理完毕。`Done()` 调用会解锁 `key`，允许它再次被添加并处理（如果将来有新的事件）。
- `c.syncToStdout(key)`: 调用核心业务逻辑。
- `c.handleErr(err, key)`: 处理业务逻辑返回的错误，决定是否需要重试。

---

### **`syncToStdout` 方法**


``` go
// syncToStdout is the business logic of the controller. In this controller it simply prints
// information about the pod to stdout. In case an error happened, it has to simply return the error.
// The retry logic should not be part of the business logic.
func (c *Controller) syncToStdout(key string) error {
	obj, exists, err := c.indexer.GetByKey(key) // 从本地缓存中获取对象
	if err != nil {
		klog.Errorf("Fetching object with key %s from store failed with %v", key, err)
		return err // 获取失败直接返回错误
	}

	if !exists {
		// 这是处理删除事件的方式，因为对象不再存在于缓存中
		// 下面我们会用一个 Pod 来预热缓存，这样我们会看到一个 Pod 的删除事件
		fmt.Printf("Pod %s does not exist anymore\n", key)
	} else {
		// 注意，如果你有一个本地控制的资源，并且它依赖于实际的实例，
		// 你还需要检查 UID 来检测一个 Pod 是否以相同的名称被重新创建。
		fmt.Printf("Sync/Add/Update for Pod %s\n", obj.(*v1.Pod).GetName())
	}
	return nil // 业务逻辑成功，返回 nil
}
```

- 这是控制器**实际的业务逻辑**所在。在这个示例中，它非常简单：
    - `c.indexer.GetByKey(key)`: 尝试从 Informer 的本地缓存 (`indexer`) 中根据 `key` 获取对应的 Pod 对象。这是控制器从缓存获取数据的主要方式。
    - **处理对象存在与否**：
        - `if !exists`: 如果对象在缓存中不存在，这通常意味着它已经被删除了（因为 `DeleteFunc` 也会将 `key` 加入队列）。
        - `else`: 如果对象存在，表示是添加或更新事件。
    - **日志输出**：简单地打印 Pod 的状态信息。
    - **错误处理**：业务逻辑只负责返回错误，不处理重试。重试逻辑由 `handleErr` 方法和 `workqueue` 负责。

---

### **`handleErr` 方法**

Go

```
// handleErr checks if an error happened and makes sure we will retry later.
func (c *Controller) handleErr(err error, key string) {
	if err == nil {
		// 在每次成功同步时，清除 key 在 AddRateLimited 中的历史记录。
		// 这确保了将来对该 key 的更新处理不会因为过时的错误历史而被延迟。
		c.queue.Forget(key) // 成功处理后，忘记 key 的错误历史
		return
	}

	// 这个控制器在出错时会重试 5 次。之后，它将停止尝试。
	if c.queue.NumRequeues(key) < 5 {
		klog.Infof("Error syncing pod %v: %v", key, err)

		// 将 key 以速率限制的方式重新加入队列。根据队列的速率限制器和重新排队历史，
		// 该 key 将在稍后再次处理。
		c.queue.AddRateLimited(key) // 重试，但会遵循速率限制
		return
	}

	c.queue.Forget(key) // 超过重试次数，忘记 key
	// 报告给外部实体，即使经过多次重试，也未能成功处理此 key
	runtime.HandleError(err) // 将无法处理的错误报告给 runtime 错误处理机制
	klog.Infof("Dropping pod %q out of the queue: %v", key, err) // 记录放弃
}
```

- 这个方法是工作队列（`workqueue`）**重试机制**的核心：
    - **无错误**：如果 `syncToStdout` 没有返回错误 (`err == nil`)，则调用 `c.queue.Forget(key)`。这会清除该 `key` 的重试历史，确保下次它被添加到队列时，不会受到之前重试延迟的影响。
    - **有错误且未达重试上限**：如果 `syncToStdout` 返回错误，并且该 `key` 的重试次数 (`c.queue.NumRequeues(key)`) 未达到 5 次，则调用 `c.queue.AddRateLimited(key)`。这会将 `key` 重新加入队列，但会根据 `workqueue` 配置的速率限制器进行延迟，实现指数退避或线性退避重试。
    - **有错误且达到重试上限**：如果重试次数已达 5 次，则调用 `c.queue.Forget(key)`（不再重试），并通过 `runtime.HandleError(err)` 报告致命错误，并打印日志表示放弃处理该 `key`。

---

### **`Run` 方法**

Go

```
// Run begins watching and syncing.
func (c *Controller) Run(ctx context.Context, workers int) {
	defer runtime.HandleCrashWithContext(ctx) // 确保在发生 panic 时捕获并处理

	// 在我们完成时让 worker 停止
	defer c.queue.ShutDown() // 确保在控制器停止时关闭队列
	klog.Info("Starting Pod controller")

	go c.informer.RunWithContext(ctx) // 启动 informer 的 List-Watch 循环

	// 在开始处理队列中的项之前，等待所有相关缓存同步完成
	if !cache.WaitForNamedCacheSyncWithContext(ctx, "pod", c.informer.HasSynced) { // 注意：这里的 "pod" 是命名，不是资源类型
		runtime.HandleError(fmt.Errorf("Timed out waiting for caches to sync"))
		return
	}

	// 启动 worker goroutine
	for i := 0; i < workers; i++ {
		go wait.UntilWithContext(ctx, c.runWorker, time.Second) // 启动 worker 循环，每秒检查一次
	}

	<-ctx.Done() // 阻塞直到上下文被取消
	klog.Info("Stopping Pod controller")
}
```

- 这是启动控制器的主入口点。
- `defer runtime.HandleCrashWithContext(ctx)`: 用于捕获和处理 `panic`，确保控制器即使在遇到不可恢复的错误时也能优雅地退出。
- `defer c.queue.ShutDown()`: 确保在控制器关闭时，工作队列也被正确关闭，释放资源。
- `go c.informer.RunWithContext(ctx)`: **关键一步**。这会在一个新的 Goroutine 中启动 Informer 的 List-Watch 循环。Informer 会开始与 Kubernetes API Server 通信，填充其本地缓存 (`indexer`)。
- `if !cache.WaitForNamedCacheSyncWithContext(ctx, "pod", c.informer.HasSynced)`: 阻塞等待 Informer 的本地缓存完成初始同步。`c.informer.HasSynced` 是一个函数，当缓存同步完成后返回 `true`。这是非常重要的，因为它确保控制器在处理事件之前，能够有一个完整且一致的集群状态视图。
- `for i := 0; i < workers; i++ { go wait.UntilWithContext(ctx, c.runWorker, time.Second) }`: 启动指定数量的**worker Goroutine**。每个 worker 都会在一个无限循环中调用 `c.processNextItem()` 来从队列中取事件并处理。`wait.UntilWithContext` 会在 `ctx` 被取消时停止这些 worker。
- `<-ctx.Done()`: 阻塞主 Goroutine，直到传入的 `ctx` 上下文被取消。这使得控制器一直运行，直到外部信号（如 `SIGTERM`）导致上下文取消。

---

### **`runWorker` 方法**

```
func (c *Controller) runWorker(ctx context.Context) {
	for c.processNextItem() {
	}
}
```

- 这是一个简单的循环函数，它会不断调用 `c.processNextItem()`，直到 `processNextItem` 返回 `false`（即队列被关闭）。每个 worker 都会运行自己的 `runWorker` 循环。

---

### **`main` 函数**

```go
func main() {
	var kubeconfig string // kubeconfig 文件路径
	var master string     // master API Server URL

	flag.StringVar(&kubeconfig, "kubeconfig", "", "absolute path to the kubeconfig file")
	flag.StringVar(&master, "master", "", "master url")
	flag.Parse() // 解析命令行参数

	// 建立连接
	config, err := clientcmd.BuildConfigFromFlags(master, kubeconfig) // 从 kubeconfig 或 master URL 构建 REST 配置
	if err != nil {
		klog.Fatal(err) // 错误处理
	}

	// 创建 clientset
	clientset, err := kubernetes.NewForConfig(config) // 基于 REST 配置创建 Clientset
	if err != nil {
		klog.Fatal(err) // 错误处理
	}

	ctx := context.Background() // 创建一个基本的上下文

	// 创建 Pod Watcher
	// NewListWatchFromClient 创建一个 ListWatcher，用于 Informer 的 Reflector 部分
	podListWatcher := cache.NewListWatchFromClient(clientset.CoreV1().RESTClient(), "pods", v1.NamespaceDefault, fields.Everything())

	// 创建工作队列
	queue := workqueue.NewTypedRateLimitingQueue(workqueue.DefaultTypedControllerRateLimiter[string]()) // 创建带速率限制的队列

	// 通过 Informer 将工作队列绑定到缓存。这样，每当缓存更新时，Pod 的 key 就会添加到工作队列中。
	// 注意，当最终从工作队列处理该项时，我们可能会看到比触发更新的 Pod 版本更新的版本。
	indexer, informer := cache.NewIndexerInformer(podListWatcher, &v1.Pod{}, 0, cache.ResourceEventHandlerFuncs{
		AddFunc: func(obj interface{}) {
			key, err := cache.MetaNamespaceKeyFunc(obj) // 获取对象的 key (namespace/name)
			if err == nil {
				queue.Add(key) // 添加到队列
			}
		},
		UpdateFunc: func(old interface{}, new interface{}) {
			key, err := cache.MetaNamespaceKeyFunc(new) // 获取新对象的 key
			if err == nil {
				queue.Add(key) // 添加到队列
			}
		},
		DeleteFunc: func(obj interface{}) {
			// IndexerInformer 使用一个 delta 队列，因此对于删除操作，我们必须使用这个 key 函数。
			key, err := cache.DeletionHandlingMetaNamespaceKeyFunc(obj) // 获取删除对象的 key
			if err == nil {
				queue.Add(key) // 添加到队列
			}
		},
	}, cache.Indexers{}) // 注意：这里没有定义自定义索引器，所以传递一个空的 Indexers map。

	controller := NewController(queue, indexer, informer) // 创建控制器实例

	// 现在我们可以预热缓存以进行初始同步。
	// 假设我们上次运行知道一个名为 "mypod" 的 Pod，因此将其添加到缓存中。
	// 如果这个 Pod 不再存在，控制器将在缓存同步后收到删除通知。
	indexer.Add(&v1.Pod{ // 预热缓存，模拟上次运行遗留的 Pod
		ObjectMeta: meta_v1.ObjectMeta{
			Name:      "mypod",
			Namespace: v1.NamespaceDefault,
		},
	})

	// 现在启动控制器
	cancelCtx, cancel := context.WithCancelCause(ctx) // 创建带 cause 的上下文，用于优雅关闭
	defer cancel(errors.New("time to stop because main has completed")) // 确保 main 函数退出时取消上下文
	go controller.Run(cancelCtx, 1) // 在新 Goroutine 中启动控制器，使用 1 个 worker

	// 永远等待（阻塞主 Goroutine）
	select {}
}
```

- **命令行参数**：定义并解析 `kubeconfig` 和 `master` 命令行参数，用于连接 Kubernetes 集群。
- **构建 `RESTConfig` 和 `Clientset`**：
    - `clientcmd.BuildConfigFromFlags`: 这是从 `kubeconfig` 文件或命令行参数构建连接 Kubernetes API Server 所需配置的标准方法。
    - `kubernetes.NewForConfig`: 使用构建好的配置创建 `kubernetes.Clientset`。这是控制器与 API Server 交互的客户端。
- **创建 `ListWatcher`**:
    - `cache.NewListWatchFromClient(clientset.CoreV1().RESTClient(), "pods", v1.NamespaceDefault, fields.Everything())`: 这是 Informer 内部 Reflector 组件所需的对象。它定义了 Informer 应该监听什么资源 (`pods`)、在哪个命名空间 (`v1.NamespaceDefault`)，以及如何筛选 (`fields.Everything()` 表示不按字段筛选)。它需要一个 `RESTClient` 才能与 API Server 交互。
- **创建 `workqueue`**:
    - `workqueue.NewTypedRateLimitingQueue(...)`: 创建一个有速率限制的工作队列，用于存储待处理的 Pod `key`。
- **创建 `Informer` 和 `Indexer`**:
    - `cache.NewIndexerInformer(...)`: 这是核心的 Informer 实例化方法。它接收：
        - `podListWatcher`: 定义了 Informer 监听哪些资源。
        - `&v1.Pod{}`: 期望监听的 Go 对象类型。
        - `0`: resync 周期（这里是 0，表示不进行周期性全量同步）。
        - `cache.ResourceEventHandlerFuncs`: 一组回调函数，当 Informer 收到 `Add`、`Update`、`Delete` 事件时，会将相应 Pod 的 `key` 添加到 `queue` 中。这是将 Informer 与 Workqueue 关联起来的关键。
        - `cache.Indexers{}`: 允许你定义自定义索引，以便在缓存中进行更复杂的查找。
- **预热缓存 (`indexer.Add(...)`)**:
    - 这是一个可选但很有趣的例子。它在 Informer 启动前，手动向 `indexer` 缓存中添加了一个名为 "mypod" 的 Pod。
    - **目的**：为了演示即使 Informer 启动时集群中不存在 "mypod"（或者状态与手动添加的不同），一旦缓存同步完成，Informer 也能检测到这个 Pod 的“缺失”或“不一致”，并触发相应的 `Delete` 或 `Update` 事件（通过 `handleErr` 最终处理）。
- **启动控制器**：
    - `go controller.Run(cancelCtx, 1)`: 在一个新的 Goroutine 中启动控制器。`1` 表示只启动一个 worker 来处理队列中的事件。
    - `select {}`: 阻塞 `main` Goroutine 永远运行，直到程序被外部信号（如 `Ctrl+C`）终止，从而导致 `cancelCtx` 被取消。

---

### **核心控制器模式总结**

这个代码示例完美地展示了 Kubernetes 控制器模式的几个核心思想：

1. **声明式 API**：控制器监听期望状态，而不是命令式地执行操作。
2. **Informer**: 作为与 API Server 交互的桥梁，高效地获取和缓存资源，并提供事件通知。
3. **Workqueue**: 异步处理事件，提供去重、重试和并发控制机制。
4. **业务逻辑 (`syncToStdout`)**: 专注于根据期望状态和当前状态之间的差异来执行调谐（reconcile）操作。
5. **韧性 (Resilience)**：通过 `workqueue` 的速率限制重试和 `runtime.HandleError` 来处理瞬时错误和致命错误，确保控制器在面对问题时依然健壮。

这是一个非常好的起点，可以让你理解如何构建自己的 Kubernetes 控制器来自动化管理集群中的资源。



# CRD 性能瓶颈
Custom Resource Definitions (CRDs) 是 Kubernetes 扩展性的基石，允许你定义自己的 API 对象类型，并像管理内置资源一样管理它们。然而，如果使用不当，CRDs 也可能成为集群的性能瓶颈。

---

### CRD 性能瓶颈的常见场景

CRD 导致的性能问题通常发生在 **Kubernetes 控制平面**，特别是 **API Server** 和作为后端存储的 **etcd**。以下是一些常见的场景：

1. **大量 Custom Resources (CRs) 实例：**
    
    - **场景：** 当你的集群中存在**数万甚至数十万个**由同一个 CRD 定义的 Custom Resources 实例时。例如，一个物联网 (IoT) 平台可能为每个设备定义一个 CR，或者一个多租户 SaaS 平台为每个客户或租户部署大量自定义资源。
        
    - **原因：** 每个 CR 实例都需要存储在 etcd 中。大量的 CRs 会显著增加 etcd 的存储负担，导致：
        
        - **etcd 写延迟增加：** 每次 CR 的创建、更新、删除都会引起 etcd 的写操作，大量操作会使 etcd 饱和。
            
        - **etcd 读延迟增加：** API Server 及其 watch 机制需要从 etcd 中读取这些大量的 CRs，即使是 List 操作（尤其是在 Informer 重新同步时）也会变得非常慢。
            
        - **etcd 内存/CPU 消耗：** etcd 需要将大量数据加载到内存中进行处理，并消耗 CPU 进行 Raft 共识协议。
            
    - **影响：** 整个集群的 API 响应变慢，控制器同步延迟增加，甚至可能导致 API Server 过载或 etcd 集群不稳定。
        
2. **频繁更新或高“变动率” (Churn Rate) 的 CRs：**
    
    - **场景：** 某些 CRs 的状态会**非常频繁地更新**。例如，如果一个 CR 包含了设备实时遥测数据，或者一个控制器频繁更新 CR 的 `status` 字段以反映其复杂状态。
        
    - **原因：** Kubernetes 的 watch 机制是基于 `resourceVersion` 的。每次 CR 更新，其 `resourceVersion` 都会递增，并在 etcd 和 API Server 的 watch cache 中产生新的事件。高变动率意味着：
        
        - **API Server Watch Cache 压力：** 大量的事件涌入 watch cache，可能导致其快速增长，消耗大量内存，甚至需要频繁清理过期事件。
            
        - **API Server CPU 消耗：** API Server 需要处理、序列化和分发这些大量的事件给所有监听的客户端。
            
        - **客户端负载：** 所有监听这些 CRs 的控制器和客户端都需要接收、处理这些事件，更新本地 Informer 缓存，并可能触发工作队列的任务。
            
    - **影响：** API Server 响应变慢，控制器的 `reconciliation loop` 变得繁忙，可能无法及时处理其他重要的集群事件。
        
3. **大型或嵌套复杂的 CRD Schema：**
    
    - **场景：** CRD 的 `spec` 中定义了非常庞大、嵌套层级深、字段众多的数据结构。
        
    - **原因：**
        
        - **序列化/反序列化开销：** API Server 在处理这些 CRs 时，需要对复杂的 YAML/JSON 结构进行序列化和反序列化操作，这会消耗更多的 CPU 资源。
            
        - **etcd 存储空间：** 大型 CRs 会占用 etcd 更多的存储空间，进一步加剧第一个场景中的存储问题。
            
        - **网络传输：** 大型 CRs 在 API Server 和客户端之间传输时，会占用更多网络带宽。
            
    - **影响：** API 请求响应延迟增加，尤其是 List 和 Get 操作。
        
4. **不当的控制器实现：**
    
    - **场景：** 尽管 CRD 本身没问题，但与其交互的**自定义控制器 (Operator)** 存在性能缺陷。
        
    - **原因：**
        
        - **不高效的 List-Watch 逻辑：** 控制器没有充分利用 Informer 缓存，而是频繁地直接向 API Server 发送 List/Get 请求。
            
        - **笨重的 Reconcile 逻辑：** 控制器在每次调谐 (reconcile) 时执行了过多、过慢或阻塞的操作（例如，同步调用外部 API，或执行复杂的计算）。
            
        - **处理事件的并发问题：** 工作队列处理不当，导致并发不足或过度并发，从而影响整体吞吐量或造成资源争抢。
            
        - **错误的重试策略：** 像您提供的 `handleErr` 函数，如果重试次数设置不合理，或者遇到无法恢复的错误时没有及时 `Forget`，会持续给队列和后端 API Server 施加压力。
            
    - **影响：** 尽管 CRD 可能没有直接问题，但其关联的控制器会成为瓶颈，间接影响集群性能。
        
5. **集群规模限制：**
    
    - **场景：** 即使 CRDs 本身设计良好，但当 Kubernetes 集群达到其**规模上限**（节点数、Pods 数、或对象总数）时，再增加 CRDs 或其 CRs 的数量就会加剧现有瓶颈。
        
    - **原因：** Kubernetes 控制平面的组件（API Server, etcd, Scheduler, Controller Manager）都有其处理能力的上限。CRDs 作为核心 API 的扩展，直接增加了这些组件的负载。
        
    - **影响：** 整个集群的稳定性下降，API 请求延迟飙升，调度器变慢，控制器无法及时响应状态变化。
        

---

### 如何缓解 CRD 性能瓶颈？

- **优化 CRD Schema：** 尽可能精简 CRD 结构，避免过深嵌套和过多字段。只存储必要的数据。
    
- **合理设计 CR 生命周期和更新频率：** 避免不必要的频繁更新。对于高频变化的字段，考虑使用独立的、更轻量级的机制（如直接的 gRPC 流或消息队列），而不是将所有状态都存储在 CR 中。
    
- **控制器优化：**
    
    - 充分利用 `client-go` 的 Informer 和 Lister 机制，从本地缓存获取数据，而不是频繁访问 API Server。
        
    - 确保 `reconciliation loop` 快速高效，避免阻塞操作。
        
    - 实现合理的限速和重试机制（如 `workqueue` 中的 `RateLimiter`）。
        
    - 使用 `status` 子资源来更新 CR 的状态，避免每次更新都触发表面的验证和变更。
        
- **分片 (Sharding) CRs：** 如果可能，将大量 CRs 分散到不同的 Kubernetes 集群或不同的 Namespace 中进行逻辑分片，减少单个集群的压力。
    
- **使用合适的 Kubernetes 版本和配置：** 关注 Kubernetes 社区在 API Server 和 etcd 性能方面的优化，例如最近版本引入的“**consistent reads from cache**”等改进。
    
- **监控和可观测性：** 密切监控 API Server 和 etcd 的指标（CPU、内存、请求延迟、etcd WAL 同步时间等），以及自定义控制器自身的性能指标，以便及时发现并解决瓶颈。
    

理解这些性能瓶颈场景对于设计和部署高性能、可扩展的云原生应用至关重要。


# CRD 瓶颈原因

造成 Kubernetes **Custom Resource Definitions (CRD)** 性能瓶颈的主要原因可以归结为以下几个核心方面。理解这些原因对于避免和解决 CRD 相关的集群性能问题至关重要。

---

### 1. 大量 Custom Resources (CRs) 实例

这是最常见也最直接的原因。

- **问题描述：** 当你的集群中存在**非常多的 CRs 实例**时（例如，数万到数十万甚至更多）。一个典型的场景是物联网应用，每个设备对应一个 CR；或者多租户平台，每个租户/项目创建大量的自定义资源。
    
- **深层原因：**
    
    - **etcd 存储压力：** Kubernetes 的所有对象（包括 CRs）都存储在 **etcd** 这个分布式键值数据库中。CRs 数量庞大直接增加了 etcd 的存储负担。大规模的数据会导致 etcd 的读写延迟增加，内存和 CPU 消耗升高。
        
    - **API Server 内存与 CPU：** API Server 需要从 etcd 中加载这些 CRs 的数据，并在内存中维护它们。大量的 CRs 会消耗大量的 API Server 内存。此外，API Server 在处理 List 请求（尤其是在控制器启动和周期性同步时）时，需要从 etcd 中检索、序列化和分发这些数据，这会消耗大量 CPU。
        
    - **网络带宽：** 无论是初次 List 还是周期性 Resync，传输大量 CRs 的数据都会占用可观的网络带宽。
        

---

### 2. CRs 实例的频繁更新（高变动率）

即使 CRs 实例的总数不多，但如果它们的状态**频繁发生变化**，也会成为瓶颈。

- **问题描述：** 某些 CRs 的 `status` 字段或 `spec` 字段更新非常频繁。例如，一个控制器可能每秒钟更新一次 CR 的状态来反映外部系统的实时数据，或者因为设计不佳导致 `reconciliation loop` 不停地更新 CR。
    
- **深层原因：**
    
    - **API Server Watch Cache 压力：** Kubernetes 的核心同步机制是基于 **List-Watch**。每次 CR 更新，都会在 etcd 和 **API Server 的 Watch Cache** 中产生一个新的事件。高变动率意味着大量的事件涌入 Watch Cache，可能导致缓存快速膨胀，占用大量内存，并需要频繁的垃圾回收。
        
    - **API Server CPU 消耗：** API Server 必须处理、序列化这些大量的事件，并通过持久连接推送到所有监听该 CRD 的客户端（Informer）。这个事件分发过程消耗大量 CPU。
        
    - **客户端 (Informer/Controller) 负载：** 所有监听这些 CRs 的 Informer 和控制器都需要接收、处理这些大量的事件，更新本地缓存，并将任务推送到工作队列。过高的事件流会使控制器始终处于繁忙状态，影响其处理其他重要任务的能力。
        

---

### 3. 不合理的 CRD Schema 设计

CRD 的结构和复杂性也会影响性能。

- **问题描述：** CRD 的 `spec` 或 `status` 中包含了**非常庞大、嵌套层级深、字段数量众多**的数据结构。
    
- **深层原因：**
    
    - **序列化/反序列化开销：** API Server 和客户端在处理这些复杂的 CRs 时，需要对大型 YAML/JSON 结构进行序列化和反序列化操作，这会消耗更多的 CPU 资源和时间。
        
    - **etcd 存储空间：** 大型 CRs 会占用 etcd 更多的存储空间，加剧了第一点提到的存储问题。
        
    - **网络传输：** 大型 CRs 在 API Server 和客户端之间传输时，需要更多的网络带宽和更长的时间。
        

---

### 4. 低效的自定义控制器 (Operator) 实现

很多时候，问题不在于 CRD 本身，而在于与其交互的**自定义控制器 (Operator)** 的实现方式。

- **问题描述：** 自定义控制器在处理 CRs 时存在性能缺陷或设计问题。
    
- **深层原因：**
    
    - **不充分利用 Informer 缓存：** 控制器没有从 Informer 的本地缓存 (`Lister`) 中获取数据，而是频繁地直接向 API Server 发送 List 或 Get 请求。这会绕过 List-Watch 的优化，给 API Server 带来巨大压力。
        
    - **笨重的 Reconcile 逻辑：** 控制器在每次调谐 (reconcile) 时，执行了过多、过慢、甚至阻塞的操作（例如，同步调用外部 API、执行复杂的计算、频繁进行数据库查询等）。这会导致工作队列积压，处理延迟。
        
    - **不当的重试策略：** 如果控制器遇到瞬时错误时，重试间隔过短或者重试次数过多，甚至对无法恢复的错误也一直重试而不调用 `Forget`，都会持续给队列和 API Server 施加不必要的压力。
        
    - **不合理的并发模型：** 工作队列的并发设置不合理，导致处理能力不足或者资源争抢。
        

---

### 5. 集群规模和配置限制

所有组件都有其处理能力上限。

- **问题描述：** 即使 CRDs 设计良好，但当 Kubernetes 集群达到其**规模上限**（例如，节点数、Pod 总数、网络流量等）时，再增加 CRDs 或其 CRs 的数量就会加剧现有瓶颈。
    
- **深层原因：** Kubernetes 控制平面的组件（API Server, etcd, Scheduler, Controller Manager）以及底层网络都有其性能限制。CRDs 作为核心 API 的扩展，直接增加了这些组件的负载。如果这些组件的资源（CPU、内存）不足或配置不当，性能问题会很快显现。
    

---

### 总结

造成 CRD 性能瓶颈的主要原因可以概括为：**CRs 数量过多**、**CRs 状态更新过于频繁**、**CRD Schema 设计过于复杂**，以及**自定义控制器实现效率低下**。这些因素往往相互影响，共同导致 API Server 和 etcd 的负载过高，最终影响整个 Kubernetes 集群的稳定性与响应速度。在设计和部署基于 CRD 的扩展时，深入理解这些潜在的瓶颈是至关重要的。