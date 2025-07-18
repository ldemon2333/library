# 引言
由于 WSL 采用的是 NAT模式，不能直接与本机 localhost 共享代理端口。

以往采取的方式就是设置`http_proxy`和`https_proxy`.

但，不方便的地方在于每次重启wsl后，由于IP会变化，很有可能需要重新进行配置。

也有办法，可以通过读取`/etc/resolve.conf`，提取其中的`nameserver`，自动提取IP，这样就可以自动化配置。

但，今天这个方案突然失效，`nameserver`提供的IP地址并不能正常使用。

很奇怪，我需要新的方案。

# 官方解决方案
链接：[networking-considerations-with-dns-tunneling](https://learn.microsoft.com/en-us/windows/wsl/troubleshooting#networking-considerations-with-dns-tunneling)，提出通过 DNS 隧道来配置 WSL 的网络

# 我的步骤
在我的 window 主机编辑 `~\.wslconfig`

去‘C:\Users\你的名字\’下面新建一个‘.wslconfig’，然后用记事本打开往里面放这些内容”
```
[wsl2]
memory=8GB
processors=8
[experimental]
autoMemoryReclaim=gradual
networkingMode=mirrored
dnsTunneling=true
firewall=true
autoProxy=true
sparseVhd=true
```

重启 wsl，代理正常工作