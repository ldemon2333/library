在 C 语言中，`fmax` 是一个用于返回两个浮点数中较大值的标准库函数，定义在 `<math.h>` 头文件中。它接受两个浮点数参数，返回这两个数中较大的一个。

### 函数原型：

```c
#include <math.h>

double fmax(double x, double y);
```

### 参数：

- `x` 和 `y`：这两个参数是要比较的浮点数（`double` 类型）。

### 返回值：

- 返回 `x` 和 `y` 中较大的一个。如果 `x` 等于 `y`，则返回其中任意一个。如果 `x` 或 `y` 是 `NaN`，则返回另一个有效数值；如果两者都是 `NaN`，则返回 `NaN`。

### 使用示例：

```c
#include <stdio.h>
#include <math.h>

int main() {
    double a = 3.5;
    double b = 2.5;
    double result;

    // 使用 fmax 比较两个浮点数
    result = fmax(a, b);

    printf("The maximum of %.2f and %.2f is %.2f\n", a, b, result);

    return 0;
}
```

### 输出：

```
The maximum of 3.50 and 2.50 is 3.50
```

### 关键点：

- `fmax` 函数返回较大的浮点数。
- 它处理 `NaN`（Not-a-Number）值时有特殊规则：如果一个数是 `NaN`，函数会返回另一个有效数；如果两个数都是 `NaN`，返回 `NaN`。

### 处理 `NaN` 的示例：

```c
#include <stdio.h>
#include <math.h>

int main() {
    double a = NAN;
    double b = 5.0;
    
    // 如果 a 是 NaN，fmax 会返回 b
    double result = fmax(a, b);
    printf("The maximum value is %.2f\n", result);  // 输出: 5.00

    return 0;
}
```

### 总结：

`fmax` 是一个非常实用的函数，用来比较两个浮点数并返回较大的那个，且能够正确处理 `NaN` 值的情况。