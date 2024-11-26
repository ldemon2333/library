以下是一个使用 Python 的 `argparse` 模块的示例代码，展示了如何解析命令行参数。

---

### 示例：基本用法

```python
import argparse

# 创建 ArgumentParser 对象
parser = argparse.ArgumentParser(description="一个 argparse 示例程序")

# 添加参数
parser.add_argument('-n', '--name', type=str, help='输入你的名字', required=True)
parser.add_argument('-a', '--age', type=int, help='输入你的年龄')
parser.add_argument('-v', '--verbose', action='store_true', help='输出详细信息')

# 解析参数
args = parser.parse_args()

# 使用参数
if args.verbose:
    print("详细模式已开启")

if args.name:
    print(f"你好, {args.name}!")

if args.age is not None:
    print(f"你的年龄是 {args.age} 岁")
```

---

### 如何运行代码

假设文件名是 `example.py`，可以通过命令行运行：

#### 运行时指定参数

```bash
python example.py --name Alice --age 25 --verbose
```

输出：

```plaintext
详细模式已开启
你好, Alice!
你的年龄是 25 岁
```

#### 忘记必需参数

如果 `--name` 是必需参数而未指定，运行时会报错：

```bash
python example.py --age 25 --verbose
```

错误提示：

```plaintext
usage: example.py [-h] -n NAME [-a AGE] [-v]
example.py: error: the following arguments are required: -n/--name
```

---

### 高级用法

#### 1. **默认值**

为参数设置默认值：

```python
parser.add_argument('-c', '--color', type=str, default='blue', help='选择颜色（默认是蓝色）')
```

未指定时，`args.color` 的值为 `'blue'`。

---

#### 2. **位置参数**

位置参数没有 `-` 或 `--` 前缀：

```python
parser.add_argument('filename', type=str, help='输入文件名')
```

调用时：

```bash
python example.py myfile.txt
```

`args.filename` 会是 `myfile.txt`。

---

#### 3. **选择固定值**

限制参数只能是某些固定值：

```python
parser.add_argument('-m', '--mode', type=str, choices=['read', 'write'], help='选择模式：read 或 write')
```

调用时：

```bash
python example.py --mode read
```

---

#### 4. **互斥组**

确保某些参数互斥（只能提供其中之一）：

```python
group = parser.add_mutually_exclusive_group()
group.add_argument('--debug', action='store_true', help='开启调试模式')
group.add_argument('--quiet', action='store_true', help='开启安静模式')
```

调用时：

```bash
python example.py --debug
python example.py --quiet
```

但不能同时指定：

```bash
python example.py --debug --quiet
```

---

### 完整示例：支持多种参数类型

```python
import argparse

def main():
    parser = argparse.ArgumentParser(description="一个综合 argparse 示例")
    parser.add_argument('input', type=str, help='输入文件')
    parser.add_argument('output', type=str, help='输出文件')
    parser.add_argument('--lines', type=int, default=10, help='要读取的行数 (默认 10)')
    parser.add_argument('--verbose', action='store_true', help='详细输出模式')
    parser.add_argument('--overwrite', action='store_true', help='是否覆盖现有文件')

    args = parser.parse_args()

    print(f"输入文件: {args.input}")
    print(f"输出文件: {args.output}")
    print(f"行数: {args.lines}")
    if args.verbose:
        print("详细模式已启用")
    if args.overwrite:
        print("将覆盖现有文件")

if __name__ == "__main__":
    main()
```

运行：

```bash
python example.py input.txt output.txt --lines 5 --verbose
```

输出：

```plaintext
输入文件: input.txt
输出文件: output.txt
行数: 5
详细模式已启用
```