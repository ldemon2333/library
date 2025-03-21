`grep` 是一个强大的文本搜索工具，常用于在文件或输入流中查找符合特定模式的行。以下是一些常见用法：

---

## 1. **基本用法**

```sh
grep "pattern" file
```

在 `file` 文件中查找包含 `"pattern"` 的行。

---

## 2. **忽略大小写 (`-i`)**

```sh
grep -i "pattern" file
```

匹配时忽略大小写，比如 `hello` 和 `HELLO` 都会被匹配到。

---

## 3. **只显示匹配的内容 (`-o`)**

```sh
grep -o "pattern" file
```

只输出匹配的内容，而不是整行。

---

## 4. **显示行号 (`-n`)**

```sh
grep -n "pattern" file
```

输出匹配行的行号，格式如：

```
12: this is a pattern
```

表示匹配的内容在第 12 行。

---

## 5. **反向匹配 (`-v`)**

```sh
grep -v "pattern" file
```

显示不包含 `"pattern"` 的行。

---

## 6. **显示匹配前后几行 (`-A` / `-B` / `-C`)**

```sh
grep -A 3 "pattern" file  # 显示匹配行及其后 3 行
grep -B 3 "pattern" file  # 显示匹配行及其前 3 行
grep -C 3 "pattern" file  # 显示匹配行及其前后 3 行
```

---

## 7. **递归搜索 (`-r` 或 `-R`)**

```sh
grep -r "pattern" directory/
grep -R "pattern" directory/
```

在 `directory/` 目录及其子目录中递归搜索 `pattern`。

---

## 8. **匹配整个单词 (`-w`)**

```sh
grep -w "word" file
```

只匹配完整的单词 `word`，不会匹配 `word123` 或 `123word`。

---

## 9. **只显示文件名 (`-l`)**

```sh
grep -l "pattern" *
```

只显示包含 `pattern` 的文件名，不显示具体内容。

---

## 10. **显示不匹配的文件名 (`-L`)**

```sh
grep -L "pattern" *
```

列出 **不包含** `pattern` 的文件名。

---

## 11. **使用正则表达式 (`-E` / `-P`)**

```sh
grep -E "pattern1|pattern2" file  # 支持扩展正则 (egrep)
grep -P "\d+" file  # 支持 Perl 兼容正则 (PCRE)
```

---

## 12. **统计匹配的行数 (`-c`)**

```sh
grep -c "pattern" file
```

只输出匹配的行数。

---

## 13. **匹配多个关键字 (`-e`)**

```sh
grep -e "pattern1" -e "pattern2" file
```

匹配 `pattern1` 或 `pattern2`。

---

## 14. **搜索二进制文件 (`--binary-files=text`)**

```sh
grep --binary-files=text "pattern" binary_file
```

防止 `grep` 在遇到二进制文件时输出 `Binary file matches`。

---

## 15. **查找整个系统中的文件**

```sh
grep -rnw '/' -e "pattern" --color=auto 2>/dev/null
```

- `-r` 递归搜索
- `-n` 显示行号
- `-w` 只匹配完整单词
- `--color=auto` 高亮显示匹配部分
- `2>/dev/null` 忽略权限不足的错误

---

## 16. **结合 `find` 使用**

```sh
find . -type f -name "*.txt" -exec grep "pattern" {} +
```

在当前目录及子目录中查找 `.txt` 文件，并在这些文件中搜索 `pattern`。

---

## 17. **与 `awk` / `sed` 结合**

```sh
grep "pattern" file | awk '{print $1}'
grep "pattern" file | sed 's/pattern/replacement/g'
```

可以用 `awk` 提取列，或用 `sed` 进行替换。

---

## 18. **在 `ps` 进程列表中过滤**

```sh
ps aux | grep "process_name"
```

查找运行中的 `process_name` 进程。

如果要排除 `grep` 本身：

```sh
ps aux | grep "[p]rocess_name"
```

或者：

```sh
pgrep -f "process_name"
```

---

## 19. **颜色高亮显示 (`--color=auto`)**

```sh
grep --color=auto "pattern" file
```

高亮匹配部分，方便查看。

---

这些 `grep` 命令涵盖了常见的使用场景，你可以根据需求组合不同的选项来提高搜索效率！🚀