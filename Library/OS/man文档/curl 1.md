# Name
transfer a URL

# Description
curl is a tool to transfer data from or to a server, using one of the supported protocols. The command is designed to work without user interaction.

`curl` 是一个非常强大的命令行工具，常用于与服务器进行数据传输，支持多种协议（如 HTTP、HTTPS、FTP、SFTP 等）。它可以用于调试 API 请求、下载文件、上传数据等。以下是一些 `curl` 的常用用法。

### **2. 发送 HTTP 请求**

**发送 GET 请求（默认）**

```bash
curl http://example.com
```

**发送 GET 请求并显示响应头**

```bash
curl -i http://example.com
```

`-i` 选项表示显示响应头。

**发送 GET 请求并只显示响应头**

```bash
curl -I http://example.com
```

**发送带查询参数的 GET 请求**

```bash
curl "http://example.com/api?name=John&age=30"
```

### **3. 发送 POST 请求**

**发送 POST 请求并传递数据**

```bash
curl -X POST -d "name=John&age=30" http://example.com/api
```

在 `curl` 命令中，`-X` 是一个选项，用来指定 HTTP 请求的方法（也叫“动词”）。`curl` 默认使用 `GET` 请求方法，但如果你需要使用其他 HTTP 方法（如 `POST`、`PUT`、`DELETE` 等），就可以使用 `-X` 选项来指定。

### 基本语法：

```bash
curl -X <HTTP_METHOD> <URL>
```

- `<HTTP_METHOD>`: 要使用的 HTTP 请求方法，例如 `POST`、`PUT`、`DELETE` 等。
- `<URL>`: 请求的目标 URL 地址。

### 示例

#### 1. **使用 `-X` 指定 `POST` 方法：**

```bash
curl -X POST https://example.com/api/resource
```

这个命令将向 `https://example.com/api/resource` 发送一个 HTTP `POST` 请求。

#### 2. **使用 `-X` 指定 `PUT` 方法：**

```bash
curl -X PUT https://example.com/api/resource
```

这个命令将向 `https://example.com/api/resource` 发送一个 HTTP `PUT` 请求，通常用于更新资源。

#### 3. **使用 `-X` 指定 `DELETE` 方法：**

```bash
curl -X DELETE https://example.com/api/resource
```

这个命令将向 `https://example.com/api/resource` 发送一个 HTTP `DELETE` 请求，用于删除指定的资源。

### 为什么使用 `-X`？

- `curl` 默认使用 `GET` 请求，如果你需要发送其他类型的请求（如 `POST`、`PUT`、`DELETE` 等），可以通过 `-X` 来显式指定请求方法。
- 也可以配合其他选项（如 `-d` 发送数据、`-H` 添加请求头等）来进一步控制请求。

### 注意：

对于 `POST` 请求，`curl` 已经有 `-d` 选项来发送请求数据，因此在这种情况下，如果你使用 `-d`，并不需要显式使用 `-X POST`，因为 `-d` 会默认将请求方法设置为 `POST`。例如：

```bash
curl -d "name=value" https://example.com/api/resource
```

这个命令等价于：

```bash
curl -X POST -d "name=value" https://example.com/api/resource
```

但如果你想使用其他方法（如 `PUT` 或 `DELETE`），则需要使用 `-X` 来指定。


**发送 JSON 格式的 POST 请求**

```bash
curl -X POST -H "Content-Type: application/json" -d '{"name": "John", "age": 30}' http://example.com/api
```

`-H` 选项用于设置请求头，`-d` 用于设置请求数据。

### **4. 上传文件**

**通过 POST 上传文件**

```bash
curl -X POST -F "file=@/path/to/file" http://example.com/upload
```

`-F` 选项用于上传文件，`@` 后跟文件路径。

### **5. 下载文件**

**下载文件并保存到指定文件名**

```bash
curl -o filename.txt http://example.com/file.txt
```

**下载文件并显示下载进度**

```bash
curl -O http://example.com/file.txt
```

`-O` 会将文件保存为与 URL 中的文件名相同的文件。

### **6. 使用身份验证**

**Basic Authentication（基本认证）**

```bash
curl -u username:password http://example.com/protected
```

或者使用 `-H` 头部来传递认证信息：

```bash
curl -H "Authorization: Basic <base64_encoded_string>" http://example.com/protected
```

**Bearer Token Authentication**

```bash
curl -H "Authorization: Bearer <token>" http://example.com/api
```

### **7. 设置请求头**

**添加自定义请求头**

```bash
curl -H "User-Agent: MyApp/1.0" http://example.com
```

**发送多个请求头**

```bash
curl -H "Authorization: Bearer <token>" -H "Accept: application/json" http://example.com/api
```

### **8. 处理 Cookies**

**保存 Cookies 到文件**

```bash
curl -c cookies.txt http://example.com
```

**使用保存的 Cookies 文件**

```bash
curl -b cookies.txt http://example.com
```

### **9. 跟踪重定向**

**自动跟踪重定向**

```bash
curl -L http://example.com
```

`-L` 选项让 `curl` 跟踪服务器的重定向。

### **10. 设置超时**

**设置连接和响应的超时时间**

```bash
curl --max-time 30 http://example.com
```

这表示最大等待时间为 30 秒。

**设置连接超时**

```bash
curl --connect-timeout 10 http://example.com
```

### **11. 显示请求的详细信息**

**显示请求和响应的详细调试信息**

```bash
curl -v http://example.com
```

`-v`（verbose）选项可以显示更多的调试信息，包括请求头和响应头。

### **12. 使用代理服务器**

**通过 HTTP 代理发送请求**

```bash
curl -x http://proxyserver:8080 http://example.com
```

**通过 SOCKS5 代理发送请求**

```bash
curl --socks5-hostname socks5://proxyserver:1080 http://example.com
```

### **13. 发送 PUT 请求**

**发送 PUT 请求**

```bash
curl -X PUT -d "name=John&age=30" http://example.com/api
```

### **14. 获取 HTTP 状态码**

**获取 HTTP 响应的状态码**

```bash
curl -o /dev/null -s -w "%{http_code}\n" http://example.com
```

- `-o /dev/null`: 忽略输出内容。
- `-s`: 静默模式，不显示进度条。
- `-w "%{http_code}"`: 输出 HTTP 状态码。

### **15. 并行请求**

**使用 `-Z` 选项并发多个请求**

```bash
curl -Z "http://example.com/api1" -Z "http://example.com/api2"
```

### **16. 指定请求方法**

**使用 `-X` 选项指定请求方法**

```bash
curl -X PATCH http://example.com/api
```

通过 `-X` 可以显式指定请求方法，像 `PATCH`、`PUT`、`DELETE` 等。
