Devices generate interrupts, which are one type of trap. The kernel trap handling code recognizes when a device has raised an interrupt and calls the driver's interrupt handler

Many device drivers execute code in two contexts: a top half that runs in a process's kernel thread, and a bottom half that executes at interrupt time. The top half is called via system calls such as `read` and `write` that want the device to perform I/O. This code may ask the hardware to start an operation (e.g., ask the disk to read a block); then the code waits for the operation to complete. Eventually the device completes the operation and raises an interrupt. The driver's interrupt handler, acting as the bottom half, figures out what operaion has completed, waked up a waiting process if appropriate, and tells the hardware to start work on any waiting next operaion.

# 5.1 Code: Console inpute
The console driver accpets characters typed by human, via the _UART_ serial-port hardware attached to the RISC-V. The console driver accumulates a line of input at a time, processing special input charactes such as backspace and control-u. User processes, such as the shell, use the `read` system call to fetch lines of input from the console. When you type input to xv6 in QEMU, youo keystrokes are delivered to xv6 by way of QEMU's simulated UART hardware.

The UART hardware that the driver talks to is a 16550 chip emulated by QEMU. On a real computer, a 16550 would manage an RS232 serial link connecting to a terminal or other computer. When running QEMU, it's connected to your keyboard and display.

The UART hardware appears to software as a set of _memory-mapped_ control registers. That is, there are some physical addresses that RISC-V hardware connects to the UART device, so that loads and stores interact with the device hardware rather than RAM. The memory-mapped addresses for the UART start at 0x10000000, or `UART0`. There are a handful of UART control registers, each the width of a byte. For example, the `LSR(line status register)` register contain bits that indicate whether input charactes are waiting to be read by the software. These charactes (if any) are available for reading from the `RHR(receive holding register)` register. Each time one is read, the UART hardware deletes it from an internal FIFO of waiting charactes, and clears the "ready" bit in `LSR` when the FIFO is empty. The UART transmit hardware is largely independent of the receive hardware; if software writes a byte to the `THR(transmit holding register)`, the UART transmit that byte.

Xv6' `main` calls `consoleinit` to initialize the UART hardware. This code configures the UART to generate a receive interrupt when the UART receives each byte of input, and a _transmit complete_ interrupt each time the UART finishes sending a byte of output.

The xv6 shell reads from the console by way of a file descriptor opened by `init.c`. Calls to the `read` system call make their way through the kernel to `consoleread`. `consoleread` waits for input to arrive (via interrupts) and be buffered in `cons.buf`, copies the input to user space, and (after a whole line has arrived) returns to the user process. If the user hasn't typed a full line yet, any reading processes will wait in the `sleep` call.

When the user types a character, the UART hardware asks the RISC-V to raise an interrupt, which activates xv6's trap handler. The trap handler calls `devintr`, which looks at the RISC-V scause register to discover that the interrupt is from an external device. Then it asks a hardware unit called the `PLIC (Platform-Level Interrupt Controller，负责对从外部设备产生的中断进行管理)` to tell it which device interrupted. If it was the UART, `devintr` calls `uartintr`.

`uartintr` reads any waiting input characters from the UART hardware and hands them to `consoleintr`; it doesn't wait for characters, since future input will raise a new interrupt. The job of `consoleintr` is to accumulate input characters in `cons.buf` until a whole line arrives. `consoleintr` treats backspace and a few other characters specially. When a newline arrives, `consoleintr` wakes up a waiting `consoleread`(if there is one).

Once woken, `consoleread` will observe a full line in `cons.buf`, copy it to user space, and return (via the system call machinery) to user space.

RISC-V对中断的支持：

`SIE`(supervisor interrupt enable) 寄存器用来控制中断，其中有一位是控制外部设备的中断（SEIE），一位控制suffer interrupt(一个CPU向另外一个CPU发出中断)(SSIE)，一位控制定时器中断(STIE)

`SSTATUS`(supervisor status)寄存器，对某一个特定的CPU核控制是否接收寄存器，在kernel/riscv.h中的`intr_on`被设置

`SIP`(supervisor interrupt pending)寄存器，可以观察这个寄存器来判断有哪些中断在pending

case study:

用户在键盘上输入了一个字符l，这个l通过键盘被发送到UART，然后通过PLIC发送到CPU的一个核，这个核产生中断，跑到`devintr`，`devintr`发现是来自UART的，调用`uartintr`，调用`uartgetc()`通过`RHR`寄存器来获取这个字符，然后调用`consoleintr`，判断这个字符是否是特殊字符(backspace等)，如果不是则将这个字符通过`consputc(c)`echo回给user，然后将其存储在`cons.buf`中，当发现整行已经输入完成后(`c=='\n' || c ==C('D'))`)，唤醒`consoleread()`

# 5.2 Code: Console output
A `write` system call on a file descriptor connected to the console eventually arrives at `uartputc`. The device driver maintains an output buffer (`uart_tx_buf`) so that writing processes do not have to wait for the UART to finish sending; instead, `uartputc` appends each character to the buffer, calls `uartstart` to start the device transmitting, and return. The only situation in which `uartputc` waits is if the buffer is already full.

Each time the UART finishes sending a byte, it generates an interrupt. `uartintr` calls `uartstart`, which checks that the device really has finished sending, and hands the device the next buffered output character. Thus if a process writes multiple bytes to the console, typically the first byte will be sent by `uartputc`'s call to `uartstart`, and the remaining buffered bytes will be sent by `uartstart` calls from `uartintr` as transmit complete interrupts arrive.

需要注意的一般模式是通过缓冲和中断将设备活动与进程活动分离。即使没有进程等待读取输入，控制台驱动程序也可以处理输入；后续读取将看到输入。同样，进程可以发送输出而不必等待设备。这种分离可以通过允许进程与设备 I/O 同时执行来提高性能，当设备速度慢（如 UART）或需要立即关注（如回显输入的字符）时尤其重要。这个想法有时称为 I/O 并发。

对console上的文件描述符进行`write`system call，最终到达kernel/uart.c的`uartputc`函数。输出的字节将缓存在`uart_tx_buf`中，这样写入进程就不需要等待UART硬件完成字节的发送，只要当这个缓存区满了的情况下`uartputc`才会等待。当UART完成了一个字符的发送之后，将产生一个中断，`uartintr`将调用`uartstart`来判断设备是否确实已经完成发送，然后将下一个需要发送的字符发送给UART。因此让UART传送多个字符时，第一个字符由`uartputc`对`uartstart`的调用传送，后面的字符由`uartintr`对`uartstart`的调用进行传送

_I/O concurrency_：设备缓冲和中断的解耦，从而让设备能够在没有进程等待读入的时候也能让console driver处理输入，等后面有进程需要读入的时候可以不需要等待。同时进程也可以不需要等待设备而直接写入字符到缓冲区。

在`consoleread`和`consoleintr`中调用了`acquire`来获取一个锁，从而保护当前的console driver，防止同时期其他进程对其的访问造成的干扰。

# 5.3 Concurrency in drivers
There are three concurrency dangers here: two processes on different CPUs might call `consoleread` at the same time; the hardware might ask a CPU to deliver a console (really UART) interrupt while that CPU is already executing inside `consoleread`; These dangers may result in race conditions or deadlocks.

并发是为了同步

# 5.4 Timer interrupts
Xv6 uses time interrupts to maintain its clock and to enable it to switch among compute-bound processes; the `yield` calls in `usertrap` and `kerneltrap` cause this switching. Timer interrupts come from clock hardware attached to each RISC-V CPU. Xv6 programs this clock hardware to interrupt each CPU periodically.

RISC-V requires that timer interrupts be taken in machine mode, not supervisor mode. RISC-V machine mode executes without paging, and with a separate set of control registers, so it's not practical to run ordinary xv6 kernel code in machine code. As a result, xv6 handles timer interrupts completely separately from the trap mechanism laid out above.

Code executed in machine mode in `start.c`, before `main`, sets up to receive timer interrupts. Part of the job is to program the CLINT hardware (core-local interruptor) to generate an interrupt after a certain delay. Finally, `start` sets `mtvec` to `timervec` and enables timer interrupts.

xv6用计时器中断来在线程间切换，`usertrap`和`kerneltrap`中的`yield`也会导致这种进程切换。RISC-V要求定时器中断的handler放在machine mode而不是supervisor mode中，而machine mode下是没有paging的，同时有另外一套完全独立的控制寄存器，因此不能将计时器中断的handler放在trap机制中执行。

`kernel/start.c`（在`main`之前）运行在machine mode下，`timerinit()`在`start.c`中被调用，用来配置CLINT(_core-local interruptor_)从而能够在一定延迟之后发送一个中断，并设置一个类似于trapframe的scratch area来帮助定时器中断handler将寄存器和CLINT寄存器的地址保存到里面，最终`start`设置`timervec`到`mtvec`(_machine-mode trap handler_)中使得在machine mode下发生中断后跳转到`timervec`然后enable定时器中断。

A timer interrupt can occur at any point when user or kernel code is executing; there's no way for the kernel to disable timer interrupts during critical operations. Thus the timer interrupt handler must do its job in a way guaranteed not to disturb interrupted kernel code. The basic strategy is for the handler to ask the RISC-V to raise a "software interrupt" and immediately return. The RISC-V delivers software interrupts to the kernel with the ordinary trap mechanism, and allows the kernel to disable them. 

由于定时器中断可能在任意时间点发生，包括kernel在执行关键的操作时，无法强制关闭定时器中断，因此定时器中断的发生不能够影响这些被中断的操作。解决这个问题的方法是定时器中断handler让RISC-V CPU产生一个"software interrupt"然后立即返回，software interrupt以正常的trap机制被送到kernel中，可以被kernel禁用。

`timervec`是一组汇编指令，将一些寄存器保存在scratch area中，告知CLINT产生下一次定时器中断的时间，让RISC-V产生一个software interrupt，恢复寄存器并返回到trap.c中，判断`which_dev==2`为定时器中断后调用`yield()`

# 5.5 Real world
Xv6 allows device and timer interrupts while executing in the kernel, as well as when executing user programs. Timer interrupts force a thread switch (a call to `yield`) from the timer interrupt handler, even when executing in the kernel. The ability to time-slice the CPU fairly among kernel threads is useful if kernel threads sometimes spend a lot of time computing, without returning to user space. How, the need for kernel code to be mindful that it might be suspended (due to a timer interrupt) and later resume on a different CPU is the source of some complexity in xv6. The kernel could be made somewhat simpler if device and timer interrupts only occured while executing user code.

The UART driver retrieves data a byte at a time by reading the UART control registers; this pattern is called _programmed I/O_, since software is driving the data movement. Programmed I/O is simple, but too slow to be used at high data rates. Devices that need to move lots of data at high speed typically use _direct memory access(DMA)_. DMA device hardware directly writes incoming data to RAM, and reads outgoing data from RAM. A driver for a DMA device would prepare data in RAM, and then use a single write to a control register to tell the device to process the prepared data.

由于中断非常耗时，因此可以用一些技巧来减少中断。1. 用一个中断来处理很多一段时间内的事件。 2. 彻底禁止设备的中断，让CPU定时去检查这些设备是否有任务需要处理，这种技巧叫做 _polling_. Polling makes sense if the device performs operations very quickly, but it wastes CPU time if the device is mostly idle. Some drivers dynamically switch between polling and interrupts depending on the current device load.

The UART driver copies incoming data first to a buffer in the kernel, and then to user space. This makes sense at low data rates, but such a double copy can significantly reduce performance for devices that generate or consume data very quickly. Some operating systems are able to directly move data between user-space buffers and device hardware, often with DMA.

如第 1 章所述，控制台对应用程序而言就像一个常规文件，应用程序使用 read 和 write 系统调用读取输入并写入输出。应用程序可能想要控制无法通过标准文件系统调用表达的设备方面（例如，在控制台驱动程序中启用/禁用行缓冲）。Unix 操作系统支持针对此类情况的 `ioctl` 系统调用。

计算机的某些用途要求系统必须在限定的时间内做出响应。例如，在安全关键系统中，错过截止时间可能会导致灾难。xv6 不适合硬实时设置。硬实时操作系统往往是与应用程序链接的库，以便进行分析以确定最坏情况的响应时间。xv6 也不适合软实时应用程序，偶尔错过截止时间是可以接受的，因为 xv6 的调度程序过于简单，并且它的内核代码路径会长时间禁用中断。

# 源码分析
我们首先从控制台(Console)的初始化和基本操作出发，然后通过两个相反的过程(==UART从键盘读取数据==、==UART向显示器输出数据==)弄懂整个串口驱动的全流程，顺带提一下==shell命令行解释器与console的关系==，最后对**UART、Console、Shell、键盘鼠标在系统中的联系和关系**做出总结。

# 1 初始化和基本操作
## 1.1 console 的初始化
在启动过程中（kernel/main.c: 14）执行的第一个任务`consoleinit`就是控制台`console`的初始化，实现如下，我们将一点点的展开这个初始化过程，了解Xv6内部对于console设备的管理：
![[Pasted image 20241218162131.png]]
这个函数的作用非常清晰，首先它==初始化了cons变量中的锁==(关于锁的细节我们到下一部分源码才会深入研究)，然后调用==uartinit函数初始化了串口芯片16550==，这部分需要参考大量16550芯片的寄存器细节，最后，consoleread和consolewrite函数被作为console结构的==标准read和write操作，被注册在devsw中==。

下面，我们首先对cons结构体和devsw进行简单的介绍，然后我们转入uartinit函数，仔细研究一下在Xv6中UART芯片是如何被初始化的，最后我们仔细阅读一下==consoleread和consolewrite的函数实现==

### 1.1.1 cons 结构体
cons结构体定义在(kernel/console.c:44)，这是==console的软件抽象==，它的定义和注释如下，我们可以看到==整个console内部其实是有一个输入缓冲区的==，并且有三枚指针来分别实时记录当前缓冲区的==读、写、编辑位置==。**读、写指针的含义都是非常容易理解的，为什么还会有编辑指针呢？**

其实这里所谓==读写指针都是有确认语义在里面的==，我们在输入命令行的时候有时候会写错，这时候我们会按退格键将之前的命令字符删除，那么其实在用户没有确认自己输入的命令完全正确之前，字符都是==有被删除的风险在的==。所以我们在使用键盘键入字符的时候，console会先使用==编辑指针试探性地记录用户输入的字符==，如果用户此时按下了Ctrl+D(EOF组合键)，或者是换行\n时，**这时写指针就会确认之前编辑指针的输入，并更新自己的位置到编辑指针的位置**。
![[Pasted image 20241218162930.png]]

### 1.1.2 UART的初始化
consoleinit中的下一个任务就是调用==userinit函数(kernel/uart.c:52)对串口芯片16550进行初始化==，这个函数中大量调用了宏**WriteReg**(kernel/uart.c:39)，这个宏的定义如下：
![[Pasted image 20241218163101.png]]
而Reg宏的定义如下，它实质上是UART==在MMIO中的基址加上对应寄存器的偏移量==来实现的，在使用时我们只需要将对应的寄存器地址传进去即可，而这些==寄存器的地址也被定义成了宏，分布在kernel/uart.c:22-36==，如下所示：
![[Pasted image 20241218163246.png]]
接下来就可以研究一下uartinit函数了，在此之前==首先贴出所有寄存器地址和功能的大致描述==，以备在后续参考：
![[Pasted image 20241218163327.png]]
以及设置波特率的一张快速查找表，我们可以==通过查表快速地确认向寄存器中写入的值==：
![[Pasted image 20241218163422.png]]

上述就是UART芯片16550的全部初始化流程，需要注意的是除了16550芯片中已经拥有的硬件FIFO，Xv6 内核中还==设置了一个软件缓冲区uart_tx_buf用来暂存UART即将要发送的数据==，与之一并定义的还有==两枚读写指针和一个用于管理进程并发的自旋锁==，代码如下所示。
![[Pasted image 20241218165501.png]]
这个数据结构借助两枚指针和一个数组**实现了一个环形队列**，用来更好地管理这个输出缓冲区，这个循环队列的==示意图和判空判满条件如下==，注意==整个循环队列的长度是32==，以下直接写的明值：
![[Pasted image 20241218165541.png]]

使用 [[ring buffer]] 技术，构建了环形队列。
### 1.1.3 devsw 数组和 console 的读写操作
在consoleinit函数中，在注册读写函数时涉及到了一个==特殊的数组devsw==，devsw的定义如下，可以看到它就是两个读写函数指针的封装，它封装了==可以对一个设备施加的所有的操作==
![[Pasted image 20241218170334.png]]
随后，使用devsw结构体在内核中声明了==一个长度为NDEV的数组==(kernel/file.c:16)，而consoleinit函数中将==console的读写函数注册在了这个数组中==：
![[Pasted image 20241218170438.png]]
这里值得补充的一点是，在UNIX系统中，有==主设备号(major device number)==和==从设备号(minor device number)==的区分，其中主设备号**用来确定设备要使用的驱动程序大类**。是的，在操作系统中==一个驱动程序可以服务多个设备==，这些设备==往往拥有类似的特征==，因此它们的驱动程序构成非常类似，不需要为每个设备都重写一遍非常相似的驱动程序。

而即便再类似的外部设备，==它们的驱动程序也一定会有细微差别，这时候就需要借助从设备号(minor device number)在驱动程序中对特定的设备加以区分和细节处理了==。所以所谓主从设备号就是操作系统内核中用于将特定驱动程序和设备关联起来的两个标识符。

在Xv6中，这个最大主设备号就是NDEV，它的值为10，这表明在Xv6内部==最多只支持注册10种不同设备的驱动程序==(事实上只定义了console一种)，且每一种设备只支持读写两种操作，接下来我们就看看console的具体对读写函数定义。

## 1.2 consolewrite 的实现
![[Pasted image 20241218170711.png]]
在这个函数中调用了==either_copyin和uartputc两个函数==，我们顺势了解一下它们的实现
- 首先是either_copyin函数，**它定义在kernel:proc.c:620的位置**，这个函数相当于==将memmove和copyin函数合二为一了==，根据**传入参数usr_dst的不同来实现将数据复制到内核/用户地址**中，
![[Pasted image 20241218171231.png]]
-  接下来是==uartputc函数(kernel/uart.c:80)==的分析，它的源码和注解如下，我们可以看到在uartputc函数中做的主要就是尝试==将一个字符放入上述的发送缓冲区(环形队列)中==，如果**缓冲区已满就让进程陷入睡眠状态，** **等到缓冲区有空位让出**，而真正的发送动作是在uartstart函数中完成的。

在这个函数中==进一步调用了uartstart函数(kernel/uart.c:133)，我们再去研究一下这个函数==，uartstart函数是**直接驱动UART芯片发送数据的函数**，它会首先检测一些条件，条件一旦满足就==向UART的THR寄存器开始写入要发送的字符==，驱动UART芯片向外发送数据，完整代码和注释如下：

到这里我们大致将consolewrite函数分析得差不多了，它做的事情很简单，首先将==数据从源地址拷贝到一个本地临时变量c中==，然后==将此字符放入输出缓冲区并驱动UART芯片将其发送出去==，而这个由qemu模拟出来的UART 16550芯片的输出通道TX**默认会连接到我们计算机的显示器上**。

## 1.3 consoleread 的实现
==在console的软件抽象cons结构体中也有一个软件缓冲区cons.buf==，**它其实和UART的输出缓冲区一样是一个环形队列**

我们可以看到，consoleread函数做的事情就是==从console缓冲区中读取指定长度的输入(或者是一行输入)==，并一个字节一个字节地拷贝到用户/内核空间的指定区域去。
![[Pasted image 20241218182748.png]]
至此，我们算是较为完整地认识了console设备的读写操作实现，以及UART的初始化过程。但是现在只是完成了==驱动的注册和硬件的初始化==，我们上面说UART芯片16550需要通过中断来通知CPU进而完成对设备的读写，因为是外部设备，所以中断信号==一定会通过PLIC来路由到CPU核心==。接下来梳理一下这部分工作。

## 1.4 PLIC 的初始化
在Xv6内核中，与外部中断相关的初始化函数有==plicinit(kernel/plic.c:11)与plicinithart(kernel/plic.c:19)==，我们结合SiFive开发板的手册，研究一下它们的行为。首先是plicinit函数，它的实现如下：
![[Pasted image 20241218183015.png]]
**优先级写入PLIC基址+4×ID号这个地址**：
![[Pasted image 20241218183203.png]]
PLIC对优先级的规定是这样的：==外部中断设备有0-7一共8个优先级，数字越大优先级越高，其中0表示“无需响应的中断”==，所以优先级设置为0表示屏蔽此中断。我们看到上述代码将虚拟磁盘中断和UART中断都设置为1，也就是最低优先级。那么，**两个中断优先级一样，同时发生时应该优先响应谁呢**，PLIC规定：==优先响应中断ID号较小的那一个==，而上述的两个中断ID定义分别如下：
![[Pasted image 20241218183330.png]]
这里需要额外说明，这里==中断ID号的设定和SiFive开发板上的ID分配方案是不一致的==，但并非是随意定义的，这里使用的是qemu中**QEMU RISC-V VirtIO machine interface**中定义好的==虚拟IO的ID号==是一致的，它们在==后续的claim/complete机制中也会再次用到==，所以不能掉以轻心，它们在qemu中的定义如下：
![[Pasted image 20241218183451 1.png]]
第二个函数是plicinithart，这个函数==对多个核心的中断使能和中断阈值做了初始化==，代码如下：
![[Pasted image 20241218183642.png]]
对于PLIC中的中断使能寄存器，只需要将中断ID对应的位设置为1，即可使能中断：
![[Pasted image 20241218183747.png]]
而对于中断阈值interrupt_threshold，**PLIC不会响应小于等于中断阈值的优先级的中断，为了让所有中断都被响应，这里将阈值设置为0，所以所有中断都会被响应**，这就是初始化PLIC的全部流程。

# 亟待解决的问题
在memorylayout.h文件中定义的==PLIC寄存器地址换算关系和手册中的寄存器地址是对应不上的==：
![[Pasted image 20241218183950.png]]
举个简单的例子，==PLIC_SCLAIM宏如果代入hart = 1，那么计算出来的结果是0x0c203004，这个地址对应到的不是hart1的CLAIM寄存器，而是M态下hart2的CLAIM寄存器==。这几个寄存器地址定义都存在类似的问题，我不知道问题出在哪里，这可能和qemu的源码有关，但是我目前还没有找到依据，望知情人告知一下，多谢！![[Pasted image 20241218184014.png]]

其实地址是可以对应上的，我之前也想了很久，后面发现只有在M模式的时候才有hart 0，在S模式的hart的编号是从hart 1开始的，没有hart 0，这样算出来的地址就是对的

==研究一下一个字符是怎么一步步被显示到我们的屏幕上的==，经过了哪些设备和步骤。和上一篇博客一样，这篇博客采用迭代深入的方法，我们会紧跟处理流程，并在这个过程中==研究和分析每一个被调用且与主线连接紧密的子函数==，直到达到被调用序列的尽头，**最终我们会给出整个过程的总览，作为总结和回顾**。

# 2. 字符是如何被显示到屏幕上的
## 2.1 printf
xv6 的内核态和用户态各有一个名为 printf.c 的文件。从用户态的 printf 出发，看看它到底做了什么，字符是如何被显示在我们的屏幕上的。

Xv6启动时会设置第一个进程init，这个进程会打开==标准输入、标准输出、标准错误这三个文件标识符(分别对应0，1，2)==，并且**这三个文件描述符其实都指向console**。代码如下(user/init.c:14)：
![[Pasted image 20241218194314.png]]

printf的函数实现如下(user/printf.c:106)，printf会调用vprintf完成一系列复杂的处理和判断，并完成字符的输出，注意==默认情况下vprintf向标准输出(fd=1)输出字符==。
![[Pasted image 20241218194433.png]]
注意到vprintf最终**所有的打印操作都会落到一个叫做putc的函数上**，我们直接去看看putc的函数实现(user/printf.c:9)，可以发现==这个函数就是write系统调用的简单封装==。
![[Pasted image 20241218194500.png]]
## 2.2 陷入内核——write 系统调用
从上面可知，用户态下所有==字符的显示任务最终都进入到了内核下的sys_write系统调用实现==，它首先==调用了一些arg\*函数来对用户态传入的参数进行解析==，随后就==调用filewrite函数进行下一步写操作==。

![[Pasted image 20241218194800.png]]

## 2.3 转发到consolewrite——filewrite 函数
filewrite函数(kernel/file.c:132)要做的事情就是==根据传入的文件句柄的种类，去分门别类地对写操作进行处理和转发==，它的代码和相应注释如下。因为我们传入的文件是console，而它在之前==使用mknod注册成了设备==，所以这里自然会落到对应的分支下。接下来，filewrite函数**会调用console驱动中已经注册好的consolewrite函数，完成写操作。**
![[Pasted image 20241218195620.png]]

## 2.4 consolewrite 驱动UART完成字符输出
接下来consolewrite函数会==首先调用either_copyin函数将用户从printf传入的字符拷贝到内核态(字符只有一个)==，然后==调用uartputc函数将此字符放入UART输出缓冲区中==，uartputc会接着调用==uartstart函数真正地将缓冲区里的字符从UART的TSR寄存器中发送出去==，而这个UART输出会经由qemu的模拟直接连接到屏幕上。

# 3. 字符是如何从键盘中被读取的
## 3.1 按下键盘——触发中断
UART是全双工的串口通信协议，我们在上面说**它的TX(输出端)连接到了我们的显示器，那么它的RX(接收端)其实就连接到了我们的键盘**。我们这里不对它们连接具体细节进行解释，因为这是qemu源码中做的事情，简单来说==它模拟了一个UART 16550芯片，并将它的输出与我们的显示器相连，输入与我们的键盘相连==

当我们按下一个键时，经过键盘的扫描等操作，会确认我们按下的是哪个字符，并==将这个字符的ASCII编码通过连线发送给串口芯片16550==，因为我们之前在初始化16550的时候设置的trigger level是1，所以16550芯片接收到==每个字符都会触发一次设备中断==(即键盘导致的串口中断)。

[[RISC-V 架构中的异常与中断详解]] 一文中详细解释了**外部中断经由PLIC路由的过程，并解释了RISC-V核心的claim/complete机制**。当一个核心收到了外部中断信号时，它就会从用户态==经由trampoline.S进入内核态==，并在usertrap函数中==根据scause寄存器的数值对陷阱进行转发==，这里不再详述。外部设备中断在usertrap中将会被devintr函数接管，它的代码如下：

可以看到，在devintr函数中我们会调用plic_claim函数使得核心响应这个中断，如果竞争成功这个函数就==会给核心返回对应外部设备的中断ID号==。之后==此函数对比ID号从而将不同设备的中断ID转发到不同的处理函数中去==，对于UART中断，将会被uartintr函数接管。

## 3.2 uartintr 接管——字符放入 console 缓冲区并回显

### 3.2.1 uartintr 函数总览
所以，经过devintr函数的处理，我们敲击键盘引起的UART中断==最终被uartintr函数接管了==(**注意：uartintr实质上会接受两种类型的中断：1.输入通道RX为满则中断(键盘输入中断) 2.输出通道TX为空则中断，这里我们只讨论前者**)，这个函数的==代码和注释==如下(kernel/uart.c:179)，在这个函数中调用了uartgetc、consoleintr以及uartstart函数。其中uartstart函数我们之前已经了解过，是==专门调用串口来异步地发送缓冲区中字符的==，接下来我们详细分析一下==剩下的两个函数uartgetc和consoleintr==。

### 3.2.2 uartgetc 函数
首先是uartgetc函数(kernel/uart.c:166)，这个函数的代码和注释如下。这个函数是面向16550芯片的简单操作，==它首先判断是否有输入等待读取，如果有就返回输入字符，反之则返回-1==，逻辑还是非常简单的。
![[Pasted image 20241218200918.png]]

### 3.2.3 consoleintr 函数
然后是consoleintr(kernel/console.c:135)函数，它的代码实现颇长，因为这里面==涉及到不同情况下组合键的处理，以及写指针和编辑指针的改动==，逻辑稍显零碎，细节都已经注释在下方。简单来说**此函数尝试将字符放入console的缓冲区cons.buf中，并适时地对一行的结束进行确认，同时它会调用consputc函数进行立即回显。**

### 3.2.4 consputc 函数
在consoleintr函数中也==大量调用了consputc函数(kernel/console.c:33)==，这个函数的功能就是直接将一个字符发送给UART显示，这个字符==不会经过缓冲区，而是直接送到UART去发送==，这是依靠uartputc_syn函数来实现的，等下我们会去对比这个函数和之前说过的uartputc函数。另外，这里还需要对**回退符**做一些==简单的判断和处理操作==，详见下面的注释。
![[Pasted image 20241218214318.png]]
最后说一嘴，注意区分consputc和consolewrite这两个函数，它们本质上都是向屏幕发送显示字符，但是也有很多不同，一定要注意区分
- **consputc是运行在内核态下的，consolewrite是从用户态读取字符并显示的。**
- **consputc输出字符是同步的，效率高，consolewrite输出字符是异步的，效率相对低。**
- **consputc调用uartputc_sync完成同步字符发送，consolewrite则调用uartputc完成异步字符发送**
正是因为consputc的同步发送，==使得在consoleintr函数中，我们输入的字符可以第一时间回显给我们。==

### 3.2.5 uartputc_sync vs uartputc
在上面consputc函数中大量调用了uartputc_sync函数，uartputc函数负责将一个字符放入发送缓冲区中，并尝试使用uartstart函数发送它。那么这里为什么又会有一个uartputc_sync呢，它们之间的区别又在哪？

其实它们的区别主要在于动作上的==同步和异步==，uartputc_sync顾名思义是一个同步输出字符的函数，它**不需要经过缓冲区，并且在发送时如果UART串口TX不是空闲的，它会阻塞在此直至发送成功**，所以我们说这是一个同步的发送动作。而uartputc函数则是一个异步的动作，**这个动作首先要经过发送缓冲区，缓冲区如果已满则要在上面睡眠等待空位出现，** 真正调用uartstart函数发送时，这个函数还要检查UART发送端是否空闲，如果不空闲则立即退出，等到下一次uartstart函数被调用时才会再次尝试发送。

因为uartputc_sync函数的同步性，使得它==非常适合用来打印一些实时性要求很高的信息==，例如==回显、内核提示的信息==等。

uartputc_sync函数实现如下：
![[Pasted image 20241218220107.png]]

# 4. 输入的数据如何被消费
## 4.1 getcmd
通过键盘我们==将字符通过上述过程输入到了console的缓冲区cons.buf里==，同时也==通过回显过程实时的将我们输入的字符显示到了屏幕上==。但是console里的字符必须得有事物来consume，否则缓冲区一定会爆掉，而这就是命令行解释器shell做的事情了。

命令行解释器(shell)位于user/sh.c文件中，显然，它在Xv6中是一个==用户态的程序==。shell 负责从内核中读取用户输入的一整行命令，然后解析它并fork 在exec

所谓==“从内核读取用户数输入的一整行命令”==，其实就是==从控制台console的缓冲区中将我们之前用键盘输入的一整行命令读出==。这个过程位于getcmd函数(user/sh.c:133)中，代码如下：
![[Pasted image 20241218220704.png]]
上述函数中调用了gets来获取字符，这个函数的实现如下(user/ulib.c:55)，这个函数中==使用了read系统调用来从标准输入(fd=0)中一个字符一个字符地进行读取==。我们在之前已经说过===在第一个线程启动时，标准输入、标准输出和标准错误都被定向到了console==，所以这里的**read操作其实也对应到了从console中读取一个字符。**
![[Pasted image 20241218221008.png]]

## 4.2 陷入内核——read系统调用
接上，gets调用了read系统调用，这个==系统调用经过usertrap的转发最终会陷入到sys_read这个系统内核的实现中==，它的代码如下(kernel/sysfile.c:69)，可以看到这个函数首先解析了用户态传入的参数，并==最终调用了fileread函数来从console(设备)文件中读取字符==。
![[Pasted image 20241218221240.png]]

## 4.3 转发到consoleread——fileread 函数
fileread函数(kernel/file.c:106)和我们之前介绍过的filewrite函数非常相似，也是==相当于一个转发中转站==，对于像console这样的设备文件，==它会直接调用已经注册好的驱动：consoleread函数来最终完成字符的读取==：
![[Pasted image 20241218221355.png]]
## 4.4 consoleread 从设备缓冲区中读取字符
故事到此为止，就回到了我们熟悉的地方，它==主要做的事情就是对输入边界进行鉴别(例如Ctrl+D，'\n’等)==，并==使用either_copyout函数将字符拷贝回用户态指定的地址中去==。这样，**console中的数据就成功被读出到了用户态，con.buf获得了释放，shell也可以解析并执行我们输入的命令了，故事到此也就彻底完整了。**

# 5. 一图

![[Pasted image 20241218202244.png]]
**console的定位就是一个软件抽象出来的设备体，它专门用来缓存用户输入的字符，并对其中输入的特殊字符和组合键进行预处理，使得串口可以正常打印，shell可以正常解析。**
