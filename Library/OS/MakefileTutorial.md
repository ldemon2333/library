# Getting Started
## Why do Makefiles exits?
Makefile 用于帮助决定大型程序的哪些部分需要重新编译。

Python、 Ruby 和原始 Javascript 等解释语言不需要类似于 Makefiles 的语言。Makefiles 的目标是根据文件的变化，编译任何需要编译的文件。但是，当解释语言中的文件发生更改时，不需要重新编译任何内容。程序运行时，使用文件的最新版本。

注意: Makefile 必须使用标签缩进，而不能使用空格或 make 将失败。


## Makefile Syntax
A Makefile consists of a set of _rules_. A rule generally looks like this:
![[Pasted image 20241019000259.png]]
- The _targets_ are file names, separated by spaces. Typically, there is only one per rule.
- The _commands_ are a series of steps typically used to make the target(s). These _need to start with a tab character_, not spaces.
- The _prerequisites_ are also file names, separated by spaces. These files need to exist before the commands for the target are run. These are also called _dependencies_


## The essence of Make
![[Pasted image 20241019000541.png]]
- We have one _target_ called `hello`
- This target has two _commands_
- This target has no _prerequisites_

运行`make hello`，只要`hello`文件不存在，这些命令就会运行。如果`hello`存在，这些命令就不会运行。

`hello` 同时是一个target 和 a file。通常，当 target 运行时（即 target 的命令运行时），这些命令将创建与 target 同名的文件。在这种情况下，`hello` target 不会创建 `hello` 文件。

![[Pasted image 20241019185221.png]]

这一次，简单地运行`make`。由于没有将 target 作为`make`命令的参数提供，因此自动运行第一个 target 。在这个例子中，只有一个target (blah)。所以第一次运行时，`blah` 会被创建。第二次运行时，会遇到`make: 'blah' is up to date`问题。这是因为`blah`文件已经存在。但这里就有一个问题：如果我们修改`blah.c`然后运行`make`，就不会编译任何事情。

通过增加prerequisite：
![[Pasted image 20241019185420.png]]
当我们运行`make`时，就会发生以下：
- 第一个target被选中，因为第一个target是默认target
- 有一个依赖`blah.c`
- Make决定是否应该运行`blah`target。只有在`blah`不存在或者`blah.c`比`blah`新的时候会运行

最后一步是关键，也是Make的本质，如果修改了`blah.c`，运行`make`会重新编译该文件。

为此，它使用文件系统时间戳作为代理来确定是否发生了更改。这是一种合理的启发式方法，因为文件时间戳通常只有在文件被修改时才会改变。但重要的是要认识到，情况并非总是如此。例如，您可以修改一个文件，然后将该文件修改后的时间戳更改为旧的内容。如果您这样做了，Make 会错误地猜测文件没有更改，因此可能会被忽略。

## 更多quick例子
接下来的Makefile 最终运行三个targets。
- Make selects the target `blah`, because the first target is the default target
- `blah` requires `blah.o`, so make searches for the `blah.o` target
- `blah.o` requires `blah.c`, so make searches for the `blah.c` target
- `blah.c` has no dependencies, so the `echo` command is run
- The `cc -c` command is then run, because all of the `blah.o` dependencies are finished
- The top `cc` command is run, because all the `blah` dependencies are finished
- That's it: `blah` is a compiled c program
```
blah:blah.o
	cc blah.o -o blah #Runs third
blah.o:blah.c
	cc -c blah.c -o blah.o #Runs second
blah.c:
	echo "int main(){retun 0;}">blah.c # Runs first
```

是从下往上依次执行target。

如果你删除`blah.c`，所以三个targets 会重新运行。如果你更改它（使
它的时间戳比 blah.o 新，前两个targets就会run。如果运行`touch blah.o`（就会改变时间戳使得它新于blah），就会执行第一个target。

`touch blah.o` 命令在 Unix/Linux 系统中有两个主要的作用：

1. **创建空文件**：如果 `blah.o` 文件不存在，`touch` 命令会创建一个名为 `blah.o` 的空文件。
    
2. **更新时间戳**：如果 `blah.o` 文件已经存在，`touch` 命令会更新该文件的“最后修改时间”和“最后访问时间”到当前时间，但不会更改文件的内容。

This next example will always run both targets, because `some_file` depends on `other_file`, which is never created.

![[Pasted image 20241019192018.png]]

## Make clean
clean 做两件事这里：
- 它不是首选目标(默认值) ，也不是先决条件。这意味着除非您明确调用 make clean，否则它永远不会运行
- 它不应该是一个文件名。如果您碰巧有一个名为 clean 的文件，那么这个目标将不会运行，这不是我们想要的。本教程稍后将介绍如何解决这个问题.PHONY
![[Pasted image 20241020135429.png]]

## 变量
变量只能是字符串。通常需要使用:=，但是=也可以。
![[Pasted image 20241020135559.png]]

单引号或双引号没有任何意义。它们只是分配给变量的字符。下列两个命令的行为是相同的：

![[Pasted image 20241020135716.png]]

使用 ${}或 $()的引用变量
![[Pasted image 20241020135756.png]]

# Targets
## The all target
Making multiple targets and you want all of them to run? Make an `all` target. Since this is the first rule listed, it will run by default if `make` is called without specifying a target.
![[Pasted image 20241020140542.png]]

## Multiple targets
When there are multiple targets for a rule, the commands will be run for each target. `$@` is an [automatic variable](https://makefiletutorial.com/#automatic-variables) that contains the target name.

![[Pasted image 20241020140711.png]]

# Automatic Variables and Wildcards
## * Wildcard
Both `*` and `%` are called 通配符 in Make, but they mean entirely different things. `*` searches your filesystem for matching filenames. I suggest that you always wrap it in the `wildcard` function, because otherwise you may fall into a common pitfall described below.
![[Pasted image 20241020140942.png]]* 可以在目标、先决条件或通配符函数中使用。
危险: * 不能直接用于变量定义
危险: 当 * 不匹配任何文件时，它保持原样(除非在通配符函数中运行)

![[Pasted image 20241020141031.png]]

## % 通配符
- 当在“匹配”模式下使用时，它匹配一个字符串中的一个或多个字符。
- 当在“替换”模式下使用时，它将获取匹配的词干，并在字符串中替换该词干。
- % 最常用于规则定义和某些特定函数中。

## Automatic Variables
There are many [automatic variables](https://www.gnu.org/software/make/manual/html_node/Automatic-Variables.html), but often only a few show up:
![[Pasted image 20241020141337.png]]
