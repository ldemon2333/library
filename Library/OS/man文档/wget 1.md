`wget` 是一个用于从网络上下载文件的命令行工具，支持 HTTP、HTTPS 和 FTP 协议。它非常强大，可以在没有图形界面的环境下通过命令行自动化文件下载任务。

以下是一些常用的 `wget` 用法：

### 1. **下载文件**

最基本的用法就是下载一个文件：

```bash
wget http://example.com/file.txt
```

这将从 `http://example.com/file.txt` 下载文件，并将其保存在当前目录。

### 2. **指定文件名**

你可以指定下载文件保存的名称：

```bash
wget -O newfile.txt http://example.com/file.txt
```

这会把下载的文件保存为 `newfile.txt`。

### 3. **下载整个网站（递归下载）**

`-r` 选项用于递归下载整个网站或目录结构：

```bash
wget -r http://example.com/
```

这将下载 `http://example.com/` 网站的整个页面和资源，递归下载链接的文件。

### 4. **限制下载深度**

如果你不想下载网站的所有页面，可以使用 `-l` 选项限制递归的深度：

```bash
wget -r -l 2 http://example.com/
```

这将递归下载网页，但最多只下载 2 层深度的链接。

### 5. **继续未完成的下载**

如果你中途断开了下载，`wget` 支持继续未完成的下载。使用 `-c` 选项：

```bash
wget -c http://example.com/file.txt
```

这会从上次中断的地方继续下载。

### 6. **下载多个文件**

你可以将多个 URL 放入一个文件中，然后批量下载：

```bash
wget -i urls.txt
```

`urls.txt` 是包含多个下载地址的文件，每个 URL 占一行。

### 7. **后台下载**

使用 `-b` 选项可以让 `wget` 在后台运行：

```bash
wget -b http://example.com/file.txt
```

这将把下载任务放入后台并继续执行其他操作，下载的日志会保存在 `wget-log` 文件中。

### 8. **设置下载速度限制**

可以使用 `--limit-rate` 选项限制下载速度：

```bash
wget --limit-rate=200k http://example.com/file.txt
```

这将限制下载速度为每秒 200KB。

### 9. **下载时设置用户代理**

有时，网站可能会限制某些用户代理的请求，可以通过 `--user-agent` 选项来模拟浏览器的用户代理：

```bash
wget --user-agent="Mozilla/5.0" http://example.com/file.txt
```

### 10. **下载并忽略证书检查（HTTPS）**

如果你在使用 `https` 协议时遇到 SSL 证书验证问题，可以使用 `--no-check-certificate` 来忽略证书验证：

```bash
wget --no-check-certificate https://example.com/file.txt
```

### 11. **限制最大下载次数**

如果你希望限制下载文件的最大尝试次数，可以使用 `--tries` 选项：

```bash
wget --tries=3 http://example.com/file.txt
```

这将最多尝试 3 次下载，若失败，则停止。

### 12. **下载带有密码的文件**

如果网站要求身份验证，你可以使用 `--user` 和 `--password` 选项：

```bash
wget --user=username --password=password http://example.com/file.txt
```

### 13. **限制下载文件的类型**

你可以使用 `-A` 或 `-R` 来限制下载的文件类型：

- **`-A`**：只下载特定类型的文件
    
    ```bash
    wget -r -A "*.jpg,*.png" http://example.com/
    ```
    
    这将递归下载 `http://example.com/` 上的所有图片文件（`.jpg` 和 `.png`）。
    
- **`-R`**：排除某些文件类型
    
    ```bash
    wget -r -R "*.mp4" http://example.com/
    ```
    
    这将递归下载所有文件，但排除 `.mp4` 文件。
    

### 14. **限制并发下载**

如果你想通过 `wget` 限制并发下载的数量，可以使用 `--wait` 和 `--random-wait` 来增加延迟，避免对服务器造成压力：

```bash
wget --wait=2 --random-wait -r http://example.com/
```

这会让 `wget` 在每次下载前等待 2 秒，并在此基础上加入随机延迟。

### 15. **保存所有文件的日志**

你可以将下载过程的日志输出到一个文件中：

```bash
wget -o download.log http://example.com/file.txt
```

这将把下载的所有信息保存到 `download.log` 文件中。

---

### 总结：

`wget` 是一个功能非常强大的下载工具，适用于批量下载、网站镜像、断点续传等任务。常用的选项包括：

- `-O`：指定保存文件的名称。
- `-r`：递归下载整个网站。
- `-c`：继续下载中断的文件。
- `-b`：后台下载。
- `--limit-rate`：限制下载速度。
- `--user-agent`：设置用户代理。

你可以根据不同的需求灵活使用这些选项来进行高效下载。