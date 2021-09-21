# DataCarrier 异步处理库

**本文主要基于 SkyWalking 3.2.6 正式版**

- [1. 概述](http://www.iocoder.cn/SkyWalking/data-carrier/)
- \2. buffer
  - [2.1 Buffer](http://www.iocoder.cn/SkyWalking/data-carrier/)
  - [2.2 Channels](http://www.iocoder.cn/SkyWalking/data-carrier/)
- [3. partition](http://www.iocoder.cn/SkyWalking/data-carrier/)
- \4. consumer
  - [4.1 IConsumer](http://www.iocoder.cn/SkyWalking/data-carrier/)
  - [4.2 ConsumerThread](http://www.iocoder.cn/SkyWalking/data-carrier/)
  - [4.3 ConsumerPool](http://www.iocoder.cn/SkyWalking/data-carrier/)
- [4. DataCarrier](http://www.iocoder.cn/SkyWalking/data-carrier/)
- [666. 彩蛋](http://www.iocoder.cn/SkyWalking/data-carrier/)

------

------

# 1. 概述

本文主要分享 **SkyWalking DataCarrier 异步处理库**。

基于生产者消费者的模式，大体结构如下图：![img](https://static.iocoder.cn/images/SkyWalking/2020_09_15/01.png)

- 实际项目中，没有 Producer 这个类。所以本文提到的 Producer ，更多的是一种**角色**。

下面我们来看看整体的项目结构，如下图所示 ：![img](https://static.iocoder.cn/images/SkyWalking/2020_09_15/02.png)

# 2. buffer

`org.skywalking.apm.commons.datacarrier.buffer` 包，主要包含 Channels 、Buffer 两个类。Channels 是 Buffer 数组的封装。

## 2.1 Buffer

`org.skywalking.apm.commons.datacarrier.buffer.Buffer` ，缓存区。

- [`buffer`](https://github.com/YunaiV/skywalking/blob/98217271d1b4d2075871b3e262c9b99bb9c1f657/apm-commons/apm-datacarrier/src/main/java/org/skywalking/apm/commons/datacarrier/buffer/Buffer.java#L35) 属性，缓冲数组。Producer 保存的数据到 `buffer` 里。
- [`strategy`](https://github.com/YunaiV/skywalking/blob/98217271d1b4d2075871b3e262c9b99bb9c1f657/apm-commons/apm-datacarrier/src/main/java/org/skywalking/apm/commons/datacarrier/buffer/Buffer.java#L39) ，缓冲策略( [`org.skywalking.apm.commons.datacarrier.buffer.BufferStrategy`](https://github.com/YunaiV/skywalking/blob/98217271d1b4d2075871b3e262c9b99bb9c1f657/apm-commons/apm-datacarrier/src/main/java/org/skywalking/apm/commons/datacarrier/buffer/BufferStrategy.java#L26) ) 。
- [`index`](https://github.com/YunaiV/skywalking/blob/98217271d1b4d2075871b3e262c9b99bb9c1f657/apm-commons/apm-datacarrier/src/main/java/org/skywalking/apm/commons/datacarrier/buffer/Buffer.java#L43) 属性，递增位置( [`org.skywalking.apm.commons.datacarrier.common.AtomicRangeInteger`](https://github.com/YunaiV/skywalking/blob/98217271d1b4d2075871b3e262c9b99bb9c1f657/apm-commons/apm-datacarrier/src/main/java/org/skywalking/apm/commons/datacarrier/common/AtomicRangeInteger.java) )。

Buffer 在保存数据时，把 `buffer` 作为一个 “**环**“，使用 `index` 记录最后存储的位置，不断向下，**循环**存储到 `buffer` 中。通过这样的方式，带来良好的存储性能，避免扩容问题。But ，存储会存在冲突的问题：`buffer` 写入位置，暂未被消费，已经存在值。此时，根据不同的 BufferStrategy 进行处理。整体流程见 [`#save(data)`](https://github.com/YunaiV/skywalking/blob/98217271d1b4d2075871b3e262c9b99bb9c1f657/apm-commons/apm-datacarrier/src/main/java/org/skywalking/apm/commons/datacarrier/buffer/Buffer.java#L61) 方法。

当 Buffer 被 Consumer 消费时，被调用 [`#obtain(start, end)`](https://github.com/YunaiV/skywalking/blob/98217271d1b4d2075871b3e262c9b99bb9c1f657/apm-commons/apm-datacarrier/src/main/java/org/skywalking/apm/commons/datacarrier/buffer/Buffer.java#L88) 方法，获得数据并清空。为什么会带 `start` 、`end` 方法参数呢？下文揭晓答案。

## 2.2 Channels

`org.skywalking.apm.commons.datacarrier.buffer.Channels` ，内嵌**多个** Buffer 的通道。

- [`bufferChannels`](https://github.com/YunaiV/skywalking/blob/98217271d1b4d2075871b3e262c9b99bb9c1f657/apm-commons/apm-datacarrier/src/main/java/org/skywalking/apm/commons/datacarrier/buffer/Channels.java#L37) 属性，Buffer 数组。
- [`dataPartitioner`](https://github.com/YunaiV/skywalking/blob/98217271d1b4d2075871b3e262c9b99bb9c1f657/apm-commons/apm-datacarrier/src/main/java/org/skywalking/apm/commons/datacarrier/buffer/Channels.java#L38) 属性，数据分区( [`org.skywalking.apm.commons.datacarrier.partition.IDataPartitioner`](https://github.com/YunaiV/skywalking/blob/98217271d1b4d2075871b3e262c9b99bb9c1f657/apm-commons/apm-datacarrier/src/main/java/org/skywalking/apm/commons/datacarrier/partition/IDataPartitioner.java) )。
- [`strategy`](https://github.com/YunaiV/skywalking/blob/98217271d1b4d2075871b3e262c9b99bb9c1f657/apm-commons/apm-datacarrier/src/main/java/org/skywalking/apm/commons/datacarrier/buffer/Buffer.java#L39) 属性，缓冲策略( [`org.skywalking.apm.commons.datacarrier.buffer.BufferStrategy`](https://github.com/YunaiV/skywalking/blob/98217271d1b4d2075871b3e262c9b99bb9c1f657/apm-commons/apm-datacarrier/src/main/java/org/skywalking/apm/commons/datacarrier/buffer/Channels.java#L42) ) 。

Channels 在保存数据时，相比 Buffer ，从 `buffer` 变成了多 `buffer` ，因此需要先选一个 `buffer` 。通过使用不同的 IDataPartitioner 实现类，进行 Buffer 的选择。当缓冲策略为 `BufferStrategy.IF_POSSIBLE` 时，根据 IDataPartitioner 定义的重试次数，进行多次保存数据直到成功。整体流程见 [`#save(data)`](https://github.com/YunaiV/skywalking/blob/98217271d1b4d2075871b3e262c9b99bb9c1f657/apm-commons/apm-datacarrier/src/main/java/org/skywalking/apm/commons/datacarrier/buffer/Channels.java#L62) 方法。

# 3. partition

[`org.skywalking.apm.commons.datacarrier.partition.IDataPartitioner`](https://github.com/YunaiV/skywalking/blob/98217271d1b4d2075871b3e262c9b99bb9c1f657/apm-commons/apm-datacarrier/src/main/java/org/skywalking/apm/commons/datacarrier/partition/IDataPartitioner.java) ，数据分配者**接口**。定义了如下方法：

- [`#partition(total, data)`](https://github.com/YunaiV/skywalking/blob/98217271d1b4d2075871b3e262c9b99bb9c1f657/apm-commons/apm-datacarrier/src/main/java/org/skywalking/apm/commons/datacarrier/partition/IDataPartitioner.java#L37) **接口**方法，获得数据被分配的分区位置。
- [`#maxRetryCount()`](https://github.com/YunaiV/skywalking/blob/98217271d1b4d2075871b3e262c9b99bb9c1f657/apm-commons/apm-datacarrier/src/main/java/org/skywalking/apm/commons/datacarrier/partition/IDataPartitioner.java#L46) **接口**方法，获得最大重试次数。

IDataPartitioner 目前有两个子类实现：

- [ProducerThreadPartitioner](https://github.com/YunaiV/skywalking/blob/98217271d1b4d2075871b3e262c9b99bb9c1f657/apm-commons/apm-datacarrier/src/main/java/org/skywalking/apm/commons/datacarrier/partition/ProducerThreadPartitioner.java) ，基于线程编号分配策略的数据分配者实现类。
- [SimpleRollingPartitioner](https://github.com/YunaiV/skywalking/blob/98217271d1b4d2075871b3e262c9b99bb9c1f657/apm-commons/apm-datacarrier/src/main/java/org/skywalking/apm/commons/datacarrier/partition/SimpleRollingPartitioner.java) ，基于顺序分配策略的数据分配者实现类。

# 4. consumer

`org.skywalking.apm.commons.datacarrier.consumer` 包，主要包含 ConsumerPool 、ConsumerThread 、IConsumer 三个类。

- ConsumerThread 使用 IConsumer ，消费数据
- ConsumerPool 是 ConsumerThread 的线程池封装

## 4.1 IConsumer

`org.skywalking.apm.commons.datacarrier.consumer.IConsumer` ，消费者接口。定义了如下方法：

- [`#init()`](https://github.com/YunaiV/skywalking/blob/fbdce6d3c0fe456629a2eef40184a3dbc9df812c/apm-commons/apm-datacarrier/src/main/java/org/skywalking/apm/commons/datacarrier/consumer/IConsumer.java#L33) **接口**方法，初始化消费者。
- [`#consume(List)`](https://github.com/YunaiV/skywalking/blob/fbdce6d3c0fe456629a2eef40184a3dbc9df812c/apm-commons/apm-datacarrier/src/main/java/org/skywalking/apm/commons/datacarrier/consumer/IConsumer.java#L40) **接口**方法，批量消费消息。
- [`#onError(List, Throwable)`](https://github.com/YunaiV/skywalking/blob/fbdce6d3c0fe456629a2eef40184a3dbc9df812c/apm-commons/apm-datacarrier/src/main/java/org/skywalking/apm/commons/datacarrier/consumer/IConsumer.java#L48) **接口**方法，处理当消费发生异常。
- [`#onExit()`](https://github.com/YunaiV/skywalking/blob/fbdce6d3c0fe456629a2eef40184a3dbc9df812c/apm-commons/apm-datacarrier/src/main/java/org/skywalking/apm/commons/datacarrier/consumer/IConsumer.java#L53) **接口**方法，处理当消费结束。此处的结束时，ConsumerThread 关闭。

我们在使用时，自定义 Consumer 类，实现 IConsumer 接口。例如：[RemoteMessageConsumer](https://github.com/YunaiV/skywalking/blob/fbdce6d3c0fe456629a2eef40184a3dbc9df812c/apm-collector/apm-collector-remote/collector-remote-grpc-provider/src/main/java/org/skywalking/apm/collector/remote/grpc/service/GRPCRemoteClient.java#L93) 。

## 4.2 ConsumerThread

`org.skywalking.apm.commons.datacarrier.consumer.ConsumerThread` ，继承 `java.lang.Thread` ，消费线程。

- [`running`](https://github.com/YunaiV/skywalking/blob/fbdce6d3c0fe456629a2eef40184a3dbc9df812c/apm-commons/apm-datacarrier/src/main/java/org/skywalking/apm/commons/datacarrier/consumer/ConsumerThread.java#L36) 属性，是否运行中。

- [`consumer`](https://github.com/YunaiV/skywalking/blob/fbdce6d3c0fe456629a2eef40184a3dbc9df812c/apm-commons/apm-datacarrier/src/main/java/org/skywalking/apm/commons/datacarrier/consumer/ConsumerThread.java#L40) 属性，消费者对象。

- `dataSources`

   

  属性，消费消息的数据源(

   

  DataSource

   

  )数组。一个 ConsumerThread ，可以消费多个 Buffer ，并且单个 Buffer 消费的分区范围可配置，即一个 Buffer 可以被多个 ConsumerThread 同时无冲突的消费。在

   

  「4.3 ConsumerPool」

   

  详细解析 ConsumerThread 分配 Buffer 的方式。

  - [`#addDataSource(sourceBuffer, start, end)`](https://github.com/YunaiV/skywalking/blob/fbdce6d3c0fe456629a2eef40184a3dbc9df812c/apm-commons/apm-datacarrier/src/main/java/org/skywalking/apm/commons/datacarrier/consumer/ConsumerThread.java#L60) 方法，添加 Buffer 部分范围。
  - [`#addDataSource(sourceBuffer)`](https://github.com/YunaiV/skywalking/blob/fbdce6d3c0fe456629a2eef40184a3dbc9df812c/apm-commons/apm-datacarrier/src/main/java/org/skywalking/apm/commons/datacarrier/consumer/ConsumerThread.java#L69) 方法，添加 Buffer 全部范围。

[`#run()`](https://github.com/YunaiV/skywalking/blob/fbdce6d3c0fe456629a2eef40184a3dbc9df812c/apm-commons/apm-datacarrier/src/main/java/org/skywalking/apm/commons/datacarrier/consumer/ConsumerThread.java#L74) **实现**方法，**不断**、**批量**的消费数据。代码如下：

- 第 78 至 88 行：

  不断

  消费，直到线程关闭(

   

  `#shutdown()`

   

  )。

  - 第 80 行：调用 [`#consume()`](https://github.com/YunaiV/skywalking/blob/fbdce6d3c0fe456629a2eef40184a3dbc9df812c/apm-commons/apm-datacarrier/src/main/java/org/skywalking/apm/commons/datacarrier/consumer/ConsumerThread.java#L103) 方法，**批量**消费数据。
  - 第 82 至 87 行：当未消费到数据，说明 `dataSources` 为空，等待 20 ms ，避免 CPU 空跑。

- 第 93 行：当线程关闭，调用 `#consume()` 方法，消费完 `dataSources` 剩余的数据。

- 第 95 行：调用 `IConsumer#onExit()` 方法，处理当消费结束。

[`#consume()`](https://github.com/YunaiV/skywalking/blob/fbdce6d3c0fe456629a2eef40184a3dbc9df812c/apm-commons/apm-datacarrier/src/main/java/org/skywalking/apm/commons/datacarrier/consumer/ConsumerThread.java#L103) 方法，**批量**消费数据。代码如下：

- 第 107 至 117 行：从 `dataSources` 中，获取要消费的数据。
- 第 120 至 126 行：当有数据可消费时，调用 `IConsumer#consume(List<T>)` 方法。当消费发生异常时，调用 `IConsumer#onError(List<T>, Throwable)` 方法。
- 第 127 行：返回是否有消费数据。

## 4.3 ConsumerPool

`org.skywalking.apm.commons.datacarrier.consumer.ConsumerPool` ，消费者池，提供了对 Channels 启动**指定数量**的 ConsumerThread 进行消费。

- [`running`](https://github.com/YunaiV/skywalking/blob/fbdce6d3c0fe456629a2eef40184a3dbc9df812c/apm-commons/apm-datacarrier/src/main/java/org/skywalking/apm/commons/datacarrier/consumer/ConsumerPool.java#L39) 属性，是否运行中。
- [`consumerThreads`](https://github.com/YunaiV/skywalking/blob/fbdce6d3c0fe456629a2eef40184a3dbc9df812c/apm-commons/apm-datacarrier/src/main/java/org/skywalking/apm/commons/datacarrier/consumer/ConsumerPool.java#L43) 属性，ConsumerThread 数组，通过构造方法的 `num` 参数进行指定。
- [`channels`](https://github.com/YunaiV/skywalking/blob/fbdce6d3c0fe456629a2eef40184a3dbc9df812c/apm-commons/apm-datacarrier/src/main/java/org/skywalking/apm/commons/datacarrier/consumer/ConsumerPool.java#L47) 属性，**数据**通道。
- [`lock`](https://github.com/YunaiV/skywalking/blob/fbdce6d3c0fe456629a2eef40184a3dbc9df812c/apm-commons/apm-datacarrier/src/main/java/org/skywalking/apm/commons/datacarrier/consumer/ConsumerPool.java#L51) 属性，锁。保证 ConsumerPool 启动或关闭时的线程安全。

[`#begin()`](https://github.com/YunaiV/skywalking/blob/fbdce6d3c0fe456629a2eef40184a3dbc9df812c/apm-commons/apm-datacarrier/src/main/java/org/skywalking/apm/commons/datacarrier/consumer/ConsumerPool.java#L96) 方法，启动 ConsumerPool ，进行数据消费。代码如下：

- 第 97 至 99 行：正在运行中，直接返回。
- 第 101 行：获得锁。
- 第 104 行：调用 `#allocateBuffer2Thread()` 方法，将 `channels` 的**多个** Buffer ，分配给 `consumerThreads` 的**多个** ConsumerThread。
- 第 107 至 109 行：启动每个 ConsumerThread ，开始消费。
- 第 112 行：标记正在运行中。
- 第 114 行：释放锁。

[`close()`](https://github.com/YunaiV/skywalking/blob/fbdce6d3c0fe456629a2eef40184a3dbc9df812c/apm-commons/apm-datacarrier/src/main/java/org/skywalking/apm/commons/datacarrier/consumer/ConsumerPool.java#L166) 方法，关闭 ConsumerPool 。代码如下：

- 第 168 行：获得锁。
- 第 169 行：标记不在运行中。
- 第 170 至 172 行：关闭每个 ConsumerThread ，结束消费。
- 第 174 行：释放锁。

------

[`#allocateBuffer2Thread()`](https://github.com/YunaiV/skywalking/blob/fbdce6d3c0fe456629a2eef40184a3dbc9df812c/apm-commons/apm-datacarrier/src/main/java/org/skywalking/apm/commons/datacarrier/consumer/ConsumerPool.java#L122) 方法，将 `channels` 的**多个** Buffer ，分配给 `consumerThreads` 的**多个** ConsumerThread。一共会有三种情况：

- Buffer 数量**等于** ConsumerThread 数量，这个十分好分配，一比一。
- Buffer 数量**大于** ConsumerThread 数量，那么按照 Buffer 数量 `%` ConsumerThread 数量进行分组，分配给 ConsumerThread ，如下图所示：![img](https://static.iocoder.cn/images/SkyWalking/2020_09_15/03.png)
- Buffer 数量**大于** ConsumerThread 数量，那么按照 ConsumerThread 数量 `%` Buffer 数量进行分组，分配给 Buffer 。其中，一个 Buffer 会被**均分**给多个 ConsumerThread ，如下图所示：![img](https://static.iocoder.cn/images/SkyWalking/2020_09_15/04.png)
  - 这个就是为什么 Buffer 里面，提供了 [`Buffer#obtain(start, end)`](https://github.com/YunaiV/skywalking/blob/fbdce6d3c0fe456629a2eef40184a3dbc9df812c/apm-commons/apm-datacarrier/src/main/java/org/skywalking/apm/commons/datacarrier/buffer/Buffer.java#L88) 方法的原因。

# 4. DataCarrier

`org.skywalking.apm.commons.datacarrier.DataCarrier` ，DataCarrier 异步处理库的**入口程序**。通过创建 DataCarrier 对象，使用**生产者消费者的模式**，执行异步执行逻辑。

[构造方法](https://github.com/YunaiV/skywalking/blob/fbdce6d3c0fe456629a2eef40184a3dbc9df812c/apm-commons/apm-datacarrier/src/main/java/org/skywalking/apm/commons/datacarrier/DataCarrier.java#L53) ，代码如下：

- [`channels`](https://github.com/YunaiV/skywalking/blob/fbdce6d3c0fe456629a2eef40184a3dbc9df812c/apm-commons/apm-datacarrier/src/main/java/org/skywalking/apm/commons/datacarrier/DataCarrier.java#L47) 属性，**数据**通道。在构造方法中，我们可以看到默认使用 SimpleRollingPartitioner 作为数据分区分配者，使用 `BufferStrategy.BLOCKING` 作为缓冲策略。
  - [`#setPartitioner(IDataPartitioner)`](https://github.com/YunaiV/skywalking/blob/fbdce6d3c0fe456629a2eef40184a3dbc9df812c/apm-commons/apm-datacarrier/src/main/java/org/skywalking/apm/commons/datacarrier/DataCarrier.java#L66) 方法，设置数据分区分配者。
  - [`#setBufferStrategy(BufferStrategy)`](https://github.com/YunaiV/skywalking/blob/fbdce6d3c0fe456629a2eef40184a3dbc9df812c/apm-commons/apm-datacarrier/src/main/java/org/skywalking/apm/commons/datacarrier/DataCarrier.java#L77) 方法，设置缓冲策略。
- [`channelSize`](https://github.com/YunaiV/skywalking/blob/fbdce6d3c0fe456629a2eef40184a3dbc9df812c/apm-commons/apm-datacarrier/src/main/java/org/skywalking/apm/commons/datacarrier/DataCarrier.java#L53) 方法参数，通道大小。
- [`bufferSize`](https://github.com/YunaiV/skywalking/blob/fbdce6d3c0fe456629a2eef40184a3dbc9df812c/apm-commons/apm-datacarrier/src/main/java/org/skywalking/apm/commons/datacarrier/DataCarrier.java#L53) 方法参数，缓冲区大小。

**设置消费者和消费线程数量**：

- [`#consume(Class>, num)`](https://github.com/YunaiV/skywalking/blob/fbdce6d3c0fe456629a2eef40184a3dbc9df812c/apm-commons/apm-datacarrier/src/main/java/org/skywalking/apm/commons/datacarrier/DataCarrier.java#L107)
- [`#consume(IConsumer, num)`](https://github.com/YunaiV/skywalking/blob/fbdce6d3c0fe456629a2eef40184a3dbc9df812c/apm-commons/apm-datacarrier/src/main/java/org/skywalking/apm/commons/datacarrier/DataCarrier.java#L124)

**生产消息**

- [`#produce(data)`](https://github.com/YunaiV/skywalking/blob/fbdce6d3c0fe456629a2eef40184a3dbc9df812c/apm-commons/apm-datacarrier/src/main/java/org/skywalking/apm/commons/datacarrier/DataCarrier.java#L88)

**关闭消费**

- [`#shutdownConsumers()`](https://github.com/YunaiV/skywalking/blob/fbdce6d3c0fe456629a2eef40184a3dbc9df812c/apm-commons/apm-datacarrier/src/main/java/org/skywalking/apm/commons/datacarrier/DataCarrier.java#L138)

# 666. 彩蛋