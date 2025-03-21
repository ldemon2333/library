![[Pasted image 20250306123411.png]]

消息header 中的 args，offset 获取任务需要被放置在哪个计算板及其具体坐标 target
![[Pasted image 20250306130657.png]]

operator 中 180
![[Pasted image 20250306123517.png]]

createTask 方法
![[Pasted image 20250306123735.png]]

执行 TaskInit
![[Pasted image 20250306123852.png]]
修改状态并调用内部初始化函数 TaskInitInner，这里读取模型文件，DPK 文件结构和解析，成员变量 NeuralTaskModel 对应神经元模型解析的结果。

![[Pasted image 20250306123926.png]]

读取模型文件，NeuralTaskModel 内部主要包含以下数据：
![[Pasted image 20250306150501.png]]
![[Pasted image 20250306150519.png]]

AllocateResource 调用资源申请的接口，申请资源

![[Pasted image 20250306124206.png]]

ResourceAllocation 类中的 AllocateResource 方法申请资源，为本次任务申请需要的资源。
![[Pasted image 20250306150146.png]]
![[Pasted image 20250306150202.png]]
![[Pasted image 20250306155937.png]]
![[Pasted image 20250306155954.png]]


如何理解脉冲数据
![[Pasted image 20250307143747.png]]
## **整体数据结构**

![[Pasted image 20250307144748.png]]

假设 `data` 的二进制格式如下：

| 偏移  | 数据                        | 说明                            |
| --- | ------------------------- | ----------------------------- |
| 0   | `tick_num`                | 总时间步数                         |
| 8   | `tick_spike_bytes_num[0]` | 第 0 个 tick 的脉冲数据大小（字节）        |
| 16  | `tick_data[0]`            | 第 0 个 tick 的脉冲数据（uint64_t 数组） |
| ... | `tick_spike_bytes_num[1]` | 第 1 个 tick 的脉冲数据大小（字节）        |
| ... | `tick_data[1]`            | 第 1 个 tick 的脉冲数据              |
| ... | ...                       | ...                           |
m_inSpk 的形状（时间步，当前时间步的脉冲数据）
当前时间步的脉冲数据，**是 `uint64_t` 数组**，存储**所有触发的神经元 ID（全局 spike ID）**

## **每个时间步的脉冲数据解析**

在 `LoadSpikeData()` 函数中，每个**时间步（tick）**的数据存储在 `data` 这个二进制缓冲区中。以下是解析逻辑的详细说明：

---

## **1. 数据的存储结构**

在 `data` 中，每个 tick（时间步）数据的存储格式如下（以 **64-bit 对齐** 存储）：

|偏移 (Byte)|数据类型|变量|说明|
|---|---|---|---|
|0|`uint64_t`|`tick_num`|**总时间步（tick）的数量**|
|8|`uint64_t`|`tick_spike_bytes_num[0]`|**第 0 个 tick 内脉冲数据所占字节数**|
|16|`uint64_t[]`|`tick_data[0]`|**第 0 个 tick 的脉冲数据**（以 64-bit 存储的 spike ID 列表）|
|...|`uint64_t`|`tick_spike_bytes_num[1]`|**第 1 个 tick 内脉冲数据所占字节数**|
|...|`uint64_t[]`|`tick_data[1]`|**第 1 个 tick 的脉冲数据**|
|...|`uint64_t`|`tick_spike_bytes_num[2]`|**第 2 个 tick 内脉冲数据所占字节数**|
|...|`uint64_t[]`|`tick_data[2]`|**第 2 个 tick 的脉冲数据**|
|...|...|...|...|

### **⚠ 关键点**

- **每个 tick 开头都有一个 `uint64_t` 变量**，它存储当前 tick 的**脉冲数据所占的字节数**（`tick_spike_bytes_num`）。
- **紧接着是 `uint64_t` 数组**，存储**所有触发的神经元 ID（spike ID）**。

---

## **2. 代码解析：如何解析每个 tick 的脉冲数据**

在 `LoadSpikeData()` 代码中，解析 tick 数据的核心逻辑如下：

```cpp
uint64_t *tick_start_data = (uint64_t *)data + sizeof(tick_num) / sizeof(uint64_t);
for (int i = 0; i < tick_num; i++)
{
    // 计算当前 tick 的脉冲数据起始地址
    auto tick_data = tick_start_data + (sizeof(tick_spike_bytes_num)) / sizeof(uint64_t);
    
    // 获取当前 tick 数据的起始和结束迭代器
    Iterator64 mual_begin = tick_data;
    Iterator64 mual_end = tick_data + ((tick_spike_bytes_num / sizeof(uint64_t)) - 1);

    // 存入 spksCmd->m_inSpk[i]
    spksCmd->m_inSpk[i].assign(std::make_move_iterator(mual_begin), std::make_move_iterator(mual_end));

    // 转移到下一个 tick
    tick_start_data += tick_spike_bytes_num / sizeof(uint64_t);
    tick_spike_bytes_num = tick_start_data[0];
}
```

---

### **3. 解析示例**

假设 `data` 以 **二进制格式** 存储了以下数据：

```plaintext
tick_num = 3    // 3 个时间步
-----------------------------------------
tick 0:
tick_spike_bytes_num = 16  // 16 字节 = 2 * uint64_t
tick_data = [1001, 1002]   // 2 个神经元触发

tick 1:
tick_spike_bytes_num = 24  // 24 字节 = 3 * uint64_t
tick_data = [2001, 2002, 2003]  // 3 个神经元触发

tick 2:
tick_spike_bytes_num = 8  // 8 字节 = 1 * uint64_t
tick_data = [3001]  // 1 个神经元触发
```

对应的 `data` 在内存中的排列如下：

```
| tick_num(3) |
| tick_spike_bytes_num(16) | tick_data(1001, 1002) |
| tick_spike_bytes_num(24) | tick_data(2001, 2002, 2003) |
| tick_spike_bytes_num(8)  | tick_data(3001) |
```

解析结果：

```
spksCmd->m_inSpk[0] = {1001, 1002}  // tick 0
spksCmd->m_inSpk[1] = {2001, 2002, 2003}  // tick 1
spksCmd->m_inSpk[2] = {3001}  // tick 2
```

---

## **4. `std::make_move_iterator` 的作用**

```cpp
spksCmd->m_inSpk[i].assign(std::make_move_iterator(mual_begin), std::make_move_iterator(mual_end));
```

**作用**：

- `std::make_move_iterator` 让 `assign()` **以移动语义（move semantics）** 方式拷贝 `tick_data`，避免不必要的拷贝，提高效率。

---

## **5. `tick_start_data` 指针如何前移**

```cpp
tick_start_data += tick_spike_bytes_num / sizeof(uint64_t);
tick_spike_bytes_num = tick_start_data[0];
```

- `tick_start_data` **向后移动 `tick_spike_bytes_num` 个字节**，即**跳过当前 tick 的所有脉冲数据**，进入下一个 tick。
- 然后 `tick_spike_bytes_num = tick_start_data[0]` 读取下一个 tick 的脉冲数据长度，循环继续。

---

## **总结**

✅ **每个 tick 数据的格式**：

- `uint64_t tick_spike_bytes_num`：该 tick 的脉冲数据大小（字节）
- `uint64_t[] tick_data`：该 tick 内所有神经元的 ID（spike ID）

✅ **数据解析流程**：

1. 读取 `tick_num`（时间步数量）。
2. **循环解析每个 tick**：
    - 读取 `tick_spike_bytes_num`（脉冲数据长度）。
    - 解析 `tick_data`（神经元触发 ID）。
    - **移动指针** 到下一个 tick。

✅ **高效的 `std::make_move_iterator`**

- **避免额外拷贝**，提升解析速度。


## 状态机变化
![[Pasted image 20250307152345.png]]
createTask 后 addTask(pNewTask)

![[Pasted image 20250307152413.png]]
WakeFSM，唤醒 0 号主状态机处理数据

![[Pasted image 20250307152447.png]]
执行 0 号状态机后执行 RunContinue

![[Pasted image 20250307152524.png]]

![[Pasted image 20250307194028.png]]
/root/darwinos_system/src/schedule/src/schedule_algorithm.cpp 是否有问题



```c++
ErrornCode TaskCreate::operator()(DrwMessage& input_msg, DrwMessage& output_msg)
{
	jsHeader = input_msg.getHeader();// 获取输入消息头
	app_id = jsHeader["app_id"]; // 提取应用 ID
	binPath = jsHeader["model_path"]; // 提取模型路径

	target = < BoardID, Coordinate >; // 任务需要被放置在哪个计算板及其具体坐标
	tick_len = jsHeader["tick_len"]; // 任务的时间步长
	global_neuron_id_map = jsHeader["global_neuron_id_map"]; //解析神经元 ID 映射表
	tarDpk = appInfo(app_id);
	pNewTask = createTask(); // 创建任务逻辑
	// 检查任务是否创建成功
	// 构造返回消息 output_msg
}
```

```c++
shared_ptr<NeuralTask> TaskMgr::createTask()
{
	TaskID taskId = TaskMgr::allocTaskId(); // 分配任务 Id
	auto pNewTask = make_shared<NeuralTask>(taskId);
	auto ret = pNewTask->TaskInit(0, global_neuron_id_maps, binFolder, target);
	TaskMgr::addTask(pNewTask);
	return pNewTask;
}
```

```c++
int32_t NeuralTask::TaskInit()
{
	SetTaskStatus(kInitting);
	ret = TaskInitInner();// 调用内部初始化函数
	// 任务成功设置状态为 kInitted
	if(ret == Ok){
		SetTaskStatus(kInitted);
	}
}
```

``` c++
int32_t NeuralTask::Task
```