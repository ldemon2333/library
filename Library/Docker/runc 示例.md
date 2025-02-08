`runc` 是一个轻量级的、基于容器的运行时，遵循 [Open Container Initiative (OCI)](https://opencontainers.org/) 的标准，它用于启动和管理容器。`runc` 可以直接运行容器，而不依赖于像 Docker 这样的高级容器管理工具。

下面是一个基本的 `runc` 示例，展示如何使用它来创建、启动和管理容器。

### 1. 安装 `runc`

在使用 `runc` 之前，首先需要安装它。你可以通过以下命令来安装：

#### 在 Ubuntu 上安装 `runc`

```bash
sudo apt update
sudo apt install runc
```

#### 在其他系统上安装

如果你使用的是其他系统，参考官方文档或者从 GitHub 上下载并编译 `runc`：

```bash
git clone https://github.com/opencontainers/runc
cd runc
make
sudo make install
```

### 2. 创建容器配置文件

`runc` 需要一个容器的配置文件，通常这个文件是一个 JSON 格式的配置，定义了容器的各种参数，如文件系统、网络、挂载点等。

可以使用 `runc` 提供的 `spec` 命令来生成默认的容器配置文件：

```bash
runc spec
```

此命令会生成一个 `config.json` 文件，包含容器的配置。

### 3. 修改 `config.json`

默认生成的 `config.json` 配置文件中有许多设置项，我们可以根据需要对其进行修改。以下是一些常见的配置：

- **`ociVersion`**：定义 OCI 规范的版本。
- **`root`**：容器根文件系统的设置，指定文件系统路径。
- **`process`**：定义要运行的进程（比如容器中的命令）。
- **`mounts`**：挂载点配置，例如挂载宿主机的目录到容器内部。

修改 `config.json` 文件来指定容器的行为。以下是一个简单的示例：

```json
{
  "ociVersion": "1.0.0",
  "process": {
    "user": {
      "uid": 1000,
      "gid": 1000
    },
    "args": ["/bin/sh"],
    "cwd": "/",
    "env": [
      "HOME=/root"
    ],
    "capabilities": [
      "CAP_SYS_ADMIN"
    ]
  },
  "root": {
    "path": "/var/lib/container_rootfs"
  },
  "mounts": [
    {
      "source": "/home/user/app",
      "destination": "/mnt/app",
      "type": "bind",
      "options": ["rbind", "ro"]
    }
  ]
}
```

### 4. 准备容器文件系统

容器运行时需要一个文件系统，`runc` 使用的是根文件系统（root filesystem）。你可以手动准备文件系统或者使用现成的基础镜像（如 BusyBox、Alpine 等）。

如果你使用的是基础镜像，确保你已经将镜像解压到指定目录，例如 `/var/lib/container_rootfs`。

### 5. 创建容器

使用 `runc` 启动容器前，确保已经有一个有效的 `config.json` 配置文件和容器文件系统。

```bash
runc create mycontainer
```

此命令会创建并启动名为 `mycontainer` 的容器，按照配置文件启动 `/bin/sh` 进程。

### 6. 启动容器

创建容器后，你可以使用 `runc start` 启动容器：

```bash
runc start mycontainer
```

容器会按照配置启动相应的进程（如上面的 `/bin/sh`）。

### 7. 进入容器

如果你需要进入正在运行的容器，可以使用 `runc exec` 命令：

```bash
runc exec mycontainer /bin/sh
```

这个命令会让你进入到容器内的 shell。

### 8. 停止和销毁容器

停止容器使用 `runc stop` 命令：

```bash
runc stop mycontainer
```

销毁容器使用 `runc delete` 命令：

```bash
runc delete mycontainer
```

### 9. 查看容器日志

你可以通过查看容器日志来获取容器的运行状态和信息。默认情况下，`runc` 会将日志输出到容器的标准输出和标准错误。

你可以通过 `journalctl` 或查看容器相关的日志文件来查看。


# Using runc
## Creating an OCI Bundle
In order to use runc you must have your container in the format of an OCI bundle. If you have Docker installed you can use its `export` method to acquire a root filesystem from an existing Docker container.
```
# create the top most bundle directory
mkdir /mycontainer
cd /mycontainer

# create the rootfs directory
mkdir rootfs

# export busybox via Docker into the rootfs directory
docker export $(docker create busybox) | tar -C rootfs -xvf -
```

After a root filesystem is populated you just generate a spec in the format of a `config.json` file inside your bundle. `runc` provides a `spec` command to generate a base template spec that you are then able to edit. To find features and documentation for fields in the spec please refer to the [specs](https://github.com/opencontainers/runtime-spec) repository.

```
runc spec
```

## Running Containers
Assuming you have an OCI bundle from the previous step you can execute the container in two different ways.

The first way is to use the convenience command `run` that will handle creating, starting, and deleting the container after it exits.
```
# run as root
cd /mycontainer
runc run mycontainerid
```



