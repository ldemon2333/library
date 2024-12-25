在 GCC 编译器中，`-O0` 和 `-O` 都是控制优化级别的选项，但它们有不同的效果。

1. **`-O0`（无优化）**：
    
    - 这个选项表示禁用优化。编译器会尽量保留源代码的结构，生成的目标代码尽可能接近源代码。这使得调试更容易，因为生成的代码与源代码之间的映射关系更直接。
    - 生成的代码通常较大，执行效率较低，但对调试非常友好。
2. **`-O`（优化级别 1）**：
    
    - 这个选项启用了基本优化。它会对代码进行一些优化，提升程序的运行效率，但不会做过多的优化，以免影响调试或引入不必要的编译时间。
    - `-O` 优化可能会包括删除未使用的变量、合并相邻的指令、常量折叠等简单的优化。
    - 相较于 `-O0`，`-O` 生成的代码通常会小一些，执行速度更快，但可能会稍微影响调试过程，因为优化后的代码可能与源代码的结构有所不同。

总结：

- `-O0`：不做优化，便于调试。
- `-O`：开启基本优化，平衡代码大小和执行效率。

### 解决方法

#### 1. **确保使用调试符号**

==在编译程序时，确保启用了**调试符号**，并禁用优化。这可以通过以下方式实现：
`gcc -g -O0 -o my_program my_program.c`

- `-g` 选项确保生成调试符号。
- `-O0` 禁用优化，以确保变量不会被删除或重新安排。

![[Pasted image 20241211164050.png]]

.gdbinit
```
set confirm off
set architecture riscv:rv64
target remote 127.0.0.1:25000
symbol-file kernel/kernel
set disassemble-next-line auto
set riscv use-compressed-breakpoints yes
```


# 命令行调试
![[Pasted image 20241211164136.png]]

另一个窗口
![[Pasted image 20241211164153.png]]

加载符号表
![[Pasted image 20241211164304.png]]

# 用 vscode 调试 xv6

https://www.cnblogs.com/KatyuMarisaBlog/p/13727565.html

下面进入本blog的第二个重头戏：告别gdb的界面，使用vscode来调试内核！

其实vscode本身仅仅是个编辑器，并不具有调试能力，它所做的不过是和gdb交互，将gdb输出的调试信息重新渲染到界面上而已。

调试xv6，需要用到gdb的remote debug模式，由qemu提供一个GDBstub，gdb需要连接到这个GDBstub上，建议阅读以下文档：http://davis.lbl.gov/Manuals/GDB/gdb_17.html

我们需要给在vscode中为xv6配置相应的launch.json文件：

```json
{

    "version": "0.2.0",

    "configurations": [

        {

            "name": "debug xv6",

            "type": "cppdbg",

            "request": "launch",

            "program": "${workspaceFolder}/kernel/kernel",

            "args": [],

            "stopAtEntry": true,

            "cwd": "${workspaceFolder}",

            "miDebuggerServerAddress": "localhost:25000",

            "miDebuggerPath": "/usr/bin/gdb-multiarch",

            "environment": [],

            "externalConsole": false,

            "MIMode": "gdb",

            "setupCommands": [

                {

                    "description": "pretty printing",

                    "text": "-enable-pretty-printing",

                    "ignoreFailures": true

                }

            ],

            "logging": {

                // "engineLogging": true,

                // "programOutput": true,

            }

        }

    ]

}
```

program就是在kernel/下的kernel

miDebuggerServerAddress设定为gdbstub的地址（我的机器上一般是localhost:26000，可以查看makefile的输出确定）

miDebuggerPath是我们调试riscv所用的gdb地址

stopAtEntry设定为true时，程序将在入口处触发一次断点，方便我们打新的断点

logging选项控制vscode端调试过程的输出，engineLogging和programOutput是两个比较重要的调试日志，如果调试出现错误，可以将这两个选项设为true，查看日志输出确认问题所在。

配置好上述文件好，先不要启动调试，先打开一个终端，输入make qemu-gdb。在项目根目录下会有一个.gdbinit文件，打开文件可以看到下面的内容：
![[Pasted image 20241211164738.png]]
.gdbinit文件gdb初始化时的配置文件。当启动gdb时，gdb会自动在根目录下搜索.gdbinit文件，如果有则一定会执行一次其中的配置。这个.gdbinit告诉我们，qemu提供了一个GDBstub(127.0.0.1:26000)。另一台机器启动时可以连接到这个GDBstub上，即可远程调试。

由于我们在vscode中已经设置了target-remote模式，因此在执行vscode中的debug时，(127.0.0.1:26000)的连接会被建立两次，一次由vscode触发，另一次由.gdbinit触发，第二次连接会强行中断第一次连接。因此执行make qemu-gdb后，要将target remote 127.0.0.1:26000这行删去，否则会爆GDBstub错误。
![[Pasted image 20241211164748.png]]
在vscode中点击调试按钮，程序即可到达内核main的入口：
![[Pasted image 20241211165009.png]]

那么在vscode中怎么切换符号表文件呢？底侧栏有一个“调试控制台”，在其中可以直接输入gdb命令。我们只需要输入 -exec file /user/\_sleep，即可切换到_sleep的符号表，现在我们的user/sleep.c下已经可以打断点了！但是如果打断点，一定要在代码侧栏打，不要再调试控制台中用 -exec b func来打，否则vscode会出现异常。
![[Pasted image 20241211165015.png]]

注意我们的断点是红的，说明断点有效。

如果vscode调试提示GDBstub出现问题，基本可以确定时因为gdb的设置出现了问题，可以将launch.json中logging的几个选项置true，然后在底端的“输出”栏看输出日志，定位问题在哪里。


虽然标题是“调试xv6的第一个进程”，但实际上我们省略了这个进程从userinit结束后到initcode被加载前的这一段过程。这段过程需要充分阅读proc.c的源码和xv6 book的相应章节后才能理解。后面我会在讲进程和进程调度时仔细讨论这段过程。


# MIT6.S081使用gdb-multiarch调试在ecall处打断点后stepi无法进入的问题
发现Lec06 Isolation & system call entry/exit (Robert)中使用gdb-multiarch只在ecall处打断点不会进入trampoline.S中的uservec，需要在手动在下一条中打断点

解决方法：
![[Pasted image 20241217163807.png]]
