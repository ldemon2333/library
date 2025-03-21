Linux的网络控制模块在内核中，叫做`netfilter`。而`iptables`是位于用户空间的一个命令行工具，它作用在`OIS7层网络模型`中的第四层，用来和内核的`netfilter`交互，配置`netfilter`进而实现对网络的控制、流量的转发 。

# iptables、防火墙
简单的说就是：iptables 是一个简单、灵活、实用的命令行工具，可以用来配置、控制 linux 防火墙。

# iptables 的五表五链及流量走向
iptables中总共有4张表还有5条链，我们可以在链上加不同的规则。

五张表：filter表、nat表、mangle表、raw表、security表

五条链：prerouting、input、output、forward、postrouting

你可以通过`iptables -t ${表名} -nL`查看表上的链
![[Pasted image 20250227161331.png]]

![[Pasted image 20250227161338.png]]
先不着急使用iptables命令，大家可以先参考下面这张图，看看如下几种流量的走向。

- 来自本机流量经过了iptables的哪些节点，最终又可以流到哪里去？
    
- 来自互联网其他主机的流量，经过了本机iptables的哪些节点，最终又可以流到哪里去？

**第一张**：是摘自iptables的wiki百科中的图，如下：
![[Pasted image 20250227161548.png]]
`raw prerouting` -> `conntrack` -> `mangle prerouting` -> `nat prerouting` - >`decision 路由选择` -> 可能是input，也可能是output。

记住：在wiki百科中的流量走向图中，mangle.prerouting和nat.prerouting之间没有任何判断逻辑就好了。路由选择判断发生在nat.prerouting之后。

**第二张：摘自github上写的一篇文章：理解 kube-proxy 中 iptables 规则

这张图是一张更为精确的流量走向图，并且他区分好了：`incoming packet`、`locally gennerated packge` 这种来源不同的流量大走向，原图如下：
![[Pasted image 20250227161855.png]]
但是我感觉他这个图稍微有点问题：你可以看下图左上角部分，mange-prerouting和 nat-prerouting之间多了一个 localhost source的判断。但是在iptables维基百科中给出的图中，它俩之间并没有这个判断。

如果你仔细看下，这个localhost source的判断和后面的for this host其实是挺重复的。而这个for this host判断正好对应着第一张图中的 bridge decsion判断。

所以，我总是感觉图应该修改成下面这样，它不一定对，但是起码能自圆其说。对整体理解的影响也不大。

而且图改成这个样子，肯定是方便会我们理解这个过程，而且错也错不了哪里去。
![[Pasted image 20250227161941.png]]
稍微解析一下上图：

1、`红`、`蓝`、`绿`、`紫`分别代表上一小节中提到的`iptables`的四张表。如果你开启着SELinux，还会多出一个`security`表

2、上图左上角写的：`incoming packet`，表示这是从互联网其他设备中来的流量。它大概的走向是：先经过各个表的`prerouting`阶段，再经由`routing decision`（可以理解成查路由表，做路由选择）决定这些流量是应该交由本机处理，还是该通过其他网口`forword`转发走。

3、再看上图中的左上部分，`incoming packet`在做`routing decision`之前会先经过`nat preroutings`阶段，我们可以在这个阶段做dnat （目标地址改写），简单来说就是：比如这个数据包原来的`dst ip`是百度的，按理说经过`routing decision`之后会进入forward转发阶段，但是在这里你可以把目标地址改写成自己，让数据流入input通路，在本机截获这个数据包。

4、上图右上角写的：`locally generated packet`，表示这是本机自己生成的流量。它会一路经过各个表的output链，然后流到output interface（网卡）上。你注意下，流量在被打包成outgoing packet之前，会有个localhost dest的判断，如果它判断流量不是发往本机的话，流量会经过nat表的postrouting阶段。一般会在这里做`DNAT`源地址改写。

所以经过对上图的简单分析，如果咱想自定义对流量进行控制，**那该怎么办？**

这并不复杂。但是在这想该怎么办之前，我们得先搞清楚，通常情况下我们会对流量做那些控制？无非如下：

1. 丢弃来自xxx的流量
    
2. 丢弃去往xxx的流量
    
3. 只接收来自xxx的流量
    
4. 在刚流量流入时，将目标地址改写成其他地址
    
5. 在流量即将流出前，将源地址改写成其他地址
    
6. 将发往A的数据包，转发给B
    

等等等等，如果你足够敏感，你就能发现，上面这六条干预策略，`filter`、`nat`这两张表已经完全能满足我们的需求了，我们只需要在这两张表的不同链上加自己的规则就行，如下：

1. 丢弃来自xxx的流量（`filter表INPUT链`）
    
2. 丢弃去往xxx的流量（`filter表OUTPUT链`）
    
3. 只接收来自xxx的流量（`filter表INPUT链`）
    
4. 在刚流量流入时，将目标地址改写成其他地址（`nat表prerouting链`）
    
5. 在流量即将流出前，将源地址改写成其他地址（`nat表postrouting链`）
    
6. 将发往A的数据包，转发给B（`filter表forward链`）

数据包在iptables中的走向还可以简化成下面这张图
![[Pasted image 20250227162438.png]]

# iptables commands
```
iptables -t ${表名}  ${Commands} ${链名}  ${链中的规则号} ${匹配条件} ${目标动作}
```

**表名**：4张表，`filter`、`nat`、`mangle`、`raw`

**Commands**：尾部追加`-A`、检查`-C`、删除`-D`、头部插入`-I`、替换`-R`、查看全部`-L`、清空`-F`、新建chain`-N`、默认规则`-P`(默认为ACCEPT)

**链名**：5条链，`PREROUTING`、`INPUT`、`FORWOARD`、`OUTPUT`、`POSTROUTING`

**匹配条件**：`-p`协议、`-4` 、`-6`、`-s 源地址`、`-d 目标地址`、`-i 网络接口名`

**目标动作**：拒绝访问`-j REJECT`、允许通过`-j ACCEPT`、丢弃`-j DROP`、记录日志 `-j LOG`、源地址转换`-j snat`、目标地址转换`-j dnat`、还有`RETURN`、`QUEUE`



# 案例：filter 的流量过滤
比如我想将发送百度的数据包丢弃，就可以在OUTPUT Chain上添加如下的规则
```
iptables -t filter -A OUTPUT -p tcp -d 220.181.38.251 -j DROP
```

# iptables 的匹配规则
**什么是匹配规则？**

这个东西并不难理解，他其实就是一种描述。比如说：我想丢弃来自A的流量，体现在iptables语法上就是：`iptables -t filter -A INPUT -s ${A的地址} -j DROP` ，这段话中的`来自A`其实就是匹配规则。

**常见的规则如下：**

源地址：`-s 192.168.1.0/24`

目标地址：`-d 192.168.1.11`

协议：`-p tcp|udp|icmp`

从哪个网卡进来：`-i eth0|lo`

从哪个网卡出去：`-o eth0|lo`

目标端口（必须制定协议）：`-p tcp|udp --dport 8080`

源端口（必须制定协议）：`-p tcp|udp --sport 8080`

# nat 表
查看NAT表，可以找到它有4条链，如下
![[Pasted image 20250227163617.png]]
通常我们会通过给nat表中的`PREROUTING`、`POSTROUTING`这两条链添加规则来实现`SNAT`、`DNAT`如下:

`-j SNAT` ，源地址转换说的时在数据包发送出去前，我们将数据包的src ip修改成期望的值。所以这个规则需要添加在`POSTROUTING`中。

`-j DNAT`，目标地址转换？比如发送者发过来的数据包的`dst ip = A`，那目标地址转换就是允许我们修改`dst ip=B`，也就是将数据包发送给B处理，这个过程对发送者来说是感知不到的，他会坚定的认为自己的数据包会发往A，而且B也感知不到这个过程，它甚至会认为这个数据包从一开始就是发给他的。

`-j MASQUERADE`，地址伪装

## 案例：使用 nat 表完成 SNAT
案例：将`src ip = 10.10.10.10`转换成`src ip = 22.22.22.22`然后将数据包发给期望的目的ip所在的机器
![[Pasted image 20250227163720.png]]
总结：源地址转换一般是数据包从私网流向公网时做的转换，将私网地址转换成公网地址，会将这个转换记录记录到地址转换表中，目的是为了数据包从公网回来时，能正确的回到私网中发送请求的机器中。