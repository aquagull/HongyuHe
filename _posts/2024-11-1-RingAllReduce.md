---
title: "Ring AllReduce"
date: 2024-11-1
mathjax: true
categories: [distributed training]
---

## 传统多卡并行

假设我们有$N$GPU进行并行训练，每张GPU上的模型梯度大小为$K$。在训练中，我们需要计算所有GPU上的梯度的平均值，然后更新参数。为此，最简单的方法，就是指定一张GPU为master，其余GPU为slave。过程是，反向传播后，master汇聚所有slave的梯度，求平均值，然后将结果发回slave。

![](https://blog-assets.unvs.cc/2021/05/ring-allreduce-fig1.webp)

这里，我们计算一下每张GPU上的发送和接受的数据量大小：
- 对于master：
    - 由于所有slave都需要发送完整梯度给master，因此收到的数据量为$(N - 1) \cdot K$。
    - master还需要将reduce后的梯度发送到每个slave上，因此其发送的数据量也为$(N - 1) \cdot K$。

- 对于slave：
    - 发送和接受的数据量均为$K$。

这种简单的并行方式有两个缺点：
1. master和slave的通信量不一样，然而在实际中，GPU之间的通信带宽是相差不大的。这种情况下，master会成为一个通信瓶颈。
2. 每个GPU的计算量是不均等的，master还负责对梯度求平均值，在此时间段，slave只能空转。

Pytorch中的`torch.nn.DataParallel`就是采用这种方式在GPU之间汇聚和发送梯度的。在GPU数量较多的情况下，通信会成为瓶颈，加速比会远低于$N$。

## Ring AllReduce 多卡并行
在2017年，百度引入了Ring AllReduce算法，解决了上面两个缺点。

在该算法中，所有GPU组成一个环。算法总共包含两个步骤，分别为scatter-reduce和allgather。

![](https://blog-assets.unvs.cc/2021/05/ring-allreduce-fig2.webp)

### scatter-reduce
为了保证每个GPU通信量和计算量尽可能靠近，模型的参数会按照GPU数量$N$来划分成大小相等的块，然后将每一数据块通过$N - 1$次数据传输汇聚到一个GPU上。

![](https://blog-assets.unvs.cc/2021/05/ring-allreduce-fig3.webp)

### all gather
接下来我们进行allgather步骤，将这些黄色数据覆盖所有GPU，总计需要$N - 1$次数据传输。

![](https://blog-assets.unvs.cc/2021/05/ring-allreduce-fig4.webp)

下面我们计算一下每块GPU的通信量，可以明显的感觉到，Ring AllReduce算法中每块GPU都是对等的，因此通信量大小都是一致的：
- scatter-reduce步骤：每一回，每块GPU从左邻居接受$K \div N$的数据，也向其右邻居发送相同大小的数据，重复$N - 1$次。

- allgather步骤：也跟scatter-reduce步骤通信量相同。

最终，每块GPU的通信量就是$2 \cdot (N - 1) \cdot (K \div N)$。
从最终的表达式分析，可以得到：
- 每块GPU的通信量与GPU的数目无关。
- 整个算法受限于最慢的GPU。 

PyTorch 中的`torch.nn.parallel.DistributedDataParallel`就是采用了 Ring Allreduce 算法。