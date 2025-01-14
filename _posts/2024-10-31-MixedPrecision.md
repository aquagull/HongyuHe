---
title: "Mixed Precision"
date: 2024-10-31
categories: 
  - distributed training
---

为什么需要混合精度训练？使用fp16训练相比于fp32，带来的好处有：
1. **减少内存占用**

2. **减少通讯开销**

3. **加快计算**

但同样的也会带来缺陷：

![](https://pic2.zhimg.com/v2-3b51acf11f4740a32446a780a46f857b_b.jpg)

1. **数据溢出**：fp16只有5位指数位，这代表他的位数范围为($2^{-15}$,$2^{16}$)，而fp32有8位指数位，能表示($2^{-127}$, $2^{128}$)。明显，使用fp16替换fp32会出现overflow和underflow的情况。

2. **舍入误差**：fp16有10位尾数位，$2^{-10} \approx 0.0009765625$，而fp32有23位尾数位$2^{-23} \approx 0.0000001192092896$。

两者trade-off的方案就是**fp16与fp32的混合精度训练**，在训练过程中还可以引入**权重备份（Weight Backup）**、**损失放大（Loss Scaling）**、**精度累加（Precision Accumulated）**这三种优化技术。

### Weight Backup

![](https://pic4.zhimg.com/v2-723c1d3de5f3730e94301735252ac581_r.jpg)

weight backup主要用于解决舍入误差的问题。在训练的后期，模型梯度可能会非常小，使用fp16可能会因为舍入的原因导致本轮训练失败，影响模型收敛。因此，在计算过程中，weight、activations、gradient都是fp16，但额外保存了一份fp32的weight，用于更新weight。weight的更新公式为：

$$
weight_{32} = weight_{32} + \alpha \cdot gradient_{16}
$$

模型的权重属于静态内存占用，训练过程中动态内存往往是静态内存的$3-4$倍。最主要的内存开销减半，总体内存占用也自然是减少的。


### Loss Scaling

loss scaling是为了解决混合精度训练中，梯度过小引起underflow导致的模型收敛差的情况。

其主要思路是：
- Scale up阶段，前向计算得到的loss值扩大$2^k$倍。
- Scale down阶段，反向传播后，梯度缩小$2^k$倍，使用fp32存储。

这个k可以固定设置，也可以动态变化。

### Precision Accumulated

![](https://pic4.zhimg.com/v2-0abb630431816d5797d341b59a38d2d9_b.jpg)

简单说，训练过程中，fp16矩阵相乘，利用fp32来进行中间的累加，然后再将fp32转化成为fp16。

混合精度训练中，还有根据实际的tensor和ops之间的关系建立黑白名单的策略。例如GEMM和CNN卷积操作对于FP16操作特别友好的计算，会把输入的数据和权重转换成FP16进行运算，而softmax、batchnorm等标量和向量在FP32操作好的计算，则是继续使用FP32进行运算。
