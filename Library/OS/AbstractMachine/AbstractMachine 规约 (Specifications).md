# 1. 基本概念
AbstractMachine 上运行的程序称为 “kernel” (内核)。这个名字通常表示直接运行在硬件上、对硬件有直接控制的代码。不仅是操作系统内核，像在 GPU 上运行的二进制文件也称为 “kernel”。

AbstractMachine 上的 kernel 被编译成一个目标体系结构/平台上可执行的文件，并可以直接在环境 (如计算机硬件、虚拟机等) 上执行，例如：

- x86 (x86 32-bit), qemu (模拟器)
- x86_64 (x86 64-bit), qemu (模拟器)
- riscv64 (RISC-V 64-bit), nemu (模拟器)
- native (作为本地进程运行)

在《操作系统》实验中，请大家以 x86_64-qemu 为基准平台，其他实现作为参考。

# 2.TRM (Turing Machine): C 语言运行环境
Kernel 是一个 C 语言书写的程序。C 程序执行的状态包括：

- 栈，概念上，包含函数调用栈帧的链表，每一帧都有独立的 PC (程序计数器)、参数和局部变量。
- 只读代码。
- 只读数据，写入只读数据 (如常量字符串) 将导致 undefined behavior。
- 读写数据，包括所有静态变量和一个可用的堆区。

以上所有的状态都存储在同一个平坦的地址空间中，并可以使用指针访问 (栈、代码、数据、堆区互不相交且连续存储)；内存的非法访问是 undefined behavior。我们不妨假定执行一条语句会从状态 $s$ 迁移到 $s^′$，记作 s→s′。

Kernel 执行的具体约定：

- Kernel 被 TRM 加载，从 `main` 开始运行；
- `main` 运行时带有一个参数 `const char *args`，允许程序启动时传递一个字符串参数：
    - 在运行 (`make run`) 时设置 `mainargs` 环境变量，即可向 kernel 传递参数
    - 参数字符串的长度 (包含末尾的 `\0`) 不超过 1024 字节
- Kernel 在运行时遵循状态机的执行，其中：
    - 可使用的堆栈大小不少于 4 KiB
    - 堆区的大小和位置在运行时确定，通过 `heap` 变量获取

Kernel 在运行时允许调用 AbstractMachine API，此时的状态迁移 s→s′ 由 AbstractMachine 定义。

## 2.1. `Area` 结构体
![[Pasted image 20241030182156.png]]
表示左闭右开区间 `[start, end)` 的一段内存。我们假设地址空间的最后一个字节 (例如 32-bit 平台下地址 `0xffffffff`) 永远不会被包含在某个 `Area` 中，因此 `end` 不会溢出。内存区间
![[Pasted image 20241030182208.png]]
的最后一个字节的地址是 `0xfffffffe`。`klib-macros.h` 提供了一些区间的构造/判断的宏：
![[Pasted image 20241030182240.png]]

## 2.2. `heap`: 物理内存堆区
![[Pasted image 20241030182346.png]]
标记一段连续的、代码可以使用的物理内存 `[heap.start, heap.end)`，这段内存 (堆区) kernel 可以任意读写。

**多处理器：安全**。`heap` 在 `main` 被调用前初始化，之后则不会改变 (任意处理器都可以自由读取它)。修改 `heap` 导致 undefined behavior；Kernel 没有任何理由需要修改它。

## 2.3 `putch`: 打印字符
![[Pasted image 20241030182436.png]]
向默认的调试终端打印 ASCII 码为 `ch` 的字符：

- 对 x86 (x86-64) QEMU 平台，向 COM1 输出，通过 QEMU 的 `-serial` 可以选择输出位置。
- 对 native 平台，输出到 stdout。
- 对真实的硬件平台，向调试串口输出。
**多处理器：安全**。putch 是最基本的调试功能，因此 AbstractMachine 的实现负责保证它在多处理器上的安全性。

## 2.4. `halt`: 终止 AbstractMachine
![[Pasted image 20241030182527.png]]
立即终止整个 AbstractMachine 的运行，并返回数字编号 `code` (0-255)：

- 对 QEMU 平台，虚拟机将直接终止，终止前会向调试终端打印信息 (例如返回代码)。
- 对 native 平台，代码将退出，进程的返回代码为 `code`。
- 对真实的硬件平台，根据硬件的支持关闭或进入死循环状态。

**多处理器：安全**。在任意处理器上执行 `halt` 都会终止整个 Kernel 的执行。

# 3. MPE (Multi-Processing Extension) 共享内存多处理器
进入共享内存多处理器模式。假设系统中有 nn 个处理器，在完成 MPE 初始化后，系统中就有多个并行执行、共享内存且拥有独立堆栈的状态机。

**数据竞争**：两个处理器同时 (在状态机模型中意味着先后) 访问一个共享内存位置，并且至少有一个是写是数据竞争。数据竞争 (data race) 在 AbstractMachine 是 undefined behavior，应当严格避免。




---
![[Pasted image 20241030195015.png]]

这段代码是一个用于处理键盘输入的函数 `print_key`，主要功能是读取键盘事件并打印按键信息。以下是对代码的详细解释：

### 代码解析：

1. **结构体定义**：
   ```c
   _DEV_INPUT_KBD_t event = { .keycode = _KEY_NONE };
   ```
   这里定义了一个键盘事件结构体 `event`，并将 `keycode` 初始化为 `_KEY_NONE`，表示没有按键被按下。

2. **宏定义**：
   ```c
   #define KEYNAME(key) \
     [_KEY_##key] = #key,
   ```
   这个宏用于生成一个键名字符串。`KEYNAME(key)` 将 `key` 转换为字符串形式，并将其映射到对应的 `_KEY_##key`（例如 `_KEY_A`）索引。`##` 操作符用于连接宏参数与其他标识符。

3. **键名数组**：
   ```c
   static const char *key_names[] = {
     _KEYS(KEYNAME)
   };
   ```
   `key_names` 数组使用宏 `_KEYS` 来填充，假设 `_KEYS` 是一个包含所有键的宏，依次将每个键的名称映射到其对应的索引。这个数组可以通过索引快速访问按键的名称。

4. **读取输入**：
   ```c
   _io_read(_DEV_INPUT, _DEVREG_INPUT_KBD, &event, sizeof(event));
   ```
   这行代码通过 `_io_read` 函数从输入设备读取键盘事件，将数据存储到 `event` 结构体中。这里 `_DEV_INPUT` 和 `_DEVREG_INPUT_KBD` 应该是与键盘相关的设备和寄存器标识符。

5. **检查键盘事件**：
   ```c
   if (event.keycode != _KEY_NONE && event.keydown) {
   ```
   这里检查读取的键码是否不是 `_KEY_NONE`（表示有键被按下），同时 `event.keydown` 为真，表示是按下事件。

6. **打印按键信息**：
   ```c
   puts("Key pressed: ");
   puts(key_names[event.keycode]);
   printf(" %d\n", event.keycode);
   ```
   - 首先打印“Key pressed: ”。
   - 然后通过 `event.keycode` 访问 `key_names` 数组，获取并打印对应的按键名称。
   - 最后，打印按键的数值（键码）。

### 总结：
`print_key` 函数的主要功能是读取键盘输入并打印按下的键及其对应的键码，适用于需要处理键盘事件的嵌入式系统或设备驱动开发。


---

这段代码定义了一个头文件 `amdev.h`，用于描述虚拟设备和相关的设备寄存器、键盘按键代码等。以下是代码的主要部分及其解释：

### 1. 宏定义

- `#ifndef __AMDEV_H__` 和 `#define __AMDEV_H__`：防止头文件被重复包含。
- `#ifdef __cplusplus` 和 `extern "C"`：确保C++编译器以C的方式链接函数，避免名称改编。

### 2. 设备常量定义

这部分定义了多个虚拟设备的标识符，例如：
- `_DEV_PERFCNT`：性能计数器
- `_DEV_INPUT`：输入设备
- `_DEV_TIMER`：定时器
- `_DEV_VIDEO`：视频控制器
- `_DEV_SERIAL`：串行设备
- `_DEV_STORAGE`：持久存储设备
- `_DEV_PCICONF`：PCI配置空间

### 3. 宏 `_AM_DEVREG`

这个宏用于定义设备的寄存器。它会创建一个结构体，包含寄存器的字段，使用 `__attribute__((packed))` 确保结构体不进行填充。

示例：
```c
_AM_DEVREG(INPUT, KBD, 1, int keydown, keycode);
```
这行代码定义了一个输入设备（`INPUT`）的键盘寄存器（`KBD`），包含 `keydown` 和 `keycode` 字段。

### 4. 设备寄存器定义

这部分定义了不同设备的寄存器及其数据结构。例如：
- `INPUT` 的键盘寄存器包含键按下状态和键码。
- `TIMER` 的 `UPTIME` 寄存器包含高低32位的时间信息。

### 5. PCI配置寄存器宏

- `_DEVREG_PCICONF(bus, slot, func, offset)`：用于生成PCI配置寄存器的地址，包含总线、插槽、功能和偏移量。

### 6. 键码定义

使用宏 `_KEYS` 定义了一系列键盘按键，包括功能键、字母、数字和控制键。然后通过宏 `_KEY_NAME` 创建一个枚举类型，列出所有的键码。

### 总结

这个头文件提供了一种结构化的方法来定义和使用虚拟设备及其寄存器，适用于底层系统编程或操作系统开发。这些定义可以帮助开发人员管理设备的交互，简化设备驱动程序的编写。