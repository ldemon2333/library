`tar` 是 Linux 中常用的归档工具，主要用于打包和解压缩文件。以下是一些常见用法：

### 1. **创建归档（打包，不压缩）**

```bash
tar -cf archive.tar file1 file2 dir1
```

- `-c`（create）：创建归档文件
- `-f`（file）：指定归档文件名

### 2. **解包归档**

```bash
tar -xf archive.tar
```

- `-x`（extract）：解压归档文件
- `-f`（file）：指定归档文件名

### 3. **创建 gzip 压缩的归档**

```bash
tar -czf archive.tar.gz file1 file2 dir1
```

- `-z`：使用 gzip 压缩

### 4. **解压 gzip 归档**

```bash
tar -xzf archive.tar.gz
```

### 5. **创建 bzip2 压缩的归档**

```bash
tar -cjf archive.tar.bz2 file1 file2 dir1
```

- `-j`：使用 bzip2 压缩

### 6. **解压 bzip2 归档**

```bash
tar -xjf archive.tar.bz2
```

### 7. **创建 xz 压缩的归档**

```bash
tar -cJf archive.tar.xz file1 file2 dir1
```

- `-J`：使用 xz 压缩

### 8. **解压 xz 归档**

```bash
tar -xJf archive.tar.xz
```

### 9. **查看归档内容**

```bash
tar -tf archive.tar
```

- `-t`（list）：列出归档文件的内容

### 10. **提取单个文件**

```bash
tar -xf archive.tar file1
```

### 11. **添加文件到已存在的归档**

```bash
tar -rf archive.tar newfile
```

- `-r`：追加文件，仅适用于未压缩的 `.tar` 文件

### 12. **删除归档中的文件**

```bash
tar --delete -f archive.tar file1
```

- 仅适用于 `.tar`，不能用于 `.tar.gz` 等压缩文件

### 13. **解压到指定目录**

```bash
tar -xf archive.tar -C /path/to/destination
```

### 14. **使用 `pv` 显示进度**

```bash
tar -cf - large_dir | pv | tar -xf -
```

### Linux 打包文件夹的常用方法及示例

---

#### 一、**使用 `tar` 命令（推荐）**

`tar` 是 Linux 最常用的归档工具，支持打包和压缩，功能强大且灵活。

1. **基本打包（不压缩）**
```
tar -cvf 包名.tar 文件夹路径/
```
- **参数说明**：
    - `-c`：创建新归档文件；
    - `-v`：显示详细过程；
    - `-f`：指定输出文件名（必须放在最后）。


**打包并压缩**

- **gzip 压缩（生成 .tar.gz ）**：
```
tar -czvf backup.tar.gz 文件夹路径/
```
- `-z`：使用 gzip 压缩。
- `-j`：使用 bzip2 压缩。

```
scp [参数] 用户名@服务器IP:服务器文件路径 本地保存路径
```

```
scp root@192.168.1.100:/data/test.txt /home/user/Downloads/
```
