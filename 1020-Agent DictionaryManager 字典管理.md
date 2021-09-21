# Agent DictionaryManager 字典管理

摘要: 原创出处 http://www.iocoder.cn/SkyWalking/agent-dictionary/ 「芋道源码」欢迎转载，保留摘要，谢谢！

- [1. 概述](http://www.iocoder.cn/SkyWalking/agent-dictionary/)
- \2. Collector 同步相关 API
  - [2.1 应用的同步 API](http://www.iocoder.cn/SkyWalking/agent-dictionary/)
  - [2.2 操作的同步 API](http://www.iocoder.cn/SkyWalking/agent-dictionary/)
- \3. Agent 调用同步 API
  - [3.1 DictionaryManager](http://www.iocoder.cn/SkyWalking/agent-dictionary/)
- [3.2 PossibleFound](http://www.iocoder.cn/SkyWalking/agent-dictionary/)
- [666. 彩蛋](http://www.iocoder.cn/SkyWalking/agent-dictionary/)

------

------

# 1. 概述

本文主要分享 **Agent DictionaryManager 字典管理**。先来简单了解下字典的定义和用途：

- 字典实际上是一个 Map 映射。目前 Agent 上有两种字典：应用编码与应用编号的映射，操作名与操作编号的映射。
  - 应用的定义：例如，Tomcat 启动的应用，或者程序里访问的 MongoDB 、MySQL 都可以认为是应用。
  - 操作的定义：例如，访问的 URL 地址，Mongo 的执行操作。
- Agent 在每次上传调用链路 Segment 给 Collector 时，Segment 里面需要包含应用和操作相关信息。考虑到减少网络流量，应用编号少于应用编号，操作编号少于操作名。

Agent 字典，会**定时**从 Collector 【**同步**】**需要**( *需要的定义，下文代码会看到* )的字典。

下面，我们分成两个小节，分别从 API 的**实现**与**调用**，分享代码的具体实现。

# 2. Collector 同步相关 API

Collector 同步相关 API 相关有四个接口：

- 2.1 应用的同步 API
- 2.2 操作的同步 API

API 处理的流程大体如下：

![img](https://static.iocoder.cn/images/SkyWalking/2020_09_25/01.png)

## 2.1 应用的同步 API

应用的同步 API ，实际使用的是**应用的注册 API**，在 [「2.1 应用的注册 API」](http://www.iocoder.cn/SkyWalking/register/?self) 有详细解析。

## 2.2 操作的同步 API

我们先来看看 API 的定义，[`DiscoveryService`](https://github.com/YunaiV/skywalking/blob/ba73b05b99a05bb67fd485188a6c6e0a4ad5fe57/apm-network/src/main/proto/DiscoveryService.proto#L11) ，如下图所示：

![img](https://static.iocoder.cn/images/SkyWalking/2020_09_28/01.png)

整体代码和 [「2.1 应用的同步 API」](https://www.iocoder.cn/SkyWalking/agent-dictionary/#) 非常相似，所以本小节，更多的是提供代码的链接地址。

### 2.2.1 ServiceNameDiscoveryServiceHandler#discovery(…)

[`ServiceNameDiscoveryServiceHandler#discovery(ServiceNameCollection, StreamObserver)`](https://github.com/YunaiV/skywalking/blob/0830d985227c42f0e0f3787ebc99a2b197486b69/apm-collector/apm-collector-agent-grpc/collector-agent-grpc-provider/src/main/java/org/skywalking/apm/collector/agent/grpc/handler/ServiceNameDiscoveryServiceHandler.java#L49)，根据操作名数组，查找操作编号数组。

### 2.2.2 IServiceNameService#getOrCreate(…)

`org.skywalking.apm.collector.agent.stream.service.register.IServiceNameService` ，继承 [Service](https://github.com/YunaiV/skywalking/blob/40823179d7228207b06b603b9a1c09dfc4f78593/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/Service.java) 接口，操作名服务接口。

- 定义了 [`#getOrCreate(applicationId, serviceName)`](https://github.com/YunaiV/skywalking/blob/a4db2c4dd5e2adc861e7fb9e9b7b7ffdc57dfb88/apm-collector/apm-collector-agent-stream/collector-agent-stream-define/src/main/java/org/skywalking/apm/collector/agent/stream/service/register/IInstanceIDService.java#L39) **接口**方法，根据应用编号 + 操作名字，获取或创建操作名( ServiceName )，并获得操作编号。

`org.skywalking.apm.collector.agent.stream.worker.register.ServiceNameService` ，实现 IServiceNameService 接口，操作名服务实现类。

- 实现了 [`#getOrCreate(applicationId, serviceName)`](https://github.com/YunaiV/skywalking/blob/0830d985227c42f0e0f3787ebc99a2b197486b69/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/register/ServiceNameService.java#L64) 方法。

### 2.2.3 Graph#start(ServiceName)

在 [`#createServiceNameRegisterGraph()`](https://github.com/YunaiV/skywalking/blob/a9873b9bf07882746bd30f29b3c64f4b44887bf2/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/graph/RegisterStreamGraph.java#L79) 方法中，我们可以看到 ServiceName 对应的 `Graph<ServiceName>` 对象的创建。

- [`org.skywalking.apm.collector.agent.stream.worker.register.ServiceNameRegisterRemoteWorker`](https://github.com/YunaiV/skywalking/blob/a9873b9bf07882746bd30f29b3c64f4b44887bf2/apm-collector/apm-collector-agent-stream/collector-agent-stream-provider/src/main/java/org/skywalking/apm/collector/agent/stream/worker/register/ServiceNameRegisterRemoteWorker.java) ，继承 AbstractRemoteWorker 抽象类，操作名注册远程 Worker 。

- `org.skywalking.apm.collector.agent.stream.worker.register.ServiceNameRegisterSerialWorker`

   

  ，继承 AbstractLocalAsyncWorker 抽象类，异步保存应用 Worker 。

  - 相同于 Application ，ServiceName 的操作编号，从 `"1"` **双向**递增。
  - [ServiceNameEsRegisterDAO#save(ServiceName)](https://github.com/YunaiV/skywalking/blob/a4db2c4dd5e2adc861e7fb9e9b7b7ffdc57dfb88/apm-collector/apm-collector-storage/collector-storage-es-provider/src/main/java/org/skywalking/apm/collector/storage/es/dao/ServiceNameEsRegisterDAO.java#L52)

### 2.2.4 ServiceName

[`org.skywalking.apm.collector.storage.table.register.ServiceName`](https://github.com/YunaiV/skywalking/blob/a9873b9bf07882746bd30f29b3c64f4b44887bf2/apm-collector/apm-collector-storage/collector-storage-define/src/main/java/org/skywalking/apm/collector/storage/table/register/ServiceName.java) ，操作名。例如记录在 ES 如下图：

![img](https://static.iocoder.cn/images/SkyWalking/2020_09_28/02.png)

# 3. Agent 调用同步 API

在 [《SkyWalking 源码分析 —— 应用于应用实例的注册》「3. Agent 调用注册 API」](https://www.iocoder.cn/SkyWalking/agent-dictionary/#) 一文中，在 [【第 170 至 173 行】的代码](https://github.com/YunaiV/skywalking/blob/bb8cb1b7dcb428c161f225f0b5d57441105f84c0/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/remote/AppAndServiceRegisterClient.java#L170)，我们可以看到，AppAndServiceRegisterClient 会定时从 Collector 同步所有字典信息。

## 3.1 DictionaryManager

[`org.skywalking.apm.agent.core.dictionary.DictionaryManager`](https://github.com/YunaiV/skywalking/blob/0830d985227c42f0e0f3787ebc99a2b197486b69/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/dictionary/DictionaryManager.java#L27) ，字典管理器。目前管理有两种字典：

- ApplicationDictionary
- OperationNameDictionary

### 3.1 ApplicationDictionary

`org.skywalking.apm.agent.core.dictionary.ApplicationDictionary` ，应用字典。

- `INSTANCE` 枚举属性，单例。
- `applicationDictionary` 属性，应用编码与应用编号的映射。
- `unRegisterApplications` 属性，未知应用编码集合。Agent 会定时从 Collector 同步。这也是文章开头说的，“**需要**”的定义。

[`#find(applicationCode)`](https://github.com/YunaiV/skywalking/blob/0830d985227c42f0e0f3787ebc99a2b197486b69/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/dictionary/ApplicationDictionary.java#L56) 方法，根据应用编码，查询应用编号。

- 第 57 行：根据应用编码，从 `applicationDictionary` 中，查询应用编号。
- 第 58 至 59 行：当应用编号查找到时，返回 Found 。Found 会在下文详细解析。
- 第 61 至 64 行：当应用编号查找不到时，添加到 `unRegisterApplications` 中，返回 NotFound 。NotFound 会在下文详细解析。

[`#syncRemoteDictionary(ApplicationRegisterServiceGrpc.ApplicationRegisterServiceBlockingStub)`](https://github.com/YunaiV/skywalking/blob/0830d985227c42f0e0f3787ebc99a2b197486b69/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/dictionary/ApplicationDictionary.java#L74) 方法，调用 [「2.1 应用的同步 API」](https://www.iocoder.cn/SkyWalking/agent-dictionary/#) ，从 Collector 同步 `unRegisterApplications` 对应的应用编号集合。

### 3.2 OperationNameDictionary

`org.skywalking.apm.agent.core.dictionary.OperationNameDictionary` ，操作名字典。

和 ApplicationDictionary 基本类似，胖友点击 [代码](https://github.com/YunaiV/skywalking/blob/0830d985227c42f0e0f3787ebc99a2b197486b69/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/dictionary/OperationNameDictionary.java) ，自己阅读理解。

# 3.2 PossibleFound

在分享 PossibleFound 之前，我们先来看一段代码，了解该类的意图：

![img](https://static.iocoder.cn/images/SkyWalking/2020_09_28/03.png)

- 我们在使用 `XXXXDictionary#find(xxx)` 方法时，返回的会是 Found 或者 NotFound 。这两个类本身是**互斥**的，并且继承 PossibleFound 。在 PossibleFound 提供 `#doInCondition(method01, method02)` 方法，**优雅**的处理两种情况。

[`org.skywalking.apm.agent.core.dictionary.PossibleFound`](https://github.com/YunaiV/skywalking/blob/0830d985227c42f0e0f3787ebc99a2b197486b69/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/dictionary/PossibleFound.java#L26) ，**抽象类**，代码如下：

- `found` 属性，是否找到。

- `value` 属性，找到的结果。

- [org.skywalking.apm.agent.core.dictionary.Found](https://github.com/YunaiV/skywalking/blob/0830d985227c42f0e0f3787ebc99a2b197486b69/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/dictionary/Found.java#L26) 实现 PossibleFound 类，`found = true` 并且 `value` 为找到的值。

- [org.skywalking.apm.agent.core.dictionary.NotFound](https://github.com/YunaiV/skywalking/blob/0830d985227c42f0e0f3787ebc99a2b197486b69/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/dictionary/NotFound.java#L26) 实现 PossibleFound 类，`found = false` 并且 `value` 不赋值。

- `#doInCondition(Found, NotFound)`

   

  方法，根据查找结果，执行不同的逻辑，【无返回】。

  - 第一个参数，[`PossibleFound.Found`](https://github.com/YunaiV/skywalking/blob/0830d985227c42f0e0f3787ebc99a2b197486b69/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/dictionary/PossibleFound.java#L72) 接口，Found 时的处理逻辑接口。
  - 第二个参数，[`PossibleFound.NotFound`](https://github.com/YunaiV/skywalking/blob/0830d985227c42f0e0f3787ebc99a2b197486b69/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/dictionary/PossibleFound.java#L79) 接口，NotFound 时的处理逻辑接口。

- `#doInCondition(FoundAndObtain, NotFoundAndObtain)`

   

  方法，根据查找结果，执行不同的逻辑，【有返回】。

  - 第一个参数，[`PossibleFound.FoundAndObtain`](https://github.com/YunaiV/skywalking/blob/0830d985227c42f0e0f3787ebc99a2b197486b69/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/dictionary/PossibleFound.java#L86) 接口，Found 时的处理逻辑接口。
  - 第二个参数，[`PossibleFound.NotFoundAndObtain`](https://github.com/YunaiV/skywalking/blob/0830d985227c42f0e0f3787ebc99a2b197486b69/apm-sniffer/apm-agent-core/src/main/java/org/skywalking/apm/agent/core/dictionary/PossibleFound.java#L93) 接口，NotFound 时的处理逻辑接口。

# 666. 彩蛋