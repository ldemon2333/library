在 C++ 中，容器（Container）是一个数据结构的集合，用于存储、管理和操作一组元素。容器的作用是提供一种通用的方式来存储和操作数据，同时封装了底层数据结构的实现细节。容器是 C++ 标准库的一部分，属于 C++ 标准模板库（STL，Standard Template Library）。

### C++ 容器的分类

C++ 标准库中的容器按照数据存储方式和操作方式的不同，可以分为两大类：

1. **顺序容器（Sequence Containers）**：元素按照特定的顺序存储，通常是线性结构。
2. **关联容器（Associative Containers）**：通过键值对来存储数据，支持高效的查找、插入和删除操作。
3. **无序容器（Unordered Containers）**：与关联容器类似，但它们不按照任何特定顺序存储元素，而是基于哈希表实现。

### 1. 顺序容器（Sequence Containers）

顺序容器存储元素的顺序和插入的顺序一致，常见的顺序容器有：

- **`vector`**：动态数组，支持快速随机访问，适合频繁在尾部插入和删除元素。`vector` 在内存中是连续存储的，因此支持高效的随机访问，但在中间插入或删除元素时效率较低。
    
    ```cpp
    std::vector<int> vec = {1, 2, 3};
    vec.push_back(4);  // 在尾部添加元素
    ```
    
- **`deque`**：双端队列，支持从两端进行高效的插入和删除操作。`deque` 在内存中不是连续存储的，因此随机访问的性能比 `vector` 差，但它在两端的操作更加高效。
    
    ```cpp
    std::deque<int> dq = {1, 2, 3};
    dq.push_front(0);  // 在头部添加元素
    dq.push_back(4);   // 在尾部添加元素
    ```
    
- **`list`**：双向链表，元素在内存中并不连续存储，每个元素包含指向前后元素的指针。`list` 支持在任意位置进行高效的插入和删除操作，但不支持随机访问。
    
    ```cpp
    std::list<int> lst = {1, 2, 3};
    lst.push_back(4);  // 在尾部添加元素
    lst.push_front(0); // 在头部添加元素
    ```
    
- **`array`**（C++11引入）：固定大小的数组容器，大小在编译时已确定。与 `std::vector` 类似，但不支持动态调整大小。
    
    ```cpp
    std::array<int, 3> arr = {1, 2, 3};
    ```
    

### 2. 关联容器（Associative Containers）

关联容器通过键值对存储元素，能够根据关键字高效查找元素。常见的关联容器有：

- **`set`**：集合，存储唯一的元素，并自动排序。`set` 根据元素的值进行排序，因此具有查找、插入、删除元素的对数时间复杂度。
    
    ```cpp
    std::set<int> s = {1, 2, 3};
    s.insert(4);  // 自动排序，元素唯一
    ```
    
- **`map`**：映射，存储键值对，键唯一并自动排序。`map` 中的键必须是唯一的，且会根据键自动排序。
    
    ```cpp
    std::map<int, std::string> m;
    m[1] = "one";  // 插入键值对
    m[2] = "two";
    ```
    
- **`multiset`**：与 `set` 类似，但允许存储重复元素。
    
    ```cpp
    std::multiset<int> ms = {1, 2, 3, 2};
    ```
    
- **`multimap`**：与 `map` 类似，但允许存储具有相同键的多个键值对。
    
    ```cpp
    std::multimap<int, std::string> mm;
    mm.insert({1, "one"});
    mm.insert({1, "uno"});
    ```
    

### 3. 无序容器（Unordered Containers）

无序容器和关联容器类似，但不对元素进行排序，而是基于哈希表实现。常见的无序容器有：

- **`unordered_set`**：无序集合，存储唯一的元素。元素没有排序，查询效率较高。
    
    ```cpp
    std::unordered_set<int> us = {1, 2, 3};
    ```
    
- **`unordered_map`**：无序映射，存储键值对。元素没有排序，查询效率较高。
    
    ```cpp
    std::unordered_map<int, std::string> um;
    um[1] = "one";
    um[2] = "two";
    ```
    
- **`unordered_multiset`**：与 `unordered_set` 类似，但允许存储重复元素。
    
    ```cpp
    std::unordered_multiset<int> ums = {1, 2, 3, 2};
    ```
    
- **`unordered_multimap`**：与 `unordered_map` 类似，但允许存储具有相同键的多个键值对。
    
    ```cpp
    std::unordered_multimap<int, std::string> umm;
    umm.insert({1, "one"});
    umm.insert({1, "uno"});
    ```
    

### 4. 适配器容器（Container Adapters）

适配器容器是对底层容器的包装，提供特定的接口，通常是队列或栈等常见的操作接口。适配器容器并不直接存储元素，而是依赖于其他容器。

- **`stack`**：栈，基于其他容器（如 `deque` 或 `vector`）实现，支持后进先出（LIFO）操作。
    
    ```cpp
    std::stack<int> s;
    s.push(1);
    s.push(2);
    s.pop();
    ```
    
- **`queue`**：队列，基于其他容器（如 `deque`）实现，支持先进先出（FIFO）操作。
    
    ```cpp
    std::queue<int> q;
    q.push(1);
    q.push(2);
    q.pop();
    ```
    
- **`priority_queue`**：优先队列，基于堆（通常是 `vector`）实现，支持按优先级出队。
    
    ```cpp
    std::priority_queue<int> pq;
    pq.push(10);
    pq.push(20);
    pq.push(5);
    ```
    

### 总结

C++ 容器是 STL 中的核心组成部分，它们封装了常见的数据结构，如数组、链表、哈希表、平衡树等，提供了丰富的接口来操作数据。根据使用场景，开发者可以选择适当的容器类型，平衡效率、内存消耗和功能需求。每个容器类型都有其适用的场景，例如 `vector` 适用于频繁访问元素，`map` 和 `set` 适用于高效查找，而 `deque` 和 `list` 适用于需要频繁在两端操作的情况。

`<stack>` 容器适配器提供了一个栈的接口，它基于其他容器（如 `deque` 或 `vector`）来实现。栈的元素是线性排列的，但只允许在一端（栈顶）进行添加和移除操作。

# stack
## 基本操作
- `push()`: 在栈顶添加一个元素。
- `pop()`: 移除栈顶元素。
- `top()`: 返回栈顶元素的引用，但不移除它。
- `empty()`: 检查栈是否为空。
- `size()`: 返回栈中元素的数量。

## 注意事项
- `<stack>` 不提供直接访问栈中元素的方法，只能通过 `top()` 访问栈顶元素。
- 尝试在空栈上调用 `top()` 或 `pop()` 将导致未定义行为。
- `<stack>` 的底层容器可以是任何支持随机访问迭代器的序列容器，如 `vector` 或 `deque`。

# unordered_set
在C++中，`<unordered_set>` 是标准模板库（STL）的一部分，提供了一种基于哈希表的容器，用于存储唯一的元素集合。

与 `set` 不同，`unordered_set` 不保证元素的排序，但通常提供更快的查找、插入和删除操作。

## 语法
- 构造函数：创建一个空的 unordered_set
- 插入：insert
- 查找：find
- 删除：erase
- 大小和空检查：使用size 和 empty
- 清空容器：clear


`unordered_set` 的 `count` 方法和 `find` 方法都用于检查元素是否存在，但是它们有一些区别：

### 1. **返回值**

- **`count`**：返回该元素在集合中出现的次数。对于 `unordered_set` 来说，由于每个元素都是唯一的，`count` 要么返回 `1`，表示元素存在；要么返回 `0`，表示元素不存在。
    
    ```cpp
    size_t count(const Key& key) const;
    ```
    
- **`find`**：返回一个迭代器，指向找到的元素。如果元素不存在，返回指向 `unordered_set` 末尾（`end()`）的迭代器。
    
    ```cpp
    iterator find(const Key& key);
    const_iterator find(const Key& key) const;
    ```
    

### 2. **用途**

- **`count`**：用于单纯检查元素是否存在，返回值是 `0` 或 `1`。
- **`find`**：用于获取元素的位置（如果存在的话），返回该元素的迭代器。如果元素不存在，返回 `end()`。

### 3. **效率**

- 对于 `unordered_set` 来说，`count` 和 `find` 都有 O(1) 的平均时间复杂度（假设哈希函数的性能良好），但 `find` 可以让你获取元素的迭代器，进而进行更多操作，而 `count` 只能告诉你元素是否存在。

### 示例代码：

```cpp
#include <iostream>
#include <unordered_set>

int main() {
    std::unordered_set<int> uset = {1, 2, 3, 4, 5};

    // 使用 count
    if (uset.count(3)) {
        std::cout << "3 is in the set." << std::endl;  // 输出 "3 is in the set."
    }

    // 使用 find
    auto it = uset.find(3);
    if (it != uset.end()) {
        std::cout << "3 found: " << *it << std::endl;  // 输出 "3 found: 3"
    }

    // 如果查找不存在的元素
    if (uset.find(6) == uset.end()) {
        std::cout << "6 is not in the set." << std::endl;  // 输出 "6 is not in the set."
    }

    return 0;
}
```

### 总结：

- 如果你只关心元素是否存在，`count` 可能更简单。
- 如果你需要获取元素的位置或进行其他操作（如修改元素），则应使用 `find`。


在 C++ 中，**迭代器**（Iterator）是一个用于访问容器（如 `vector`、`list`、`unordered_set` 等）元素的对象。迭代器提供了一种通用的方式来遍历容器中的元素，它是容器和算法之间的桥梁，允许你在不同容器之间统一地访问元素。

### 迭代器的作用

- **访问元素**：迭代器提供访问容器中元素的能力。
- **遍历容器**：可以通过迭代器逐一遍历容器中的每个元素。
- **与容器分离**：算法与容器分离，算法不需要关心具体的容器类型，只需要依赖迭代器。

### 迭代器的类型

C++ 标准库中的迭代器根据操作特性有不同的分类，常见的类型包括：

1. **输入迭代器（Input Iterator）**：只能向前读取元素。
2. **输出迭代器（Output Iterator）**：只能写入元素。
3. **前向迭代器（Forward Iterator）**：可以读取和写入元素，可以向前遍历。
4. **双向迭代器（Bidirectional Iterator）**：可以前后双向遍历。
5. **随机访问迭代器（Random Access Iterator）**：可以随机访问任何位置的元素（如 `vector` 或 `deque` 容器的迭代器）。

### 迭代器的常用操作

1. **解引用（`*`）**：通过迭代器访问指向的元素。
    
    ```cpp
    *it;  // 获取迭代器 it 指向的元素
    ```
    
2. **递增（`++`）**：将迭代器移动到容器中的下一个元素。
    
    ```cpp
    ++it;  // 将迭代器指向下一个元素
    ```
    
3. **递减（`--`）**：将迭代器移动到容器中的前一个元素（适用于双向迭代器和随机访问迭代器）。
    
    ```cpp
    --it;  // 将迭代器指向前一个元素
    ```
    

### 迭代器的使用示例

```cpp
#include <iostream>
#include <vector>

int main() {
    std::vector<int> vec = {10, 20, 30, 40, 50};

    // 使用迭代器遍历 vector
    std::vector<int>::iterator it = vec.begin();
    while (it != vec.end()) {
        std::cout << *it << " ";  // 解引用迭代器，访问元素
        ++it;  // 移动到下一个元素
    }
    std::cout << std::endl;

    return 0;
}
```

### 解释：

- `vec.begin()` 返回一个指向容器第一个元素的迭代器。
- `vec.end()` 返回一个指向容器最后一个元素之后位置的迭代器，表示容器的结束位置。
- 迭代器 `it` 用 `++it` 递增，访问每个元素并输出。

### 迭代器的类别

- **普通迭代器**：如上面的例子。
- **常量迭代器**：如果你不希望修改元素，可以使用常量迭代器，保证容器元素不被修改：
    
    ```cpp
    std::vector<int>::const_iterator cit = vec.begin();
    ```
    
- **反向迭代器**：可以从容器的末尾向前遍历：
    
    ```cpp
    std::vector<int>::reverse_iterator rit = vec.rbegin();
    ```
    

### 迭代器的优势

- **通用性**：迭代器提供统一的方式访问不同容器的数据，无论是 `vector`、`list`、`unordered_set` 等，迭代器操作方式一致。
- **抽象性**：不需要关心容器内部的实现细节，迭代器允许你在算法中专注于数据操作而不必考虑容器的底层结构。

### 总结：

迭代器是 C++ 中访问和操作容器元素的工具，它允许你以统一的方式遍历不同类型的容器。通过迭代器，C++ 提供了一种与容器类型无关的通用方法来操作数据。
