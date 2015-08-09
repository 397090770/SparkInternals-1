# RDD 内部结构
## 从 RDD 的名字说起

RDD 作为 Apache Spark 中最为重要的一类数据抽象，同时也是 Apache Spark 程序开发者接触最多的数据结构，自然而然地，也就成为我理解 Apache Spark 工作原理的最佳入口之一。


1. 数据集：顾名思义，说明 RDD 是数据集合的抽象，从外部来看，RDD 的确可被看待成带扩展特性（如容错性等）的数据集合。
2. 分布式：数据的计算并非只局限于单个节点，而是多个节点之间协同计算得到。
3. 弹性：RDD 内部数据是只读的，但 RDD 却具有弹性这一特性，实际上，RDD 可以在不改变内部存储数据记录的前提下，去调整并行计算计算单元的划分结构，弹性这一特性，也是为并行计算服务的。

我把 RDD 归纳为一句话：能够进行并行计算的数据集，其中最重要的是__并行计算__这一特征，基于它，可以进一步往下思考 RDD 在设计上的一些问题。



传统方法的容错机制有两种，一是创建__数据检查点__，即将某个节点的数据保存在存储介质当中，二是__记录更新__，即记录下内部数据所遭遇过的所有的更新。对于前者，在网络中传输与复制数据集的带宽开销显然是非常庞大的，对于后者，如果要记录每一个数据记录的所有更新，成本自然也是不小。使用过 RDD 做过开发的话，自然知道 Apache Spark 最终采用的是第二种办法，而为了避免巨大的开销，RDD 只支持__粗粒度__的转换操作，一个操作会应用到多个数据而非单个记录。那么，在 RDD 内部，应该如何去记录数据的更新，丢失的数据又是通过何种方式恢复的呢？

本章后面的小节都将是围着上面的这些问题进行展开，每一节都会回答一个或者多个问题，探索这些问题答案的过程中会配合解析相应的源码实现。我希望通过这种方式，能够从整体的角度去理解 Apache Spark 开发者们如此设计 RDD 的目的，而非单纯机械地一行一行去解释代码的含义。

## RDD 内部接口
在 Apache Spark 源码级别，`RDD` 是一个抽象类，我们所使用的 `RDD` 实例，都是 `RDD` 的子类，例如执行 `map` 转换操作之后可以得到一个 `MappedRDD` 实例，执行 `groupByKey` 转换操作之后可以得到一个 `ShuffledRDD` 实例。不同的 `RDD` 子类会根据实际需求实现各自的功能，但无论如何，一个 `RDD` 内部都会包含如下几类接口的全部或者一部分，在后面的小节中，我们会看到，这些接口究竟是如何为实现我们的并行计算目标服务的。