NAT（**Network Address Translation**，网络地址转换）是计算机网络中一个非常核心的机制，尤其是在 Kubernetes、容器网络、出入公网等场景下非常关键。下面我们来系统地讲一讲 NAT。

---

## 🧠 一句话理解 NAT：

> NAT 允许一个或多个**私有 IP 地址**通过一个**公共 IP**访问互联网，或者在局域网内实现 IP 地址转换。

---

## 🧱 NAT 的三种基本类型：

|类型|英文名|作用|
|---|---|---|
|**源地址转换**|**SNAT**（Source NAT）|改变**源 IP 地址**，常用于 **内网出网**|
|**目标地址转换**|**DNAT**（Destination NAT）|改变**目标 IP 地址**，常用于 **将公网请求转发给内网服务器**|
|**双向 NAT**|Full NAT|同时改变源 IP 和目标 IP（较少见）|

---

## 📦 例子讲解：

### 🔁 SNAT 示例（内网出网）：

假设你有一台内网机器 `192.168.1.10`，要访问百度 `www.baidu.com`：

```text
192.168.1.10 → [NAT设备] → 123.45.67.89（公网 IP）→ 百度
```

这时 NAT 设备会：

- 把源地址 `192.168.1.10` 改成 `123.45.67.89`（公网 IP）；
    
- 并记录 NAT 映射表，等百度返回时再映射回来给 `192.168.1.10`。
    

> Linux 中通常在 `POSTROUTING` 链里进行 SNAT。

---

### 🎯 DNAT 示例（公网访问服务）：

你有一个内网 Web 服务器 `192.168.1.100:80`，公网访问使用 `123.45.67.89:8080`。

```text
用户请求 → 123.45.67.89:8080 → [NAT设备] → 转发到 192.168.1.100:80
```

这时 NAT 设备会：

- 将目标地址 `123.45.67.89:8080` 改为 `192.168.1.100:80`；
    
- 同样记录映射，方便反向转换。
    

> Linux 中通常在 `PREROUTING` 链进行 DNAT。

---

## 🔧 在 Linux 中实现 NAT（iptables）：

### SNAT：

```bash
iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -o eth0 -j SNAT --to-source 123.45.67.89
```

### DNAT：

```bash
iptables -t nat -A PREROUTING -d 123.45.67.89 -p tcp --dport 8080 -j DNAT --to-destination 192.168.1.100:80
```

---

## 🚀 在 Kubernetes 中的 NAT 应用场景：

|场景|NAT 类型|说明|
|---|---|---|
|Pod 出网访问外部互联网|SNAT|由于 Pod 用的是虚拟 IP（如 10.244.x.x），出网时必须被 NAT 成 Node 的 IP|
|NodePort 访问 Service|DNAT|访问 NodeIP:Port 时 kube-proxy 用 DNAT 把请求转发到对应 Pod|
|Ingress、LoadBalancer 入口转发|DNAT|将流量从公网入口转发到后端服务 Pod|

---

## 🎯 NAT 的限制

- NAT 会**打破端到端通信模型**（客户端无法直接知道真正的服务器 IP）；
    
- 会导致一些协议（如 FTP、SIP、P2P）复杂化；
    
- 会让 Debug 更麻烦（比如抓包时看到的是 NAT 后的地址）；
    
- 在 Kubernetes 中，NAT 会影响 Service 的来源 IP（需要开启 `externalTrafficPolicy=Local` 才保留原始 IP）。
    

---

## 🧾 小结

|概念|说明|
|---|---|
|SNAT|改源地址，一般用于内网出网|
|DNAT|改目标地址，一般用于公网访问内网服务|
|使用链|SNAT 用 POSTROUTING，DNAT 用 PREROUTING|
|Kubernetes|Service 转发和 Pod 出网都依赖 NAT 实现|
