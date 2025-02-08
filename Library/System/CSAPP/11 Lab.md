# 11.10
`strchr` 函数是 C 标准库中的一个字符串处理函数，用于查找一个字符在字符串中的第一次出现。它在 `<string.h>` 头文件中定义。

### 函数原型

```c
char *strchr(const char *str, int c);
```

### 参数说明

- **`str`**：指向要搜索的 C 字符串（以 `\0` 结尾的字符数组）。
- **`c`**：要查找的字符。虽然参数类型是 `int`，但实际上它会被转换为 `unsigned char`，并且通常是一个单个字符（`char` 类型）。

### 返回值

- 如果在字符串 `str` 中找到字符 `c`，`strchr` 会返回指向字符第一次出现位置的指针。
- 如果没有找到字符 `c`，则返回 `NULL`。
- 如果 `c` 是字符串的终止字符 `\0`，则返回指向字符串末尾的指针。

### 示例

下面是一个使用 `strchr` 的例子，演示了如何查找字符在字符串中的位置：

```c
#include <stdio.h>
#include <string.h>

int main() {
    const char *str = "Hello, world!";
    char ch = 'o';

    // 查找字符 'o' 在字符串中的第一次出现
    char *result = strchr(str, ch);

    if (result != NULL) {
        printf("Found '%c' at position: %ld\n", ch, result - str);
    } else {
        printf("Character '%c' not found.\n", ch);
    }

    return 0;
}
```

### 运行结果

```
Found 'o' at position: 4
```

### 解释

- `strchr(str, ch)` 查找字符 `'o'` 在字符串 `"Hello, world!"` 中的第一次出现。返回的是一个指向字符 `'o'` 的指针。
- `result - str` 计算的是字符 `'o'` 在字符串中的索引位置，这里输出的是 `4`，表示 `'o'` 第一次出现的位置是字符串的第 4 个位置（从 0 开始计数）。

### 注意事项

1. **字符比较**：`strchr` 查找的是字符的“值”，而不是字符的“位置”，它会返回指向第一个匹配字符的指针。如果字符出现在多个地方，只会返回第一个匹配的位置。
    
2. **查找字符串结束符**：如果 `c` 是 `'\0'`，则 `strchr` 返回的是指向字符串结束符的指针，即 `str + strlen(str)`。
    
3. **区分大小写**：`strchr` 是区分大小写的，它只会找到完全匹配的字符。例如，如果要查找字符 `'a'`，它不会返回字符 `'A'` 的位置。
    

### `strchr` 示例：查找字符串结尾的 `\0` 字符

```c
#include <stdio.h>
#include <string.h>

int main() {
    const char *str = "Hello, world!";
    char *result = strchr(str, '\0'); // 查找字符串结束符

    if (result != NULL) {
        printf("Found '\\0' at position: %ld\n", result - str);
    } else {
        printf("'\\0' not found.\n");
    }

    return 0;
}
```

### 运行结果

```
Found '\0' at position: 13
```

### 总结

- `strchr` 用于查找指定字符在字符串中首次出现的位置。
- 返回值是一个指针，指向字符的位置（如果找到了），如果没找到，返回 `NULL`。
- 该函数可以查找任何字符，包括字符串的结束符 `'\0'`。
# 11.11 
扩展 TINY，以支持 HTTP HEAD 方法。使用 TELNET 作为 Web 客户端来验证你的工作。

`HEAD` 方法是 HTTP 协议中的一种请求方法，它与 `GET` 方法非常相似，但有一个关键的区别：`HEAD` 请求不会返回消息体，只返回响应头部。

### 用途：

`HEAD` 方法主要用于获取响应的元数据（例如，响应头），而不需要获取实际的响应内容（如 HTML 文本、图片等）。通常用于检查服务器上的资源是否存在、检查资源的修改时间或大小等。

### 特点：

- **无消息体**：`HEAD` 请求的响应仅包含状态行和头部信息，不包含消息体内容。
- **性能优化**：由于没有返回消息体，`HEAD` 请求的响应通常比 `GET` 请求更小，因此在某些情况下能提高性能（例如，用于检查文件是否存在，或查看文件的修改日期等）。

### 例子：

假设你想查看某个网页的响应头而不需要其内容，你可以使用 `HEAD` 请求。

```bash
curl -I https://www.example.com
```

输出的结果将是类似于：

```http
HTTP/1.1 200 OK
Date: Tue, 26 Dec 2024 12:00:00 GMT
Server: Apache
Last-Modified: Wed, 25 Dec 2024 10:00:00 GMT
Content-Type: text/html; charset=UTF-8
Content-Length: 1024
Connection: keep-alive
```

### 常见用例：

1. **检查资源是否存在**：客户端可以通过 `HEAD` 请求获取资源的元信息，如文件大小或最后修改时间，来判断资源是否发生变化。
    
2. **检查修改日期**：可以用来检查文件的 `Last-Modified` 头部，判断文件是否自上次访问以来已经被修改。
    
3. **检查文件类型**：通过查看 `Content-Type` 头部，客户端可以确认文件的 MIME 类型，而无需下载整个文件。
    


`strcasecmp` 是一个 C 标准库函数，用于比较两个字符串（不区分大小写）。它的函数原型如下：

```c
int strcasecmp(const char *str1, const char *str2);
```

### 返回值：

- 如果 `str1` 和 `str2` 相等（忽略大小写），返回 0。
- 如果 `str1` 在字典顺序上小于 `str2`（忽略大小写），返回负值。
- 如果 `str1` 在字典顺序上大于 `str2`（忽略大小写），返回正值。

### 示例代码：

```c
#include <stdio.h>
#include <string.h>

int main() {
    char str1[] = "hello";
    char str2[] = "HELLO";
    char str3[] = "world";

    // 使用 strcasecmp 比较两个字符串
    if (strcasecmp(str1, str2) == 0) {
        printf("str1 和 str2 是相等的（忽略大小写）。\n");
    } else {
        printf("str1 和 str2 不相等。\n");
    }

    // 比较两个不同的字符串
    if (strcasecmp(str1, str3) == 0) {
        printf("str1 和 str3 是相等的（忽略大小写）。\n");
    } else {
        printf("str1 和 str3 不相等。\n");
    }

    return 0;
}
```

### 输出：

```
str1 和 str2 是相等的（忽略大小写）。
str1 和 str3 不相等。
```

### 解释：

- `strcasecmp` 函数会忽略大小写来比较两个字符串。
- 在示例中，`str1` 和 `str2` 内容相同，但大小写不同，因此它们被认为是相等的。
- `str1` 和 `str3` 内容不同，所以它们被认为是不相等的。

### 使用场景：

- 在进行用户输入验证时，如果不关心输入的大小写差异，可以使用 `strcasecmp` 来比较字符串。
- 在处理不区分大小写的文本数据时，`strcasecmp` 可以有效避免因为大小写不同而导致的错误比较结果。


# 11.12 
以 HTTP POST 方式请求动态内容。

HTTP `POST` 方法是用于向服务器提交数据的请求方法。通常用来提交表单数据、上传文件或提交 JSON 数据等。与 `GET` 请求不同，`POST` 请求将数据放在请求体中，而不是 URL 参数中，这使得它可以处理更大的数据量和更复杂的数据格式。

### 1. **`POST` 请求的基本概念**

- **请求体（Request Body）**：`POST` 请求的核心是请求体，数据是通过请求体传递给服务器的，而不是通过 URL。
- **用途**：`POST` 请求通常用于以下几种情况：
    - 提交表单数据
    - 上传文件
    - 创建或更新资源
    - 发送 JSON 数据

### 2. **`POST` 请求的结构**

- **请求行**：包含请求方法（`POST`），请求的 URL 以及 HTTP 版本。
- **请求头部**：
    - `Content-Type`: 表示请求体的数据格式。常见的类型包括 `application/x-www-form-urlencoded`（表单数据）、`application/json`（JSON 数据）、`multipart/form-data`（上传文件）。
    - `Content-Length`: 请求体的大小。
    - 其他自定义的请求头信息（如 `Authorization`，`User-Agent` 等）。
- **请求体**：包含发送的数据内容，如表单字段、文件、JSON 数据等。

### 3. **`POST` 请求的常见用途**

#### a) 提交表单数据（`application/x-www-form-urlencoded`）

这是最常见的 `POST` 请求用途，表单数据以键值对的形式编码。

**HTML 表单示例：**

```html
<form action="https://www.example.com/submit" method="POST">
    <label for="name">Name:</label>
    <input type="text" id="name" name="name"><br><br>

    <label for="age">Age:</label>
    <input type="text" id="age" name="age"><br><br>

    <input type="submit" value="Submit">
</form>
```

在这种情况下，提交的表单数据会被编码为：

```
name=John&age=30
```

#### b) 发送 JSON 数据（`application/json`）

发送 JSON 数据通常用于与 Web API 交互，特别是 RESTful API。

**发送 JSON 数据的示例（使用 `fetch`）：**

```javascript
const data = {
    name: "John",
    age: 30
};

fetch("https://www.example.com/submit", {
    method: "POST",
    headers: {
        "Content-Type": "application/json"
    },
    body: JSON.stringify(data)  // 将 JavaScript 对象转为 JSON 字符串
})
.then(response => response.json())
.then(data => console.log("Success:", data))
.catch(error => console.error("Error:", error));
```

#### c) 上传文件（`multipart/form-data`）

`multipart/form-data` 是一个专门用于上传文件的编码类型。此方法允许表单同时包含文本字段和文件字段。

**HTML 表单上传文件的示例：**

```html
<form action="https://www.example.com/upload" method="POST" enctype="multipart/form-data">
    <label for="file">Choose a file:</label>
    <input type="file" id="file" name="file"><br><br>
    
    <input type="submit" value="Upload">
</form>
```

- `enctype="multipart/form-data"`: 这一属性是告诉浏览器表单数据包含文件上传。

### 4. **使用 `curl` 发送 `POST` 请求**

`curl` 是一个强大的命令行工具，用于与服务器进行交互，可以非常方便地发送 `POST` 请求。

**发送表单数据的 `POST` 请求：**

```bash
curl -X POST -d "name=John&age=30" https://www.example.com/submit
```

**发送 JSON 数据的 `POST` 请求：**

```bash
curl -X POST -H "Content-Type: application/json" -d '{"name": "John", "age": 30}' https://www.example.com/submit
```

### 5. **使用 Python `requests` 库发送 `POST` 请求**

Python 中的 `requests` 库是一个非常流行的库，用于发送 HTTP 请求。它支持发送 `POST` 请求，通常用于与 API 交互。

**发送表单数据的 `POST` 请求：**

```python
import requests

url = "https://www.example.com/submit"
data = {"name": "John", "age": 30}

response = requests.post(url, data=data)
print(response.text)
```

**发送 JSON 数据的 `POST` 请求：**

```python
import requests
import json

url = "https://www.example.com/submit"
data = {"name": "John", "age": 30}

response = requests.post(url, json=data)  # 发送 JSON 数据
print(response.text)
```

### 6. **使用 Node.js `axios` 发送 `POST` 请求**

在 Node.js 中，`axios` 是一个流行的库，适用于发送 HTTP 请求。

**发送 JSON 数据的 `POST` 请求：**

```javascript
const axios = require('axios');

const url = 'https://www.example.com/submit';
const data = {
    name: 'John',
    age: 30
};

axios.post(url, data)
    .then(response => {
        console.log('Response:', response.data);
    })
    .catch(error => {
        console.error('Error:', error);
    });
```

### 7. **`POST` 请求与 `GET` 请求的对比**

|特性|`POST` 请求|`GET` 请求|
|---|---|---|
|**数据发送位置**|请求体（Request Body）|URL 查询字符串（Query String）|
|**数据长度限制**|理论上没有限制|URL 长度限制，通常为 2048 字符左右|
|**安全性**|数据隐藏在请求体中，相对安全（但仍需加密）|数据附加在 URL 中，容易被窃取|
|**用途**|创建、更新资源，发送大量数据|获取数据，通常用于获取资源|
|**缓存**|不缓存数据（通常需要禁用缓存）|可以缓存，浏览器会缓存 `GET` 请求的结果|
|**幂等性**|非幂等：同样的请求可能导致不同的结果|幂等：相同的请求始终返回相同的结果|

### 8. **`POST` 请求的安全性**

- **加密**：虽然 `POST` 请求的数据在请求体中而非 URL 中传输，但依然可以被拦截。因此，建议通过 **HTTPS** 协议发送 `POST` 请求，以确保数据的机密性和完整性。
- **CSRF 攻击**：使用 `POST` 请求时，要小心跨站请求伪造（CSRF）攻击。为了防止此类攻击，通常需要使用 CSRF token。

### 总结

- **用途**：`POST` 请求用于向服务器提交数据，通常用于表单提交、文件上传、数据创建/更新等。
- **请求体**：数据通过请求体发送，可以传输大量数据，且数据不会暴露在 URL 中。
- **常见格式**：`POST` 请求的数据格式包括 `application/x-www-form-urlencoded`（表单数据）、`application/json`（JSON 数据）、`multipart/form-data`（文件上传）等。
- **与 `GET` 的对比**：`POST` 请求适用于提交数据，而 `GET` 请求用于获取数据。

`POST` 方法广泛应用于 Web 开发和 API 交互，能够有效地处理多种类型的客户端请求。