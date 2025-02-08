![[Pasted image 20250109213406.png]]

# Docker 架构的工作流程
- 构建镜像：使用 Dockerfile 创建镜像
- 推送镜像到注册表：将镜像上传到 Docker Hub 或私有注册表中
- 拉取镜像：通过 docker pull 从注册表中拉取镜像。
- 运行容器：使用镜像创建并启动容器
- 管理容器：使用 Docker 客户端命令管理正在运行的容器（例如查看日志、停止容器、查看资源使用情况等）。
- 网络与存储：容器之间通过 Docker 网络连接，数据通过 Docker 卷或绑定挂载进行持久化。

# 1. docker 客户端
Docker 客户端是用户与 Docker 守护进程交互的命令行界面（CLI）。它是用户与 Docker 系统的主要交互方式，用户通过 Docker CLI 发出命令，这些命令被发送到 Docker 守护进程，由守护进程执行相应的操作。
- **功能**：允许用户使用命令与 Docker 守护进程通信，如创建容器、构建镜像、查看容器状态等。
- **交互方式**：Docker 客户端与 Docker 守护进程之间通过 REST API 或 Unix 套接字通信。常用的命令行工具是 `docker`，通过它，用户可以发出各种 Docker 操作命令。

# 2. docker 守护进程（Docker Daemon）
Docker 守护进程（通常是 `dockerd`）是 Docker 架构的核心，负责管理容器生命周期、构建镜像、分发镜像等任务。

守护进程通常以后台进程的方式运行，等待来自 Docker 客户端的 API 请求。
**功能**：
- 启动和停止容器。
- 构建、拉取和推送镜像。
- 管理容器的网络和存储。
- 启动、停止、查看容器日志等。
- 与 Docker 注册表进行通信，管理镜像的存储与分发。

Docker 守护进程监听来自 Docker 客户端的请求，并且通过 Docker API 执行这些请求。守护进程将负责容器、镜像等 Docker 对象的管理，并根据请求的参数启动容器、删除容器、修改容器配置等。

启动 Docker 守护进程（通常是自动启动的）：

# 3. Docker 引擎 API
Docker 引擎 API 是 Docker 提供的 RESTful 接口，允许外部客户端与 Docker 守护进程进行通信。通过这个 API，用户可以执行各种操作，如启动容器、构建镜像、查看容器状态等。API 提供了 HTTP 请求的接口，支持跨平台调用。

**功能**：

- 向 Docker 守护进程发送 HTTP 请求，实现容器、镜像的管理。
- 提供 RESTful 接口，允许通过编程与 Docker 进行交互。

可以通过 `curl` 或其他 HTTP 客户端访问 Docker 引擎 API。例如，查询当前 Docker 守护进程的版本：
```
curl --unix-socket /var/run/docker.sock http://localhost/version
```

# 4. Docker 容器 （Docker Containers）
容器是 Docker 的执行环境，它是轻量级、独立且可执行的软件包。容器是从 Docker 镜像启动的，包含了运行某个应用程序所需的一切——从操作系统库到应用程序代码。容器在运行时与其他容器和宿主机共享操作系统内核，但容器之间的文件系统和进程是隔离的。
**功能**：

- 提供独立的运行环境，确保应用程序在不同的环境中具有一致的行为。
- 容器是临时的，通常在任务完成后被销毁。

容器的生命周期是由 Docker 守护进程管理的。容器可以在任何地方运行，因为它们不依赖于底层操作系统的配置，所有的运行时依赖已经封装在镜像中。

启动一个容器：
```
docker run -d ubuntu
```

# 5. Docker 镜像
Docker 镜像是容器的只读模板。每个镜像都包含了应用程序运行所需的操作系统、运行时、库、环境变量和应用代码等。镜像是静态的，用户可以根据镜像启动容器。
**功能**：
- 镜像是构建容器的基础，每个容器实例化时都会使用镜像。
- 镜像是只读的，不同容器使用同一个镜像时，容器中的文件系统层是独立的。
Docker 镜像可以通过 `docker pull` 从 Docker Hub 或私有注册表拉取，也可以通过 `docker build` 从 Dockerfile 构建。

# 6. Docker Registries
Docker 仓库是用来存储 Docker 镜像的地方，最常用的公共仓库是 **Docker Hub**。用户可以从 Docker Hub 下载镜像，也可以上传自己的镜像分享给其他人。除了公共仓库，用户也可以部署自己的私有 Docker 仓库来管理企业内部的镜像。

# 7. Docker Compose
Docker Compose 是一个用于定义和运行多容器 Docker 应用的工具。通过 Compose，用户可以使用一个 `docker-compose.yml` 配置文件定义多个容器（服务），并可以通过一个命令启动这些容器。Docker Compose 主要用于开发、测试和部署多容器的应用。
**功能**：
- 定义和运行多个容器组成的应用。
- 通过 YAML 文件来配置应用的服务、网络和卷等。
创建一个简单的 `docker-compose.yml` 文件来配置一个包含 Web 服务和数据库服务的应用：
![[Pasted image 20250109215303.png]]
启动 Compose 定义的所有服务：
![[Pasted image 20250109215311.png]]

# 8. Docker Swarm
Docker Swarm 是 Docker 提供的集群管理和调度工具。它允许将多个 Docker 主机（节点）组织成一个集群，并通过 Swarm 集群管理工具来调度和管理容器。Swarm 可以实现容器的负载均衡、高可用性和自动扩展等功能。
**功能**：
- 管理多节点 Docker 集群。
- 通过调度器管理容器的部署和扩展。
初始化 Swarm 集群：
```
docker swarm init
```

# 9. Docker Networks
Docker 网络允许容器之间相互通信，并与外部世界进行连接。Docker 提供了多种网络模式来满足不同的需求，如 `bridge` 网络（默认）、`host` 网络和 `overlay` 网络等。

**功能**：
- 管理容器间的网络通信。
- 支持不同的网络模式，以适应不同场景下的需求。

创建一个自定义网络并将容器连接到该网络：
```
docker network create my_network
docker run -d --network my_network ubuntu
```

# 10. Docker Volumes
Docker 卷是一种数据持久化机制，允许数据在容器之间共享，并且独立于容器的生命周期。与容器文件系统不同，卷的内容不会随着容器的销毁而丢失，适用于数据库等需要持久存储的应用。

**功能**：
- 允许容器间共享数据。
- 保证数据持久化，独立于容器的生命周期。

创建并挂载卷：
```
docker volume create my_volume
docker run -d -v my_volume:/data ubuntu
```


# 进入容器
在使用 **-d** 参数时启动容器时，容器会运行在后台，这时如果要进入容器，可以通过以下命令进入：

- **docker attach**：允许你与容器的标准输入（stdin）、输出（stdout）和标准错误（stderr）进行交互。如果从这个容器退出，会导致容器的停止。
    
- **docker exec**：推荐大家使用 docker exec 命令，因为此命令会退出容器终端，但不会导致容器的停止。

# 运行一个 web 应用
接下来让我们尝试使用 docker 构建一个 web 应用程序。

我们将在docker容器中运行一个 Python Flask 应用来运行一个web应用。
![[Pasted image 20250109220931.png]]
参数说明:
- -d:让容器在后台运行。
- -P:将容器内部使用的网络端口随机映射到我们使用的主机上。

# 查看 web 应用容器
使用 docker ps 来查看我们正在运行的容器：
![[Pasted image 20250109221018.png]]
这里多了端口信息。
![[Pasted image 20250109221026.png]]
Docker 开放了 5000 端口（默认 Python Flask 端口）映射到主机端口 32769 上。
这时我们可以通过浏览器访问WEB应用
![[Pasted image 20250109221115.png]]
我们也可以通过 -p 参数来设置不一样的端口：
![[Pasted image 20250109221131.png]]
**docker ps**查看正在运行的容器
![[Pasted image 20250109221142.png]]
容器内部的 5000 端口映射到我们本地主机的 5000 端口上。

# 网络端口的快捷方式
**docker** 还提供了另一个快捷方式 **docker port**，使用 **docker port** 可以查看指定 （ID 或者名字）容器的某个确定端口映射到宿主机的端口号。
![[Pasted image 20250109221258.png]]

# 查看 web 应用程序日志
docker logs [ID或者名字] 可以查看容器内部的标准输出。
![[Pasted image 20250109221441.png]]
**-f:** 让 **docker logs** 像使用 **tail -f** 一样来输出容器内部的标准输出。

从上面，我们可以看到应用程序使用的是 5000 端口并且能够查看到应用程序的访问日志。

# 查看 web 应用程序容器的进程
我们还可以使用 docker top 来查看容器内部运行的进程
![[Pasted image 20250109221607.png]]

# 检查 web 应用程序
使用 **docker inspect** 来查看 Docker 的底层信息。它会返回一个 JSON 文件记录着 Docker 容器的配置和状态信息。
![[Pasted image 20250109221644.png]]

# 创建镜像
当我们从 docker 镜像仓库中下载的镜像不能满足我们的需求时，我们可以通过以下两种方式对镜像进行更改。

- 1、从已经创建的容器中更新镜像，并且提交这个镜像
- 2、使用 Dockerfile 指令来创建一个新的镜像

## 更新镜像
更新镜像之前，我们需要使用镜像来创建一个容器。
![[Pasted image 20250109222034 1.png]]
此时 ID 为 e218edb10161 的容器，是按我们的需求更改的容器。我们可以通过命令 docker commit 来提交容器副本。
![[Pasted image 20250109222108.png]]

各个参数说明：

- **-m:** 提交的描述信息
    
- **-a:** 指定镜像作者
    
- e218edb10161：容器 ID
    
- **runoob/ubuntu:v2:** 指定要创建的目标镜像名

我们可以使用 **docker images** 命令来查看我们的新镜像 **runoob/ubuntu:v2**：
![[Pasted image 20250109222905.png]]

## 构建镜像
我们使用命令 **docker build** ， 从零开始来创建一个新的镜像。为此，我们需要创建一个 Dockerfile 文件，其中包含一组指令来告诉 Docker 如何构建我们的镜像。
```dockerfile
runoob@runoob:~$ cat Dockerfile 
FROM    centos:6.7
MAINTAINER      Fisher "fisher@sudops.com"

RUN     /bin/echo 'root:123456' |chpasswd
RUN     useradd runoob
RUN     /bin/echo 'runoob:123456' |chpasswd
RUN     /bin/echo -e "LANG=\"en_US.UTF-8\"" >/etc/default/local
EXPOSE  22
EXPOSE  80
CMD     /usr/sbin/sshd -D
```
每一个指令都会在镜像上创建一个新的层，每一个指令的前缀都必须是大写的。
第一条FROM，指定使用哪个镜像源
RUN 指令告诉docker 在镜像内执行命令，安装了什么。。。
然后，我们使用 Dockerfile 文件，通过 docker build 命令来构建一个镜像。
![[Pasted image 20250109224057.png]]
参数说明：
- **-t** ：指定要创建的目标镜像名
- **.** ：Dockerfile 文件所在目录，可以指定Dockerfile 的绝对路径

使用docker images 查看创建的镜像已经在列表中存在,镜像ID为860c279d2fec

## 设置镜像标签
我们可以使用 docker tag 命令，为镜像添加一个新的标签。
![[Pasted image 20250109224255.png]]

# Docker 容器连接
容器中可以运行一些网络应用，要让外部也可以访问这些应用，可以通过 -P 或 -p 参数来指定端口映射。

下面我们来实现通过端口连接到一个 docker 容器。

要在 Docker 中启动一个网络应用，你需要按照以下步骤进行操作：

### 1. 创建 Docker 网络（如果需要）

首先，创建一个自定义网络，以便容器之间可以通过这个网络进行通信。

```bash
docker network create my_network
```

### 2. 编写 Dockerfile（如果你还没有镜像）

如果你没有现成的镜像，可以编写一个 Dockerfile 来构建应用镜像。假设你有一个简单的网络应用（比如基于 Node.js 的应用），你可以编写一个 Dockerfile：

```Dockerfile
# 使用官方 Node.js 镜像
FROM node:14

# 设置工作目录
WORKDIR /app

# 复制 package.json 和 package-lock.json
COPY package*.json ./

# 安装依赖
RUN npm install

# 复制项目文件
COPY . .

# 暴露应用的端口
EXPOSE 3000

# 启动应用
CMD ["npm", "start"]
```

### 3. 构建 Docker 镜像

在包含 Dockerfile 的目录下运行以下命令来构建镜像：

```bash
docker build -t my_network_app .
```

### 4. 启动容器

启动容器并连接到你创建的网络上：

```bash
docker run -d --name my_network_container --network my_network -p 3000:3000 my_network_app
```

在这个命令中：

- `-d`：后台运行容器。
- `--name`：为容器指定一个名称。
- `--network`：将容器连接到你创建的自定义网络 `my_network`。
- `-p 3000:3000`：将容器的 3000 端口映射到宿主机的 3000 端口。
- `my_network_app`：使用之前构建的镜像。

### 5. 访问应用

此时，应用已经运行在容器内并暴露端口。你可以通过访问宿主机的 IP 或 `localhost`（如果你在本地运行 Docker）来访问应用：

```bash
http://localhost:3000
```

### 6. 验证容器

你可以查看正在运行的容器，并验证网络配置：

```bash
docker ps
docker network inspect my_network
```

这样，你就可以通过 Docker 启动并运行一个网络应用了。如果有多个容器需要进行通信，你可以在同一个网络中启动它们，它们可以通过容器名相互访问。


# Docker 容器互联
## 新建网络
下面先创建一个新的 Docker 网络
![[Pasted image 20250110140404.png]]
参数说明：

**-d**：参数指定 Docker 网络类型，有 bridge、overlay。

其中 overlay 网络类型用于 Swarm mode，在本小节中你可以忽略它。

## 连接容器
运行一个容器并连接到新建的 test-net 网络:
![[Pasted image 20250110140458.png]]
![[Pasted image 20250110140503.png]]

下面通过 ping 来证明 test1 容器和 test2 容器建立了互联关系。

如果 test1、test2 容器内中无 ping 命令，则在容器内执行以下命令安装 ping（即学即用：可以在一个容器里安装好，提交容器到镜像，在以新的镜像重新运行以上俩个容器）。
```
apt-get update
apt install iputils-ping
```

在test1 容器里输入以下命令：
![[Pasted image 20250110142327.png]]
同理在 test2 容器也会成功连接到:
![[Pasted image 20250110142341.png]]

这样，test1 容器和 test2 容器建立了互联关系。
如果你有多个容器之间需要互相连接，推荐使用 Docker Compose，后面会介绍。

# 配置 DNS
我们可以在宿主机的 /etc/docker/daemon.json 文件中增加以下内容来设置全部容器的 DNS：
```json
{
  "dns" : [
    "114.114.114.114",
    "8.8.8.8"
  ]
}
```
设置后，启动容器的 DNS 会自动配置为 114.114.114.114 和 8.8.8.8。
配置完，需要重启 docker 才能生效。
查看容器的 DNS 是否生效可以使用以下命令，它会输出容器的 DNS 信息：

```
$ docker run -it --rm  ubuntu  cat etc/resolv.conf
```
![[Pasted image 20250110145409.png]]

## 手动指定容器的配置
如果只想在指定的容器设置 DNS，则可以使用以下命令：
![[Pasted image 20250110145440.png]]
参数说明：
**--rm**：容器退出时自动清理容器内部的文件系统。

**-h HOSTNAME 或者 --hostname=HOSTNAME**： 设定容器的主机名，它会被写到容器内的 /etc/hostname 和 /etc/hosts。

**--dns=IP_ADDRESS**： 添加 DNS 服务器到容器的 /etc/resolv.conf 中，让容器用这个服务器来解析所有不在 /etc/hosts 中的主机名。

**--dns-search=DOMAIN**： 设定容器的搜索域，当设定搜索域为 .example.com 时，在搜索一个名为 host 的主机时，DNS 不仅搜索 host，还会搜索 host.example.com。