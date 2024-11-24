C 内联汇编是指在 C 代码中嵌入汇编代码，从而直接操作底层硬件或优化性能。以下是关于 C 内联汇编的详细讲解，包括基本语法、用法以及实例。

---

## **1. 基础语法**
C 内联汇编使用 `asm` 或 `__asm__` 关键字（GCC/Clang）。基本格式如下：

```c
asm ("assembly_code" : output_operands : input_operands : clobbers);
```

### **语法结构解析**
1. **`assembly_code`**:
   - 汇编代码，通常是字符串，可以包含多行，用 `\n\t` 分隔。
   - 格式与目标架构（如 x86、ARM）的汇编指令一致。

2. **`output_operands`**:
   - 输出操作数列表，用逗号分隔。格式为 `["constraint"] (C_variable)`。
   - 例如，`"=r"(result)` 表示将结果存储到 C 变量 `result` 中。

3. **`input_operands`**:
   - 输入操作数列表，用逗号分隔。格式为 `["constraint"] (C_variable)`。
   - 例如，`"r"(a)` 表示将 C 变量 `a` 的值作为寄存器输入。

4. **`clobbers`**:
   - 声明被汇编代码修改的寄存器或内存。常见值包括 `"memory"` 和 `"cc"`（条件码）。

---

## **2. 示例讲解**

### **2.1 简单的加法**
计算两个整数的和并存储到结果变量：

```c
#include <stdio.h>

int main() {
    int a = 5, b = 3, result;
    
    asm ("addl %1, %0"
         : "=r" (result)   // 输出: result 存储在通用寄存器
         : "r" (a), "0" (b) // 输入: a 和 b 作为寄存器输入
         : "cc");          // Clobber: 修改了条件码寄存器

    printf("Result: %d\n", result); // 输出 8
    return 0;
}
```

#### **解释**:
- **`addl %1, %0`**: 汇编指令，表示 `%0 += %1`。
- **`"=r"(result)`**: 输出到寄存器并存储到 `result` 中。
- **`"r"(a)` 和 `"0"(b)`**: 将 `a` 和 `b` 分别加载到寄存器中。

---

### **2.2 访问寄存器**
直接访问特定寄存器，例如读取栈指针：

```c
#include <stdio.h>

int main() {
    unsigned long rsp;

    asm ("mov %%rsp, %0"
         : "=r" (rsp));  // 输出到变量 rsp

    printf("Stack Pointer: 0x%lx\n", rsp);
    return 0;
}
```

#### **解释**:
- **`mov %%rsp, %0`**: 将 `rsp` 寄存器的值移动到输出操作数 `%0`。
- **`%%rsp`**: 双百分号用于区分 C 和汇编的标识符。

---

### **2.3 修改标志寄存器**
通过汇编代码影响条件跳转，例如设置条件码：

```c
#include <stdio.h>

int main() {
    int a = 5, b = 10, is_greater;

    asm ("cmp %1, %2\n\t"
         "setg %0"
         : "=r" (is_greater)
         : "r" (a), "r" (b)
         : "cc");  // 修改条件码寄存器

    printf("Is a > b? %d\n", is_greater); // 输出 0
    return 0;
}
```

#### **解释**:
- **`cmp %1, %2`**: 比较 `%1 - %2`，设置条件码。
- **`setg %0`**: 如果条件码 `>` 为真，则将 `%0` 置为 1，否则为 0。

---

### **2.4 带内存操作的代码**
实现内存读取或写入，例如交换两个整数：

```c
#include <stdio.h>

int main() {
    int a = 5, b = 10;

    asm ("xchg %0, %1"
         : "=r"(a), "=r"(b) // 输出: 交换后的值
         : "0"(a), "1"(b)   // 输入: a 和 b 的初始值
         : "memory");       // Clobber: 可能修改了内存

    printf("a = %d, b = %d\n", a, b); // 输出 a=10, b=5
    return 0;
}
```

#### **解释**:
- **`xchg %0, %1`**: 交换 `%0` 和 `%1` 的值。
- **`"memory"`**: 提示可能修改了内存以避免编译器优化。

在 GCC 的内联汇编中，`"0"` 和 `"1"` 是对约束编号的引用。它们并不是字面值 `0` 和 `1`，而是用于引用前面已经声明的操作数。

---

### **约束编号的解释**
1. **约束编号**  
   编译器为每个操作数分配一个编号，从上到下按顺序编号。
   - 输出操作数从 `0` 开始编号。
   - 输入操作数从前面输出操作数的编号之后继续编号。

2. **`"0"` 和 `"1"`**  
   这些是对输出操作数编号的引用，用于告诉编译器：某个输入操作数与特定编号的输出操作数共享同一个寄存器或存储单元。

---

### **你的代码中的用法**
在你的原代码中：
```c
asm ("xchg %0, %1"
     : "=r"(a), "=r"(b) // 输出: 交换后的值
     : "0"(a), "1"(b)   // 输入: a 和 b 的初始值
     : "memory");
```

#### **逐项解析**
1. **`=r`**  
   表示 `a` 和 `b` 的值在汇编代码执行后会被修改，结果会存入寄存器。

2. **`"0"(a)`**  
   表示输入操作数 `a` 必须与输出操作数 `a` 使用同一个寄存器。

3. **`"1"(b)`**  
   表示输入操作数 `b` 必须与输出操作数 `b` 使用同一个寄存器。

通过这种引用方式，编译器确保输入变量 `a` 和 `b` 共享输出变量的寄存器，从而避免额外的寄存器分配。

---

### **改进后的建议**
虽然 `"0"` 和 `"1"` 的用法是合法的，但它显得较为冗余，直接使用 `+r` 来绑定输入和输出会更直观。例如：
```c
asm volatile (
    "xchg %0, %1"
    : "+r"(a), "+r"(b)  // 输入和输出绑定
    :
    : "memory"
);
```
这段代码表达了同样的逻辑，更加简洁和现代。

---

## **3. 常用约束符**
约束符指定输入/输出变量如何与寄存器或内存绑定：
- **`r`**: 寄存器。
- **`m`**: 内存地址。
- **`i`**: 常数立即数。
- **`=r`**: 输出，只写寄存器。
- **`+r`**: 输入输出，读写寄存器。
- **`&r`**: 输出，避免与输入重叠的寄存器。


---

```
 asm [ volatile ] (   
        assembler template  
        [ : output operands ]                /* optional */  
        [ : input operands  ]                /* optional */  
        [ : list of clobbered registers ]    /* optional */  
        );
```

# 2 assembler template
双引号括起来，以便 gcc 以字符串形式将这些命令传给汇编器 AS

有时候，汇编命令可能有多个，则通常分多行写，每行的命令都用双引号括起来，命令后紧跟"\n\t"之类的分隔符（当然，也可以只用1对双引号将多行命令括起来，从语法来说，两种写法均有效，我们可自行决定用哪种格式来写）。示例代码如下所示：

```
__asm__ __volatile__ ( "movl %eax, %ebx\n\t"  
                      "movl %ecx, 2(%edx, %ebx, $8)\n\t"  
                      "movb %ah, (%ebx)"  
                    );
```

出现类似 Magic Number 的操作数，比如下面的代码
![[Pasted image 20241120123048.png]]

我们看到，movl指令的操作数（operand）中，出现了%1、%0，这往往让新手摸不着头脑。**其实只要知道下面的规则就不会产生疑惑了：**

在内联汇编中，操作数通常用数字来引用，具体的编号规则为：若命令共涉及n个操作数，则第1个输出操作数（the first output operand）被编号为0，第2个output operand编号为1，依次类推，最后1个输入操作数（the last input operand）则被编号为n-1。

还需要注意：当命令中同时出现寄存器和以 %num 来引用的操作数时，会以 \%\%reg
来引用寄存器，以便帮助 gcc 来区分寄存器和由 C 语言提供的操作数。

# 3 output operands
该字段为可选项，用以指明输出操作数，典型的格式为：
```
: "=a" (out_var)
```
其中，"=a"指定output operand的应遵守的约束（constraint），out_var为存放指令结果的变量，通常是个C语言变量。本例中，“=”是output operand字段特有的约束，表示该操作数是只写的（write-only）；“a”表示先将命令执行结果输出至%eax，然后再由寄存器%eax更新位于内存中的out_var。

若输出有多个，则典型格式示例如下：
![[Pasted image 20241120123615.png]]

# 4 input operands
该字段为可选项，用以指明输入操作数，其典型格式为：
```
: "constraints" (in_var)
```
其中，constraints可以是gcc支持的各种约束方式，in_var通常为C语言提供的输入变量。

当然，input operands + output operands的总数通常是有限制的，考虑到每种指令集体系结构对其涉及到的指令支持的最多操作数通常也有限制，此处的操作数限制也不难理解。此处具体的上限为max(10, max_in_instruction)，其中max_in_instruction为ISA中拥有最多操作数的那条指令包含的操作数数目。

需要明确的是，在指明input operands的情况下，即使指令不会产生output operands，其:也需要给出。例如asm ("sidt %0\n" : :"m"(loc)); 该指令即使没有具体的output operands也要将:写全，因为有后面跟着: input operands字段。

# 5 **list of clobbered registers**
该字段为可选项，用于列出指令中涉及到的且没出现在output operands字段及input operands字段的那些寄存器。若寄存器被列入clobber-list，则等于是告诉gcc，这些寄存器可能会被内联汇编命令改写。因此，执行内联汇编的过程中，这些寄存器就不会被gcc分配给其它进程或命令使用。

# 6 常用约束**commonly used constraints**
前面介绍output operands和input operands字段过程中，我们已经知道这些operands通常需要指明各自的constraints，以便更明确地完成我们期望的功能（试想，如果不明确指定约束而由gcc自行决定的话，一旦代码执行结果不符合预期，调试将变得很困难）。

下面开始介绍一些常用的约束项。
1）寄存器操作数约束 (register operand constraint, r)
当操作数被指定为这类约束时，表明汇编指令执行时，操作数被将存储在指定的通用寄存器（General Purpose Registers, GPR）中。例如：
```
asm ("movl %%eax, %0\n" : "=r"(out_val));
```

该指令的作用是将%eax的值返回给%0所引用的C语言变量out_val，根据"=r"约束可知具体的操作流程为：先将%eax值复制给任一GPR，最终由该寄存器将值写入%0所代表的变量中。"r"约束指明gcc可以先将%eax值存入任一可用的寄存器，然后由该寄存器负责更新内存变量。

通常还可以明确指定作为“中转”的寄存器，约束参数与寄存器的对应关系为：

a : %eax, %ax, %al

b : %ebx, %bx, %bl

c : %ecx, %cx, %cl

d : %edx, %dx, %dl

S : %esi, %si

D : %edi, %di

例如，如果想指定用%ebx作为中转寄存器，则命令为：asm ("movl \%\%eax, %0\n" : "=b"(out_val));

# 7 内存操作数约束(Memory operand constraint, m)
当我们不想通过寄存器中转，而是直接操作内存时，可以用"m"来约束。例如：

asm volatile ( "lock; decl %0" : "=m" (counter) : "m" (counter));

该指令实现原子减一操作，输入、输出操作数均直接来自内存（也正因如此，才能保证操作的原子性）。

# 8 关联约束（matching constraint）
在有些情况下，如果命令的输入、输出均为同一个变量，则可以在内联汇编中指定以matching constraint方式分配寄存器，此时，input operand和output operand共用同一个“中转”寄存器。例如：

asm ("incl %0" :"=a"(var):"0"(var));

该指令对变量var执行incl操作，由于输入、输出均为同一变量，因此可用"0"来指定都用%eax作为中转寄存器。注意"0"约束修饰的是input operands。





