- Separate compilation of programs
- Storage duration, scope, and linkage
- Placement `new`
- Namespaces

C++ offers many choices for storing data in memory. You have choices for how long data remains in memory (storage duration) and choices for which parts of a program have access to data (scope and linkage)

# Separate Compilation
Divide the original program into three parts:
- A header file that contains the structure declarations and prototypes for functions that use those structures
- A source code file that contains the code for the structure-related functions
- A source code file that contains the code that calls the structure-related functions

Here are some things commonly found in header files:
- Function prototypes
- Symbolic constants defined using `#define` or `const`
- Structure declarations
- Class declarations
- Template declarations
- Inline functions

It's okay to put structure declarations in a header file because they don't create variables; Data declared `const` and inline functinos have special linkage properties (described shortly) that allow them to placed in header files without causing problems.

```cpp
//7.12 strctfun.cpp -- functions with a structure argument
#include <iostream>
#include <cmath>

struct polar{
	double distance;
	double angle;
};
struct rect{
	double x;
	double y;
};
// prototypes
polar rect_to_polar(rect xypos);
void show_polar(polar dapos);

int main(){
	using namespace std;
	rect rplace;
	polar pplace;
	while(cin>>rplace.x >> rplace.y){
		pplace = rect_to_polar(rplace);
		show_polar(pplace);
	}
	cout<< "Done.\n";
	return 0;
}
polar rect_to_polar(rect xypos){
	using namespace std;
	polar answer;
	answer.distance = sqrt(xypos.x * xypos.x);
	answer.angle = ....;
	return answer;
}
void show_polar(polar dapos){
	....;
	
}
```

Listings 9.1 9.2, and 9.3 show the result of dividing Listing 7.12 into separate parts.
```cpp
// 9.1 coordin.h
#ifndef COORDIN_H
#define COORDIN_H
struct polar{
	double distance;
	double angle;
};
struct rect{
	double x;
	double y;
};
// prototypes
polar rect_to_polar(rect xypos);
void show_polar(polar dapos);
#endif
```
![[Pasted image 20250107151321.png]]
Figure 9.1 outlines the steps for putting this program this program together on a Unix system. Note that you only add source code files, not header files, to projects. That's because the `#include` directive manages the header files. 

>Header file Management
>You should include a header file just once in a file. There's a standard C/C++ technique for avoiding multiple inclusions of header files. It's based on the preprocessor `#ifndef` directive. 
>The first time the compiler encounters the file, the name `COORDIN_H` should be undefined. That being the case, the compiler looks at the material between the `#ifndef` and the `#endif`, which is what you want. If it then encounters a second inclusion of `coordin.h` in the same file, the compiler notes that `COORDIN_H` is defined and skips to the line following the `#endif`. Note that this method doesn't keep the compiler from including a file twice. Most of the standard C and C++ header files use this guarding scheme. Otherwise you might get the same structure defined twice in one file, and that will produce a compiler error.

```cpp
// Listing 9.2 file1.cpp
#include <iostream>
#include "coordin.h"
using namespace std;
int main(){
	rect rplace;
	polar pplace;
	...;
	return 0;
}
```
```cpp
//Listing 9.3 file2.cpp
#include <iostream>
#include <cmath>
#include "coordin.h"

polar rect_to_polar (rect xypos){
	...;
}
```

By the way, although we've discussed separate compilation in terms of files, the C++ Standard uses the term _translation unint_ instead of _file_ in order to preserve greater generality;
> Multiple Library Linking
> C++ 标准允许每个编译器设计者根据自己的需要自由地实现名称修饰或修改（请参阅第 8 章“函数历险记”中的边栏“什么是名称修饰？”），因此您应该知道，使用不同编译器创建的二进制模块（目标代码文件）很可能无法正确链接。也就是说，两个编译器将为同一函数生成不同的修饰名称。此名称差异将阻止链接器将一个编译器生成的函数调用与第二个编译器生成的函数定义进行匹配。尝试链接已编译的模块时，您应该确保每个目标文件或库都是使用同一编译器生成的。如果您获得了源代码，通常可以通过使用编译器重新编译源代码来解决链接错误。

# Storage Duration, Scope, and Linkage
C++ uses three separate schemes (four under C++11) for storing data, and the schemes differ in how long they preserve data in memory:
- **Automatic storage duration**, Variables declared inside a function definition.
- **Static storage duration** —— Variables defined outside a function definition or else by using the keyword `static` have static storage duration. They persist for the entire time a program is running.
- **Thread storage duration** —— This allows a program to split computations into separate `threads` that can be processed concurrently. Variables declared with the `thread_local` keyword have storage that persists for as long as the containing thread lasts.
- **Dynamic storage duration**—— Memory allocated by the `new` operator persists until it is freed with the `delete` operator or until the program ends.

## Scope and Linkage
_Scope_ describes how widely visible a name is in a file (translation unit). _Linkage_ describes how a name can be shared in different units. A name with _external linkage_ can be shared across files, and a name with _internal linkage_ can be shared by functions within a single file. Names of automatic variables have no linkage because they are not shared.

_local scope_ (also _block scope_) is known only within the block in which it is defined. Recall that a block is a series of statements enclosed in braces. A variable that has _global scope_ (also termed _file scope_) is known throughout the file after the point where it is defined.

C++ functions can have class scope or namespace scope, including global scope, but they can't have local scope.

If you define a variable inside a block, the variable's persistence and scope are confined to that block. `teledeli` is visible in both the outer and inner blocks, whereas `websight` exists only in the inner block.
```cpp
int main(){
	int teledeli = 5;
	{
		int websight = -1;
		cout<<websight;
	}
	cout << teledeli;
	...
}
```

But what if you name the variable in the inner block `teledeli` instead of `websight` so that you have two variables of the same name, with one in the outer block and one in the inner block? 
![[Pasted image 20250107155714.png]]

## Register Variables
C orginally introduced the `register` keyword to suggest that the compiler use a CPU register to store an automatic variable:
```
register int count_fast;// request for a register variable
```
The idea was that this would allow faster access to the variable.

在 C++11 之前，C++ 以相同的方式使用关键字，只是随着硬件和编译器的复杂化，该提示被泛化为表示变量被大量使用，并且编译器可能提供某种特殊处理。在 C++11 中，甚至该提示也被弃用，只留下 register 作为明确标识变量为自动变量的一种方式。鉴于 register 只能用于本来就是自动变量的变量，因此使用此关键字的一个原因是表明您确实想要使用自动变量，也许是与外部变量同名的变量。这与最初使用 auto 的目的相同。然而，保留 register 更重要的原因是避免使使用该关键字的现有代码无效。

## Static Duration Variables
C++ provides static storage duration variables with three kinds of linkage: external linkage (accessible across files), internal linkage (accessible to functions with a single file), and no linkage (accessible to just one function or to one block within a function). All three last for the duration of the program; The compiler allocates a fixed block of memory to hold all the static variables. Also, if you don't explicitly initialize a static variable, the compiler sets it to 0. Static arrays and structures have all the bits of each element or member set to 0 by default.

在C++中，`static` 变量有三种链接类型：**外部链接（external linkage）**、**内部链接（internal linkage）**和**无链接（no linkage）**。这些变量的生命周期是整个程序运行期间，直到程序结束时才会被销毁。与自动变量（local variables）不同，`static` 变量不会在函数调用结束时被销毁，它们在整个程序的执行过程中保持有效。

### 1. **外部链接（External Linkage）**：

外部链接的变量是可以在多个文件之间共享的，通常会在一个源文件中定义，并在其他源文件中引用。

#### 示例：

```cpp
// file1.cpp
#include <iostream>

int global_var = 10;  // 外部链接的全局变量

void printGlobalVar() {
    std::cout << "Global variable: " << global_var << std::endl;
}

int main() {
    printGlobalVar();
    global_var = 20;
    printGlobalVar();
    return 0;
}
```

```cpp
// file2.cpp
extern int global_var;  // 声明外部链接的变量

void modifyGlobalVar() {
    global_var = 50;
}
```

- `global_var` 是具有外部链接的变量，意味着它在程序的任何地方都是可访问的。
- `extern` 关键字用于在 `file2.cpp` 中声明它，使得 `file2.cpp` 可以访问 `file1.cpp` 中定义的 `global_var`。

### 2. **内部链接（Internal Linkage）**：

内部链接的变量仅在同一文件中的不同函数之间共享。它的作用域仅限于文件内部。

#### 示例：

```cpp
// file1.cpp
#include <iostream>

static int internal_var = 30;  // 内部链接的静态变量

void printInternalVar() {
    std::cout << "Internal variable: " << internal_var << std::endl;
}

int main() {
    printInternalVar();
    internal_var = 40;
    printInternalVar();
    return 0;
}
```

- `internal_var` 是一个静态变量，具有内部链接，意味着它只能在 `file1.cpp` 文件中的函数之间访问。
- 如果你尝试在另一个源文件中引用 `internal_var`，会产生链接错误，因为它的作用域被限制在当前文件内。

### 3. **无链接（No Linkage）**：

无链接的变量是局部静态变量，仅在其声明的函数或块内有效。它的作用域限于所在的函数或代码块，但它的生命周期贯穿整个程序执行过程。

#### 示例：

```cpp
#include <iostream>

void myFunction() {
    static int counter = 0;  // 无链接的静态局部变量
    counter++;
    std::cout << "Counter: " << counter << std::endl;
}

int main() {
    myFunction();  // 输出：Counter: 1
    myFunction();  // 输出：Counter: 2
    myFunction();  // 输出：Counter: 3
    return 0;
}
```

- `counter` 是一个静态局部变量，具有无链接，意味着它只在 `myFunction` 内部有效，但它的值在函数调用之间会保持。
- 即使 `myFunction` 被多次调用，`counter` 变量也不会被重新初始化，每次调用时它都会保留上次调用时的值。

### 静态变量的初始化

如你所说，静态变量如果没有显式初始化，编译器会将它们初始化为 0。下面是一些默认初始化的例子。

#### 示例：

```cpp
#include <iostream>

static int static_var;  // 默认为0
static int static_array[5];  // 默认为全0

int main() {
    std::cout << "Static var: " << static_var << std::endl;  // 输出 0
    std::cout << "Static array first element: " << static_array[0] << std::endl;  // 输出 0
    return 0;
}
```

- `static_var` 和 `static_array` 都是静态变量，如果没有显式初始化，它们的初始值会是 0。
- 对于静态数组和结构体，其每个元素或成员都会被设置为 0。

### 总结：

- **外部链接**：可以跨多个源文件访问，通常用于全局变量。
- **内部链接**：仅在同一源文件内部有效，适用于文件内共享的变量。
- **无链接**：作用于局部变量，但它们的生命周期贯穿整个程序的执行过程。

这三种静态变量在不同的应用场景下都非常有用，可以控制变量的可见性和生命周期，适应不同的需求。

