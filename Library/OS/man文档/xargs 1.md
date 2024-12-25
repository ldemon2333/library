给其他命令传递参数的一个过滤器

**xargs 命令** 是给其他命令传递参数的一个过滤器，也是组合多个命令的一个工具。它擅长将标准输入数据转换成命令行参数，xargs 能够处理管道或者 stdin 并将其转换成特定命令的命令参数。xargs 也可以将单行或多行文本输入转换为其他格式，例如多行变单行，单行变多行。xargs 的默认命令是 echo，空格是默认定界符。这意味着通过管道传递给 xargs 的输入将会包含换行和空白，不过通过 xargs 的处理，==换行和空白将被空格取代。==xargs 是构建单行命令的重要组件之一。

# xargs 命令用法
xargs 用作替换工具，读取输入数据重新格式化后输出。

定义一个测试文件，内有多行文本数据：
```
cat test.txt

a b c d e f g
h i j k l m n
o p q
r s t
u v w x y z

```
多行输入单行输出：
```
cat test.txt | xargs

a b c d e f g h i j k l m n o p q r s t u v w x y z

```

# 使用 -n 进行多行输出
-n 选项 多行输出：
```
cat test.txt | xargs -n3

a b c
d e f
g h i
j k l
m n o
p q r
s t u
v w x
y z

```
# 使用 -d 分割输入
-d 选项 可以自定义一个定界符：
```
echo "nameXnameXnameXname" | xargs -dX

name name name name

```
```
echo "nameXnameXnameXname" | xargs -dX -n2

name name
name name

```

# 结合 -I 选项
xargs 的一个 **选项 -I** ，使用 -I 指定一个替换字符串{}，这个字符串在 xargs 扩展时会被替换掉，当 -I 与 xargs 结合使用，每一个参数命令都会被执行一次：
```
cat arg.txt | xargs -I {} ./sk.sh -p {} -l

-p aaa -l
-p bbb -l
-p ccc -l

```

复制所有图片文件到 /data/images 目录下：
```
ls *.jpg | xargs -n1 -I{} cp {} /data/images
```


`xargs` 是一个非常有用的命令行工具，通常用来将输入转换为命令行参数，并传递给其他命令。它主要用于处理输入流，特别是在管道中使用时，可以避免一次性传递过多参数。

### 基本用法

```bash
echo "a b c" | xargs echo
```

这个命令会把 `"a b c"` 转换为 `echo a b c`，并执行它。

### 常见选项

1. **`-n`**  
    `-n` 选项指定每次传递给命令的参数个数。例如：
    
    ```bash
    echo "a b c d e" | xargs -n 2 echo
    ```
    
    输出为：
    
    ```
    a b
    c d
    e
    ```
    
2. **`-I`**  
    使用 `-I` 可以指定占位符来替换输入流中的每一项。例如：
    
    ```bash
    echo "a b c" | xargs -I {} echo "Item: {}"
    ```
    
    输出为：
    
    ```
    Item: a
    Item: b
    Item: c
    ```
    
3. **`-d`**  
    `-d` 用于指定输入的分隔符。例如：
    
    ```bash
    echo "a,b,c" | xargs -d, echo
    ```
    
    输出为：
    
    ```
    a b c
    ```
    
4. **`-p`**  
    在每次执行命令之前，提示用户确认。例如：
    
    ```bash
    echo "a b c" | xargs -p echo
    ```
    
    会在执行每次 `echo` 前提示用户是否执行。
    
5. **`-0`**  
    使用 `-0` 选项时，`xargs` 会将输入分隔符设为 null 字符（`\0`）。这通常与 `find` 命令的 `-print0` 选项一起使用，用于处理文件名中包含空格或特殊字符的情况。
    
    ```bash
    find . -type f -print0 | xargs -0 rm
    ```
    
    这样可以安全地删除包含空格或特殊字符的文件。
    
6. **`-L`**  
    `-L` 选项指定每次传递多少行输入。例如：
    
    ```bash
    echo -e "line1\nline2\nline3" | xargs -L 1 echo
    ```
    
    输出为：
    
    ```
    line1
    line2
    line3
    ```
    
7. **`-t`**  
    `-t` 会在执行命令前打印出命令。例如：
    
    ```bash
    echo "a b c" | xargs -t echo
    ```
    
    输出为：
    
    ```
    echo a b c
    a b c
    ```
    

### 示例

#### 示例 1：批量删除文件

```bash
find . -type f -name "*.log" | xargs rm
```

这个命令会找到当前目录及其子目录下的所有 `.log` 文件，并删除它们。

#### 示例 2：为每个文件添加后缀

```bash
ls *.txt | xargs -I {} mv {} {}.bak
```

这个命令会将所有 `.txt` 文件重命名为 `.txt.bak`。

### 总结

`xargs` 是处理大量命令行参数的一个非常强大的工具，特别是在管道和文件操作中非常常用。通过合理使用选项，可以控制如何传递和处理输入数据。