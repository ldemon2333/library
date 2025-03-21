### **`enum class` 解释**

`enum class` 是 C++11 引入的**强类型枚举**（Scoped Enumeration），用于定义一组有命名的整数常量。相比传统 `enum`，`enum class` 具有**更好的类型安全性**，避免命名冲突，并且支持显式指定底层存储类型。

---

## **代码解析**

```cpp
enum class ErrornCode : int32_t
{
    // 通用 0 ~ 99
    kOk = 0,           // 结果正常
    kInvalidArgument,  // 参数错误

    // 调度 100~199
    kScheGeneral,
    kScheStep,
    kScheContinue,
    kScheDriver,

    // 资源分配 200 ~ 249
    kNraBigSmallCoreInvalid = 200,  // 指定坐标分配大小核偏移错误
    kNraOutOfRange,                 // 指定坐标分配超出范围
    kNraCoverRiscvCore,             // 指定坐标分配覆盖了 Riscv 核心
    kNraCoverUsedCore,              // 指定坐标分配的核心被占用
    kNraLack,                       // 资源不足

    // 资源管理 250~299
    kNraInitInvalid = 250,  // 机箱节点上线流程不正确，WEST 方向的节点未上线，EAST 方向的节点上线无法初始化

    // 任务运行 300 ~ 399

};
```

### **关键点**

1. **`enum class ErrornCode : int32_t`**
    
    - `enum class` 定义了一个**强类型枚举**，避免了与其他枚举或变量名称冲突。
    - `: int32_t` 明确指定了底层存储类型（默认是 `int`）。
    - 这样可以确保枚举值使用 `int32_t` 类型，保证跨平台一致性。
2. **枚举值的定义**
    
    - `kOk = 0`：显式赋值为 `0`。
    - `kInvalidArgument` **未赋值**，则默认为 `1`（比前一个值 `+1`）。
    - `kScheGeneral` **未赋值**，则默认为 `100`（前面 `kInvalidArgument = 1`，但 `kNraBigSmallCoreInvalid = 200`，所以 `kScheGeneral` 按照 `100~199` 区间递增）。
3. **分区管理**
    
    - `0 ~ 99`：通用错误。
    - `100 ~ 199`：调度相关错误。
    - `200 ~ 249`：资源分配错误。
    - `250 ~ 299`：资源管理错误。
    - `300 ~ 399`：任务运行错误。
    - 这样分类有助于**错误码管理**，方便调试与查找错误。

---

## **传统 `enum` vs `enum class`**

|特性|`enum`|`enum class`|
|---|---|---|
|**作用域**|直接暴露到全局|需要 `EnumName::` 访问|
|**类型安全**|允许隐式转换为 `int`，可能产生冲突|不能隐式转换，避免命名冲突|
|**底层类型指定**|由编译器决定，可能是 `int`、`unsigned int`|可以手动指定，如 `: int32_t`|

### **示例**

```cpp
enum OldEnum { A = 1, B = 2 };  // 传统枚举
enum class NewEnum { A = 1, B = 2 };  // 强类型枚举
```

- `OldEnum x = A;` ✅ 允许直接使用 `A`，容易命名冲突。
- `NewEnum y = NewEnum::A;` ✅ 需要 `NewEnum::A`，更安全。

---

## **如何使用 `enum class`？**

### **1. 访问枚举值**

```cpp
ErrornCode err = ErrornCode::kOk;
if (err == ErrornCode::kOk) {
    std::cout << "Operation successful!" << std::endl;
}
```

必须使用 `ErrornCode::kOk` 访问，防止命名污染。

### **2. 转换为整数**

`enum class` 不能隐式转换为 `int`，需要显式转换：

```cpp
int code = static_cast<int>(ErrornCode::kNraLack);
std::cout << "Error Code: " << code << std::endl;  // 输出 204
```

---

## **总结**

✅ **使用 `enum class` 的好处：**

1. **避免命名冲突**（必须使用 `EnumName::` 访问）。
2. **提高类型安全性**（不会隐式转换为 `int`）。
3. **可以指定底层类型**（如 `int32_t`，保证跨平台一致性）。
4. **便于分类和管理**（不同错误码有不同范围）。

🚀 **推荐**：C++ 开发中**优先使用 `enum class` 而不是 `enum`**！