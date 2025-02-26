# 1. const 含义
常类型是指使用类型修饰符**const**说明的类型，常类型的变量或对象的值是不能被更新的。

# 2. const 作用
1. 定义常量
```
const int a=100;
```

2. 类型检查
const常量与#define宏定义常量的区别：**const常量具有类型，编译器可以进行安全检查；#define宏定义没有数据类型，只是简单的字符串替换，不能进行安全检查。**

### 1. **`const` 修饰的变量**

`const` 关键字用于修饰变量，表示该变量是**只读的**，一旦初始化后，它的值不能再修改。然而，这并不意味着 `const` 变量只能是整数或枚举类型。`const` 变量可以是任何类型（如基本数据类型、结构体、类等）。

#### 例如：

```cpp
const int a = 10;         // 整数类型的常量
const double b = 3.14;    // 浮点类型的常量
const char c = 'A';       // 字符常量
const Vector2D v(1.0, 2.0); // 用户定义类型的常量
```

`const` 关键字确保了这些变量在初始化之后不能被修改。如果你尝试修改它们，编译器会报错。

### 2. **常量表达式（`constexpr`）**

`constexpr` 是 C++11 引入的一个关键字，表示一个变量、函数或构造函数在编译时就能被求值为常量。常量表达式是编译时计算的，必须满足特定的条件，通常用于提高性能（如在数组大小、`switch` 语句、`constexpr` 函数等中使用）。

`constexpr` 变量必须是 **常量表达式**，并且它的值必须在编译时已知。这是与 `const` 的一个重要区别：

- **`const`**：表示一个常量，值在运行时确定，可以是任意类型（即使它不能是常量表达式）。`const` 变量的值可以在运行时计算得出。
- **`constexpr`**：表示常量表达式，值必须在编译时就能确定，必须是编译时常量。`constexpr` 变量必须初始化为常量表达式，并且在编译时可以被求值。

#### `const` 与 `constexpr` 的关系：

- 如果你定义一个 `const` 变量，它的值是固定的，但编译器并不要求它是常量表达式。在某些情况下，`const` 变量可以在运行时计算出来。
- 如果你使用 `constexpr` 定义一个变量，它要求变量的值在编译时已知，因此是常量表达式。

### 3. **关于 `const` 变量与常量表达式的初始化**

你的问题中提到的“`const` 定义的变量只有类型为整数或枚举，且以常量表达式初始化时才能作为常量表达式”这个描述并不完全正确。实际上，`const` 变量**不要求**其值必须是常量表达式。`const` 变量的初始化可以是运行时值，只要该值在初始化时确定即可。

### 4. **示例**

#### `const` 变量初始化

```cpp
const int x = 10;          // 常规常量，值为10
const double y = x * 2.0;  // 使用常规常量初始化
```

在这个例子中，`x` 和 `y` 都是 `const` 变量，但它们不需要是常量表达式。`y` 是在运行时由 `x` 计算得出的，但它的值不能被修改。

#### `constexpr` 变量初始化

```cpp
constexpr int x = 10;      // 常量表达式
constexpr int y = x * 2;   // 使用常量表达式初始化
```

在这个例子中，`x` 和 `y` 都是 `constexpr` 变量，值必须是编译时可计算的常量表达式。`y` 的值也必须在编译时就能求出。

### 5. **常量表达式初始化的限制**

`constexpr` 变量要求在编译时可确定其值，因此不能使用运行时计算得到的值。换句话说：

- **`const` 变量**可以在运行时计算，并且它的值可以是非常量表达式。
- **`constexpr` 变量**要求初始化时必须是常量表达式，且不能在运行时求得。

#### 示例：

```cpp
const int x = 10;    // 运行时常量
int runtime_value = 20;
const int y = x + runtime_value;  // 合法，因为运行时值可以赋给 const 变量

constexpr int a = 10;
int runtime_value2 = 20;
// constexpr int b = a + runtime_value2; // 错误！无法将运行时值用于 constexpr 变量初始化
```

### 总结

- `const` 关键字用于声明只读变量，但该变量的值可以在运行时计算，不要求是常量表达式。
- `constexpr` 关键字用于声明编译时常量，要求变量的值在编译时已知，必须是常量表达式。
- `const` 变量不一定要是常量表达式，可以在运行时确定其值，而 `constexpr` 变量则必须是常量表达式，并且它的值必须在编译时已知。

3. 防止修改，起保护作用，增加程序健壮性
```
void f(const int i){ i++; //error! }
```

4. 可以节省空间，避免不必要的内存分配

const定义常量从汇编的角度来看，只是给出了对应的内存地址，而不是像#define一样给出的是立即数，所以，const定义的常量在程序运行过程中只有一份拷贝，而#define定义的常量在内存中有若干个拷贝。

# 3. const 对象默认为文件局部变量
注意：非const变量默认为extern。要使const变量能够在其他文件中访问，必须在文件中显式地指定它为extern。const 变量默认为static，只能在当前文件中访问

- 未被const修饰的变量在不同文件的访问
![[Pasted image 20241207151340.png]]
- const 常量在不同文件的访问
![[Pasted image 20241207151358.png]]
==小结：可以发现未被const修饰的变量不需要extern显式声明！而const常量需要显式声明extern，并且需要做初始化！因为常量在定义后就不能被修改，所以定义时必须初始化。==

# 4. 定义常量
![[Pasted image 20241207151425.png]]
上述有两个错误，第一：b为常量，不可更改！第二：i为常量，必须进行初始化！(因为常量在定义后就不能被修改，所以定义时必须初始化。)

# 5. 指针与const
与指针相关的const有四种：
![[Pasted image 20241207151547.png]]
小结：如果 const 位于`*`的左侧，则 const 就是用来修饰指针所指向的变量，即指针指向为常量；如果const位于`*`的右侧，const 就是修饰指针本身，即指针本身是常量。

当指针被加上 const 特性，则指针不可改变指向的地址，但是指针指向的内容可以被修改。
当指向的目标为const char，则内容不可通过指针修改。

具体使用如下：
1. 指向常量的指针
```
const int *ptr; 
*ptr = 10; //error
```

ptr是一个指向int类型const对象的指针，const定义的是int类型，也就是ptr所指向的对象类型，而不是ptr本身，所以ptr可以不用赋初始值。但是不能通过ptr去修改所指对象的值。

除此之外，也不能使用void`*`指针保存const对象的地址，必须使用const void`*`类型的指针保存const对象的地址。
![[Pasted image 20241207151919.png]]

另外一个重点是：**允许把非const对象的地址赋给指向const对象的指针**。

将非const对象的地址赋给const对象的指针:
![[Pasted image 20241207152018.png]]
我们不能通过ptr指针来修改val的值，即使它指向的是非const对象!

我们不能使用指向const对象的指针修改基础对象，然而如果该指针指向了非const对象，可用其他方式修改其所指的对象。可以修改const指针所指向的值的，但是不能通过const对象指针来进行而已！如下修改：
![[Pasted image 20241207152123.png]]

小结：
- 对于指向常量的指针，不能通过指针来修改对象的值。  
- 也不能使用void`*`指针保存const对象的地址，必须使用const void`*`类型的指针保存const对象的地址。  
- 允许把非const对象的地址赋值给const对象的指针，如果要修改指针所指向的对象值，必须通过其他方式修改，不能直接通过当前指针直接修改。

![[Pasted image 20241231105943.png]]

### 常指针
const 指针必须进行初始化，且 const 指针指向的值能修改，但指针本身不能修改。

当把一个 const 常量的地址赋给 ptr 时候，由于 ptr 指向的是一个变量，而不是 const 常量，所有会报错，出现 const int * -> int * 错误
![[Pasted image 20250109130445.png]]
上述若改为 const int `*`ptr或者改为const int `*`const ptr，都可以正常！

### 指向常量的常指针
理解完前两种情况，下面这个情况就比较好理解了：
![[Pasted image 20250109130516.png]]
ptr是一个const指针，然后指向了一个int 类型的const对象。

# 6.函数中使用 const
## const int
![[Pasted image 20250109130903.png]]
无意义

const 修饰函数参数
传递过来的参数及指针本身在函数内不可变，无意义！
![[Pasted image 20250109130944.png]]
表明参数在函数体内不能被修改，但此处没有任何意义，var本身就是形参，在函数内不会改变。包括传入的形参是指针也是一样。

输入参数采用“值传递”，由于函数将自动产生临时变量用于复制该参数，该输入参数本来就无需保护，所以不要加const 修饰。

## 参数指针所指向内容为常量不可变
![[Pasted image 20250109131105.png]]
其中src 是输入参数，dst 是输出参数。给src加上const修饰后，如果函数体内的语句试图改动src的内容，编译器将指出错误。这就是加了const的作用之一。

## 参数为引用，为了增加效率同时防止修改
![[Pasted image 20250109131146.png]]
对于非内部数据类型的参数而言，像void func(A a) 这样声明的函数注定效率比较低。因为函数体内将产生A 类型的临时对象用于复制参数a，而临时对象的构造、复制、析构过程都将消耗时间。

为了提高效率，可以将函数声明改为void func(A &a)，因为“引用传递”仅借用一下参数的别名而已，不需要产生临 时对象。

但是函数void func(A &a) 存在一个缺点：  
  
“引用传递”有可能改变参数a，这是我们不期望的。解决这个问题很容易，加const修饰即可，因此函数最终成为 void func(const A &a)。

>小结：  
1.对于非内部数据类型的输入参数，应该将“值传递”的方式改为“const 引用传递”，目的是提高效率。例如将void func(A a) 改为void func(const A &a)。  
  2.对于内部数据类型的输入参数，不要将“值传递”的方式改为“const 引用传递”。否则既达不到提高效率的目的，又降低了函数的可理解性。例如void func(int x) 不应该改为void func(const int &x)。

以上解决了两个面试问题：

- 如果函数需要传入一个指针，是否需要为该指针加上const，把const加在指针不同的位置有什么区别；
- 如果写的函数需要传入的参数是一个复杂类型的实例，传入值参数或者引用参数有什么区别，什么时候需要为传入的引用参数加上const。

# 7. 类中使用 const
在一个类中，任何不会修改数据成员的函数都应该声明为const类型。如果在编写const成员函数时，不慎修改数据成员，或者调用了其它非const成员函数，编译器将指出错误，这无疑会提高程序的健壮性。

使用const关键字进行说明的成员函数，称为常成员函数。只有常成员函数才有资格操作常量或常对象，没有使用const关键字明的成员函数不能用来操作常对象。

对于类中的 const 成员变量必须通过初始化列表进行初始化，如下所示：
![[Pasted image 20250109132019.png]]

这段代码定义了一个名为 `Apple` 的类的构造函数。我们可以逐步解释每个部分的含义：

### 1. 类名和构造函数

```cpp
Apple::Apple(int i)
```

- `Apple` 是类的名字，构造函数是与类同名的特殊成员函数。
- `Apple(int i)` 是构造函数的声明，表示这个构造函数接受一个类型为 `int` 的参数 `i`。

### 2. 初始化列表

```cpp
: apple_number(i)
```

- `apple_number(i)` 是 **初始化列表**，它用于在构造函数体执行之前初始化类的成员变量。
- 这里，它表示将传入的参数 `i` 的值赋给类成员 `apple_number`。

### 3. 构造函数体

```cpp
{
}
```

- `{}` 中为空，说明这个构造函数没有其他代码，构造函数仅通过初始化列表来初始化成员变量。

### 总结

这段代码的意思是，当创建 `Apple` 类的对象时，构造函数将接收一个 `int` 类型的参数，并用这个参数初始化成员变量 `apple_number`。初始化列表用于在构造函数体执行之前对成员变量进行初始化，它比在构造函数体中赋值更加高效。

### 假设的完整代码

为了更好理解，我们可以假设类的定义如下：

```cpp
class Apple {
private:
    int apple_number;  // 类的成员变量

public:
    // 构造函数
    Apple(int i): apple_number(i) {
        // 此构造函数使用初始化列表初始化 apple_number
    }

    // 可以添加其他成员函数和方法
};
```

在上面的代码中，`apple_number` 是 `Apple` 类的成员变量，构造函数通过初始化列表将传入的 `int i` 参数赋值给它。

### 示例用法：

```cpp
Apple myApple(10);  // 创建一个 Apple 对象，apple_number 会被初始化为 10
```

在这里，`myApple` 对象的 `apple_number` 成员变量将被初始化为 10。

---
const对象只能访问const成员函数,而非const对象可以访问任意的成员函数,包括const成员函数.

`#pragma once` 是一种预处理指令，用于防止头文件被多次包含。在 C++ 中，头文件通常会被多个源文件或其他头文件引用，如果没有适当的防护，头文件可能会被多次包含，从而导致重复定义的问题。`#pragma once` 解决了这一问题，确保一个头文件只被包含一次。

### 工作原理

当编译器遇到 `#pragma once` 时，它会记住该头文件已被包含，并在后续的编译过程中跳过该文件的重复包含。这与传统的 `#ifndef`、`#define` 和 `#endif` 宏保护方法类似，但使用起来更加简洁。

### 例子

#### 使用 `#pragma once` 的头文件：

```cpp
// Apple.h
#pragma once

class Apple {
private:
    int apple_number;

public:
    Apple(int i) : apple_number(i) {}
    int getAppleNumber() const { return apple_number; }
};
```

#### 使用传统的宏保护：

```cpp
// Apple.h
#ifndef APPLE_H
#define APPLE_H

class Apple {
private:
    int apple_number;

public:
    Apple(int i) : apple_number(i) {}
    int getAppleNumber() const { return apple_number; }
};

#endif // APPLE_H
```

### 区别

- `#pragma once` 更简洁，只需在文件的开头添加一行即可。
- 传统的宏保护方法需要定义一个唯一的宏（例如 `APPLE_H`），并且每次包含时都进行检查。

### 优点

1. **简洁性**：`#pragma once` 比传统的宏保护方式更为简洁，代码更少。
2. **避免错误**：它不需要手动定义宏标识符，因此可以避免由于宏名冲突或者疏漏导致的问题。

### 缺点

1. **可移植性问题**：虽然大多数现代编译器都支持 `#pragma once`，但并不是所有编译器都支持它。例如，较旧的编译器可能不支持这个特性。在这种情况下，传统的宏保护方法仍然是最可靠的解决方案。

### 总结

- `#pragma once` 是一种预处理指令，用来防止头文件被多次包含。
- 它比传统的 `#ifndef` 宏保护方式简洁，但需要确保编译器支持 `#pragma once`。
- 对于现代编译器，`#pragma once` 提供了一个更加简洁且不容易出错的方式来实现头文件保护。


在你提供的 `Apple` 类定义中，`const` 关键字用于不同的上下文，具体含义如下：

### 2. `void take(int num) const;`

```cpp
void take(int num) const;
```

- 在这个成员函数声明中，**`const`** 出现在函数的末尾，表示这个成员函数是 **常量成员函数**。
- 常量成员函数意味着该函数承诺不会修改类的任何成员变量。也就是说，`take` 函数中不能修改 `apple_number` 或其他非 `mutable` 成员变量。
- 常量成员函数通常用于访问成员变量而不改变它们。

举个例子：

```cpp
void Apple::take(int num) const {
    // apple_number = num;  // 错误！不能修改成员变量
    std::cout << "Taking " << num << " apples." << std::endl;
}
```


如果你尝试在 `const` 成员函数中修改对象的成员变量，编译器会报错：

```cpp
int Apple::add(int num) const {
    apple_number += num;  // 错误！不能修改 const 成员变量
    return apple_number;
}
```

### 总结

- **`const int apple_number;`**：声明成员变量 `apple_number` 为常量，表示它一旦被初始化后，不能再改变。
- **`void take(int num) const;`** 和 **`int add(int num) const;`**：这些函数是常量成员函数，表示它们不能修改类的任何非 `mutable` 成员变量。
- 常量成员函数（带 `const`）主要用于保证在调用函数时，类的状态不会被改变。这有助于提高代码的安全性和可预测性。

const对象只能访问const成员函数,而非const对象可以访问任意的成员函数,包括const成员函数.

const成员函数只能访问const成员函数。

我们除了上述的初始化const常量用初始化列表方式外，也可以通过下面方法：

第一：将常量定义与static结合，也就是：
![[Pasted image 20250109134116.png]]

第二：在外面初始化：
![[Pasted image 20250109134130.png]]

当然，如果你使用c++11进行编译，直接可以在定义出初始化，可以直接写成：
![[Pasted image 20250109134142.png]]
这两种都在c++11中支持！

编译的时候加上`-std=c++11`即可！

这里提到了static，下面简单的说一下：

在C++中，static静态成员变量不能在类的内部初始化。在类的内部只是声明，定义必须在类定义体的外部，通常在类的实现文件中初始化。

在类中声明：
![[Pasted image 20250109134225.png]]



