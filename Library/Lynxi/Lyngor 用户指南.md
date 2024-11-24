# 1 产品介绍
## 1.1 什么是 Lyngor
是计算图编程框架

## 1.2 系统架构
![[Pasted image 20241119214431.png]]
# 4 操作说明
Lyngor 主要包括 2 个使用场景：
- 首次构建引擎
- 重复加载引擎
![[Pasted image 20241119221420.png]]


## 4.1 首次构建引擎





## 6.3 生成计算图

### 6.3.1 构建计算图

### 6.3.2 导入模型转换成计算图


## 6.5 构建推理引擎
用户在使用 Lyngor 提供的 API 自行生成计算图时，有可能会同时生成一个携带参数信息的字典，这个字典也可以单独独立构建。参数字典用于减少计算图的体积，方便计算图展示。

用户在使用第三方框架模型进行计算图转换时，除了计算图外，也会生成一个携带参数信息的字典，字典用于存放部分不在计算图中的权重。

计算图转化后，如果参数字典不为空，可使用带参数的 build 方法构建引擎。也有可能参数字典为空，只生成图，这时可使用不带参数的 build 方法构建引擎，具体示例如下。


### 6.5.1 构造无参数引擎
在使用 Lyngor 算子创建计算图后，使用 Lyngor 接口生成推理引擎。

### 6.5.2 构建带参数引擎
在导入已有网络模型，转换为计算图和参数后，使用 Lyngor 接口构建带参数推理引擎。

## 6.7 反序列化引擎
使用 Lyngor 接口对已序列化的引擎进行反序列化操作，加载已保存的推理引擎，节省重复编译时间。

## 6.8 执行推理



## 7.5 Pytorch 模型编译推理实例
Pytorch 网络模型推理实例，导入 Pytorch 框架模型通过 lyngor API 生成推理引擎，并得到运行结
果的完整过程。

1）使用 `lyn.DLModel()` 加载模型
model.load()
2）通过`lyn.Builder` 对算子图进行编译，得到推理引擎
offline_builder.build()
3）使用`lyn.load` 加载推理引擎
r_engine = lyn.load()





总体流程
```
#2. 加载模型
model = lyn.DLModel()
model.load()

#3.创建一个Builder 来编译计算图，产生 runtime 引擎
offline_builder = lyn.Builder()
out_path = offline_builder.build(model.graph, model.params, out_path=path)

#4. 获取引擎
r_engine = lyn.load(path = out_path+'Net_0', device=0)

r_engine.run(data=image_extend.astype(dtype))
result = r_engine.get_output()

```

## 7.6 模型训练实例


# 10 编译生成物说明
## 10.1 net_params.json
