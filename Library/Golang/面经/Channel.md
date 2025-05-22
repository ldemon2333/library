多线程使用共享内存传递数据

并发模型：CSP。goroutine 和 channel 分别对应 CSP 中的实体和传递信息的媒介。

先入先出

目前的 Channel 收发操作均遵循了先进先出的设计，具体规则如下：

- 先从 Channel 读取数据的 Goroutine 会先接收到数据；
- 先向 Channel 发送数据的 Goroutine 会得到先发送数据的权利；
