1在一个状态机内，实现多个状态机（类似线程切换）的效果。

如何用一个线程模拟“两个线程”执行，在一个线程里通过 malloc 创建若干份堆栈，然后在多个线程之间主动切换。

在 Python 中，`yield` 是生成器（generator）的核心关键字，它用于在函数中暂停函数的执行并返回一个值，同时保留函数的运行状态。与普通函数的区别在于，普通函数每次调用都会从头开始，而带有 `yield` 的函数是一个生成器函数，每次调用时会从上次暂停的地方继续执行。

以下是 `yield` 的运行机制和相关原理：

---

### 1. **生成器函数**
任何包含 `yield` 的函数都是生成器函数。调用生成器函数不会执行函数体，而是返回一个生成器对象。生成器对象是一个迭代器，支持惰性求值。

```python
def my_generator():
    yield 1
    yield 2
    yield 3

gen = my_generator()  # 返回生成器对象
```

---

### 2. **惰性求值**
生成器函数的代码只会在调用 `next()` 方法或在循环中被迭代时执行。`yield` 会暂停函数的执行并将值返回给调用者。

```python
print(next(gen))  # 输出 1，生成器暂停在第一个 yield 处
print(next(gen))  # 输出 2，生成器继续执行到下一个 yield 处
print(next(gen))  # 输出 3
```

当生成器耗尽时，会抛出 `StopIteration` 异常。

---

### 3. **保存状态**
生成器函数内部有一个状态机，`yield` 会保存以下信息：
- **局部变量的值**
- **程序的执行位置**
- **栈帧信息**

因此，生成器可以从暂停的地方继续执行。

```python
def counter():
    x = 0
    while x < 3:
        yield x
        x += 1

gen = counter()
print(next(gen))  # 输出 0，暂停在 yield 处
print(next(gen))  # 输出 1，恢复时 x 是 1
print(next(gen))  # 输出 2
```

---

### 4. **`for` 循环和生成器**
`for` 循环会自动处理 `StopIteration` 异常，因此更推荐用 `for` 来迭代生成器。

```python
for num in counter():
    print(num)  # 依次输出 0, 1, 2
```

---

### 5. **双向通信**
生成器不仅可以通过 `yield` 返回值，还可以通过 `send()` 接收值，实现双向通信。

```python
def echo():
    while True:
        value = yield
        print(f"Received: {value}")

gen = echo()
next(gen)          # 启动生成器
gen.send("Hello")  # 输出 "Received: Hello"
gen.send("World")  # 输出 "Received: World"
```

---

### 6. **`yield from`**
`yield from` 用于简化子生成器的调用，将一个生成器的产生过程委托给另一个生成器。

```python
def sub_generator():
    yield 1
    yield 2

def main_generator():
    yield from sub_generator()  # 简化子生成器调用
    yield 3

for value in main_generator():
    print(value)  # 输出 1, 2, 3
```

---

### 总结
`yield` 的运行机制关键点：
1. 创建生成器对象。
2. 每次调用 `next()`，生成器运行到下一个 `yield` 停止。
3. `yield` 暂停并返回值，保存上下文。
4. 支持通过 `send()` 双向通信。
5. 使用 `yield from` 简化子生成器调用。

它非常适合处理惰性求值、流式数据处理或协程的场景。


调用 generator function 会创建一个闭包 (closure)，而不是直接陷入死循环：

闭包的内部会存储所有局部变量和 PC，遇到 yield 时执行流返回，但闭包仍然可以继续使用，就好像 `stack<Frame>` 还被保留下来，随时可以继续执行。

# 2. 实验描述
使用一个 OS 线程实现主动切换的多个执行流，在这个实验中，实现轻量级的用户态协程（coroutine），可以在一个不支持线程的 OS 上实现共享内存多任务并发。即我们希望实现 C 语言的“函数”，它能够：
- 被 `co_start()` 调用，从头开始运行；
- 在运行到中途时，调用 `co_yield()` 被“切换”出去；
- 稍后有其他协程调用 `co_yield()` 后，选择一个先前被切换的协程继续执行。

协程实现了“可以暂停和恢复执行”的函数。

## 2.1 Coroutine APIs
实现协程库 co.h 中定义的API：
```
struct co *co_start(const char *name, void (*func)(void *), void *arg);

void co_yield();
void co_wait(struct co *co);
```
- co_start(name, func, arg) 创建一个新的协程，并返回一个指向 struct co 的指针
	- 新创建的协程从函数 func 开始执行，并传入参数 arg。新创建的协程不会立即执行，而是调用 `co_start` 的协程继续执行。
	- 使用协程的应用程序不需要知道 `struct co` 的具体定义，因此请把这个定义留在 `co.c` 中；框架代码中并没有限定 `struct co` 结构体的设计，所以你可以自由发挥。
	- `co_start` 返回的 `struct co` 指针需要分配内存。我们推荐使用 `malloc()` 分配。
- `co_wait(co)` 表示当前协程需要等待，直到 `co` 协程的执行完成才能继续执行 (类似于 `pthread_join`)。
	- 在被等待的协程结束后、 `co_wait()` 返回前，`co_start` 分配的 `struct co` 需要被释放。如果你使用 `malloc()`，使用 `free()` 释放即可。
	- 因此，每个协程只能被 `co_wait` 一次 (使用协程库的程序应当保证除了初始协程外，其他协程都必须被 `co_wait` 恰好一次，否则会造成内存泄漏)。
- `co_yield()` 实现协程的切换。协程运行后一直在 CPU 上执行，直到 `func` 函数返回或调用 `co_yield` 使当前运行的协程暂时放弃执行。`co_yield` 时若系统中有多个可运行的协程时 (包括当前协程)，你应当随机选择下一个系统中可运行的协程。
-  `main` 函数的执行也是一个协程，因此可以在 `main` 中调用 `co_yield` 或 `co_wait`。`main` 函数返回后，无论有多少协程，进程都将直接终止。

## 2.2 使用协程库
首先，所有的协程是共享内存的——就是协程所在进程的地址空间。此外，协程就是用户线程，每个协程想要执行，就需要拥有独立的堆栈和寄存器。一个协程的寄存器、堆栈、共享内存就构成了当且协程的状态机执行，用户自己要完成用户线程的上下文切换。
- `co_start` 会在共享内存中创建一个新的状态机 (堆栈和寄存器也保存在共享内存中)，仅此而已。新状态机的 `%rsp` 寄存器应该指向它独立的堆栈，`%rip` 寄存器应该指向 `co_start` 传递的 `func` 参数。根据 32/64-bit，参数也应该被保存在正确的位置 (x86-64 参数在 `%rdi` 寄存器，而 x86 参数在堆栈中)。`main` 天然是个状态机，就对应了一个协程；
- `co_yield` 会将当前运行协程的寄存器保存到共享内存中，然后选择一个另一个协程，将寄存器加载到 CPU 上，就完成了 “状态机的切换”；
- `co_wait` 会等待状态机进入结束状态，即 `func()` 的返回。

## 2.3 协程和线程
唯一不同的是，线程的调度不是由线程决定的 (由操作系统和硬件决定)，但协程除非执行 `co_yield()` 主动切换到另一个协程运行，当前的代码就会一直执行下去。

协程会在执行 `co_yield()` 时主动让出处理器，调度到另一个协程执行。因此，如果能保证 `co_yield()` 的定时执行，我们甚至可以在进程里实现线程。这就有了操作系统教科书上所讲的 “用户态线程”——线程可以看成是每一条语句后都 “插入” 了 `co_yield()` 的协程。这个 “插入” 操作是由两方实现的：操作系统在中断后可能引发上下文切换，调度另一个线程执行；在多处理器上，两个线程则是真正并行执行的。

协程与线程的区别在于协程是完全在应用程序内 (低特权运行级) 实现的，不需要操作系统的支持，占用的资源通常也比操作系统线程更小一些。协程可以随时切换执行流的特性，用于实现状态机、actor model, goroutine 等。在实验材料最前面提到的 Python/Javascript 等语言里的 generator 也是一种特殊的协程，它每次 `co_yield` 都将控制流返回到它的调用者，而不是像本实验一样随机选择一个可运行的协程。


# 4. 实验指南
## 4.1 编译和运行
我们的框架代码的 `co.c` 中没有 `main` 函数——它并不会被编译成一个能直接在 Linux 上执行的二进制文件。编译脚本会生成共享库 (shared object, 动态链接库) `libco-32.so` 和 `libco-64.so`：

共享库可以有自己的代码、数据，甚至可以调用其他共享库 (例如 libc, libpthread 等)。共享库中全局的符号将能被加载它的应用程序调用。共享库中不需要入口 (`main` 函数)。我们的 Makefile 里已经写明了如何编译共享库：

![[Pasted image 20241119182152.png]]

其中 `-fPIC -fshared` 就代表编译成位置无关代码的共享库。除此之外，共享库和普通的二进制文件没有特别的区别。虽然这个文件有 `+x` 属性并且可以执行，但会立即得到 Segmentation Fault (可以试着用 gdb 调试它)。

## 4.3 实现协程：问题分析
分析如何实现 co_yield()
![[Pasted image 20241119182610.png]]

这个程序在 `co_yield()` 之后可能会切换到其他协程执行，但最终它依然会完成 1, 2, 3, ... 1000 的打印。分析 `co_yield` 函数：

- 不能返回到 foo 继续执行，否则切换功能就仿佛没有实现过。
- 不能把正在执行的 foo “返回”，否则 `co_yield` 函数的调用栈帧 (stack frame) 会被摧毁，变量 ii 也就消失了。

我们不妨回到状态机的视角。 `co_yield` 的执行 (call 指令，包括函数的执行)，应该看作一个状态迁移，把当前代码执行的状态机切换到另一段代码：

![[Pasted image 20241119182724.png]]

那么，调用 `co_yield` 瞬间的协程 “状态” 是什么呢？是当前协程栈中的所有数据，外加处理器上所有的寄存器——有它们，我们就可以解释执行任意函数了。所以我们把寄存器 (包括 Stack Pointer) 都保存下来，然后恢复另一个协程保存的寄存器 (和 Stack Pointer)，就完成了任务！

没错！你 “发明” 了分时操作系统内核中具有奠基性意义的机制：“上下文切换”。我们可以为线程 (或协程) 在内存中分配堆栈，让 Stack Pointer 指向各自分配的堆栈中。那么，我们就只要保存寄存器就可以了：

## 4.4 实现协程：数据结构
当然，我们需要全局的指针，指向当前运行的协程 (初始时，我们需要为 main 分配协程对象)：

```
struct co* current;
```

为了实现 `co_yield`，我们：

1. 为每一个协程分配独立的堆栈 (stack)；栈顶的指针由 `current->context` 中的 `%rsp` 寄存器确定；
2. 在 `co_yield` 发生时，将寄存器保存到属于该协程的 `current->context` 中 (包括 `%rsp`)；
3. 切换到另一个协程执行，找到系统中的另一个协程，然后恢复它 `struct co` 中的寄存器现场 (包括 `%rsp`)。

这个过程称为 “上下文切换”。POSIX 为我们提供了 [ucontext.h](https://pubs.opengroup.org/onlinepubs/7908799/xsh/ucontext.h.html) 用于处理线程的寄存器快照，大家可以在实验中使用，或是用之后介绍更高效的方法。

## 4.5 实现协程切换
注意到 `co_yield` 是一个函数调用。编译器在为调用 `co_yield` 生成代码时，会遵循 x86-64 的 calling conventions (System V ABI)：

注意到 `co_yield` 是一个函数调用。编译器在为调用 `co_yield` 生成代码时，会遵循 x86-64 的 calling conventions (System V ABI)：

- 前 6 个参数分别保存在 rdi, rsi, rdx, rcx, r8, r9。当然，`co_yield` 没有参数，这些寄存器里的数值也没有意义。
- `co_yield` 可以任意改写上面的 6 个寄存器的值以及 rax (返回值), r10, r11，返回后 `foo` 仍然能够正确执行。
- `co_yield` 返回时，必须保证 rbx, rsp, rbp, r12, r13, r14, r15 的值和调用时保持一致 (这些寄存器称为 callee saved/non-volatile registers/call preserved, 和 caller saved/volatile/call-clobbered 相反)。

换句话说，`co_yield` 这个 “函数调用” 使编译器在帮助我们减少实现上下文切换的工作量——我们并不需要像 ucontext.h 那样，保存 `rdi` 等 6 个寄存器；更不需要保存浮点数相关的寄存器。而更有趣的是，setjmp/longjmp 恰好帮我们实现了 caller save register 的保存和恢复。

我们实际实现协程切换的代码是

![[Pasted image 20241119183156.png]]

在上面的代码中，`setjmp` 会返回两次

- 在 `co_yield()` 被调用时，`setjmp` 保存寄存器现场后会立即返回 `0`，此时我们需要选择下一个待运行的协程 (相当于修改 `current`)，并切换到这个协程运行。
- `setjmp` 是由另一个 `longjmp` 返回的，此时一定是因为某个协程调用 `co_yield()`，此时代表了寄存器现场的恢复，因此不必做任何操作，直接返回即可。


## 4.6 创建新协程
每当 `co_yield()` 发生时，我们都会选择一个协程继续执行，此时必定为以下两种情况之一 (思考为什么)：

1. 选择的协程是新创建的，此时该协程还没有执行过任何代码——它的 context 是空的。
2. 选择的协程是调用 `yield()` 切换出来的，此时该协程已经调用过 `setjmp` 保存寄存器现场，我们直接 `longjmp` 恢复寄存器现场即可。

真正麻烦的是第一种情况。AbstractMachine 的实现中有一个小巧的 `stack_switch_call` (`x86.h`)，可以用于切换堆栈后并执行函数调用，且能传递一个参数：

![[Pasted image 20241119184038.png]]

我们可以借助这个函数，切换到全新的协程栈并执行函数。注意，这里的堆栈切换是不可避免的——我们也无法用 C/C++ 提供的机制安全地实现它。框架代码里有一行奇怪的 `CFLAGS += -U_FORTIFY_SOURCE`，是用来防止 `__longjmp_chk` 代码检查到堆栈切换以后报错 (当成是 stack smashing)。Google 的 sanitizer [也遇到了相同的问题](https://github.com/google/sanitizers/issues/721)。

## 4.7 资源初始化、管理和释放

### 实现程序运行前的初始化？
许多库函数都会提供一个 init 函数，例如 `co_init()`，在希望使用时调用。但协程库实验中没有。如果你希望在程序运行前完成一系列的初始化工作 (例如分配一些内存)，可以定义 `__attribute__((constructor))` 属性的函数，它们会在 `main` 执行前被运行。

实验中最后一个小小的挑战是资源的合理分配和回收——资源管理是长时间运行软件 (和库) 十分重要的主题。在面向 OJ 编程时，大家养成了很糟糕的习惯：只管申请、不管释放，依赖操作系统在进程结束后自动释放资源。但如果是长期运行的程序，这些没有释放但又不会被使用的**泄露**资源就成了很大但问题，例如在 Windows XP 时代，桌面 Windows 是没有办法做到开机一星期的，一周之后机器就一定会变得巨卡无比。

管理内存说起来轻巧——一次分配对应一次回收即可，但协程库中的资源管理有些微妙 (但并不复杂)，因为 `co_wait` 执行的时候，有两种不同的可能性：

1. 此时协程已经结束 (`func` 返回)，这是完全可能的。此时，`co_wait` 应该直接回收资源。
2. 此时协程尚未结束，因此 `co_wait` 不能继续执行，必须调用 `co_yield` 切换到其他协程执行，直到协程结束后唤醒。

![[Pasted image 20241119184313.png]]

希望大家仔细考虑好每一种可能的情况，保证你的程序不会在任何一种情况下 crash 或造成资源泄漏。然后你会发现，假设每个协程都会被 `co_wait` 一次，且在 `co_wait` 返回时释放内存是一个几乎不可避免的设计：如果允许在任意时刻、任意多次等待任意协程，那么协程创建时分配的资源就无法做到自动回收了——即便一个协程结束，我们也无法预知未来是否还会执行对它的 `co_wait`，而对已经回收的 (非法) 指针的 `co_wait` 将导致 undefined behavior。C 语言中另一种常见 style 是让用户管理资源的分配和释放，显式地提供 `co_free` 函数，在用户确认今后不会使用时释放资源。

资源管理一直是计算机系统世界的难题，至今很多系统还受到资源泄漏、use-after-free 的困扰。例如，顺着刚才资源释放的例子，你可能会感觉 `pthread` 线程库似乎有点麻烦：`pthread_create()` 会修改一个 `pthread_t` 的值，线程返回以后资源似乎应该会被释放。那么：

- 如果 `pthread_join` 发生在结束后不久，资源还未被回收，函数会立即返回。
- 如果 `pthread_join` 发生在结束以后一段时间，可能会得到 `ESRCH` (no such thread) 错误。
- 如果 `pthread_join` 发生在之后很久很久很久很久，资源被释放又被再次复用 (`pthread_t` 是一个的确可能被复用的整数)，我不就 join 了另一个线程了吗？这恐怕要出大问题。

实际上，pthread 线程默认是 “joinable” 的。joinable 的线程只要没有 join 过，资源就一直不会释放。特别地，文档里写明：

> Failure to join with a thread that is joinable (i.e., one that is not detached), produces a "zombie thread". Avoid doing this, since each zombie thread consumes some system resources, and when enough zombie threads have accumulated, it will no longer be possible to create new threads (or processes).

这就是实际系统中各种各样的 “坑”。在《操作系统》这门课程中，我们尽量不涉及这些复杂的行为，而是力图用最少的代码把必要的原理解释清楚。当大家对基本原理有深入的理解后，随着经验的增长，就会慢慢考虑到更周全的系统设计。


---

在 C 语言中，`setjmp` 和 `longjmp` 是一对用来实现非本地跳转（Non-local Jump）的函数，通常用于处理错误、异常或中断流程。这对函数是标准库 `<setjmp.h>` 中提供的。

### **`setjmp` 和 `longjmp` 的作用**

- **`setjmp`**: 保存当前的程序状态（寄存器、堆栈指针等）到一个 `jmp_buf` 类型的变量中，以便稍后通过 `longjmp` 恢复这个状态。
- **`longjmp`**: 从之前保存的状态恢复程序的执行（类似跳转到之前调用 `setjmp` 的地方），同时可以传递一个返回值。

这种机制允许程序从某个嵌套的函数调用中直接返回到更高层的调用栈。

---

### **`setjmp` 的语法**
```c
#include <setjmp.h>

int setjmp(jmp_buf env);
```

- **参数**: 
  - `env`: `jmp_buf` 类型变量，用于保存当前的执行上下文。
- **返回值**: 
  - 初次调用时，总是返回 `0`。
  - 如果通过 `longjmp` 恢复，则返回 `longjmp` 的第二个参数（非零）。

---

### **`longjmp` 的语法**
```c
#include <setjmp.h>

void longjmp(jmp_buf env, int val);
```

- **参数**:
  - `env`: 一个通过 `setjmp` 保存的上下文。
  - `val`: 从 `setjmp` 返回的值，非零值（如果传入 0，则会被强制改为 1）。
- **作用**:
  - 恢复程序到 `setjmp` 调用时保存的上下文，并返回给 `setjmp`。

---

### **基本工作原理**

1. 使用 `setjmp` 保存当前的上下文信息（程序计数器、栈指针、寄存器状态等）。
2. 当某个异常或需要跳转的条件发生时，调用 `longjmp` 跳转到之前的上下文。
3. 通过 `longjmp` 跳转后，程序从 `setjmp` 返回，但返回值不是 `0`。

---

### **代码示例**

#### **简单的 `setjmp` 和 `longjmp` 示例**
```c
#include <stdio.h>
#include <setjmp.h>

jmp_buf env;

void error_handler() {
    printf("Error occurred, jumping back to main...\n");
    longjmp(env, 1);  // 跳回 setjmp，并返回值 1
}

int main() {
    if (setjmp(env) == 0) {
        printf("First time setjmp is called.\n");
        error_handler();  // 触发错误
    } else {
        printf("Returned from longjmp.\n");
    }
    return 0;
}
```

**输出：**
```
First time setjmp is called.
Error occurred, jumping back to main...
Returned from longjmp.
```

#### **错误处理场景**
```c
#include <stdio.h>
#include <setjmp.h>

jmp_buf env;

void process_file(const char *filename) {
    printf("Processing file: %s\n", filename);
    if (filename == NULL) {
        printf("Error: NULL filename!\n");
        longjmp(env, 1);  // 跳回 setjmp
    }
    printf("File processed successfully.\n");
}

int main() {
    if (setjmp(env) == 0) {
        process_file("data.txt");
        process_file(NULL);  // 触发错误
    } else {
        printf("Error handled in main.\n");
    }
    return 0;
}
```

**输出：**
```
Processing file: data.txt
File processed successfully.
Processing file: (null)
Error: NULL filename!
Error handled in main.
```

---

### **`setjmp`/`longjmp` 的常见用途**

1. **错误处理**:
   - 在嵌套函数中发生错误时，可以跳转到一个通用的错误处理器。
   - 类似于 C++ 的异常处理机制，但没有资源清理能力。

2. **状态恢复**:
   - 在协程、上下文切换中使用，以便恢复某些状态。

3. **避免深层嵌套**:
   - 当函数调用嵌套很深时，用 `setjmp`/`longjmp` 实现直接跳转到某个特定的上层函数。

---

### **注意事项**
1. **资源清理**: 
   - `longjmp` 不会调用栈上已展开的函数的析构操作或清理逻辑（例如 C++ 的 RAII）。
   - 因此，在使用中要注意管理动态分配的内存或文件句柄。

2. **跨线程不可用**:
   - `setjmp` 和 `longjmp` 在多线程环境中不安全，因为不同线程有独立的栈。

3. **调试困难**:
   - 非本地跳转可能使程序执行流程不易预测，调试时可能较难追踪代码路径。

---

### **总结**
`setjmp` 和 `longjmp` 提供了一个强大的非本地跳转机制，可以在没有现代异常处理机制的环境中用于错误处理或状态恢复。然而，其强大也伴随着复杂性和潜在的风险，建议谨慎使用，尤其是在代码复杂度较高的情况下。


正如你所描述的，GDB 会因为找不到库而拒绝运行程序。这种情况下，确实需要配置与 `make test` 完全一致的环境。以下是详细步骤：

---

### 1. **确认 `LD_LIBRARY_PATH` 设置正确**
在终端中运行：
```bash
export LD_LIBRARY_PATH=/path/to/your/libs:$LD_LIBRARY_PATH
```
其中，`/path/to/your/libs` 是包含 `libco-64.so` 或其他动态库的目录。

要确认是否正确配置，检查环境变量：
```bash
echo $LD_LIBRARY_PATH
```

---

### 2. **验证环境配置**
在设置完环境变量后，尝试直接运行程序，确保没有库加载错误：
```bash
./libco-test-64
```
如果程序可以正常运行，说明 `LD_LIBRARY_PATH` 已配置正确。

---

### 3. **启动 GDB 并继承环境变量**
确保当前终端已经设置好 `LD_LIBRARY_PATH`，然后启动 GDB：
```bash
gdb ./libco-test-64
```
GDB 会继承当前 Shell 环境中的 `LD_LIBRARY_PATH`，从而能够找到所需的动态库。

---

### 4. **为 GDB 单独设置 `LD_LIBRARY_PATH`**
如果需要在 GDB 中单独设置库路径，可以使用以下命令：
```bash
(gdb) set environment LD_LIBRARY_PATH /path/to/your/libs:$LD_LIBRARY_PATH
(gdb) run
```

---

### 5. **将环境配置脚本化**
为了避免每次都手动配置环境，可以将设置写入脚本。例如，创建一个 `run_gdb.sh` 脚本：
```bash
#!/bin/bash
export LD_LIBRARY_PATH=/path/to/your/libs:$LD_LIBRARY_PATH
gdb ./libco-test-64
```
给脚本添加执行权限：
```bash
chmod +x run_gdb.sh
```
然后通过运行脚本启动 GDB：
```bash
./run_gdb.sh
```

---

### 6. **使用 GDB 的 `file` 和 `set solib-search-path` 命令**
如果仍然遇到问题，可以显式告诉 GDB 去哪里找共享库：
```bash
(gdb) set solib-search-path /path/to/your/libs
(gdb) file ./libco-test-64
(gdb) run
```
