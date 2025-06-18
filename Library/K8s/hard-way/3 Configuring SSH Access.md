# Enable root SSH Access
If `root` SSH access is enabled for each of your machines you can skip this section.

By default, a new `debian` install disables SSH access for the `root` user. This is done for security reasons as the `root` user has total administrative control of unix-like systems. If a weak password is used on a machine connected to the internet, well, let's just say it's only a matter of time before your machine belongs to someone else. As mentioned earlier, we are going to enable `root` access over SSH in order to streamline the steps in this tutorial. Security is a tradeoff, and in this case, we are optimizing for convenience. Log on to each machine via SSH using your user account, then switch to the `root` user using the `su` command:

```
su - root
```

Edit the `/etc/ssh/sshd_config` SSH daemon configuration file and set the `PermitRootLogin` option to `yes`:
```
sed -i \
  's/^#*PermitRootLogin.*/PermitRootLogin yes/' \
  /etc/ssh/sshd_config
```


```
sed -i \
  's/^#*PermitRootLogin.*/PermitRootLogin yes/' \
  /etc/ssh/sshd_config 
```

这个命令用于**修改 `/etc/ssh/sshd_config` 文件**，将其中控制 `root` 用户登录权限的设置进行调整。

### 命令分解：

1. **`sed`**:    
    - 这是一个“流编辑器”（stream editor），用于对文本文件进行行处理。它会读取文件，对每一行应用指定的编辑操作，然后输出结果。
2. **`-i` (第一个 `-i`)**:
    - 这个选项是 `--in-place` 的简写，它告诉 `sed` **直接修改原始文件，而不是将结果输出到标准输出**。
    - **重要提示：** 如果没有 `-i` 选项，`sed` 会将修改后的内容打印到屏幕上，而不会改变原始文件。使用 `-i` 则会原地修改文件，这在脚本中非常有用，但也意味着操作是不可逆的（除非您有备份）。
    - 在一些 `sed` 版本中，您可以在 `-i` 后面跟一个字符串（例如 `sed -i.bak`），这样 `sed` 会在修改前创建一个备份文件（例如 `sshd_config.bak`）。但在这个命令中，`-i` 后面没有跟字符串，所以它会直接修改文件，不创建备份。
3. **`'s/^#*PermitRootLogin.*/PermitRootLogin yes/'`**:
    
    - 这是 `sed` 的**编辑操作**，用单引号括起来以防止 Shell 对其中的特殊字符进行解释。
    - **`s`**: 表示 `substitute`（替换）操作。
    - **`^`**: 匹配行的开头。
    - **`#*`**: 匹配零个或多个 `#` 字符。这使得命令能够匹配被注释掉的行（例如 `#PermitRootLogin no`）以及未注释的行（例如 `PermitRootLogin prohibit-password`）。
    - **`PermitRootLogin`**: 匹配字面字符串 "PermitRootLogin"。
    - **`.*`**: 匹配任意数量（包括零个）的任意字符。这会匹配 `PermitRootLogin` 后面直到行尾的所有内容（例如 `yes`、`no`、`prohibit-password`）。
    - `/`: 分隔符，将匹配模式和替换字符串分开。
    - **`PermitRootLogin yes`**: 这是替换字符串。它会将匹配到的整个模式（包括可能存在的 `#` 和 `PermitRootLogin` 后的旧值）替换为 `PermitRootLogin yes`。
    - `/`: 替换命令的结束分隔符。
4. **`/etc/ssh/sshd_config`**:
    
    - 这是 `sed` 命令要操作的**目标文件**。它是 SSH 服务器的配置文件。

### 命令的最终效果：

这个命令会打开 `/etc/ssh/sshd_config` 文件，查找所有以 `PermitRootLogin` 开头（无论前面是否有 `#` 注释）的行，然后将其替换为 `PermitRootLogin yes`。

**例如：**

- 如果文件中有 `#PermitRootLogin no`，它会变成 `PermitRootLogin yes`。
- 如果文件中有 `PermitRootLogin prohibit-password`，它会变成 `PermitRootLogin yes`。
- 如果文件中有 `PermitRootLogin yes`，它会保持不变。

**目的：**

通常，这个命令的目的是**允许 SSH 服务器接受 `root` 用户通过密码直接登录**。这在 Kubernetes The Hard Way 这样的教程中可能用于简化部署和管理，但在生产环境中，直接允许 `root` 密码登录通常被认为是**不安全的**，通常会建议禁用 `root` 密码登录，而使用密钥登录或限制特定用户登录。

Restart the `sshd` SSH server to pick up the updated configuration file:
```
systemctl restart sshd
```

# Generate and Distribute SSH Keys
In this section you will generate and distribute an SSH keypair to the `server`, `node-0`, and `node-1`, machines, which will be used to run commands on those machines throughout this tutorial. Run the following commands from the `jumpbox` machine.

Generate a new SSH key:
```
ssh-keygen
```
```
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /root/.ssh/id_rsa
Your public key has been saved in /root/.ssh/id_rsa.pub
```

Copy the SSH public key to each machine:
```
while read IP FQDN HOST SUBNET; do
  ssh-copy-id root@${IP}
done < machines.txt
```

`ssh-copy-id` 是一个非常方便的命令行工具，用于**将你的 SSH 公钥（public key）安全地复制到远程服务器上**。它的主要目的是自动化 SSH 无密码登录的设置过程。

### `ssh-copy-id` 的核心功能

1. **自动化公钥部署：** 它将本地用户的 SSH 公钥（通常是 `~/.ssh/id_rsa.pub` 或 `~/.ssh/id_ed25519.pub`）添加到远程服务器上目标用户的 `~/.ssh/authorized_keys` 文件中。
2. **处理目录和权限：** 如果远程服务器上目标用户的 `~/.ssh` 目录不存在，`ssh-copy-id` 会自动创建它。它还会确保 `~/.ssh` 和 `~/.ssh/authorized_keys` 文件的权限设置正确（这对于 SSH 密钥认证的安全性和功能性至关重要）。
3. **防止重复添加：** 它会检查目标 `authorized_keys` 文件中是否已经存在要复制的公钥，以避免重复添加。

### 为什么需要 `ssh-copy-id`？

在不使用 `ssh-copy-id` 的情况下，手动设置 SSH 无密码登录需要执行以下繁琐且容易出错的步骤：

1. 生成 SSH 密钥对 (`ssh-keygen`)。
2. 通过密码登录到远程服务器。
3. 在远程服务器上创建 `~/.ssh` 目录（如果不存在）。
4. 设置 `~/.ssh` 目录的正确权限（`chmod 700 ~/.ssh`）。
5. 在远程服务器上创建或编辑 `~/.ssh/authorized_keys` 文件。
6. 将本地的公钥内容复制粘贴到 `authorized_keys` 文件中。
7. 设置 `authorized_keys` 文件的正确权限（`chmod 600 ~/.ssh/authorized_keys`）。
8. 退出远程服务器。

`ssh-copy-id` 将所有这些步骤自动化，大大简化了设置过程，并减少了因权限设置错误而导致登录失败的可能性。

### `ssh-copy-id` 如何工作？

当你运行 `ssh-copy-id user@remote_host` 时：

1. `ssh-copy-id` 会尝试**使用密码**（或你已有的 SSH 密钥）登录到远程服务器的 `user@remote_host`。
2. 登录成功后，它会在远程服务器上执行一系列命令：
    - 检查并创建 `/home/user/.ssh` 目录，并设置其权限为 `700`（只有所有者可读写执行）。
    - 将你的本地公钥内容追加到 `/home/user/.ssh/authorized_keys` 文件中。
    - 设置 `authorized_keys` 文件的权限为 `600`（只有所有者可读写）。
3. 完成这些操作后，`ssh-copy-id` 会退出。

从这时起，你就可以直接使用 `ssh user@remote_host` 进行无密码登录了（前提是你的本地私钥是可用的，并且如果私钥有密码，你已经解锁了它，例如通过 `ssh-agent`）。

### 基本语法
```
ssh-copy-id [options] [user@]hostname
```

**常用示例：**

- **复制默认公钥到远程主机：**
    
    ```
    ssh-copy-id username@remote_host
    ```
    
    （这会尝试复制 `~/.ssh/id_rsa.pub` 或 `~/.ssh/id_ed25519.pub` 等默认公钥。）
    
- **指定要复制的公钥文件：**
    
    ```
    ssh-copy-id -i ~/.ssh/my_custom_key.pub username@remote_host
    ```
    
- **指定 SSH 端口（如果不是默认的 22）：**

    
    ```
    ssh-copy-id -p 2222 username@remote_host
    ```
    

### 前提条件

1. **本地存在 SSH 密钥对：** 你必须已经使用 `ssh-keygen` 命令生成了你的 SSH 密钥对（公钥和私钥）。
2. **首次需要密码或现有密钥登录：** 在执行 `ssh-copy-id` 时，它会尝试首次登录远程服务器。这意味着你需要知道远程用户的密码，或者你的本地机器上已经有一个密钥可以让你登录到该服务器。

`ssh-copy-id` 是实现 SSH 无密码登录的第一步，也是最重要的一步，它让这个过程变得高效和安全。

简而言之，`ssh -n` 就像是告诉 `ssh`：“你不需要我的任何输入，尽管去远程执行命令吧，我可以把你放进后台。”这对于那些启动后不需要持续交互的远程命令，特别是图形界面程序，非常有用。

## 不加 -n 的后果
根据您提供的 `while read IP FQDN HOST SUBNET; do ssh root@${IP} hostname done < machines.txt` 命令，**只会有一次输出**的原因在于：

**`ssh` 命令的标准输入 `< machines.txt` 仅在 `ssh` 命令的第一个执行周期被消耗。**

让我们分解一下这个命令的执行流程：

1. **`while read IP FQDN HOST SUBNET; do ... done < machines.txt`**:
    
    - 最外层的 `while read` 循环的**标准输入源**是 `machines.txt` 文件。这意味着 `read` 命令会从 `machines.txt` 中逐行读取数据，并将每行的字段分别赋值给 `IP`, `FQDN`, `HOST`, `SUBNET` 变量。
2. **`ssh root@${IP} hostnamedone < machines.txt`**:
    
    - 这是循环内部的命令。
    - `ssh root@${IP} hostnamedone`: 这部分命令会连接到远程服务器（`root@${IP}`），并在远程服务器上执行 `hostnamedone` 这个命令或脚本。
    - **`< machines.txt`**: 这是问题的关键点。**这个 `< machines.txt` 是 `ssh` 命令的标准输入重定向，而不是 `while read` 循环的标准输入重定向。**

**执行步骤分析：**

- **第一次循环迭代：**
    
    1. `while read IP FQDN HOST SUBNET` 从 `machines.txt` 的**第一行**读取数据，例如：`192.168.1.1 node1.example.com node1 192.168.1.0/24`。此时 `IP` 变量被赋值为 `192.168.1.1`。
    2. 执行 `ssh root@192.168.1.1 hostnamedone < machines.txt`。
    3. **注意：** 此时 `ssh` 命令的标准输入被重定向到 `machines.txt`。这意味着远程服务器上执行的 `hostnamedone` 命令，它的**标准输入将是 `machines.txt` 文件的全部内容**。
    4. `hostnamedone` 命令会处理 `machines.txt` 的内容并产生一次输出。
    5. **重要：** `ssh` 命令在执行过程中会完全消耗掉 `machines.txt` 文件作为其标准输入。
- **第二次循环迭代及以后：**
    
    1. `while read IP FQDN HOST SUBNET` 尝试再次从 `machines.txt` 读取数据。
    2. 然而，由于在第一次循环中，**`ssh` 命令已经将 `machines.txt` 的内容从头到尾读取了一遍**，导致文件的“读取指针”已经到达了文件末尾。
    3. 因此，当 `while read` 再次尝试从 `machines.txt` 读取时，它会发现文件已经没有更多内容可读了。
    4. `read` 命令读取失败，循环条件不再满足，`while` 循环随即终止。

**结论：**
导致只有一个输出的原因是，内层的 `ssh` 命令通过 `< machines.txt` 劫持并消耗了整个 `machines.txt` 文件作为其标准输入，使得外层的 `while read` 循环在第一次迭代后就无法再从 `machines.txt` 读取到数据了。

# Hostnames
In this section you will assign hostnames to the `server`, `node-0`, and `node-1` machines. The hostname will be used when executing commands from the `jumpbox` to each machine. The hostname also plays a major role within the cluster. Instead of Kubernetes clients using an IP address to issue commands to the Kubernetes API server, those clients will use the `server` hostname instead. Hostnames are also used by each worker machine, `node-0` and `node-1` when registering with a given Kubernetes cluster.

To configure the hostname for each machine, run the following commands on the `jumpbox`.

Set the hostname on each machine listed in the `machines.txt` file:
```
while read IP FQDN HOST SUBNET; do
    CMD="sed -i 's/^127.0.1.1.*/127.0.1.1\t${FQDN} ${HOST}/' /etc/hosts"
    ssh -n root@${IP} "$CMD"
    ssh -n root@${IP} hostnamectl set-hostname ${HOST}
    ssh -n root@${IP} systemctl restart systemd-hostnamed
done < machines.txt
```

`127.0.1.1` 是一个特殊的 IP 地址，它通常与 **Debian 及其派生发行版**（如 Ubuntu）的系统配置有关，用于处理**主机名解析**。

### 1. 回环地址 (Loopback Address) 的范围

首先，我们需要理解回环地址的概念：

- **`127.0.0.0/8`**：这是一个特殊的 IP 地址块，被称为**回环地址范围**。所有以 `127` 开头的 IP 地址都是回环地址。
- **`127.0.0.1`**：这是最常用的回环地址，通常被称为 `localhost`。它代表“本机”，任何发送到 `127.0.0.1` 的流量都会被操作系统路由回本机，而不会发送到网络上。它主要用于应用程序在同一台机器上进行通信，或用于测试网络服务。

### 2. `127.0.1.1` 的特殊用途（主要是 Debian/Ubuntu）

在大多数 Linux 发行版中，主机名通常直接映射到 `127.0.0.1`。但 Debian 及其派生系统（如 Ubuntu）在安装时，为了处理一些特定的情况，可能会将主机名映射到 `127.0.1.1`，而不是 `127.0.0.1`。

这种配置的目的是：

- **解决无固定 IP 地址或断网环境下的主机名解析问题：** 如果系统没有连接到网络，或者没有分配到静态 IP 地址，`127.0.1.1` 充当一个“占位符”，确保系统可以通过其完整限定域名 (FQDN) 或短主机名进行自我解析。
- **避免与 `127.0.0.1` 混淆：** 某些情况下，`127.0.0.1` 可能被其他服务或应用程序（例如数据库服务、Web 服务器等）绑定，或者被用于更严格的本地网络测试。使用 `127.0.1.1` 作为主机名的解析地址可以避免潜在的冲突。
- **兼容性：** 这种做法与一些特定的软件或脚本有关，这些软件或脚本可能依赖于主机名解析到一个非 `127.0.0.1` 的回环地址。

### 3. 如何查看和修改

这个映射通常会在 `/etc/hosts` 文件中找到。例如，你的 `/etc/hosts` 文件可能包含类似以下内容：

```
127.0.0.1       localhost
127.0.1.1       your_hostname.your_domain your_hostname
# The following lines are desirable for IPv6 capable hosts
::1             localhost ip6-localhost ip6-loopback
ff02::1         ip6-allnodes
ff02::2         ip6-allrouters
```

这里的 `127.0.1.1` 就被映射到了 `your_hostname.your_domain` (FQDN) 和 `your_hostname` (短主机名)。

### 总结

`127.0.1.1` 是一个回环地址，在 **Debian/Ubuntu 等系统**中，它是一个惯例性的配置，用于将系统的主机名映射到本地回环接口，以确保在没有外部网络连接的情况下也能正确解析本机主机名，并避免与 `127.0.0.1` 的潜在冲突。对于普通用户或应用程序来说，它的行为与 `127.0.0.1` 在功能上非常相似，都是指代本机。

Verify the hostname is set on each machine:
```
while read IP FQDN HOST SUBNET; do
  ssh -n root@${IP} hostname --fqdn
done < machines.txt
```

```
server.kubernetes.local
node-0.kubernetes.local
node-1.kubernetes.local
```

# Host Lookup Table
In this section you will generate a `hosts` file which will be appended to `/etc/hosts` file on the `jumpbox` and to the `/etc/hosts` files on all three cluster members used for this tutorial. This will allow each machine to be reachable using a hostname such as `server`, `node-0`, or `node-1`.

Create a new `hosts` file and add a header to identify the machines being added:
```shell
echo "" > hosts
echo "# Kubernetes The Hard Way" >> hosts
```

Generate a host entry for each machine in the `machines.txt` file and append it to the `hosts` file:

```shell
while read IP FQDN HOST SUBNET; do
    ENTRY="${IP} ${FQDN} ${HOST}"
    echo $ENTRY >> hosts
done < machines.txt
```

Review the host entries in the `hosts` file:

```shell
cat hosts
```

```

# Kubernetes The Hard Way
XXX.XXX.XXX.XXX server.kubernetes.local server
XXX.XXX.XXX.XXX node-0.kubernetes.local node-0
XXX.XXX.XXX.XXX node-1.kubernetes.local node-1
```

# Adding `/etc/hosts` Entries To The Remote Machines
In this section you will append the host entries from `hosts` to `/etc/hosts` on each machine listed in the `machines.txt` text file.

Copy the `hosts` file to each machine and append the contents to `/etc/hosts`:

```shell
while read IP FQDN HOST SUBNET; do
  scp hosts root@${HOST}:~/
  ssh -n \
    root@${HOST} "cat hosts >> /etc/hosts"
done < machines.txt
```

At this point, hostnames can be used when connecting to machines from your `jumpbox` machine, or any of the three machines in the Kubernetes cluster. Instead of using IP addresses you can now connect to machines using a hostname such as `server`, `node-0`, or `node-1`.

