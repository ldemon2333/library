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

