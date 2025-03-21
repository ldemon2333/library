随着微服务架构和容器的兴起，Zabbix对容器监控显得力不从心。为解决监控容器的问题 Prometheus 应运而生。

Prometheus 是一套开源的系统监控报警框架，采用Go语言开发。得益于Google与k8s的强力支持，自带云原生的光环，天然能够友好协作，使得Prometheus 在开源社区异常火爆。
![[Pasted image 20250208210304.png]]

# 2.3 综合对比
下表通过多维度展现了各自监控系统的优缺点：
![[Pasted image 20250208202411.png]]

# 3. 使用 Prometheus + grafana 搭建监控系统
前面，我们了解了一些监控系统的区别和优缺点，下面我们以Prometheus为例，带大家一步一步搭建监控系统。

## 3.1 下载
Prometheus需要下载prometheus（Prometheus主服务）、node_exporter（服务器监控）、mysqld_exporter（Mysql数据库监控-可选）、pushgateway（数据网关-可选）、alertmanager（告警组件-可选）

下载地址：[https://prometheus.io/download/](https://prometheus.io/download/)

Grafana为数据展示界面，下载地址：[https://grafana.com/grafana/download](https://grafana.com/grafana/download)

## 3.2 架构图
![[Pasted image 20250208202535.png]]

## 3.3 安装 Prometheus Server
Prometheus 的架构设计中，Prometheus Server 主要负责数据的收集，存储并且对外提供数据查询支持。下面开始安装Prometheus Server。

**step1**：首先，下载prometheus，并上传到服务器

**setp2**：启动prometheus Server 服务。prometheus启动非常简单，只需要一个命令即可，进入到/usr/local/prometheus/prometheus-2.37.0后执行如下命令：

**step3**：验证prometheus是否启动成功，prometheus默认端口为：9090，我们在浏览器中输入：[http://10.2.1.231:9090/graph](http://10.2.1.231:9090/graph)，进入prometheus数据展示页面，说明prometheus启动成功。


## 3.4 安装 Node Exporter
实际的监控样本数据的由 Exporter 负责收集，如node_exporter 就是负责服务器的资源信息，同时提供了对外访问的HTTP服务地址（通常是/metrics）给prometheus拉取监控样本数据。下面开始安装node_exporter。

