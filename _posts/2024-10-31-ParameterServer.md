<head>
    <script src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML" type="text/javascript"></script>
    <script type="text/x-mathjax-config">
        MathJax.Hub.Config({
            tex2jax: {
            skipTags: ['script', 'noscript', 'style', 'textarea', 'pre'],
            inlineMath: [['$','$']]
            }
        });
    </script>
</head>

---
title: "ParameterServer"
date: 2024-10-31
categories: 
  - distributed training
---

之前几代的Parameter Server都没能够给出让人接受的答案，主要是因为分布式训练参数的中心化存储有着以下几个问题：

1. **同步通信**：在完成一个训练任务的时候，worker需要发送梯度给参数服务器，server将这些梯度平均计算，然后用来更新参数，再将参数发回给worker，拿到参数worker才能进行下一个batch的训练。这可能会带来以下的问题：
    - 木桶效应明显，整个系统的训练速度完全取决于最慢的那一台机器。
    - 计算资源利用率低。


2. **一致性**：简单来说，采用异步通信，某些worker可能使用了过时的参数进行训练，并将计算的梯度结果发送给了server，这会影响模型的收敛效果。

3. **热点**：受数据传输带宽限制，训练瓶颈在master节点上。

对于以上问题，李沐的第三代parameter server给出了新解决方法。

这是Parameter Server的新架构

![](https://pica.zhimg.com/v2-622874fc4d30a12de71b7678068a97fe_b.webp?consumer=ZHI_MENG)

可以看到，PS分为两大部分：server group和多个worker group，另外resource manager负责总体的资源分配调度。
- server group内部包含了多个server node，每个node维护了一部分参数，server manager负责分配和维护server资源；
- 每个worker group对应一个application，worker group只与对应的server node 通信。

在这里，我们解决了**热点**这个问题。

下图描述了PS并行训练的流程

![PS并行训练流程示意图](https://pic4.zhimg.com/v2-fbde5a01a296542047c5492346391467_b.webp?consumer=ZHI_MENG)

概括为以下步骤：

1. 每个worker载入一部分的训练数据
2. worker从server节点pull完整的、最新的模型参数
3. 计算梯度
4. push梯度到server上
5. server节点汇总梯度，更新参数
6. goto step2

以下是两种更新梯度的方式

![同步更新梯度](https://pic1.zhimg.com/v2-d68b6f8e6d57a7bed9c8fd5e85cca56c_b.gif?consumer=ZHI_MENG)

![异步更新梯度](https://pic4.zhimg.com/v2-75dc0f59d2630e85a79090c1c7cd7dd5_b.gif?consumer=ZHI_MENG)


对于梯度的更新，PS采取的方法是**异步非阻断式**。
以下图为例，在做iter11的时候，iter10的push&pull并没有结束，也就是说，最新的参数还没有拉取到本地上，worker仍然是使用iter10的参数进行梯度运算。

![](https://pic3.zhimg.com/v2-7cd40e19dd7adba7d1d2e9ee26b106fa_b.webp?consumer=ZHI_MENG)

提高了计算资源的利用率，解决木桶效应这个问题。当然，这都是取舍，虽然加快了训练速度，但带来的是模型一致性的损失，也就是说**分布式并行训练的结果与单点串行训练的结果是不一致的**，这会影响模型的收敛速度。

在同步和异步之间，trade-off的方案就是**最大延迟**，规定在k轮迭代内，worker必须停下来等待pull操作的完成。
