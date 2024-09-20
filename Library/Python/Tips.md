1. **`s.strip()`**：  
   `strip()` 方法用于去除字符串 `s` 两端的空白字符（包括空格、换行符等）。它返回一个去除了首尾空白的新字符串。

2. **`.split(' ')`**：  
   `split(' ')` 方法将字符串按空格分割成一个列表，每个元素都是一个单词或一个空字符串（如果有连续的空格）。  

综上所述，**`len(s.strip().split(' ')[-1])`** 的作用是：
- 去除字符串首尾的空白字符，
- 按照空格将字符串分割成单词列表，
- 取列表中最后一个单词，
- 计算并返回这个单词的长度。

### 示例：
```python
s = "   Hello world!   "
result = len(s.strip().split(' ')[-1])
print(result)  # 输出 6
```

在这个例子中，`"   Hello world!   "` 去除首尾空白后变为 `"Hello world!"`，分割后得到列表 `['Hello', 'world!']`，最后一个单词是 `"world!"`，其长度为 6。


---

`join()` 是一个字符串方法，用于将一个可迭代对象（如列表、元组等）中的元素连接成一个字符串，元素之间由指定的分隔符分隔。

### 语法：
```python
separator.join(iterable)
```

- **`separator`**：用于分隔元素的字符串。这是你希望放在每个元素之间的字符（比如逗号、空格、连字符等）。
- **`iterable`**：要连接的元素序列（如列表、元组、字符串等）。序列中的每个元素必须是字符串。

### 示例用法：

1. **使用空格连接列表中的单词**：
   ```python
   words = ["Hello", "world", "from", "Python"]
   result = " ".join(words)
   print(result)  # 输出: "Hello world from Python"
   ```

2. **使用逗号连接列表中的元素**：
   ```python
   items = ["apple", "banana", "cherry"]
   result = ", ".join(items)
   print(result)  # 输出: "apple, banana, cherry"
   ```

3. **连接字符串中的字符**：
   ```python
   s = "ABC"
   result = "-".join(s)
   print(result)  # 输出: "A-B-C"
   ```

### 注意：
- `join()` 只能连接字符串类型的元素，如果 `iterable` 中包含非字符串类型的元素，会引发 `TypeError`。如果你要连接不同类型的元素，需要先将它们转换为字符串：
  ```python
  mixed = ["The number is", 3]
  result = " ".join(map(str, mixed))
  print(result)  # 输出: "The number is 3"
  ```

`join()` 通常用于将列表中的字符串连接成一整个字符串，比如生成一个句子，或是将文件路径中的各部分合并。

---

`reversed()` 是一个内置函数，用于反转一个序列（如字符串、列表、元组等），返回一个反转后的迭代器。

### 语法：
```python
reversed(sequence)
```

- **`sequence`**：一个可以被反转的序列类型，例如字符串、列表、元组，或者其他实现了 `__reversed__()` 或 `__getitem__()` 方法的对象。

### 返回值：
`reversed()` 返回一个迭代器，该迭代器可以用于遍历反转后的序列。

### 示例用法：

1. **反转字符串**：
   ```python
   s = "Hello"
   reversed_s = reversed(s)
   result = ''.join(reversed_s)
   print(result)  # 输出: "olleH"
   ```

2. **反转列表**：
   ```python
   numbers = [1, 2, 3, 4, 5]
   reversed_numbers = list(reversed(numbers))
   print(reversed_numbers)  # 输出: [5, 4, 3, 2, 1]
   ```

3. **反转元组**：
   ```python
   my_tuple = (10, 20, 30)
   reversed_tuple = tuple(reversed(my_tuple))
   print(reversed_tuple)  # 输出: (30, 20, 10)
   ```

4. **遍历反转后的序列**：
   ```python
   for char in reversed("Python"):
       print(char, end=" ")  # 输出: n o h t y P
   ```

### 注意：
- `reversed()` 返回的是一个迭代器，不是一个序列类型（如列表或字符串）。如果你需要一个具体的序列，可以将其转换为 `list`、`tuple` 或使用 `''.join()` 将其转换为字符串。
- 反转操作不会修改原序列本身，而是返回一个新的迭代器。

### 示例：直接使用反转的迭代器
```python
# 不需要转换为列表或字符串，直接使用反转后的迭代器
numbers = [1, 2, 3]
for num in reversed(numbers):
    print(num, end=" ")  # 输出: 3 2 1
```

`reversed()` 是一个常用的工具函数。

`list` 类型有一个 `reverse()` 方法，用于就地反转列表中的元素顺序。与 `reversed()` 不同，`reverse()` 是一个原地操作，它直接修改列表，而不会返回新的列表或迭代器。

### 语法：
```python
list.reverse()
```

- **`list`**：这是你要反转的列表。

### 返回值：
`reverse()` 方法没有返回值。它直接修改原始列表，使其元素顺序倒置。

### 示例用法：

1. **反转列表**：
   ```python
   numbers = [1, 2, 3, 4, 5]
   numbers.reverse()
   print(numbers)  # 输出: [5, 4, 3, 2, 1]
   ```

2. **在原列表上操作**：
   ```python
   fruits = ["apple", "banana", "cherry"]
   fruits.reverse()
   print(fruits)  # 输出: ["cherry", "banana", "apple"]
   ```

### 注意：
- `reverse()` 直接修改原列表，因此它是一个就地操作（in-place operation）。
- `reverse()` 方法不适用于不可变序列（如元组、字符串）。对于不可变序列，可以使用 `reversed()` 来获取反转后的迭代器。

### 示例：与 `reversed()` 的对比
```python
# 使用 reverse() 就地反转
lst = [1, 2, 3]
lst.reverse()
print(lst)  # 输出: [3, 2, 1]

# 使用 reversed() 生成反转后的迭代器，不修改原列表
lst = [1, 2, 3]
reversed_lst = list(reversed(lst))
print(reversed_lst)  # 输出: [3, 2, 1]
print(lst)           # 输出: [1, 2, 3]
```

在需要修改原列表的情况下，`list.reverse()` 是一个简洁有效的方法。如果你需要保留原列表，并生成一个新的反转序列，可以使用 `reversed()`。

---
**结论：split()的时候，多个空格当成一个空格；split(' ')的时候，多个空格都要分割，每个空格分割出来空。**

---
- `s`: 输入字符串。
- `ch.lower()`: 将字符 `ch` 转换为小写。
- `ch.isalnum()`: 检查字符 `ch` 是否是字母或数字。

下面是这个代码片段的具体用法示例：

```python
s = "Hello, World! 123"
sgood = "".join(ch.lower() for ch in s if ch.isalnum())
print(sgood)  # 输出: "helloworld123"
```

在这个例子中，原始字符串 `"Hello, World! 123"` 被转换为 `"helloworld123"`，所有非字母数字字符（如逗号和空格）都被去掉了。

