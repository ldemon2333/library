![[Pasted image 20250624152409.png]]

# components
- Reflector: A reflector, which is defined in [type _Reflector_ inside package _cache_](https://github.com/kubernetes/client-go/blob/master/tools/cache/reflector.go), watches the Kubernetes API for the specified resource type (kind). The function in which this is done is _ListAndWatch_. The watch could be for an in-built resource or it could be for a custom resource. When the reflector receives notification about existence of new resource instance through the watch API, it gets the newly created object using the corresponding listing API and puts it in the Delta Fifo queue inside the _watchHandler_ function.
- Informer: An informer defined in the [base controller inside package _cache_](https://github.com/kubernetes/client-go/blob/master/tools/cache/controller.go) pops objects from the Delta Fifo queue. The function in which this is done is _processLoop_. The job of this base controller is to save the object for later retrieval, and to invoke our controller passing it the object.
- Indexer: An indexer provides indexing functionality over objects. It is defined in [type _Indexer_ inside package _cache_](https://github.com/kubernetes/client-go/blob/master/tools/cache/index.go). A typical indexing use-case is to create an index based on object labels. Indexer can maintain indexes based on several indexing functions. Indexer uses a thread-safe data store to store objects and their keys. There is a default function named _MetaNamespaceKeyFunc_ defined in [type _Store_ inside package _cache_](https://github.com/kubernetes/client-go/blob/master/tools/cache/store.go) that generates an object’s key as `<namespace>/<name>` combination for that object.

# Custom Controller components
- Informer reference: This is the reference to the Informer instance that knows how to work with your custom resource objects. Your custom controller code needs to create the appropriate Informer.
- Indexer reference: This is the reference to the Indexer instance that knows how to work with your custom resource objects. Your custom controller code needs to create this. You will be using this reference for retrieving objects for later processing.

The base controller in client-go provides the _NewIndexerInformer_ function to create Informer and Indexer. In your code you can either [directly invoke this function](https://github.com/kubernetes/client-go/blob/master/examples/workqueue/main.go#L174) or [use factory methods for creating an informer.](https://github.com/kubernetes/sample-controller/blob/master/main.go#L61)

- Resource Event Handlers: These are the callback functions which will be called by the Informer when it wants to deliver an object to your controller. The typical pattern to write these functions is to obtain the dispatched object’s key and enqueue that key in a work queue for further processing.
    
- Work queue: This is the queue that you create in your controller code to decouple delivery of an object from its processing. Resource event handler functions are written to extract the delivered object’s key and add that to the work queue.
    
- Process Item: This is the function that you create in your code which processes items from the work queue. There can be one or more other functions that do the actual processing. These functions will typically use the [Indexer reference](https://github.com/kubernetes/client-go/blob/master/examples/workqueue/main.go#L73), or a Listing wrapper to retrieve the object corresponding to the key.


---

In the architecture of a Kubernetes controller using `client-go`, the **Informer** and the **Workqueue (工作队列)** are two indispensable components that work in tandem. They form the backbone of how your controller efficiently and reliably reacts to changes in Kubernetes resources.

---

### **Informer: The Event Source and Cache Manager**

The **Informer** acts as the controller's **eyes and ears** on the Kubernetes API Server. Its primary roles are:

1. **Efficiently Observing Changes (List-Watch)**:
    - It performs an initial `List` operation to fetch all existing resources of a specific type (e.g., all Pods).
    - It then establishes a persistent `Watch` connection to the API Server, receiving real-time notifications for any `Add`, `Update`, or `Delete` events for those resources. This approach is far more efficient than constantly polling the API Server.
2. **Maintaining a Local Cache (Indexer)**:
    - The Informer keeps an in-memory, up-to-date **cache (Indexer)** of the resources it's watching. This cache is crucial because your controller can quickly read the current state of any resource from local memory without needing to make a network call to the API Server every time. This significantly reduces API Server load and improves your controller's performance.
3. **Broadcasting Events**:
    - When the Informer detects a change (via `Watch` events or periodic `resync`), it **broadcasts** these events to all registered **Event Handlers**. This is where the connection to the Workqueue begins.

**Role Positioning**: The Informer is responsible for **data acquisition** (from the API Server) and **data provision** (to the local cache and event handlers). It provides a _stream of events_ and a _consistent view of the cluster state_.

---

### **Workqueue: The Reliable Task Processor**

The **Workqueue** (specifically `workqueue.TypedRateLimitingInterface`) is a specialized queue that serves as the controller's **todo list** and **error-handling mechanism**. Its primary roles are:

1. **Decoupling Event Reception from Processing**:
    - The Informer's event handlers don't directly process the business logic. Instead, they simply take the unique identifier (the **key**, typically `namespace/name`) of the changed resource and add it to the Workqueue. This immediately frees up the Informer's event-handling thread, ensuring it can continue to receive and process new events without being blocked by potentially long-running or error-prone business logic.
2. **De-duplication**:
    - If multiple events for the same resource happen quickly (e.g., a Pod is updated twice in a short span), the Workqueue will automatically **de-duplicate** them. It ensures that the same `key` is not processed concurrently or unnecessarily added multiple times. When the controller finally processes that `key`, it will fetch the _latest_ version of the resource from the Informer's cache.
3. **Retries with Rate Limiting**:
    - If the controller's business logic fails to process an item (returns an error), the Workqueue handles **retries**. It allows you to re-add the `key` to the queue with a built-in **rate limiter** (e.g., exponential back-off). This prevents a faulty item from constantly flooding the controller and overwhelming the API Server or external services.
4. **Concurrency Control**:
    - While multiple worker Goroutines can be launched to process items from the Workqueue concurrently, the queue ensures that **only one worker processes a given `key` at any time**. This prevents race conditions when multiple workers try to reconcile the same resource simultaneously.

**Role Positioning**: The Workqueue is responsible for **managing reconciliation tasks**, ensuring **reliability**, **de-duplication**, and **controlled retries**. It transforms a stream of raw events into a structured, retryable processing pipeline.

---

### **The Association: How They Work Together**

The **`cache.ResourceEventHandlerFuncs`** is the **glue** that binds the Informer and the Workqueue:

``` go
indexer, informer := cache.NewIndexerInformer(
    podListWatcher,             // Watches pods
    &v1.Pod{},                  // Type of object
    0,                          // No resync period
    cache.ResourceEventHandlerFuncs{ // ⬅️ The key association point
        AddFunc: func(obj interface{}) {
            key, err := cache.MetaNamespaceKeyFunc(obj)
            if err == nil {
                queue.Add(key) // Informer detects Add -> add key to Workqueue
            }
        },
        UpdateFunc: func(old interface{}, new interface{}) {
            key, err := cache.MetaNamespaceKeyFunc(new)
            if err == nil {
                queue.Add(key) // Informer detects Update -> add key to Workqueue
            }
        },
        DeleteFunc: func(obj interface{}) {
            key, err := cache.DeletionHandlingMetaNamespaceKeyFunc(obj)
            if err == nil {
                queue.Add(key) // Informer detects Delete -> add key to Workqueue
            }
        },
    },
    cache.Indexers{},
)
```

Here's the interaction flow:

1. The **Informer** (via `podListWatcher`) establishes a `List` and `Watch` connection to the API Server.
2. As `Add`, `Update`, or `Delete` events occur for Pods on the cluster, the **Informer** receives them.
3. Upon receiving an event, the **Informer** first updates its **local cache (Indexer)**.
4. Simultaneously, the **Informer** invokes the corresponding function in the `ResourceEventHandlerFuncs` you've provided (e.g., `AddFunc` for an `Add` event).
5. Inside these handler functions, the unique **`key`** of the affected Pod (`namespace/name`) is extracted.
6. This `key` is then immediately added to the **Workqueue** using `queue.Add(key)`. This is a non-blocking operation.
7. Separately, your controller's **worker Goroutines** (`c.runWorker` calling `c.processNextItem`) are continuously pulling `keys` from the Workqueue.
8. When a worker gets a `key`, it uses the **Informer's Indexer** (`c.indexer.GetByKey(key)`) to fetch the _latest_ version of the resource from the local cache and then executes the business logic (`c.syncToStdout`).

This pattern ensures that your controller is both **responsive** to cluster changes and **resilient** to transient failures, making it a robust way to build Kubernetes operators and controllers.


# # Kubernetes: Controllers, Informers, Reflectors and Stores
![[Pasted image 20250630205045.png]]
At some point we noticed some behavior we didn’t expect. For example: _UpdateFunc_ was called every 30 seconds, while there actually were no updates on the upstream. This is when I decided to dig into the workings of _Informers_, _Controllers_, _Reflectors_, and _Stores_. I’ll start by explaining how Controllers work, then I’ll explain how Controllers use a Reflector and a _DeltaFIFO_ store internally, and lastly how Informers are just a convenient wrapper to sync your upstream with a downstream store.

Please note that Kubernetes is under heavy development. The code base will probably have changed between the point of writing and the time you’re reading this article. I will refer to code at this [blob](https://github.com/kubernetes/kubernetes/tree/82cb4c17581752ae9f00bd746a63e529149c04b4/pkg).



Excellent question. You've provided a very precise and insightful piece of text that describes a core mechanism of the Kubernetes `Reflector`. Let's break down that explanation and connect it to the code example you provided earlier.

### Translation of the Provided Text

In simpler terms, the text is explaining the **difference between how a controller learns about objects at startup versus how it learns about them in real-time.**

- **Startup (Initial Synchronization):** When a controller first starts, it doesn't know anything. Its first job is to get a complete list of all existing objects it's supposed to manage from the API server. This initial "catch-up" process uses the `List` and `Replace` mechanism, which generates `Sync` events.
    
- **Real-time (Ongoing Changes):** After the initial list is complete, the controller "watches" the API server for any changes that happen from that moment forward. A new object being created triggers an `Added` event. An object being deleted triggers a `Deleted` event.
    

---

### Deconstructing the `Replace` Method's Logic

The `Replace` method is the key here. It's called by the `Reflector` at the very beginning of its life (and periodically for resyncs). It has a two-phase job:

**Phase 1: Announce Existing Items**

The `Reflector` calls the API server's `List` endpoint and gets a complete list of all current Pods. Let's say it finds `[pod-A, pod-B, pod-C]`. It passes this list to the `Replace` method of the queue (`DeltaFIFO`).

`Replace` then iterates through this list and for each item, it enqueues a `Delta` with `DeltaType: Sync`.

- Queue receives: `(Sync, pod-A)`
    
- Queue receives: `(Sync, pod-B)`
    
- Queue receives: `(Sync, pod-C)`
    

This is why your controller's `Process` function would receive items of type `Sync`.

**Phase 2: Detect Deletions That Happened Offline**

This is the clever part. The `Replace` method needs to account for the possibility that an object was deleted while the controller was down or between the `List` call and now.

It uses the `knownObjects` list (which in your example is a reference to the `downstream` store) to do this.

Let's imagine a scenario:

1. Your controller was running and its `downstream` store knew about `[pod-A, pod-B, pod-D]`.
    
2. Your controller crashes.
    
3. While it's down, an admin deletes `pod-D`.
    
4. Your controller restarts. Its `Reflector` does a `List` and gets the current list `[pod-A, pod-B, pod-C]`.
    
5. It calls `Replace` with this new list.
    
6. `Replace` compares the new list `[A, B, C]` with the `knownObjects` list `[A, B, D]`.
    
7. It sees that `D` is in `knownObjects` but not in the new list. It concludes `D` must have been deleted.
    
8. It then explicitly enqueues a `Delta` with `DeltaType: Deleted` for `pod-D`.
    

This ensures the controller's cache becomes perfectly consistent with the API server.

---

### `Sync` vs. `Added`: A Crucial Distinction

|DeltaType|When It's Used|Meaning|
|---|---|---|
|**`Sync`**|During the initial `List` & `Replace` operation when a controller starts (or during a full resync).|"This object already existed when I started. I am just syncing my state."|
|**`Added`**|During an ongoing `Watch` operation, after the initial sync is complete.|"A brand new object just appeared that wasn't here a moment ago."|

In your example controller, the processing logic doesn't care about the difference:

```
if newest.Type != cache.Deleted {
    // This code block runs for BOTH 'Added' and 'Sync' events.
    source.Delete(newest.Object.(runtime.Object))
}
```

The distinction is more for understanding the state of the system and for more complex controllers that might treat a pre-existing object differently from a brand new one.

### Applying This to Your Code Example

The text you provided perfectly explains what happens in your `Example()` function:

1. **`source.Add(...)`**: The main goroutine adds three pods to the fake API server _before_ the controller is fully up and running its watch loop.
    
2. **`go framework.New(cfg).Run(stop)`**: The controller starts in a separate goroutine.
    
3. **Initial List**: Its `Reflector` immediately performs its first action: a `List()` on the `source`. It gets back the list of three pods you just added.
    
4. **Call to `Replace()`**: The `Reflector` passes this list of three pods to the `DeltaFIFO`'s internal `Replace` method.
    
5. **`Sync` Events Queued**: `Replace` sees these three items and enqueues them as `Sync` events. It checks the `downstream` store (which is empty), finds no discrepancies, and so it queues no `Deleted` events.
    
6. **Controller Processes `Sync`**: Your `Process` function pops these items one by one. For each, `newest.Type` is `cache.Sync`. The condition `cache.Sync != cache.Deleted` is true, so `source.Delete()` gets called.
    

You are correct that if the goroutine started and the `Reflector` managed to run its `List` _before_ you added the pods, and _then_ you added the pods, the `Watch` mechanism would have picked them up and you would have seen `Added` events instead. However, given the typical program flow, the adds happen first, leading to the `List`/`Replace`/`Sync` scenario.



# Two way
好的，这是一个非常经典且重要的问题，尤其是在使用 `client-go` 开发 Kubernetes Operator 或 Controller 时。

简单来说，这两者的核心区别在于**它们所能操作的资源类型不同**：

1. `kubeClient, err := kubernetes.NewForConfig(cfg)`: 用于创建与 Kubernetes **内置核心资源**进行交互的客户端。
    
2. `exampleClient, err := clientset.NewForConfig(cfg)`: 用于创建与你的**自定义资源 (CRD)** 进行交互的客户端。
    

下面我们来详细拆解这个区别。

---

### `kubeClient, err := kubernetes.NewForConfig(cfg)`

这个函数来自于 `k8s.io/client-go/kubernetes` 包。它返回的 `kubeClient`（一个 `*kubernetes.Clientset` 类型的对象）是一个**标准客户端集**。

用途：

这个客户端被设计用来访问所有 Kubernetes 原生、内置 的 API 资源。你可以把它想象成 Go 代码版本的 kubectl，能够操作你所熟知的各种资源。

**能操作的资源示例：**

- Pods (`core/v1`)
    
- Deployments (`apps/v1`)
    
- Services (`core/v1`)
    
- ConfigMaps (`core/v1`)
    
- Secrets (`core/v1`)
    
- Nodes (`core/v1`)
    
- ...以及其他所有标准的 Kubernetes 资源。
    

**代码示例：**

```
// 使用标准 kubeClient 来获取 "default" 命名空间下的所有 Pod
pods, err := kubeClient.CoreV1().Pods("default").List(context.TODO(), metav1.ListOptions{})
if err != nil {
    panic(err.Error())
}
fmt.Printf("There are %d pods in the cluster\n", len(pods.Items))

// 使用标准 kubeClient 创建一个 Deployment
_, err = kubeClient.AppsV1().Deployments("default").Create(context.TODO(), myDeployment, metav1.CreateOptions{})
```

**总结：** 当你的程序需要与 Kubernetes 的标准组件进行交互时（比如，你的 Controller 需要根据一个自定义资源的状态来创建一个 Deployment），你就需要使用这个标准 `Clientset`。

---

### `exampleClient, err := clientset.NewForConfig(cfg)`

这个函数来自于一个**自动生成的包**。在 `sample-controller` 的例子中，这个包的路径通常是 `k8s.io/sample-controller/pkg/client/clientset/versioned`。

背景：

当你定义了一个自定义资源（CRD，Custom Resource Definition）后，比如一个叫 Foo 的资源，Kubernetes 原生的 kubeClient 并不知道这个 Foo 是什么，它没有 kubeClient.FooV1().Foos() 这样的方法。

为了能用强类型、代码提示友好的方式来操作你的自定义资源，Kubernetes 提供了一个代码生成工具 (`k8s.io/code-generator`)。你需要在你的项目上运行这个工具，它会扫描你定义的 `Foo` 资源的 Go 类型，并为你**自动生成**一套专属的客户端代码，其中就包括了这个 `clientset.NewForConfig` 函数。

用途：

这个客户端（exampleClient）是专门用来操作你在项目中定义的特定 CRD 的。

**能操作的资源示例：**

- 在 `sample-controller` 例子中，它只能操作 `Foo` 这种自定义资源。
    

**代码示例：**


```
// 这里的 "samplecontroller.k8s.io/v1alpha1" 是你在 CRD 中定义的 Group 和 Version
// 使用 exampleClient 来获取 "default" 命名空间下的所有 Foo 资源
foos, err := exampleClient.SamplecontrollerV1alpha1().Foos("default").List(context.TODO(), metav1.ListOptions{})
if err != nil {
    panic(err.Error())
}
```

**总结：** 当你编写一个处理 CRD 的 Controller 时，你必须使用这个为你自己的 CRD 生成的专属 `Clientset` 来与你的自定义资源进行交互。

---

### 为什么在 Controller 中两者都需要？

在一个典型的自定义 Controller 中，你的核心逻辑通常是：**“监听我的自定义资源（`Foo`）的状态变化，然后根据这些变化去操作一些内置的 Kubernetes 资源（比如 `Deployment`）。”**

这就完美解释了为什么你需要两个客户端：

- **`exampleClient` (自定义客户端)**: 用来 `Get`、`List`、`Watch` 你的 `Foo` 资源，了解它的 `spec`（期望状态）和 `status`（实际状态）。
    
- **`kubeClient` (标准客户端)**: 用来执行具体的操作，比如根据 `Foo` 的 `spec` 去 `Create` 或 `Update` 一个 `Deployment`，或者去更新一个 `ConfigMap`。
    

**在 `sample-controller` 的 `main.go` 中，你会看到这样的初始化代码，这正是最佳实践：**


```
// ... (加载 kubeconfig)

// 1. 创建标准的 clientset
kubeClient, err := kubernetes.NewForConfig(cfg)
if err != nil {
    // ... handle error
}

// 2. 创建用于我们自己的 CRD (Foo) 的 clientset
exampleClient, err := clientset.NewForConfig(cfg)
if err != nil {
    // ... handle error
}

// 3. 将这两个 clientset 传递给你创建的 Controller
//    Controller 内部就可以根据需要，使用正确的客户端来操作正确的资源了。
controller := NewController(kubeClient, exampleClient, ...)
```

### 对比总结

| 特性       | `kubernetes.NewForConfig(cfg)`            | `clientset.NewForConfig(cfg)` (生成的)                  |
| -------- | ----------------------------------------- | ---------------------------------------------------- |
| **来源包**  | `k8s.io/client-go/kubernetes` (官方标准包)     | `your-project/pkg/client/clientset/versioned` (自动生成) |
| **操作对象** | Kubernetes 内置核心资源 (Pod, Deployment, etc.) | 项目中定义的自定义资源 (CRD)                                    |
| **通用性**  | **高**，所有 K8s 项目通用                         | **低**，仅为特定 CRD 生成，项目专属                               |
| **使用场景** | 几乎所有与 K8s API 交互的程序                       | 专门用于编写操作 CRD 的 Controller/Operator                   |


# Controller 工作流程
![[Pasted image 20250703194845.png]]

# Informer 封装 list-watch
Kubernetes 的 **List-Watch 机制**是其控制平面和各种控制器能够实现**声明式 API** 和**最终一致性**的基石。它允许组件高效地监听集群状态的变化，并对其做出响应，而无需不断地轮询 API Server。

---

### 1. 为什么需要 List-Watch 机制？

想象一下 Kubernetes 中有成千上万个 Pod、Deployment、Service 等资源，并且它们的状态时刻在变化。如果每个控制器都通过频繁地向 API Server 发送 `GET /pods` 或 `GET /deployments` 等请求来获取最新状态（即 **轮询 polling**），将会带来以下问题：

- **API Server 负载过高：** 大量的轮询请求会给 API Server 带来巨大的压力，影响其性能和稳定性。
    
- **网络带宽浪费：** 即使资源状态没有变化，轮询也会传输大量重复数据。
    
- **延迟：** 轮询间隔决定了发现变化的速度，无法做到实时响应。
    

为了解决这些问题，Kubernetes 设计了 List-Watch 机制，它结合了**初始状态同步**和**事件驱动更新**的优势。

---

### 2. List-Watch 机制的核心组件和流程

List-Watch 机制的核心参与者包括：

- **etcd：** Kubernetes 的分布式一致性键值存储，所有集群状态的“事实来源”。etcd 自身提供了高效的 `watch` 机制。
    
- **Kubernetes API Server：** 作为所有组件与 etcd 交互的唯一接口。它将 etcd 的 `watch` 功能暴露给 Kubernetes 内部和外部的客户端。
    
- **Controller (控制器)：** Kubernetes 中负责协调和维护集群期望状态的逻辑单元（例如 Deployment Controller、ReplicaSet Controller 等）。
    
- **Informer (信息器)：** Kubernetes `client-go` 库提供的高级抽象，封装了 List-Watch 机制的复杂性，供控制器使用。
    

**基本工作流程：**

一个典型的控制器（通过 Informer）监听特定资源类型（如 Pods）的变化，其过程如下：

1. **List (初始同步)：**
    
    - 当控制器启动时，它首先向 API Server 发送一个 `GET` 请求，以**获取（List）**该资源类型（例如，所有 Pods）的**当前所有对象**的完整快照。
        
    - API Server 从 etcd 中读取数据并返回。这个响应中会包含一个名为 `resourceVersion` 的特殊字段。`resourceVersion` 是一个不透明的字符串，代表了 etcd 中这个资源集合的最新版本。
        
    - 控制器接收到快照后，将其加载到自己的**本地缓存**中（通常称为 **Lister** 或 **Indexer**）。
        
2. **Watch (持续监听)：**
    
    - 在成功获取初始快照后，控制器（或 Informer）会立即向 API Server 发送一个 **`GET` 请求，并带上 `watch=true` 参数以及从上一步 List 操作中获得的 `resourceVersion`**。
        
    - API Server 接收到 `watch` 请求后，会建立一个**持久的连接**（通常是 HTTP Long-Polling 或 HTTP/2 Streaming）。
        
    - **API Server 内部的 Watch Cache：** API Server 自身维护了一个内存中的“观察缓存”（Watch Cache）。它会监听 etcd 的变化，并将这些变化存储在 Watch Cache 中。当有客户端发起 Watch 请求时，API Server会从Watch Cache中获取事件并推送到客户端。
        
    - **事件推送：** 一旦 etcd 中与被监听资源相关的对象发生任何变化（创建、更新、删除），etcd 会通知 API Server。API Server 就会将这些变化封装成**事件 (Event)**，并通过之前建立的持久连接**推送 (Push)** 给监听的客户端（控制器）。这些事件包含了变化的类型（`ADDED`, `MODIFIED`, `DELETED`, `BOOKMARK` 等）和受影响的对象的新/旧状态。
        
    - **更新本地缓存：** 控制器接收到事件后，会根据事件类型更新其本地缓存。例如，如果收到一个 `ADDED` 事件，它会将新对象添加到缓存；如果收到 `DELETED` 事件，它会从缓存中删除对象。
        
3. **Reflector 和 DeltaFIFO (客户端实现细节)：**
    
    - 在 `client-go` 库中，**Reflector (反射器)** 是实际执行 List 和 Watch 操作的组件。它负责维护与 API Server 的连接，处理断开重连，并确保本地缓存是最新的。
        
    - **DeltaFIFO (差量 FIFO 队列)** 是 Informer 内部使用的队列。当 Reflector 接收到事件时，它会将这些事件推送到 DeltaFIFO。DeltaFIFO 会对事件进行去重和合并，并按照顺序传递给控制器。它还负责处理所谓的 **"resync" (重新同步)** 机制。
        
4. **Resync (重新同步/全量同步)：**
    
    - 为了防止因网络抖动、API Server 重启或 etcd 数据丢失等原因导致事件丢失，Informer 会周期性地（默认 30 分钟）执行一次**全量 List 操作**。
        
    - 它会将最新的全量数据与本地缓存进行比对，纠正任何可能存在的差异，确保最终一致性。如果发现本地缓存与 API Server 的最新状态不一致，它会生成相应的 `ADDED`、`MODIFIED` 或 `DELETED` 事件推送到 DeltaFIFO。
        
5. **Event Handler (事件处理器) 和 Workqueue (工作队列)：**
    
    - 控制器会注册**事件处理器**（例如 `AddFunc`, `UpdateFunc`, `DeleteFunc`）到 Informer。
        
    - 当 Informer 通过 DeltaFIFO 处理事件后，它会将相应的对象（或其键 `namespace/name`）添加到控制器的**工作队列 (Workqueue)** 中。
        
    - 控制器从工作队列中取出任务，并执行其核心的**协调 (Reconciliation)** 逻辑，即检查实际状态是否与期望状态一致，并采取相应行动。
        

---

### 3. `resourceVersion` 的作用

`resourceVersion` 是 List-Watch 机制高效运行的关键：

- **标记时间点：** 每个 Kubernetes 对象都有一个 `resourceVersion` 字段，表示该对象在 etcd 中的版本号。每次对象发生变化，其 `resourceVersion` 都会更新。
    
- **断点续传：** 当客户端发起 `watch` 请求时，带上一个 `resourceVersion` 参数，API Server 就只会发送从该 `resourceVersion` 之后发生的所有事件。这使得客户端可以在连接中断后，从上次同步到的版本继续接收事件，而无需从头开始。
    
- **解决“丢失事件”问题：** 如果客户端的 `resourceVersion` 过旧，API Server 的 Watch Cache 中可能已经没有对应的数据了（Watch Cache 只保留最近一段时间的事件）。在这种情况下，API Server 会返回一个 `410 Gone` 错误。客户端（Reflector）收到 `410 Gone` 后，就知道它丢失了太多的事件，无法从上次断点续传。此时，它会**清除本地缓存，重新执行一次 List 操作，获取最新的 `resourceVersion`，然后从新的 `resourceVersion` 开始一个新的 Watch 连接**。这个过程是自动处理的，确保了数据的最终一致性。
    

---

### 4. 优势和影响

- **高效性：** 避免了大量重复的轮询请求，显著降低了 API Server 和网络负载。事件驱动的方式使得变化能够实时传播。
    
- **最终一致性：** 通过 List-Watch 结合 Resync 和 `resourceVersion`，控制器能够确保其本地缓存始终与集群的实际状态保持最终一致。
    
- **解耦：** API Server 专注于提供统一的 API 接口和事件流，而各个控制器则独立地监听和响应自己关心的资源，降低了组件间的耦合度。
    
- **声明式管理：** 这种机制是 Kubernetes 声明式 API 的核心支撑。控制器通过不断地将实际状态调整到期望状态来实现声明式管理，而 List-Watch 机制提供了驱动这种调整的事件流。
    

### 总结

Kubernetes 的 List-Watch 机制是一个精妙的设计，它有效地解决了分布式系统中状态同步和事件通知的挑战。通过利用 etcd 的 Watch 能力、API Server 的 Watch Cache、以及 `client-go` 中的 Informer 抽象，Kubernetes 实现了高效、可靠且最终一致的集群状态管理，为构建复杂的自动化控制器提供了坚实的基础。


![[Pasted image 20250703195040.png]]
![[Pasted image 20250703195212.png]]
![[Pasted image 20250703195311.png]]
![[Pasted image 20250703195355.png]]

![[Pasted image 20250703195622.png]]
![[Pasted image 20250703200254.png]]
![[Pasted image 20250703200335.png]]


# Forget 作用
在您提供的 `handleErr` 函数中，`c.queue.Forget(key)` 的作用是：

**清除或重置与给定 `key` 在队列中的速率限制历史记录。**

让我们分两种情况来理解它的具体作用：

### 1. `if err == nil` 时的 `Forget(key)`

```
func (c *Controller) handleErr(err error, key string) {
    if err == nil {
        // Forget about the #AddRateLimited history of the key on every successful synchronization.
        // This ensures that future processing of updates for this key is not delayed because of
        // an outdated error history.
        c.queue.Forget(key) // 第一次 Forget
        return
    }
    // ... 其他逻辑
}
```

- **场景：** 当 `handleErr` 函数被调用，并且 `err` 为 `nil`（表示**同步操作成功**）时。
    
- **作用：** 此时调用 `c.queue.Forget(key)` 的目的是，**将这个 `key` 从队列内部的速率限制器中“释放”出来，清除其所有的重试计数和延迟信息。**
    
- **原因：** 如果一个 `key` 之前因为错误而多次被 `AddRateLimited` 加入队列，它会被标记为需要延迟处理。一旦同步成功，就表明这个 `key` 已经“恢复正常”，不再需要基于之前的错误历史来限制其处理速度。`Forget` 确保了该 `key` 在下次被添加到队列时，会像一个全新的、没有任何错误历史的 `key` 一样被对待，不会因为之前的问题而被不必要地延迟。
    
- **概括：** 成功处理后，清除历史遗留的“惩罚”，让 `key` 在队列中获得“清白”。
    

### 2. `c.queue.NumRequeues(key) >= 5` 时的 `Forget(key)`

```
func (c *Controller) handleErr(err error, key string) {
    // ...
    if c.queue.NumRequeues(key) < 5 {
        // ...
        c.queue.AddRateLimited(key)
        return
    }

    c.queue.Forget(key) // 第二次 Forget
    // Report to an external entity that, even after several retries, we could not successfully process this key
    runtime.HandleError(err)
    klog.Infof("Dropping pod %q out of the queue: %v", key, err)
}
```

- **场景：** 当 `handleErr` 函数被调用，并且 `err` 不为 `nil`（表示**同步操作失败**），而且该 `key` 的重试次数已经**达到或超过**了预设的阈值（这里是 5 次）。
    
- **作用：** 此时调用 `c.queue.Forget(key)` 的目的是，**显式地将这个 `key` 从队列中完全移除，并清除其速率限制历史。**
    
- **原因：** 尽管同步失败，但控制器已经尝试了最大重试次数。这意味着控制器认为再尝试下去可能也无法成功，或者说已经达到了其处理此问题的能力极限。此时，`Forget` 的作用是将 `key` 从队列的活跃处理列表中移除，停止对其进行进一步的自动重试。然后，通过 `runtime.HandleError(err)` 将错误报告给外部系统，并通过 `klog.Infof` 记录该 Pod 被“丢弃”的信息。
    
- **概括：** 达到最大重试次数后，放弃自动重试，清除历史，并将问题上报。
    

### 总结 `Forget` 的核心作用：

`Forget` 方法主要用于**管理队列中特定 `key` 的重试状态和延迟策略**：

- **成功时调用：** 用于**重置** `key` 的错误历史，确保后续处理不受之前失败的影响。
    
- **放弃重试时调用：** 用于**清除** `key` 的所有重试信息，并将其从队列中逻辑上移除，防止其被继续自动重试。
    

可以说，`Forget` 方法是 `workqueue.RateLimitingInterface`（或其他类似队列接口）中一个非常重要的操作，它与 `AddRateLimited` 共同协作，实现了一种带有指数退避或其他策略的重试机制，同时又能保证在问题解决或达到重试上限后，能够及时地清理状态。