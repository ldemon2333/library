GPIO（General Purpose Input/Output）是一种用于微控制器和其他数字电路的接口，允许设备与外部世界进行交互。GPIO 引脚可以配置为输入模式或输出模式，具体功能通常由用户控制。

### GPIO 的主要功能：

1. **输入（Input）**：
    
    - **读取状态**：GPIO 输入引脚用于接收外部信号（例如传感器、按钮等）的电平状态，通常是 0（低电平）或 1（高电平）。
    - **应用场景**：检测按钮是否被按下、读取温度传感器的数据等。
2. **输出（Output）**：
    
    - **发送信号**：GPIO 输出引脚用于向外部设备发送电平信号，通常可以是高电平或低电平。
    - **应用场景**：控制 LED 灯的开关、驱动继电器、控制马达等。

### GPIO 引脚的控制：

- **输入模式**：当引脚被设置为输入模式时，微控制器读取外部信号的电平，并根据其高低（1 或 0）做出反应。
    
- **输出模式**：当引脚被设置为输出模式时，微控制器可以将其状态设置为高（1）或低（0），从而驱动外部电路。
    
- **中断模式**：某些平台支持 GPIO 引脚的中断模式，允许程序在引脚电平发生变化时立即做出反应（如按钮按下时触发中断）。
    

### 常见的 GPIO 使用场景：

1. **控制 LED**： GPIO 引脚可以用来控制 LED 的开关，通常将 LED 连接到 GPIO 输出引脚。
    
2. **读取按钮状态**： 通过设置 GPIO 引脚为输入模式，可以检测按钮的按下与否，通常将按钮连接到 GPIO 输入引脚。
    
3. **控制继电器或马达**： GPIO 可以控制高电流设备，如继电器或马达。通过输出高电平或低电平信号，控制外部设备的启停。
    
4. **信号采集**： 在嵌入式系统中，GPIO 输入引脚可以用于采集传感器数据（如温度传感器、光传感器等）。
    

### GPIO 在 Linux 系统中的使用：

在 Linux 系统中，GPIO 引脚通常通过 `/sys/class/gpio` 接口或 `libgpiod` 库进行操作。

#### 使用 `/sys/class/gpio`：

1. **导出 GPIO 引脚**： 在 `/sys/class/gpio` 下，GPIO 引脚的操作通常通过“导出”来启用。例如，导出 GPIO 17 引脚：
    
    ```bash
    echo 17 > /sys/class/gpio/export
    ```
    
2. **设置 GPIO 引脚方向**（输入或输出）： 导出后，可以设置引脚的方向：
    
    ```bash
    echo "in" > /sys/class/gpio/gpio17/direction  # 设置为输入
    echo "out" > /sys/class/gpio/gpio17/direction  # 设置为输出
    ```
    
3. **读取 GPIO 引脚的值**： 如果 GPIO 引脚配置为输入模式，可以读取其电平：
    
    ```bash
    cat /sys/class/gpio/gpio17/value
    ```
    
4. **设置 GPIO 引脚的值**： 如果 GPIO 引脚配置为输出模式，可以设置其值（高电平或低电平）：
    
    ```bash
    echo 1 > /sys/class/gpio/gpio17/value  # 设置为高电平
    echo 0 > /sys/class/gpio/gpio17/value  # 设置为低电平
    ```
    
5. **卸载 GPIO 引脚**： 在使用完毕后，可以通过“unexport”将 GPIO 引脚卸载：
    
    ```bash
    echo 17 > /sys/class/gpio/unexport
    ```
    

#### 使用 `libgpiod`：

`libgpiod` 是一个 C 库，提供了对 Linux 内核 GPIO 子系统的高级访问接口，推荐用于更复杂的 GPIO 操作，如事件处理和中断。

示例：

```c
#include <gpiod.h>
#include <stdio.h>

int main() {
    struct gpiod_chip *chip;
    struct gpiod_line *line;
    int value;

    // 打开 GPIO 芯片
    chip = gpiod_chip_open_by_name("/dev/gpiochip0");
    if (!chip) {
        perror("Open GPIO chip");
        return 1;
    }

    // 获取 GPIO 17 引脚
    line = gpiod_chip_get_line(chip, 17);
    if (!line) {
        perror("Get GPIO line");
        return 1;
    }

    // 设置为输入模式
    gpiod_line_request_input(line, "GPIO_input");

    // 读取引脚值
    value = gpiod_line_get_value(line);
    printf("GPIO 17 value: %d\n", value);

    // 清理
    gpiod_line_release(line);
    gpiod_chip_close(chip);

    return 0;
}
```

### GPIO 中的高级功能：

1. **中断（Interrupts）**：
    
    - 当引脚的电平变化时，可以通过中断处理程序响应这些变化。Linux 下，GPIO 的中断通常可以通过 `libgpiod` 或 `/sys/class/gpio` 来配置。
    - 中断可以提高响应速度，避免轮询（polling）的方式消耗 CPU 时间。
2. **PWM（脉宽调制）**：
    
    - 有些 GPIO 引脚支持 PWM 功能，可以用来控制电机速度、亮度等。例如，在树莓派等设备上，GPIO 可以产生 PWM 信号控制 LED 的亮度。
3. **模拟输入（ADC）**：
    
    - 许多微控制器提供模拟输入功能，可以通过 ADC 将模拟信号转换为数字信号。然而，大多数单板计算机（如树莓派）没有内建的 ADC 功能，但可以通过外部模块实现。

### 总结：

GPIO 是与外部设备交互的重要接口，常用于嵌入式系统和单板计算机中。通过 GPIO，可以控制 LED、读取按钮状态、驱动继电器等。通过 `/sys/class/gpio` 或 `libgpiod` 库，可以在 Linux 系统中轻松控制 GPIO 引脚。理解如何操作 GPIO 是嵌入式编程和硬件接口开发的重要步骤。
