`gcc -c`命令用于编译源代码文件（如`.c`文件）为目标文件（`.o`文件），但不会进行链接。具体来说：

- **编译**：将源代码转换为汇编代码。
- **汇编**：将汇编代码转换为机器代码，生成目标文件（`.o`）。

使用`-c`选项的好处是你可以分别编译多个源文件，然后在最后一步进行链接，以生成可执行文件。例如：

```bash
gcc -c file1.c
gcc -c file2.c
gcc -o myprogram file1.o file2.o
```
