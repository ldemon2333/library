gcc -g 选项增加debugging symbols for the debugger

gdb -tui 选项start gdb with the text user interface

## 1.1 TUI Mode(recommended)
The _Text User Interface_ (TUI) is enabled by running `gdb` with the `-tui` option. It shows

- Commands and history towards the bottom
- Source code position towards the top

The screen will occasionally get "messed up" which can be corrected by pressing `Control-L` which will force a redraw of the terminal screen.

## 1.2 Normal Mode
这与通常的gdb 模式形成了对比，后者除非发出list命令，否则不显示任何源代码。大多数人发现这很难处理，除非他们将其作为另一个编辑器(如 emacs)的子进程运行。

## 1.3 切换模式
在normal mode，可以进入TUI模型使用
```
tui enable
```
切换回来
```
tui disable
```

# 3.标准GDB命令

## 3.1 设置断点
breakpoint 指示调试器将在程序中停止执行的位置。
![[Pasted image 20241017113253.png]]

## 3.2 Arguments and Running
![[Pasted image 20241017113841.png]]

## 3.3 Stepping
单步跟踪
![[Pasted image 20241017114345.png]]

## 3.4 打印内存和栈中的值

| Command      | Effect                                                                   | Notes         |
| ------------ | ------------------------------------------------------------------------ | ------------- |
| print a      | 打印变量 a 的值，该值必须在当前函数中                                                     | 根据其 C 类型格式化 a |
| print/x a    | Print value of `a` as a hexadecimal number                               |               |
| print/o a    | Print value of `a` as a octal number                                     |               |
| print/t a    | Print value of `a` as a binary number (show all bits)                    |               |
| print/s a    | Print value of `a` as a string even if it is not one                     |               |
| print arr[2] | Print value of `arr[2]` according to its C type                          |               |
| print 0x4A25 | Print decimal value of hex constant `0x4A25` which is 18981              |               |
|              |                                                                          |               |
| x a          | 检查内存地址a存储的值                                                              | a是一个指针        |
| x/d a        | 检查内存地址a存储的值，输出十进制                                                        |               |
| x/s a        |                                                                          |               |
| x/s (a+4)    | 检查内存地址(a+4)存储的值                                                          |               |
|              |                                                                          |               |
| x $rax       | 打印寄存器 rax 指向的内存单元的值                                                      |               |
| x $rax+8     | Print memory 8 bytes above where register `rax` points                   |               |
| x /wx $rax   | Print as "words" of 32-bit numbers in hexadecimal format                 |               |
| x /gx $rax   | Print as "giant" 64-bit numbers in hexadecimal format                    |               |
| x /5gd $rax  | Print 5 64-bit numbers starting where `rax` points; use decimal format\| |               |
| x /3wd $rsp  | Print 3 32-bit numbers at the top of the stack (starting at `rsp`)       |               |
|              |                                                                          |               |

## 3.6 Command 历史 和 Screen 管理 in TUI Mode
![[Pasted image 20241017122108.png]]

# 4. 用汇编进行调试
## 4.1

| Command                  | TUI -tui mode                                      | Normal Mode                                                      |
| ------------------------ | -------------------------------------------------- | ---------------------------------------------------------------- |
| `Ctrl-l` (control L)     | Re-draw the screen to remove cruft (do this a LOT) | Clear the screen                                                 |
| layout asm               | Show assembly code window if not currently shown   |                                                                  |
| layout reg               | Show a window holding register contents            |                                                                  |
| winheight regs +2        | Make the registers window 2 lines bigger           |                                                                  |
| winheight regs -1        |                                                    |                                                                  |
| info reg                 |                                                    |                                                                  |
| list                     |                                                    |                                                                  |
| `disassemble` OR `disas` |                                                    | Show lines of assembly in binary, includes instruction addresses |
## 4.2编译和运行程序集文件(源代码可用)
编译带有调试标志的程序集文件将导致 gdb 在 TUI 源窗口中默认显示该程序集代码，并导致 step 命令一次移动一条指令。
![[Pasted image 20241017131823.png]]

## 4.3 Debug Binaries (no source available)
如果没有调试符号，gdb 就不知道要显示什么源代码。由于二进制文件对应于程序集，因此总是可以让调试器用 TUI 显示具有布局等同性的程序集代码。此外，step i 命令应该用于一次执行单个汇编指令，因为 step 可能会尝试找出要执行的 C 级操作，其中可能包括多个汇编指令。

![[Pasted image 20241017132213.png]]

# 5.二进制文件中的断点
使用 C 和 Assembly 源文件设置断点很容易通过文件/行号完成。但是，如果没有可用的源，就会变得有点棘手，因为必须根据其他条件设置断点。这里有一些技巧。