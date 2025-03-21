`GOROOT`和`GOPATH`都是环境变量，其中`GOROOT`是我们安装go开发包的路径，而从Go 1.8版本开始，Go开发包在安装完成后会为`GOPATH`设置一个默认目录，并且在Go1.14及之后的版本中启用了Go Module模式之后，不一定非要将代码写到GOPATH目录下，所以也就**不需要我们再自己配置GOPATH**了，使用默认的即可。

Go1.14版本之后，都推荐使用`go mod`模式来管理依赖环境了，也不再强制我们把代码必须写在`GOPATH`下面的src目录了，你可以在你电脑的任意位置编写go代码。（网上有些教程适用于1.11版本之前。）

默认GoPROXY配置是：`GOPROXY=https://proxy.golang.org,direct`，由于国内访问不到`https://proxy.golang.org`，所以我们需要换一个PROXY，这里推荐使用`https://goproxy.io`或`https://goproxy.cn`。

可以执行下面的命令修改GOPROXY：

```
go env -w GOPROXY=https://goproxy.cn,direct
```

# go mod init
使用go module模式新建项目时，我们**需要**通过`go mod init 项目名`命令对项目进行初始化，该命令会在项目根目录下生成`go.mod`文件。例如，我们使用`hello`作为我们第一个Go项目的名称，执行如下命令。
```
go mod init hello
```

# 编译
`go build`命令表示将源代码编译成可执行文件。

在hello目录下执行：
```
go build
```

我们还可以使用`-o`参数来指定编译后得到的可执行文件的名字。

# go run
`go run main.go`也可以执行程序，该命令本质上是先在临时目录编译程序然后再执行。

# go install
`go install`表示安装的意思，它先编译源代码得到可执行文件，然后将可执行文件移动到`GOPATH`的bin目录下。因为我们**把`GOPATH`下的`bin`目录添加到了环境变量中**，所以我们就可以在任意地方直接执行可执行文件了。

在 Fish Shell 中，您可以通过以下几种方式添加环境变量：

### 1. **临时设置环境变量**

如果只想在当前会话中添加环境变量，可以使用 `set` 命令：

```fish
set -x MY_VARIABLE "my_value"
```

- `-x` 选项会将变量导出为环境变量，因此它对所有子进程可见。
- 这个设置在当前会话中有效，关闭终端后会丢失。

**`set -x`**：使用此选项设置的变量将被导出到环境中。也就是说，它不仅存在于当前 shell 会话中，还能被该会话中启动的任何程序访问。

### 2. **永久设置环境变量**

要永久添加环境变量，可以将 `set` 命令添加到 Fish 配置文件中。这样，变量会在每次启动 Fish 时自动加载。

1. 编辑 Fish 配置文件 `config.fish`（通常位于 `~/.config/fish/config.fish`）。
    
    ```bash
    nano ~/.config/fish/config.fish
    ```
    
2. 在文件末尾添加如下命令：
    
    ```fish
    set -x MY_VARIABLE "my_value"
    ```
    
3. 保存文件并退出编辑器。
    
4. 重新加载配置文件以使更改生效：
    
    ```fish
    source ~/.config/fish/config.fish
    ```
    

### 3. **添加多个环境变量**

如果需要添加多个环境变量，可以在 `config.fish` 中逐行设置它们。例如：

```fish
set -x PATH $PATH /path/to/your/directory
set -x MY_API_KEY "your_api_key"
set -x MY_VARIABLE "my_value"
```

### 4. **查看环境变量**

要查看某个环境变量的值，可以使用 `echo` 命令：

```fish
echo $MY_VARIABLE
```

### 5. **删除环境变量**

如果需要删除环境变量，可以使用 `set -e` 命令：

```fish
set -e MY_VARIABLE
```

### 总结：

- 使用 `set -x` 临时添加环境变量（只在当前会话有效）。
- 在 `~/.config/fish/config.fish` 中添加 `set -x` 命令以永久添加环境变量。
- 使用 `echo $VARIABLE_NAME` 查看环境变量。

在 Fish Shell 中，`-g` 选项用于将变量设置为 **全局变量**。这意味着该变量不仅在当前函数或块中可见，还可以在整个 Shell 会话中访问，包括任何函数和子进程。

### 解释：

- **局部变量**：默认情况下，在函数内部使用 `set` 命令设置的变量是局部的，只能在该函数内部使用。
- **全局变量**：通过使用 `-g`，您可以将变量定义为全局变量，这样它可以在函数外部和整个会话中使用。

### 示例：

```fish
function my_function
    set -g MY_VAR "Hello, World!"
end
```

在上面的示例中，`MY_VAR` 被设置为全局变量。即使在 `my_function` 函数外部，`MY_VAR` 也可以访问。

```fish
my_function
echo $MY_VAR  # 输出 "Hello, World!"
```

### 对比：

- `set MY_VAR "value"`：此设置为局部变量，通常在当前代码块或函数中有效。
- `set -g MY_VAR "value"`：此设置为全局变量，可以在整个会话中使用，包括所有函数和代码块。

### 常见用法：

- **全局变量**：`-g` 选项通常用于需要在不同函数或脚本之间共享的变量。
- **临时使用**：您可以在全局范围内设置一些值，然后在整个脚本中访问它们。

### 总结：

- `-g` 是设置全局变量的选项，使得变量在整个 Fish Shell 会话中都可见，无论是在函数内部还是外部。

# 跨平台编译
默认我们`go build`的可执行文件都是当前操作系统可执行的文件，Go语言支持跨平台编译——在当前平台（例如Windows）下编译其他平台（例如Linux）的可执行文件。

## Windows 编译 Linux 可执行文件
只需要在编译时指定目标操作系统的平台和处理器架构即可。

注意：无论你在Windows电脑上使用VsCode编辑器还是Goland编辑器，都要注意你使用的终端类型，因为不同的终端下命令不一样！！！目前的Windows通常默认使用的是`PowerShell`终端。

如果你的`Windows`使用的是`cmd`，那么按如下方式指定环境变量。
```
SET CGO_ENABLED=0  // 禁用CGO
SET GOOS=linux  // 目标平台是linux
SET GOARCH=amd64  // 目标处理器架构是amd64
```

如果你的`Windows`使用的是`PowerShell`终端，那么设置环境变量的语法为

```
$ENV:CGO_ENABLED=0
$ENV:GOOS="linux"
$ENV:GOARCH="amd64"
```

在你的`Windows`终端下执行完上述命令后，再执行`go build`，得到的就是能够在Linux平台运行的可执行文件了。

## Linux 编译 Mac 可执行文件
Linux平台下编译Mac平台64位可执行程序：
```
CGO_ENABLED=0 GOOS=darwin GOARCH=amd64 go build
```

## Linux 编译 Windows 可执行文件
Linux平台下编译Windows平台64位可执行程序：
```
CGO_ENABLED=0 GOOS=windows GOARCH=amd64 go build
```


# 依赖管理

# godep
Go语言从v1.5开始开始引入`vendor`模式，如果项目目录下有vendor目录，那么go工具链会优先使用`vendor`内的包进行编译、测试等。

`godep`是一个通过vender模式实现的Go语言的第三方依赖管理工具，类似的还有由社区维护准官方包管理工具`dep`。


## go module
`go module`是Go1.11版本之后官方推出的版本管理工具，并且从Go1.13版本开始，`go module`将是Go语言默认的依赖管理工具。

### GO111MODULE

要启用`go module`支持首先要设置环境变量`GO111MODULE`，通过它可以开启或关闭模块支持，它有三个可选值：`off`、`on`、`auto`，默认值是`auto`。

1. `GO111MODULE=off`禁用模块支持，编译时会从`GOPATH`和`vendor`文件夹中查找包。
2. `GO111MODULE=on`启用模块支持，编译时会忽略`GOPATH`和`vendor`文件夹，只根据 `go.mod`下载依赖。
3. `GO111MODULE=auto`，当项目在`$GOPATH/src`外且项目根目录有`go.mod`文件时，开启模块支持。

简单来说，设置`GO111MODULE=on`之后就可以使用`go module`了，以后就没有必要在GOPATH中创建项目了，并且还能够很好的管理项目依赖的第三方包信息。

使用 go module 管理依赖后会在项目根目录下生成两个文件`go.mod`和`go.sum`。

## go mod 命令
常用的`go mod`命令如下：
```
go mod download    下载依赖的module到本地cache（默认为$GOPATH/pkg/mod目录）
go mod edit        编辑go.mod文件
go mod graph       打印模块依赖图
go mod init        初始化当前文件夹, 创建go.mod文件
go mod tidy        增加缺少的module，删除无用的module
go mod vendor      将依赖复制到vendor下
go mod verify      校验依赖
go mod why         解释为什么需要依赖
```

## go.mod
go.mod文件记录了项目所有的依赖信息，其结构大致如下：
```sh
module github.com/Q1mi/studygo/blogger

go 1.12

require (
	github.com/DeanThompson/ginpprof v0.0.0-20190408063150-3be636683586
	github.com/gin-gonic/gin v1.4.0
	github.com/go-sql-driver/mysql v1.4.1
	github.com/jmoiron/sqlx v1.2.0
	github.com/satori/go.uuid v1.2.0
	google.golang.org/appengine v1.6.1 // indirect
)
```

其中，

- `module`用来定义包名
- `require`用来定义依赖包及版本
- `indirect`表示间接引用

### go get

在项目中执行`go get`命令可以下载依赖包，并且还可以指定下载的版本。

1. 运行`go get -u`将会升级到最新的次要版本或者修订版本(x.y.z, z是修订版本号， y是次要版本号)
2. 运行`go get -u=patch`将会升级到最新的修订版本
3. 运行`go get package@version`将会升级到指定的版本号version

如果下载所有依赖可以使用`go mod download` 命令。

### 整理依赖

我们在代码中删除依赖代码后，相关的依赖库并不会在`go.mod`文件中自动移除。这种情况下我们可以使用`go mod tidy`命令更新`go.mod`中的依赖关系。

### 新项目

对于一个新创建的项目，我们可以在项目文件夹下按照以下步骤操作：

1. 执行`go mod init 项目名`命令，在当前项目文件夹下创建一个`go.mod`文件。
2. 手动编辑`go.mod`中的require依赖项或执行`go get`自动发现、维护依赖。


