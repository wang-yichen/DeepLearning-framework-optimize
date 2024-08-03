
# 模型框架优化总结

![总图](图/框架总流程.png)

## 参考资料

https://blog.csdn.net/sgyuanshi/article/details/123536579（tensorrt）

https://github.com/ihomeava/PaddleSeg2/blob/release/2.3/docs/slim/quant/quant.md（模型量化）

https://developer.aliyun.com/article/1161611（Tensorrt导出engine模型进行推理）

https://www.jos.org.cn/html/2021/1/6096.htm#outline_anchor_36（模型优化与压缩，综述）

https://www.cnblogs.com/shixiangwan/p/9015010.html（模型优化与压缩，综述）

https://github.com/cedrickchee/awesome-ml-model-compression（模型优化与压缩，论文整理）

https://tvm.hyper.ai/docs/tutorial/intro（TVM）

https://zhuanlan.zhihu.com/p/344442534（openvino）

https://onnxruntime.ai/docs/execution-providers/  （ONNX Runtime）

https://blog.csdn.net/hh1357102/article/details/131831741（NCNN）

https://github.com/alibaba/MNN（MNN）

## 1.Tensorrt转中间格式对模型的优化

### 1.优化方法

##### 1.模型量化

支持 FP32、FP16 和 INT8 的精度，通过降低精度进行存储和计算，可以降低计算量，加快推理速度

##### 2.层融合

融合多个网络节点，减少计算和内存访问次数，提高运算效率

##### 3.内核自动调优

针对不同内核的特点，根据不同的任务选择不同的内核

##### 4.动态张量内存分配

自动分配内存机制，可以最大化减少显存的利用，并提高内存构建和释放的效率，提高速度

##### 5.图优化

合并可以合并的操作，删除无关紧要的操作，减少计算量

##### 6.流控制优化

并行操作，同时进行多次推理

### 2.优化步骤

由最初的.pt模型转换为onnx模型，onnx模型进入Tensorrt的某种解析器转化为一种中间表示IR，进行全部优化后生成一个序列化的模型，可以被Tensorrt引擎加载推理。

### 3.triton和tensorrt

类似于TensorFlow Serving以及docker，适用于TensorRT、ONNX、TensorFlow、Torch、OpenVINO、DALI等框架，提供http和grpc协议，实现模型推理和其他业务分离，模型统一部署在triton server，其他业务通过triton client进行模型推理请求

## 2.模型优化与压缩方法总结

#### 1.参数剪枝

参数剪枝是指在预训练好的大型模型的基础上, 设计对网络参数的评价准则, 以此为根据删除“冗余”参数.根据剪枝粒度粗细, 参数剪枝可分为非结构化剪枝和结构化剪枝

##### 非结构化剪枝

非结构化剪枝的粒度比较细, 可以无限制地去掉网络中期望比例的任何“冗余”参数, 但这样会带来裁剪后网络结构不规整、难以有效加速的问题

##### 结构化剪枝

结构化剪枝的粒度比较粗, 剪枝的最小单位是filter内参数的组合, 通过对filter或者feature map设置评价因子, 甚至可以删除整个filter或者某几个channel, 使网络“变窄”, 从而可以直接在现有软/硬件上获得有效加速, 但可能会带来预测精度(accuracy)的下降, 需要通过对模型微调(fine-tuning)以恢复性能

#### 2.参数量化

参数量化是指用较低位宽表示典型的32位浮点网络参数, 网络参数包括权重、激活值、梯度和误差等等, 可以使用统一的位宽(如16-bit、8-bit、2-bit和1-bit等), 也可以根据经验或一定策略自由组合不同的位宽.

参数量化的优点是:

(1)能够显著减少参数存储空间与内存占用空间, 将参数从32位浮点型量化到8位整型, 从而缩小75%的存储空间, 这对于计算资源有限的边缘设备和嵌入式设备进行深度学习模型的部署和使用都有很大的帮助; 

(2)能够加快运算速度, 降低设备能耗, 读取32位浮点数所需的带宽可以同时读入4个8位整数, 并且整型运算相比浮点型运算更快, 自然能够降低设备功耗.

但其仍存在一定的局限性, 网络参数的位宽减少损失了一部分信息量, 会造成推理精度的下降, 虽然能够通过微调恢复部分精确度, 但也带来时间成本的增加; 量化到特殊位宽时, 很多现有的训练方法和硬件平台不再适用, 需要设计专用的系统架构, 灵活性不高

##### 二值化

##### 三值化

##### 聚类量化

##### 混合位宽

#### 3.低秩分解

神经网络的filter可以看作是四维张量:宽度*w*×高度*h*×通道数*c*×卷积核数*n*, 由于*c*和*n*对网络结构的整体影响较大, 所以基于卷积核(*w*×*h*)矩阵信息冗余的特点及其低秩特性, 可以利用低秩分解方法进行网络压缩.

低秩分解是指通过合并维数和施加低秩约束的方式稀疏化卷积核矩阵, 由于权值向量大多分布在低秩子空间, 所以可以用少数的基向量来重构卷积核矩阵, 达到缩小存储空间的目的.低秩分解方法在大卷积核和中小型网络上有不错的压缩和加速效果, 过去的研究已经比较成熟, 但近两年已不再流行.原因在于:除了矩阵分解操作成本高、逐层分解不利于全局参数压缩, 需要大量的重新训练才能达到收敛等问题之外, 近两年提出的新网络越来越多地采用1×1卷积, 这种小卷积核不利于低秩分解方法的使用, 很难实现网络压缩与加速.

##### 二元分解

##### 多元分解

#### 4.参数共享

参数共享是指利用结构化矩阵或聚类等方法映射网络参数, 减少参数数量.参数共享方法的原理是利用参数存在大量冗余的特点, 设计一种映射形式, 将全部参数映射到少量数据上, 减少对存储空间的需求.

由于全连接层参数数量较多, 参数存储占据整个网络模型的大部分, 所以参数共享对于去除全连接层冗余性能够发挥较好的效果; 也由于其操作简便, 适合与其他方法组合使用.但其缺点在于不易泛化, 如何应用于去除卷积层的冗余性仍是一个挑战.同时, 对于结构化矩阵这一常用映射形式, 很难为权值矩阵找到合适的结构化矩阵, 并且其理论依据不够充足

##### 循环矩阵

##### 聚类共享

#### 5.紧凑网络

##### 卷积核级别

深度可分离卷积

组卷积

##### 层级别

##### 网络结构级别

#### 6.知识蒸馏

知识蒸馏用于训练带有伪数据标记的强分类器的压缩模型和复制原始分类器的输出.

知识蒸馏法需要两种类型的网络:教师模型和学生模型.预先训练好的教师模型通常是一个大型的神经网络模型, 具有很好的性能.

![1](图/1.jpg)

将教师模型的softmax层输出作为soft target与学生模型的softmax层输出作为hard target一同送入total loss计算, 指导学生模型训练, 将教师模型的知识迁移到学生模型中, 使学生模型达到与教师模型相当的性能.学生模型更加紧凑高效, 起到模型压缩的目的.

知识蒸馏法可使深层网络变浅, 极大地降低了计算成本, 但也存在其局限性.由于使用softmax层输出作为知识, 所以一般多用于具有softmax损失函数的分类任务, 在其他任务的泛化性不好; 并且就目前来看, 其压缩比与蒸馏后的模型性能还存在较大的进步空间.

## 3.框架优化对比

### TVM

![1](图/1.png)

##### 自动调度

通过自动调度寻找最优的计算配置和调度策略

##### 图优化

自动识别并合并多个连续的算子（如层）到一个循环中，减少内存访问次数和减少数据读写开销

##### 多级内存

利用缓存等内存结构降低访问延迟

##### 向量化

硬件SIMD指令方式加速数据处理

##### 并行化

并行处理，提高数据吞吐量

##### 模型量化

##### 硬件特定优化

针对特定硬件的优化，为 GPU 生成特定的内核、为 CPU 利用高级指令集等。

通过硬件后端（如 CUDA、OpenCL、ROCM、Metal 等）实现。

##### 缺点或问题

较高的学习曲线，需要对编译和底层优化有一定了解。

### 框架介绍

| 模型推理部署框架 | 框架介绍                                                     |
| ---------------- | ------------------------------------------------------------ |
| TVM              | TVM是一个开源的机器学习编译器，旨在为各种硬件平台提供高效的深度学习模型部署解决方案。它支持自动优化，能够将模型从各种框架（如PyTorch, TensorFlow等）转换并优化到CPU、GPU、FPGA等多种硬件上。 |
| Paddle inference | Paddle Inference是百度开发的PaddlePaddle深度学习框架的推理库。它专为生产环境优化，支持快速部署和高效运行PaddlePaddle训练的模型，支持多种硬件平台，包括CPU和GPU。 |
| PyTorch          | PyTorch是一个广泛使用的开源机器学习库，由Facebook的AI研究团队开发。它主要用于应用如计算机视觉和自然语言处理的研究与开发，支持动态计算图，使得模型易于构建和修改。 |
| OpenVINO         | OpenVINO是由Intel开发的一套免费工具集，用于加速深度学习应用的部署。它特别优化了在Intel硬件（如CPU、GPU和VPU）上的性能，支持广泛的深度学习模型和框架。 |
| TensorRT         | TensorRT是NVIDIA推出的高性能深度学习推理（inference）引擎，专为生产环境优化，可提供低延迟和高吞吐量的服务。TensorRT特别适合在NVIDIA GPU上进行图像和视频处理任务。 |
| ONNX Runtime     | ONNX Runtime是一个用于执行机器学习模型推理的性能优化引擎。支持Open Neural Network Exchange (ONNX)格式的模型，允许模型从各种框架互操作，支持多种硬件平台。 |
| TFLite           | TensorFlow Lite是一个为移动和嵌入式设备优化的开源深度学习框架。它支持快速推理，并可以在不牺牲准确度的情况下减少模型的大小和复杂度。 |
| NCNN             | NCNN是由腾讯优化的一个高性能神经网络推理引擎，特别适用于移动设备，具有极低的延迟和高效率。它广泛用于移动端的人脸识别、图像分类等AI应用。 |
| MNN              | MNN是阿里巴巴开源的移动端深度学习框架。它支持模型的快速部署到各种移动设备上，优化了运行速度和资源消耗，使其适用于多种实时应用场景。 |

### 推理框架优化方法总结

| 模型推理部署框架 | 层融合/图优化 | 量化 | 并行 | 内存优化 | 内核优化 | 硬件优化 |
| ---------------- | ------------- | ---- | ---- | -------- | -------- | -------- |
| TVM              | √             | √    | √    | √        | √        | √        |
| Paddle inference | √             | √    | √    | √        | √        | √        |
| pytorch          | √             | √    | √    |          |          |          |
| OpenVINO         | √             | √    | √    | √        | √        | √        |
| TensorRT         | √             | √    | √    | √        | √        | √        |
| ONNX Runtime     | √             | √    | √    |          |          |          |
| TFLite           | √             | √    | √    | √        | √        | √        |
| NCNN             | √             | √    | √    | √        | √        | √        |
| MNN              | √             | √    | √    | √        | √        | √        |



### 支持的深度学习转换框架

| 模型推理部署框架 | TensorFlow | Caffe | Mxnet | Pytorch | ONNX | self         |
| ---------------- | ---------- | ----- | ----- | ------- | ---- | ------------ |
| TVM              | √          | √     | √     | √       | √    |              |
| Paddle inference |            |       |       |         | √    | PaddlePaddle |
| pytorch          |            |       |       | √       | √    | √            |
| OpenVINO         | √          | √     | √     |         | √    |              |
| TensorRT         | √          | √     | √     | √       | √    | √            |
| ONNX Runtime     | √          | √     | √     | √       | √    |              |
| TFLite           | √          |       |       |         |      | TensorFlow   |
| NCNN             | √          | √     |       |         | √    | Caffe        |
| MNN              | √          | √     | √     | √       | √    |              |

### 框架的缺点或问题

| 模型推理部署框架 | 缺点或问题                                                   |
| ---------------- | ------------------------------------------------------------ |
| TVM              | **高学习曲线**：TVM涉及底层的编译技术和优化策略，需要用户对编译原理和硬件优化有深入了解。**环境配置复杂**：设置TVM环境需要多个步骤，包括LLVM、CUDA等依赖项的配置，可能对新用户不友好。 |
| Paddle inference | **框架限制**：主要优化和支持PaddlePaddle框架，对从其他深度学习框架迁移的模型可能需要额外的转换工作。**功能局限**：与其他成熟的推理框架相比，社区支持和功能扩展可能较少。 |
| PyTorch          | **研究优先**：虽然PyTorch在研究和开发中非常流行，但其在生产环境中的部署需要额外的工具，如TorchScript，这可能增加部署复杂性。**性能变化**：模型转换过程中可能出现性能损失，特别是在动态图特性转换为静态图时。 |
| OpenVINO         | **硬件依赖**：虽然在Intel硬件上表现出色，但在AMD或其他非Intel平台上可能不提供同等级的性能优化。**功能局限性**：某些先进的神经网络操作和层在OpenVINO中可能不受支持，限制了某些模型的使用。 |
| TensorRT         | **硬件限制**：只能在NVIDIA GPU上使用，限制了在其他类型硬件平台上的使用。**模型兼容性**：某些特定的深度学习模型和操作可能需要特定的调整才能在TensorRT上高效运行。 |
| ONNX Runtime     | **模型兼容性**：虽然支持多种框架通过ONNX格式，但在转换过程中仍可能出现兼容性问题，特别是对于复杂的自定义层或操作。**性能依赖**：最终性能高度依赖于模型如何与ONNX和底层硬件交互。 |
| TFLite           | **性能限制**：针对移动设备优化，可能在处理大型或复杂深度学习模型时性能不足。**功能支持**：相对于TensorFlow，TFLite可能不支持所有操作和特性，需要调整或简化模型。 |
| NCNN             | **专注移动端**：虽然在移动设备上表现优异，但在服务器或高性能计算环境中可能不是最佳选择。**模型支持**：主要优化支持Caffe和ONNX，其他框架的模型可能需要转换，可能带来额外的工作负担。 |
| MNN              | **社区和资源**：作为一个较新的框架，MNN的文档和社区支持相对有限，新用户的学习曲线可能较陡峭。**跨平台支持**：虽然支持多平台，但在某些平台上的优化和性能调整可能不如专门的框架。 |



## 参考资料：

https://www.cnblogs.com/RSran/p/17721841.html（模型与框架）

https://openmlsys.github.io/chapter_frontend_and_ir/intermediate_representation.html（中间表示）

## 4.模型与框架

### 1.模型与框架的区分

- 深度学习模型可以看作是房屋的设计图。设计图描述了房屋的结构、布局和各个部分的组成。有了设计图，建筑工人可以根据图上的指示来建造房屋。
- 深度学习框架则像是一群建筑工人。建筑工人可以按照设计图把房屋建造出来，他们知道如何搭建房屋的各个部分，并能够处理各种技术问题，如水电、土木等。同样，深度学习框架提供了一个“工人”的角色，帮助你实现深度学习的各种技术，如数据预处理、神经网络的构建和训练、模型的优化等。

**框架是一个开发环境，提供了构建和训练深度学习模型的工具和接口；而模型是针对特定任务的学习算法，由框架实现和支持。**

### 2.框架优化流程

![优化全流程](图/优化全流程.png)

### 3.中间表示IR

中间表示是编译器的核心数据结构之一

![中间表示IR设计需求](图/中间表示IR设计需求.png)



#### 中间表示优化方式

![编译器优化方式](图/编译器优化方式.png)

#### 编译器前端优化流程

![编译器前端优化流程](图/编译器前端优化流程.png)

#### 编译器后端优化流程

![编译器后端基础架构](图/编译器后端基础架构.png)

#### 前端编译优化

##### 1.无用与不可达代码消除

![编译优化-无用代码消除](图/编译优化-无用代码消除.svg)

##### 2.常量传播、常量折叠

![编译优化-常量传播与常量折叠](图/编译优化-常量传播与常量折叠.svg)

##### 3.公共子表达式消除

![编译优化-公共子表达式消除](图/编译优化-公共子表达式消除.svg)

#### 计算图优化

![2](图/2.png)

![静态图和动态图](图/静态图和动态图.png)

![计算图优化](图/计算图优化.png)

#### 算子选择

对IR图上的每个节点进行算子选择，才能生成真正在设备上执行的算子序列

![算子选择](图/算子选择.png)

#### 内存分配

![内存分配](图/内存分配.png)

#### 后端编译优化

![后端编译优化](图/后端编译优化.png)

### 4.模型部署优化

![模型部署优化](图/模型部署优化.png)
