AbstractMachine 是裸机上的 C 语言运行环境，提供 5 组 (15 个) 主要 API，可以实现各类系统软件 (如操作系统)：

- (TRM) `putch`/`halt` - 最基础的计算、显示和停机
- (IOE) `ioe_read/ioe_write` - I/O 设备管理
- (CTE) `ienabled`/`iset`/`yield`/`kcontext` - 中断和异常
- (VME) `protect`/`unprotect`/`map`/`ucontext` - 虚存管理
- (MPE) `cpu_count`/`cpu_current`/`atomic_xchg` - 多处理器

# 1.从一个例子说起
怎样让C程序运行起来：
- 为了使程序能运行，当然需要经过编译链接的过程。就假设最简单的情况：生成静态链接的 ELF 格式的二进制文件好了。
- 二进制文件假设代码、数据存在于地址空间的指定位置。那么是谁来完成这件事？
- `main` 在二进制文件中的地址是不固定的。是谁调用的 `main()`？
- 我们需要自己动手实现各种库函数，那 `printf` (输出到屏幕), `malloc` (动态分配内存)又是如何实现的？

为了理解 C 程序是如何运行在裸机 (bare-metal) 上的，我们首先理解 C 程序是如何从源代码 (文本文件) 最终在操作系统上运行起来的。为此，我们讲解一个小例子 (`say.c` 和 `main.c` 组成的小项目) 在操作系统上以及 AbstractMachine 上的编译、链接和加载运行的过程。

![[Pasted image 20241030123354.png]]
![[Pasted image 20241030123400.png]]

以下完整的流程是操作系统上 (hosted) 和 bare-metal 上共同的：
![[Pasted image 20241030123439.png]]

# 2. OS上的C程序
## 2.1 编译
我们使用gcc 把源代码编译成可重定位的二进制文件：
![[Pasted image 20241030123536.png]]
“relocatable” 的含义是虽然生成了指令序列，但暂时还不确定它们在二进制文件中的位置。我们可以查看生成的指令序列：
![[Pasted image 20241030123649.png]]
可以看到 relocatable 的代码从 0 开始编址；因为 `main` 并不知道 `say` 的代码在何处，所以虽然生成了 opcode 为 `0xe8` 的 `call` 指令 (对应 `say("...")` 的函数调用)，但没有生成跳转的偏移量 (`say.c` 中向 `putchar` 的调用也生成同样的 `call` 指令)：

![[Pasted image 20241030123822.png]]

类似的，`say` 的第一个参数 (通过 `%rdi` 寄存器传递) 是通过如下 `lea` 指令获取的，它的位置同样暂时没有确定：
![[Pasted image 20241030123912.png]]

## 2.2 链接
通常，我们使用 gcc 帮助我们完成链接：
![[Pasted image 20241030123934.png]]
如果直接使用 `ld` 命令链接，则会报错：
![[Pasted image 20241030124000.png]]
首先，我们的程序没有入口 (`_start`)，其次，我们链接的对象中没有 `putchar` 函数。我们可以给 gcc 传递额外的参数，查看 `ld` 的选项：
![[Pasted image 20241030124054.png]]
你会发现链接的过程比想象中复杂得多。用以下简化了的命令可以得到可运行的 hello 程序：
![[Pasted image 20241030124103.png]]
为什么这么复杂？还有好多没见过的文件！这是因为我们的二进制文件要运行在操作系统上，就必须遵循操作系统的规则，调用操作系统提供的 API 完成加载。加载器也是代码的一部分，当然应该被链接进来。链接文件的具体解释：

- `ld-linux-x86-64.so` 负责动态链接库的加载，没有它就无法加载动态链接库 (libc)。
- `crt*.o` 是 C Runtime 的缩写，即 C 程序运行所必须的一些环境，例如程序的入口函数 `_start` (二进制文件并不是从 `main` 开始执行的！)、`atexit` 注册回调函数的执行等。
- `-lc` 表示链接 glibc。

链接后得到一个 ELF 格式的可执行文件：
## 2.3 加载
完成链接后，在操作系统的终端程序中使用 `./a.out` 运行我们的程序，流程大致如下 (学完《操作系统》课后，你将会对这个流程有更深入的认识)：

- Shell 接收到命令后，在操作系统中使用 `fork()` 创建一个新的进程。
- 在子进程中使用 `execve()` 加载 `a.out`。操作系统内核中的加载器识别出 `a.out` 是一个动态链接文件，做出必要的内存映射，从 `ld-linux-x86-64.so` 的代码开始执行，把动态链接库映射到进程的地址空间中，然后跳转到 `a.out` 的 `_start` 执行，初始化 C 语言运行环境，最终开始执行 `main`。
- 程序运行过程中，如需进行输入/输出等操作 (如 libc 中的 `putchar`)，则会使用特殊的指令 (例如 x86 系统上的 `int` 或`syscall`) 发出系统调用请求操作系统执行。典型的例子是 `printf` 会调用 `write` 系统调用，向编号为 `1` 的文件描述符写入数据。

怎么在实际的系统上观察上述的行为？我们有调试器！gdb 为我们提供了 `starti` 指令，可以在程序执行第一条指令时就停下：
![[Pasted image 20241030124432.png]]
操作系统的加载器完成了 `ld-linux-x86-64.so.2` 的加载，并给它传递了相应的参数。我们可以查看此时的进程信息 (这些内存都是操作系统加载的)：

![[Pasted image 20241030124530.png]]

如果我们在 `_start` 设置断点，会发现此时已经加载完成：
![[Pasted image 20241030124803.png]]
地址空间中已有 `a.out`, libc, 堆区、栈区等我们熟悉的东西，libc 的 `_start` 完成初始化后会调用 `main()`。这真是一段漫长的旅途！

# 3. Bare-Metal 上的 C 程序
对于 AbstractMachine 上的程序，我们需要一个 Makefile，就能把 hello 程序编译到 bare-metal 执行：
![[Pasted image 20241030124900.png]]
在终端中执行 `make -nB ARCH=x86_64-qemu` 可以查看完整的编译、链接到 x86-64 的过程 (不实际进行编译)。

## 3.1 编译
编译器的功能是把 `.c` 文件翻译成可重定位的二进制目标文件 (`.o`)。这一步对于有无操作系统来说差别并不大，最主要的区别是在 bare-metal 是 “freestanding” 的运行环境，没有办法调用依赖于操作系统的库函数——你依然可以声明它们 (例如 `printf`, `malloc` 等)，但最终它们的代码无法链接。

编译器 (gcc) 提供了选项帮我们生成不依赖操作系统的目标文件，例如对 `-ffreestanding` (`-fno-hosted`) 选项的文档：

![[Pasted image 20241030125157.png]]

## 3.2 链接
可能出乎意料，因为没有操作系统，bare-metal 程序的链接比操作系统上的情况还简单一些！我们使用的链接命令是：
![[Pasted image 20241030125248.png]]
只链接了 `main.o`, `say.o` 和必要的库函数 (AbstractMachine 和 klib；在这个例子中，我们甚至可以不链接 klib 也能正常运行)。使用的链接选项：

- `-melf_x86_64`：指定链接为 x86_64 ELF 格式；
- `-N`：标记 `.text` 和 `.data` 都可写，这样它们可以一起加载 (而不需要对齐到页面边界)，减少可执行文件的大小；
- `-Ttext-segment=0x00100000`：指定二进制文件应加载到地址 `0x00100000`。

使用 `readelf` 命令查看 `hello-x86_64-qemu.o` 文件的信息：
![[Pasted image 20241030125608.png]]
其中的 program headers 描述了需要加载的部分：加载这个文件的加载器需要把文件中从 `0xb0` (Offset) 开始的 `0x29ac` 字节 (`FileSiz`) 加载到内存的 `0x1000b0` 虚拟/物理地址 (VirtAddr/PhysAddr)，内存中的大小 `0x23f98` 字节 (`MemSiz`，超过 `FileSiz` 的内存清零)，标志为 RWE (可读、可写、可执行)。
![[Pasted image 20241030132654.png]]
实际上，这个程序可以直接在操作系统上被运行！如果你试着用 gdb 调试它，会发现程序从 `_start` (`0x100100`) 开始执行，但在执行了若干条指令后，在 `movabs %eax,0xb900001000` 时发生了 Segmentation Fault。
![[Pasted image 20241030132743.png]]
只不过因为运行环境不同，执行到系统指令时，属于非法操作 crash 了。我们需要在 bare-metal 上加载它。所以我们会创建 `hello-x86_64-qemu` 的镜像文件：
![[Pasted image 20241030132815.png]]
镜像文件是由 512 字节的 “MBR”、1024 字节的空白 (用于存放 `main` 函数的参数) 和 `hello-x86_64-qemu.o` 组成的。用 `file` 类型可以识别出它：
![[Pasted image 20241030132834.png]]

## 3.3 加载
我们在 QEMU 全系统模拟器中运行完整的镜像 `hello-x86_64-qemu` (包含 `hello-x86_64-qemu.o`)。如果用一些特别的选项，就能近距离观察模拟器的执行：
![[Pasted image 20241030132913.png]]
其中：

- `-S` 在模拟器初始化完成 (CPU Reset) 后暂停
- `-s` 启动 gdb 调试服务器，可以使用 gdb 调试模拟器中的程序
- `-serial none` 忽略串口输入/输出
- `-nographics` 不启动图形界面

我们可以在终端里启动一个 monitor (Online Judge 就处于这个模式)。在这里，我们就可以像代码课中讲解的那样，直接调试整个 QEMU 虚拟机了！

### 3.3.1 CPU Reset
就像 NEMU 一样调试虚拟机，我们使用 `info registers` 查看 CPU Reset 后的寄存器状态。
![[Pasted image 20241030133211.png]]

CPU Reset 后的状态涉及很多琐碎的硬件细节，这也是大家感到为 bare-metal 编程很神秘的原因。不过简单来讲，我们关心的状态只有两个：

- `%cr0 = 0x60000010`，最低位 `PE`-bit 为 0，运行在 16-bit 模式 (现在 CPU 的行为就像 8086)
- `%cs = 0xf000`, `%ip = 0xfff0`，相当于 PC 指针位于 `0xffff0`

CPU Reset 后，我们的计算机系统就是一个状态机，按照 “每次执行一条指令” 的方式工作。

### 3.3.2 Firmware: 加载 Master Boot Record
位于 `0xffff0` 的代码以内存映射的方式映射到只读的存储器 (固件，firmware，也称为 BIOS) 中。固件代码会进行一定的计算机状态检查 (比如著名的 “Keyboard not found, press any key to continue...”)。如果我们在 gdb 中使用 `target remote localhost:1234` 连接到 qemu (默认端口为 1234)，就可以开始单步调试固件代码。

在比较简单的 Legacy BIOS Firmware (Legacy Boot) 模式，固件会依次扫描系统中的存储设备 (磁盘、优盘等，Boot Order 通常可以设置)，然后将第一个可启动磁盘的前 512 字节 (主引导扇区, Master Boot Record, MBR) 加载到物理内存的 `0x7c00` 地址。以下是 AMI 公司在 1990s 的 BIOS Firmware 界面：
![[Pasted image 20241030133359.png]]
今天的 Firmware 有了 [UEFI 标准](https://wiki.osdev.org/UEFI)，能更好地提供硬件的抽象、支持固件应用程序——今天的 “BIOS” 甚至有炫酷的图形界面 (不要小看图形界面，能显示图形意味着你得有总线、键盘、鼠标的驱动程序……)，固件复杂程度可比小型操作系统；UEFI 加载器也不再仅仅加载是 512 字节的 MBR，而是能加载任意 GPT 分区表上的 FAT 分区中存储的应用。今天的计算机默认都通过 UEFI 引导。

从容易理解的角度，我们编写操作系统时，依然从 Legacy Boot 启动，你只需要知道，在 x86 系统上，AbstractMachine 和 Firmware 的约定是磁盘镜像的前 512 字节将首先会被加载到物理内存中执行。

我们可以通过 gdb 连接到已经启动 (但暂停) 的 qemu 模拟器：

![[Pasted image 20241030133441.png]]

使用 `layout asm` 进行指令级的调试 (调试 16-bit code 时 disassembler 会遇到一些小麻烦，但调试的基本功能是没问题的；借助其他工具如 nasm 可以正确查看代码)。

## 3.3  Boot Loader: 解析并加载 ELF 文件
在 gdb 中看到的 `0x7c00` 地址的指令序列是我们的 boot loader 代码，存储在磁盘的前 512 字节。x86-64 的 AbstractMachine 在这 512 字节内完成 ELF 文件的加载。这部分代码位于 `am/src/x86/qemu/boot`，由一部分 16-bit 汇编 (`start.S`)，主要部分如下：
![[Pasted image 20241030133535.png]]

这段代码 (不需要理解) 就是做一些必要的处理器设置，切换到 32-bit 模式，设置初始的栈顶指针 (`0xa000`)，然后跳转到 32-bit C 代码 `load_kernel` 执行。`load_kernel` 位于 `main.c` (有简化)：
![[Pasted image 20241030133557.png]]
我们现在只需要知道我们编写的这段代码会被编译链接，然后被放置在磁盘的 MBR，从而被固件自动加载执行。在 `load_elf64` 中，我们根据 ELF 格式的规定将文件内容载入内存，然后跳转到 ELF 文件的入口，就算完成了 “hello 的加载”。

## 3.4 `_start`: 初始化 64-bit Long Mode
此时我们的 hello (以及未来的 “操作系统”) 代码已经开始执行了，不过此时还不能立即执行 `main` 函数——我们还处于 32-bit 模式。`am/src/x86/qemu/start64.S` 的代码会完成最后的设置，比较重要的是启动分页 (正确设置四级页表)、切换到 x86-64 Long Mode，然后进入以下 64-bit 代码执行：
![[Pasted image 20241030133727.png]]
之后，我们就进入了 C 代码的世界；但此时并未完成所有的初始化，在 `trm.c` 中的代码还要完成一系列硬件/运行环境的初始化：
![[Pasted image 20241030133849.png]]
最后，完成堆栈切换，然后调用 `call_main` 函数：
![[Pasted image 20241030133901.png]]


### 3.3.4 Bare-Metal 上的程序
C 程序编译后得到的指令序列终于在 bare-metal 上开始运行。

和操作系统上的 C 程序不同，AbstractMachine 上的程序对计算机硬件系统有完整的控制，甚至可以链接汇编代码或使用内联汇编 (inline assembly) 访问系统指令。我们可以在 bare-metal 上编写各种 non-trival 的程序，例如任何小游戏 (参考 OS Lab 0)，其中使用 I/O 指令和 memory-mapped I/O 直接和物理设备交互，读取系统时间和按键、更新屏幕。