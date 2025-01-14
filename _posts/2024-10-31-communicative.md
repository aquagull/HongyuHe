---
title: "通信原语"
date: 2024-10-31
categories: 
  - distributed training
---
### Broadcast
 broadcast，表示某一个节点将自己的数据发送给其他节点。
 
![](https://pica.zhimg.com/v2-c9aa7762a6ec00d370c58de183441362_b.jpg)
![](https://picx.zhimg.com/v2-1ff295f93679ebe9a03ad510259ead8b_b.jpg)

### Scatter
scatter，将某个节点，<mark>数据的不同部分</mark>，按需发送给其他节点。

![](https://pica.zhimg.com/v2-f17bd118677f919e255d5b1689fc66dc_b.jpg)

### Reduce
reduce，从多个节点获得数据，这些数据进行<mark>简单运算</mark>后，只保存在一个节点上。

![](https://pic4.zhimg.com/v2-c7bdad601780f9798a62c2dfb1bbef4d_b.jpg)

### All Reduce
all reduce，则是在<mark>所有节点上</mark>进行同样的reduce操作。这也可以通过单节点的Reduce + Broadcast来完成。

![](https://pic3.zhimg.com/v2-80b1bd60a2fdefb19f792fdf193c6d76_b.jpg)

### Gather
gather，从多个节点收集数据到一个节点。

![](https://picx.zhimg.com/v2-dc3fcf248c39b4a76947bcea140840d1_b.jpg)

### All Gather
all gather，相当于gather + broadcast。

![](https://pic2.zhimg.com/v2-831e0b04646c78f9e74bf4f29c35b8af_b.jpg)

### Reduce Scatter
reduce scatter，先将各个节点的数据分块，然后按块做reduce，此时做reduce的这个节点再将这些块发出去。

![](https://pic3.zhimg.com/v2-14cdd631faae00452885a116dd36737c_b.jpg)
