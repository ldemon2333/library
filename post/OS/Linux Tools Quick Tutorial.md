[Linux工具快速教程](https://linuxtools-rst.readthedocs.io/zh-cn/latest/base/index.html)

# Linux 基础
## 1. 学会使用命令帮助
### 1.1. 概述
在Linux终端，面对命令不知道怎么用，或不记得命令的拼写及参数时，我们需要求助于系统的帮助文档； Linux系统内置的帮助文档很详细，通常能解决我们的问题，我们需要掌握如何正确的去使用它们；

- 在只记得部分命令关键字的场合，我们可通过man -k来搜索；
- 需要知道某个命令的简要说明，可以使用whatis；而更详细的介绍，则可用info命令；
- 查看命令在哪个位置，我们需要使用which；
- 而对于命令的具体参数及使用方法，我们需要用到强大的man；

下面介绍这些命令；

### 1.2. 命令使用

#### 查看命令的简要说明

简要说明命令的作用（显示命令所处的man分类页面）:

```sh
$whatis command
```


正则匹配:

```sh
$whatis -w "loca"
```

更加详细的说明文档:

```sh
$info command
```

#### 使用man
查询命令command的说明文档：

```sh
$man command
```

在man的帮助手册中，将帮助文档分为了9个类别，对于有的关键字可能存在多个类别中，我们就需要指定特定的类别来查看；

```sh
$man man
```

man页面所属的分类标识

1. Executable programs or shell commands
2. System calls
.....

前面说到使用whatis会显示命令所在的具体的文档类别，我们学习如何使用它

![[Pasted image 20240626215403.png]]

我们看到printf在分类1和分类3中都有；分类1中的页面是命令操作及可执行文件的帮助；而3是常用函数库说明；

#### 查看路径
查看程序的binary文件所在路径:

```sh
$which command
```

eg:查找make程序安装路径:

```sh
$which make
/opt/app/openav/soft/bin/make install
```

查看程序的搜索路径:

```sh
$whereis command
```

当系统中安装了同一软件的多个版本时，不确定使用的是哪个版本时，这个命令就能派上用场；

## 2. 文件及目录管理
文件管理不外乎文件或目录的创建、删除、查询、移动，有mkdir/rm/mv

文件查询是重点，用find来进行查询；find的参数丰富，也非常强大；

查看文件内容是个大的话题，文本的处理有太多的工具供我们使用，在本章中只是点到即止，后面会有专门的一章来介绍文本的处理工具；

有时候，需要给文件创建一个别名，我们需要用到ln，使用这个别名和使用原文件是相同的效果；

### 2.1. 创建和删除
查看当前目录下文件个数：

```sh
$find ./ | wc -l
```

复制目录：

```sh
$cp -r source_dir dest_dir
```