In C++11 and later, a lambda expression -- often called a _lambda_ -- is a convenient way of defining an anonymous function object (a _closure_) right at the location where it's invoked or passed as an argument to a function.
下面是一个简单的 Lambda，它作为第三个参数传递给 `std::sort()` 函数：
![[Pasted image 20241221141302.png]]
此图显示了 Lambda 语法的组成部分：
![[Pasted image 20241221141308.png]]
1. capture 子句（在 C++ 规范中也称为 Lambda 引导。）
    
2. 参数列表（可选）。 （也称为 Lambda 声明符）
    
3. mutable 规范（可选）。
    
4. exception-specification（可选）。
    
5. trailing-return-type（可选）。
    
6. Lambda 体。

# capture clause
Lambda 可在其主体中引入新的变量（用 C++14），它还可以访问（或“捕获”）周边范围内的变量。 Lambda 以 capture 子句开头。 它指定捕获哪些变量，以及捕获是通过值还是通过引用进行的。 有与号 (`&`) 前缀的变量通过引用进行访问，没有该前缀的变量通过值进行访问。

空 capture 子句 `[ ]` 指示 lambda 表达式的主体不访问封闭范围中的变量。

可以使用默认捕获模式来指示如何捕获 Lambda 体中引用的任何外部变量：`[&]` 表示通过引用捕获引用的所有变量，而 `[=]` 表示通过值捕获它们。 可以使用默认捕获模式，然后为特定变量显式指定相反的模式。 例如，如果 lambda 体通过引用访问外部变量 `total` 并通过值访问外部变量 `factor`，则以下 capture 子句等效：
![[Pasted image 20241221141811.png]]
使用默认捕获时，只有 Lambda 体中提及的变量才会被捕获。

如果 capture 子句包含默认捕获 `&`，则该 capture 子句的捕获中没有任何标识符可采用 `&identifier` 形式。 同样，如果 capture 子句包含默认捕获 `=`，则该 capture 子句没有任何捕获可采用 `=identifier` 形式。 标识符或 **`this`** 在 capture 子句中出现的次数不能超过一次。 以下代码片段给出了一些示例：
![[Pasted image 20241221141927.png]]
在 C++ 中，**lambda 表达式**是一种允许你在代码中定义匿名函数（没有名字的函数）的机制。它在某些地方非常有用，比如作为 `std::sort` 这种函数的比较函数，或者用作某些算法中的回调函数。lambda 表达式在 C++11 引入，并且其语法非常简洁。下面我将通过具体的例子和解释来帮助你理解 lambda 表达式的工作原理。

### Lambda 表达式的基本语法

```cpp
[捕获列表](参数列表) -> 返回类型 { 函数体 }
```

- **捕获列表 (`[]`)**：指定 lambda 表达式能访问的外部变量（包括全局变量、局部变量等）。通过捕获列表，lambda 可以访问在其作用域外部定义的变量。
- **参数列表 (`(参数列表)`)**：和普通函数一样，lambda 也可以接受参数，形式类似于函数的参数列表。
- **返回类型 (`-> 返回类型`)**：这是一个可选部分，指定 lambda 表达式的返回类型。如果 C++ 能够推断返回类型，可以省略这个部分。
- **函数体 (`{ 函数体 }`)**：lambda 表达式的实际实现部分。

### 例子：简单的 Lambda 表达式

#### 1. **没有捕获变量的简单 lambda**

```cpp
auto add = [](int a, int b) -> int {
    return a + b;
};
std::cout << add(3, 5);  // 输出 8
```

- 这个例子定义了一个匿名函数 `add`，它接受两个整数参数 `a` 和 `b`，返回它们的和。`-> int` 显式指定了返回类型是 `int`。如果你省略 `-> int`，编译器会自动推断返回类型。

#### 2. **带有捕获变量的 lambda**

```cpp
int x = 10;
auto multiply = [x](int a) -> int {
    return a * x;
};
std::cout << multiply(5);  // 输出 50
```

- 这个例子中，lambda 捕获了外部变量 `x`。捕获的变量会被传递给 lambda，从而使得 lambda 可以在函数体内访问这些变量。`[x]` 是捕获列表，表示将外部的变量 `x` 捕获到 lambda 中。
- 需要注意的是，捕获的变量是 **按值捕获**（即 `x` 的拷贝），因此即使 `x` 在外部发生改变，lambda 中使用的依然是 `x` 的初始值。

#### 3. **按引用捕获**

```cpp
int x = 10;
auto multiply = [&x](int a) -> int {
    return a * x;
};
x = 20;
std::cout << multiply(5);  // 输出 100
```

- 通过 `[&x]` 捕获列表，lambda 会 **按引用** 捕获外部的变量 `x`。这意味着 lambda 表达式使用的是原始的 `x` 变量（而不是 `x` 的拷贝）。所以即使 `x` 在外部被修改，lambda 仍然能够访问修改后的值。

#### 4. **捕获所有变量**

```cpp
int x = 10, y = 20;
auto add = [=]() -> int {
    return x + y;
};
std::cout << add();  // 输出 30
```

- `[=]` 表示捕获所有外部变量，并且是按值捕获。这里，lambda 捕获了 `x` 和 `y`，并返回它们的和。即使 `x` 和 `y` 在 lambda 外部发生变化，lambda 仍然使用它们的初始值。

#### 5. **混合捕获方式**

```cpp
int x = 10, y = 20;
auto add = [x, &y]() -> int {
    return x + y;
};
x = 30;
y = 40;
std::cout << add();  // 输出 50
```

- `[x, &y]` 表示 **按值捕获 `x`** 和 **按引用捕获 `y`**。这样，lambda 使用 `x` 的拷贝（即 `x = 10`），但使用原始的 `y` 变量（即 `y` 发生变化后，lambda 仍然看到的是修改后的 `y`）。

### Lambda 表达式中的 `this` 指针

如果你在类的成员函数中使用 lambda 表达式，它会自动捕获 `this` 指针。这样，你可以在 lambda 中访问类的成员变量和成员函数。

```cpp
class MyClass {
public:
    int value = 10;
    void printValue() {
        auto lambda = [this]() {
            std::cout << value << std::endl;
        };
        lambda();
    }
};
```

- `this` 是隐式捕获的，允许 lambda 访问类的成员变量 `value`。

### 解释给定的 `std::sort` 使用的 Lambda 表达式

```cpp
sort(intervals.begin(), intervals.end(), [](const vector<int>& a, const vector<int>& b){
    if(a[1]==b[1]){
        return a[0]>b[0];
    }
    return a[1]<b[1];
});
```

这段代码是一个 **排序函数**，它使用了一个 **lambda 表达式** 来比较区间（`a` 和 `b`）。我们可以逐步分析其结构：

- `[](const vector<int>& a, const vector<int>& b)`：
    
    - 这是一个没有捕获的 lambda（因为捕获列表是空的 `[]`），接受两个参数 `a` 和 `b`，它们是 `vector<int>` 类型的引用，表示区间。
- `if(a[1] == b[1]){ return a[0] > b[0]; }`：
    
    - 如果两个区间的 **结束位置** 相同（即 `a[1] == b[1]`），则按 **起始位置** 降序排序（`a[0] > b[0]`）。
- `return a[1] < b[1];`：
    
    - 如果两个区间的 **结束位置** 不同，则按 **结束位置** 升序排序（`a[1] < b[1]`）。

### 小结

- **捕获列表**：指定 lambda 表达式可以访问哪些外部变量。可以按值或按引用捕获。
- **参数列表**：指定 lambda 接受的参数类型。
- **返回类型**：指定 lambda 的返回类型（可选，编译器可以自动推断）。
- **函数体**：执行具体的操作或逻辑。

