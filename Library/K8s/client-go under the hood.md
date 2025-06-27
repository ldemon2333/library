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