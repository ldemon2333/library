在 C++ 中，如果你想动态分配内存并为指针变量创建一个对象，可以使用 `new` 运算符。`new` 运算符会在堆（heap）上分配内存，并返回该内存块的地址。

### 为基本类型创建指针变量：

如果你想为一个基本类型（如 `int`、`float` 等）创建指针变量，可以这样做：

```cpp
int* p = new int;  // 为一个整数分配内存
*p = 10;  // 设置值为 10
```

这里：

- `new int` 会在堆上分配一个 `int` 类型的内存空间，返回这个内存空间的地址，并赋值给指针 `p`。
- 然后你可以通过 `*p` 来访问这个内存空间，给它赋值（例如这里赋值为 `10`）。

### 为结构体或类创建指针变量：

如果你想为一个结构体（如 `ListNode`）或类对象创建指针变量，可以按如下方式操作：

假设你有一个 `ListNode` 结构体：

```cpp
struct ListNode {
    int val;
    ListNode* next;
    ListNode(int x) : val(x), next(nullptr) {}
};
```

然后你可以使用 `new` 来为它创建一个指针：

```cpp
ListNode* node = new ListNode(5);  // 动态分配一个 ListNode，值为 5
```

这里：

- `new ListNode(5)` 会在堆上分配一个 `ListNode` 类型的内存，并通过构造函数将 `val` 设置为 5，`next` 默认为 `nullptr`。

### 为数组创建指针变量：

如果你需要创建一个数组，并且数组的大小在运行时确定，可以使用 `new[]` 运算符：

```cpp
int* arr = new int[10];  // 创建一个包含 10 个整数的数组
arr[0] = 1;  // 设置第一个元素为 1
```

这里：

- `new int[10]` 会在堆上分配一个包含 10 个 `int` 元素的数组。

### 释放动态分配的内存：

在 C++ 中，使用 `new` 分配的内存需要手动释放，避免内存泄漏。使用 `delete` 来释放单个对象，使用 `delete[]` 来释放数组：

```cpp
delete p;  // 释放单个对象
delete[] arr;  // 释放数组
```

例如：

```cpp
delete node;  // 释放 ListNode 对象
```

### 完整例子：

```cpp
#include <iostream>
using namespace std;

struct ListNode {
    int val;
    ListNode* next;
    ListNode(int x) : val(x), next(nullptr) {}
};

int main() {
    // 使用 new 创建 ListNode 对象
    ListNode* node = new ListNode(10);
    cout << "Node value: " << node->val << endl;

    // 释放内存
    delete node;

    return 0;
}
```

### 总结：

- `new` 用来动态分配内存，返回的是该内存块的指针。
- 使用 `delete` 或 `delete[]` 释放内存，防止内存泄漏。