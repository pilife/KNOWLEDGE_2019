# Hadoop Learning RoadMap

## PLAN

关于Hadoop的学习，本人计划分为***三个阶段***：

- Step 1: 基本使用和基本原理

  学习目标: 

  - Hdfs: 
    - 熟悉API的使用
    - 了解基本架构和各个模块的功能
  - MapReduce
    - 熟悉API的使用
    - 了解partition，shuffle，sort等工作原理，可以自己在纸上完整个画完mapreduce的流程( the more detailed, the better)
  - Yarn
    - //todo

- Step 2: 源代码分析（将基本概念映射到代码）

  学习目标：对hadoop源代码整体架构和局部的很多细节，有了一定的了解。比如你知道MapReduce Scheduler是怎样实现的，MapReduce shuffle过程中，map端做了哪些事情，reduce端做了哪些事情，是如何实现的，等等。这个阶段完成后，当你遇到问题或者困惑点时，可以迅速地在Hadoop源代码中定位相关的类和具体的函数，通过阅读源代码解决问题，这时候，hadoop源代码变成了你解决问题的参考书。

  学习建议：

  - 首先，你要摸清hadoop的代码模块，知道client，master，slave各自对应的模块（hadoop中核心系统都是master/slave架构，非常类似），并在阅读源代码过程中，时刻谨记你当前阅读的代码属于哪一个模块，会在哪个组件中执行；
  - 之后你需要摸清各个组件的交互协议，也就是分布式中的RPC，这是hadoop自己实现的，你需要对hadoop RPC的使用方式有所了解，然后看各模块间的RPC protocol，到此，你把握了系统的骨架，这是接下来**阅读源代码的基础**；
  - 接着，你要选择一个模块开始阅读，我一般会选择Client，这个模块相对简单些，会给自己增加信心，为了在阅读代码过程中，不至于迷失自己，建议在纸上画出类的调用关系，边看边画，我记得我阅读hadoop源代码时，花了一叠纸。注意，看源代码过程中，很容易烦躁不安，建议经常起来走走，不要把自己逼得太紧。
  - 在这个阶段，建议大家多看一些源代码分析博客和书籍，比如《Hadoop技术内幕》系列丛书（相关网站：[Hadoop技术内幕](https://link.zhihu.com/?target=http%3A//hadoop123.com/)）就是最好的参考资料。借助这些博客和书籍，你可以在前人的帮助下，更快地学习hadoop源代码，节省大量时间，注意，目前博客和书籍很多，建议大家广泛收集资料，找出最适合自己的参考资料。

- Step 3: 根据需求修改源代码（可作为毕业论文的方向）

  ​

---

## 基本使用和基本原理

### Hdfs

### Map Reduce

### Yarn

Reference: [Hadoop2原理介绍](https://blog.csdn.net/rxt2012kc/article/details/72644873)

---

## 源代码分析



---

## 改进源代码（可作为发论文的方向）



---

# QA:

1. 为什么我要阅读Hadoop源码？

   阅读hadoop源代码的目的不一定非是工作的需要，你可以把他看成一种修养，通过阅读hadoop源代码，加深自己对分布式系统的理解，培养自己踏实做事的心态。——摘自董西成（Hadoop内幕系列丛书作者）的知乎回答