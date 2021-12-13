# NVIDIA相关
业内一般采用N卡进行AI相关的机器学习模型训练，主要原因在于软件层面NVIDIA CUDA以及CUDNN的支持。
- CUDA：NVIDIA通用并行计算架构及工具
- CUDNN：NVIDIA深度学习神经网络GPU加速库

额外原因在于硬件层面NVIDIA TENSOR CORE对于训练/推理的性能加持。

Tensor core:

_https://www.nvidia.cn/data-center/tensor-cores/_

专业级显卡相比消费级显卡拥有更高的算力，更低的功耗，而消费级显卡拥有更便宜的价格。

官方提供的显卡算力与计算速度无关，只与显卡的功能和特性有关

_https://developer.nvidia.com/cuda-gpus#compute_

## HPL Benchmark
HPL (High-Performance Linpack Benchmark)
主要用于对高性能计算机集群，采用高斯消元法求解一元N次稠密线性代数方程组进行浮点计算能力测试。

由于显卡不能像cpu一样同时支持不同精度的浮点运算，所以gpu里的单精度与双精度需要不同的计算单元，通常支持单精度运算的计算单元称为FP32 core或简称为core，
而双精度的计算单元称之为DP unit(Double Precision)或FP64 core,不同架构，不同型号的显卡中，两种core的数量比例差别很大。

显卡理论Flops计算公式

_https://zh.wikipedia.org/wiki/每秒浮點運算次數_

单精度FLOPS = FP32cores * 主频 (GHz)* 单核单周期计算次数(2)

双精度FLOPS = FP64cores * 主频 (GHz)* 单核单周期计算次数(2)

半精度运算同样是借助单精度的core，但是需要确认显卡架构中每个单精度的core支持几个半精度的运算，若一个单精度core支持两个半精度运算，半精度的FLOPS就是单精度的双倍。

官方提供的3090数据，并未提供算力相关数据(可能是因为消费级显卡，专业级显卡提供了数据)
_https://www.nvidia.cn/geforce/graphics-cards/30-series/rtx-3090/_

根据官方数据计算单精度FLOPS

FP32FLOPS = 10496 * 1.70 * 2 = 35.6864 TFLOPS


非官方提供的3090算力数据（这个数据并未包含Tensorcore的加持，所以认为可以用作HPC浮点计算能力的衡量指标）
_https://www.techpowerup.com/gpu-specs/geforce-rtx-3090.c3622_

根据非官方的数据可以看出，3090中

FP16core:FP32core:FP64core=64:64:1

算力单位(显卡默认以tera为单位TFLOPS)
- FLOPS(floating-point operations per second)每秒浮点运算次数
- MFLOPS (mega) 一百万 =10^6
- GFLOPS (giga) 十亿 =10^9
- TFLOPS (tera) 一万亿 =10^12
- PFLOPS (peta) 一千万亿 =10^15

测试精度
- FP64 双精度浮点数 64位
- FP32 单精度浮点数 32位
- FP16 半精度浮点数 16位
- TF32 是NVIDIA基于Tensorcore推出的新数值，与FP16相同精度(10位)，与FP32同样指数(8位)

TF32
_https://blogs.nvidia.com/blog/2020/05/14/tensorfloat-32-precision-format/_

NVIDIA官方打包了对应CUDA版本的HPL容器镜像用于测试，依赖需要NVIDIA-container-toolkit以及NVIDIA显卡驱动

官方安装/使用参考

_https://catalog.ngc.nvidia.com/orgs/nvidia/containers/hpc-benchmarks_
_https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html#pre-requisites_
_https://docs.nvidia.com/ngc/ngc-private-registry-user-guide/index.html#generating-api-key_

其他参考

_http://hpl-calculator.sourceforge.net/Howto-HPL-GPU.pdf_

## 深度学习Benchmark
深度学习Benchmark，主要是使用Google Tensorflow和Facebook PyTorch等深度学习框架，进行几种典型卷积神经网络模型的训练，在网络收敛至所需的预测准确水平时，对吞吐量或收敛时间进行对比。
或使用TensorRT等框架进行推理性能测试。（主要用于MLPerf）

lambda测试数据

_https://lambdalabs.com/gpu-benchmarks_

lambda测试数据使用V100为度量值，其他显卡对比其加速度，如图3090的吞吐量加速度为V100的1.31倍。


Tensorflow与PyTorch需要对应CUDA以及CUDNN版本使用。

_https://www.tensorflow.org/install/source#gpu_
- TTS (Time to Sloution)/TTC (Time to Convergence)/TTT (Time to Train) 完成时间/收敛时间/训练时间
- Throughput (吞吐量)
- Batch_size(一次训练抓取的数据样本数)
- Tensor 张量(高维矩阵)

卷积网络模型
- resnet
- AlexNet
- VGG
- goolenet 主要使用 inception模块

官方提供深度学习测试结果(只看到了企业级显卡的数据)

_https://developer.nvidia.com/deep-learning-performance-training-inference#:~:text=NVIDIA landed top performance spots on all MLPerf™,inference performance across multiple application areas and models._

官方的测试结果中分别提供了基于不同硬件芯片MLPerf模型训练时间对比，单卡相同硬件芯片不同架构模型的训练性能对比，以及两者相对应的推理时间及性能对比。

官方测试介绍

_https://developer.nvidia.com/blog/updating-ai-product-performance-from-throughput-to-time-to-solution/_

官方Tensorflow容器镜像

_https://catalog.ngc.nvidia.com/orgs/nvidia/containers/tensorflow_

官方PyTorch容器镜像
_https://catalog.ngc.nvidia.com/orgs/nvidia/containers/tensorflow_

官方提供的深度学习训练例子
_https://github.com/NVIDIA/DeepLearningExamples_

## MLPerf
是一整套权威的AI芯片/软件/服务的训练和推理性能测试基准，其中训练包含了8个类别的机器学习任务。

训练任务

v1.1包括以下分类训练任务。
_https://mlcommons.org/en/training-normal-11/_

图片处理类：
1. 图片分类
- 医学图像分割
2. 计算机视觉类：
- 目标检测（轻量）
- 目标检测（重量）
3. 计算机语言处理类：
- 语音识别
- 自然语言处理
4. 商业类：
- 推荐系统
5. 科研类：
- 强化学习

MLPerf包括两种赛制：

封闭模型（Closed Model Division）
开放模型（Open Model Division）

封闭模型要求使用相同的模型网络、数据集以及优化，限制batch_size和其他参数，主要目的在于公平比较各AI芯片性能。

开放模型则只限制使用相同的数据集解决相同的问题，对于模型和算法的优化不会加以限制，它有助于推动ML模型和优化的创新。

对于v1.1封闭模型的规则限制，要求使用以下模型完成对应任务，并达到指定的训练目标。（MLPerf要求达到的目标只是基础易完成的程度，便于测试性能，而非state of the art）

_https://github.com/mlcommons/training_policies/blob/master/training_rules.adoc#closed-division_


推理任务


MLPerf使用参考

_https://github.com/mlcommons
https://segmentfault.com/a/1190000022834920?utm_source=tag-newest_