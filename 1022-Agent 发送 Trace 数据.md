# Agent 发送 Trace 数据

**本文主要基于 SkyWalking 3.2.6 正式版**

- [1. 概述](http://www.iocoder.cn/SkyWalking/agent-send-trace/)
- \2. TraceSegmentServiceClient
  - [2.1 实现 BootService 接口](http://www.iocoder.cn/SkyWalking/agent-send-trace/)
  - [2.2 实现 GRPCChannelListener 接口](http://www.iocoder.cn/SkyWalking/agent-send-trace/)
  - [2.3 实现 TracingContextListener 接口](http://www.iocoder.cn/SkyWalking/agent-send-trace/)
  - [2.4 实现 IConsumer 接口](http://www.iocoder.cn/SkyWalking/agent-send-trace/)
- [666. 彩蛋](http://www.iocoder.cn/SkyWalking/agent-send-trace/)

------

------

# 1. 概述

分布式链路追踪系统，链路的追踪大体流程如下：

1. Agent 收集 Trace 数据。
2. **Agent 发送 Trace 数据给 Collector** 。
3. Collector 接收 Trace 数据。
4. Collector 存储 Trace 数据到存储器，例如，数据库。

本文主要分享【第二部分】 **SkyWalking Agent 发送 Trace 数据**。

考虑到减少**外部组件**的依赖，Agent 收集到 Trace 数据后，不是写入外部消息队列( 例如，Kafka )或者日志文件，而是 Agent 写入**内存消息队列**，**后台线程**【**异步**】发送给 Collector 。

本文涉及的类非常少，如下图所示：

![img](https://static.iocoder.cn/images/SkyWalking/2020_10_05/01.png)

# 2. TraceSegmentServiceClient

[`org.skywalking.apm.agent.core.remote.TraceSegmentServiceClient`](https://github.com/YunaiV/skywalking/blob/0fe81f39054634a0b9a04fca41e6889f0e175b4a/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/remote/TraceSegmentServiceClient.java) ，TraceSegment 发送服务客户端。它是一个服务，也是一个客户端，负责将 TraceSegment **异步**发送到 Collector 。

我们先来看看 TraceSegmentServiceClient 的**属性**：

- `TIMEOUT` **静态**属性，发送等待超时时长，单位：毫秒。
- `lastLogTime` 属性，最后打印日志时间。该属性主要用于开发调试。
- `segmentUplinkedCounter` 属性，TraceSegment 发送数量。
- `segmentAbandonedCounter` 属性，TraceSegment 被丢弃数量。在 Agent 未连接上 Collector 时，产生的 TraceSegment 将被丢弃。
- `carrier` 属性，内存队列。在 [《SkyWalking 源码分析 —— DataCarrier 异步处理库》](http://www.iocoder.cn/SkyWalking/data-carrier/?self) 有对 DataCarrier 的详细解析。
- `serviceStub` 属性，**非阻塞** Stub 。
- `status` 属性，连接状态。

下面，我们来介绍 TraceSegmentServiceClient 实现的**接口**以及对应的方法。

## 2.1 实现 BootService 接口

[`#beforeBoot()`](https://github.com/YunaiV/skywalking/blob/0fe81f39054634a0b9a04fca41e6889f0e175b4a/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/remote/TraceSegmentServiceClient.java#L85) 方法，代码如下：

- 第 86 行：调用 `GRPCChannelManager#addChannelListener(this)` 方法，将自己添加到 GRPCChannelManager 中，作为一个监听器，从而调用 `#statusChanged(GRPCChannelStatus)` 方法，实现对**连接状态**( `status` )的监听处理。

[`#boot()`](https://github.com/YunaiV/skywalking/blob/0fe81f39054634a0b9a04fca41e6889f0e175b4a/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/remote/TraceSegmentServiceClient.java#L90) 方法，代码如下：

- 第 95 至 97 行：创建 DataCarrier 对象，作为**内存队列**，并设置自己作为消费者，从而调用 `#consume(List<TraceSegment> )` 方法，实现**异步**发送 TraceSegment 到 Collector 。

[`#afterBoot()`](https://github.com/YunaiV/skywalking/blob/0fe81f39054634a0b9a04fca41e6889f0e175b4a/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/remote/TraceSegmentServiceClient.java#L101) 方法，代码如下：

- 第 102 行：调用 `TracingContext.ListenerManager#add(this)` 方法，将自己添加到 ListenerManager 中，作为一个监听器，从而调用 `#afterFinished(TraceSegment)` 方法，实现**收集**到新的 TraceSegment ，添加到内存队列。

[`#shutdown()`](https://github.com/YunaiV/skywalking/blob/0fe81f39054634a0b9a04fca41e6889f0e175b4a/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/remote/TraceSegmentServiceClient.java#L106) 方法，代码如下：

- 第 107 行：调用 `DataCarrier#shutdownConsumers()` 方法，停止消费。

## 2.2 实现 GRPCChannelListener 接口

[`#statusChanged(GRPCChannelStatus)`](https://github.com/YunaiV/skywalking/blob/0fe81f39054634a0b9a04fca41e6889f0e175b4a/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/remote/TraceSegmentServiceClient.java#L209) 方法，代码如下：

- 第 211 至 214 行：连接成功，创建 Stub 对象。
- 第 215 行：记录连接状态。

## 2.3 实现 TracingContextListener 接口

[`#afterFinished(TraceSegment)`](https://github.com/YunaiV/skywalking/blob/0fe81f39054634a0b9a04fca41e6889f0e175b4a/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/remote/TraceSegmentServiceClient.java#L196) 方法，代码如下：

- 第 197 至 199 行：当 `TraceSegment.ignore = true` 时，忽略该 TraceSegment 。
- 第 201 行：提交 TraceSegment 到内存队列。

## 2.4 实现 IConsumer 接口

[`#consume(List)`](https://github.com/YunaiV/skywalking/blob/0fe81f39054634a0b9a04fca41e6889f0e175b4a/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/remote/TraceSegmentServiceClient.java#L116) 方法，代码如下：

- —— **连接中** ——

- 第 119 行：创建 [`org.skywalking.apm.agent.core.remote。GRPCStreamServiceStatus`](https://github.com/YunaiV/skywalking/blob/0fe81f39054634a0b9a04fca41e6889f0e175b4a/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/remote/GRPCStreamServiceStatus.java) 对象。

- 第 122 至 141 行：创建 StreamObserver 对象。在下面，我们可以看到 Agent 发送 TraceSegment 给 Collector 是

  非阻塞

  的方式，通过该对象，

  观察

  执行结果。

  - 第 130 行 || 第 139 行：当发生错误或者完成时，调用 [`GRPCStreamServiceStatus#finished()`](https://github.com/YunaiV/skywalking/blob/0fe81f39054634a0b9a04fca41e6889f0e175b4a/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/remote/GRPCStreamServiceStatus.java#L44) 方法，**标记完成**。为什么呢？下面会看到。
  - 第 134 行：调用 `GRPCChannelManager#reportError(Throwable)` 方法，汇报错误。如果是连接错误，GRPCChannelManager 会负责断开重连。

- 第 144 至 151 行：逐条

  非阻塞

  发送 TraceSegment 请求。

  - 第 146 行：调用

     

    `TraceSegment#transform()`

     

    方法，将 TraceSegment 转换成

     

    ```
    org.skywalking.apm.network.proto.UpstreamSegment
    ```

     

    对象，用于 gRPC 传输，参见

     

    `TraceSegmentService.proto`

     

    的数据结构定义。

    - [`DistributedTraceId#toUniqueId()`](https://github.com/YunaiV/skywalking/blob/0fe81f39054634a0b9a04fca41e6889f0e175b4a/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/context/ids/DistributedTraceId.java#L65)
    - [`ID#transform()`](https://github.com/YunaiV/skywalking/blob/0fe81f39054634a0b9a04fca41e6889f0e175b4a/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/context/ids/ID.java#L120)
    - [`AbstractTracingSpan#transform()`](https://github.com/YunaiV/skywalking/blob/0fe81f39054634a0b9a04fca41e6889f0e175b4a/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/context/trace/AbstractTracingSpan.java#L262)
    - [`ExitSpan#transform()`](https://github.com/YunaiV/skywalking/blob/0fe81f39054634a0b9a04fca41e6889f0e175b4a/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/context/trace/ExitSpan.java#L129)
    - [`LogDataEntity#transform()`](https://github.com/YunaiV/skywalking/blob/0fe81f39054634a0b9a04fca41e6889f0e175b4a/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/context/trace/LogDataEntity.java#L72)
    - [`TraceSegmentRef#transform()`](https://github.com/YunaiV/skywalking/blob/0fe81f39054634a0b9a04fca41e6889f0e175b4a/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/context/trace/TraceSegmentRef.java#L160)
    - [`KeyValuePair#transform()`](https://github.com/YunaiV/skywalking/blob/0fe81f39054634a0b9a04fca41e6889f0e175b4a/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/context/util/KeyValuePair.java#L45)

  - 第 154 行：调用 `StreamObserver#onCompleted()` 方法，标记全部请求发送完成。

  - 第 157 至 159 行：调用

     

    `GRPCStreamServiceStatus#wait4Finish(maxTimeout)`

     

    方法，等待 Collector 处理完成。这就是为什么上面需要调用

     

    ```
    GRPCStreamServiceStatus#finished()
    ```

     

    方法。完成后，记录数量到

     

    ```
    segmentUplinkedCounter
    ```

     

    。

    - **注意**，此处若等待完成超时，TraceSegment **依然**在发送，或者被 Collector 处理中，直到最终的成功或失败。

- —— **未连接** ——

- 第 161 行：记录数量到 `segmentAbandonedCounter` 。

- —— **ALL** ——

- 调用 [`#printUplinkStatus()`](https://github.com/YunaiV/skywalking/blob/0fe81f39054634a0b9a04fca41e6889f0e175b4a/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/remote/TraceSegmentServiceClient.java#L170) 方法，每三十秒，打印一次 segmentUplinkedCounter 和 segmentAbandonedCounter 数据。主要用于开发调试。另外，该方法会重置 `segmentUplinkedCounter` 和 `segmentAbandonedCounter` 计数。

ps：目前 DataCarrier **最长**每 20 秒消费一次。

[`#onError(List, Throwable)`](https://github.com/YunaiV/skywalking/blob/0fe81f39054634a0b9a04fca41e6889f0e175b4a/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/remote/TraceSegmentServiceClient.java#L186) 方法，当消费发生**异常**时，打印日志。

# 666. 彩蛋