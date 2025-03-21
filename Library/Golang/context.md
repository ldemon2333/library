在 Go 语言中，`context` 包是用于协调多个并发任务的工具，可以理解为一个「传令官」，负责在不同协程（goroutine）之间传递控制信号和共享数据。

`context`的作用就是在不同的`goroutine`之间同步请求特定的数据、取消信号以及处理请求的截止日期。

# 什么是 Context
是一个跨 API 和进程用来传递截止日期、取消信号和请求相关值的接口。

定义如下：
```
type Context interface{
	Deadline() (deadline time.Time, ok bool)
	Done() <-chan struct{}
	Err() error
	Value(key interface{}) interface{}
}
```

Deadline() 返回一个完成工作的截止时间，表示上下文应该被取消的时间。如果 ok== false 表示没有设置截止时间。

Done() 返回一个 Channel，这个 Channel 会在当前工作完成时被关闭，表示上下文应该被取消。如果无法取消此上下文，则 Done 可能返回 nil。

Err() 返回 Context 结束的原因，它只会在 Done 方法对应的 Channel 关闭时返回非空值。如果 Context 被取消，会返回 `context.Canceled` 错误；如果 Context 超时，会返回 `context.DeadlineExceeded` 错误。

`Value()` 从 Context 中获取键对应的值。如果未设置 key 对应的值则返回 nil。

两个创建默认上下文的函数：
![[Pasted image 20250314235819.png]]
还有四个基于父级创建不同类型上下文的函数：
![[Pasted image 20250314235843.png]]

# 为什么要有 Context
Go 为后台服务而生，如只需几行代码，便可以搭建一个 HTTP 服务。

在 Go 的服务里，通常每来一个请求都会启动若干个 goroutine 同时工作：有些执行业务逻辑，有些去数据库拿数据，有些调用下游接口获取相关数据…
![[Pasted image 20250314235947.png]]
协程 a 生 b c d，c 生 e，e 生 f。父协程与子孙协程之间是关联在一起的，他们需要共享请求的相关信息，比如用户登录态，请求超时时间等。如何将这些协程联系在一起，context 应运而生。

话说回来，为什么要将这些协程关联在一起呢？以超时为例，当请求被取消或是处理时间太长，这有可能是使用者关闭了浏览器或是已经超过了请求方规定的超时时间，请求方直接放弃了这次请求结果。此时所有正在为这个请求工作的 goroutine 都需要快速退出，因为它们的“工作成果”不再被需要了。在相关联的 goroutine 都退出后，系统就可以回收相关资源了。

==总的来说 context 的作用是为了在一组 goroutine 间传递上下文信息（cancel signal，deadline，request-scoped value）以达到对它们的管理控制。==

# 使用
## 创建`context`

`context`包主要提供了两种方式创建`context`:

- `context.Backgroud()`
    
- `context.TODO()`
    

这两个函数其实只是互为别名，没有差别，官方给的定义是：

- `context.Background` 是上下文的默认值，所有其他的上下文都应该从它衍生（Derived）出来。
- `context.TODO` 应该只在不确定应该使用哪种上下文时使用；

所以在大多数情况下，我们都使用`context.Background`作为起始的上下文向下传递。

上面的两种方式是创建根`context`，不具备任何功能，具体实践还是要依靠`context`包提供的`With`系列函数来进行派生：

![[Pasted image 20250315002138.png]]

这四个函数都要基于父`Context`衍生，通过这些函数，就创建了一颗Context树，树的每个节点都可以有任意多个子节点，节点层级可以有任意多个，画个图表示一下：

![[Pasted image 20250315002202.png]]

## `WithValue`携带数据

我们日常在业务开发中都希望能有一个`trace_id`能串联所有的日志，这就需要我们打印日志时能够获取到这个`trace_id`，在`python`中我们可以用`gevent.local`来传递，在`java`中我们可以用`ThreadLocal`来传递，在`Go`语言中我们就可以使用`Context`来传递，通过使用`WithValue`来创建一个携带`trace_id`的`context`，然后不断透传下去，打印日志时输出即可，来看使用例子：
