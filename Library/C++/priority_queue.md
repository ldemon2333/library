在 C++ 中，`<priority_queue>` 是标准模板库（STL）的一部分，用于实现优先队列。

优先队列是一种特殊的队列，它允许我们快速访问队列中具有最高（或最低）优先级的元素。

在 C++ 中，`priority_queue` 默认是一个最大堆，这意味着队列的顶部元素总是具有最大的值。

`priority_queue` 是一个容器适配器，它提供了对底层容器的堆操作。它不提供迭代器，也不支持随机访问。

# 语法
```
#include <queue>

// 声明一个整型优先队列
priority_queue<int> pq;

// 声明一个自定义类型的优先队列，需要提供比较函数
struct compare {
    bool operator()(int a, int b) {
        return a > b; // 这里定义了最小堆
    }
};
priority_queue<int, vector<int>, compare> pq_min;
```

# 常用操作
- `empty()`: 检查队列是否为空。
- `size()`: 返回队列中的元素数量。
- `top()`: 返回队列顶部的元素（不删除它）。
- `push()`: 向队列添加一个元素。
- `pop()`: 移除队列顶部的元素。


这段代码中的 `struct compare` 和 **函数重载** 之间的关系在于，`operator()` 重载了 `()` 运算符，使得 `compare` 结构体能够被用作自定义的比较器。

### 函数重载与 `operator()` 解释：

1. **函数重载**： 函数重载是 C++ 中的一种特性，它允许多个函数同名，但参数不同。每个函数通过不同的参数类型或数量来区分。
    
2. **`operator()` 重载**： 在 C++ 中，可以重载许多运算符，包括 `()`（函数调用运算符）。这意味着你可以让对象像函数一样被调用。通过重载 `operator()`，你可以为一个类或结构体定义函数调用的行为。
    
    在你的代码中：
    
    ```cpp
    struct compare {
        bool operator()(int a, int b) {
            return a > b; // 这里定义了最小堆
        }
    };
    ```
    
    - **`operator()`** 被重载为一个接受两个 `int` 类型参数，并返回一个 `bool` 类型的函数。通过这种方式，我们使得 `compare` 结构体的实例能够像函数一样被调用，比较两个整数 `a` 和 `b`。
    - **最小堆的定义**：`a > b` 返回 `true` 表示 `a` 比 `b` 大，这在堆的实现中会将较小的元素优先放在堆顶，从而构成一个最小堆。

### 关系：函数重载和 `operator()` 的重载

1. **函数重载**：函数重载允许你定义多个同名的函数，使用不同的参数类型、顺序或数量来区分它们。
    
2. **`operator()` 的重载**：这是一种特定的函数重载，用于改变对象的函数调用行为。在你的例子中，`operator()` 重载允许 `compare` 对象作为函数调用，比较两个整数。
    

### 通过函数重载实现自定义比较器

在 C++ 中，我们可以使用函数重载来定义一个比较器，并将它传递给 `std::priority_queue` 等容器来定制元素的排序方式。具体来说，你的 `compare` 结构体提供了一个重载的 `operator()`，它定义了两个整数的比较规则。

例如，`std::priority_queue` 默认是最大堆，但通过使用自定义的比较器，你可以改变堆的行为，创建一个最小堆：

```cpp
#include <iostream>
#include <queue>
#include <vector>

struct compare {
    bool operator()(int a, int b) {
        return a > b; // 定义最小堆
    }
};

int main() {
    std::vector<int> data = {3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5};
    
    // 创建一个最小堆
    std::priority_queue<int, std::vector<int>, compare> pq;

    // 将数据插入堆中
    for (int num : data) {
        pq.push(num);
    }

    // 输出堆中的元素（从小到大）
    while (!pq.empty()) {
        std::cout << pq.top() << " ";
        pq.pop();
    }

    return 0;
}
```

### 总结：

- **函数重载** 是指允许同名的函数根据不同的参数类型、数量、顺序来区分。
- **`operator()` 重载** 是函数重载的一种特殊形式，用于重载运算符 `()`，使得对象可以像函数一样被调用。在你的例子中，通过重载 `operator()` 实现了自定义的比较规则，使得 `compare` 对象能够被用作最小堆的比较器。

这两个概念都涉及到多态和函数调用的定制，`operator()` 重载实际上是函数重载的一个具体应用。


重载operator<的例子：返回true时，说明左边形参的优先级低于右边形参


