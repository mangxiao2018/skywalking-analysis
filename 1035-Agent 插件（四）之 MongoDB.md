# Agent 插件（四）之 MongoDB

**本文主要基于 SkyWalking 3.2.6 正式版**

- [1. 概述](http://www.iocoder.cn/SkyWalking/agent-plugin-mongodb/)
- \2. MongoDBInstrumentation
  - [2.1 MongoDBMethodInterceptor](http://www.iocoder.cn/SkyWalking/agent-plugin-mongodb/)
- [666. 彩蛋](http://www.iocoder.cn/SkyWalking/agent-plugin-mongodb/)

------

------

# 1. 概述

本文主要分享 **SkyWalking Agent MongoDB 插件**。涉及到的代码不多，如下图：

![img](https://static.iocoder.cn/images/SkyWalking/2020_12_01/02.png)

考虑到大多数团队目前使用的 MongoDB 3.X 版本，本文解析 `mongodb-3.x-plugin` 插件。

# 2. MongoDBInstrumentation

在 [`skywalking-plugin.def`](https://github.com/apache/incubator-skywalking/blob/ea37839950c58b77114fbfebffc13f051e1caeb7/apm-sniffer/apm-sdk-plugin/mongodb-3.x-plugin/src/main/resources/skywalking-plugin.def) 里，定义了插件，如下图：

![img](https://static.iocoder.cn/images/SkyWalking/2020_12_01/03.png)

------

[`org.apache.skywalking.apm.plugin.mongodb.v3.define.MongoDBInstrumentation`](https://github.com/apache/incubator-skywalking/blob/ea37839950c58b77114fbfebffc13f051e1caeb7/apm-sniffer/apm-sdk-plugin/mongodb-3.x-plugin/src/main/java/org/apache/skywalking/apm/plugin/mongodb/v3/define/MongoDBInstrumentation.java) ，实现 ClassInstanceMethodsEnhancePluginDefine 抽象类，定义了方法切面，代码如下：

![img](https://static.iocoder.cn/images/SkyWalking/2020_12_01/01.png)

## 2.1 MongoDBMethodInterceptor

[`org.apache.skywalking.apm.plugin.mongodb.v3.MongoDBMethodInterceptor`](https://github.com/YunaiV/skywalking/blob/0128349b40592b8ae329443c52f43577cc9fa16b/apm-sniffer/apm-sdk-plugin/dubbo-plugin/src/main/java/org/skywalking/apm/plugin/dubbo/DubboInterceptor.java) ，实现 InstanceMethodsAroundInterceptor 和 InstanceConstructorInterceptor 接口，MongoDBInstrumentation 的拦截器。代码如下：

- `DB_TYPE` 静态属性，数据库类型。

- `MONGO_DB_OP_PREFIX` 静态属性，操作前缀。

- ```
  FILTER_LENGTH_LIMIT
  ```

   

  静态属性，操作语句长度限制，

  256

   

  。

  - `EMPTY` 静态属性，未知操作的参数。
  - 当开启 `Config.Plugin.MongoDB.TRACE_PARAM = true` 时，记录 MongoDB 操作语句。可以通过 `agent.config` 配置，如下图所示：![img](https://static.iocoder.cn/images/SkyWalking/2020_12_01/04.png)

- [`#onConstruct(objInst, allArguments)`](https://github.com/YunaiV/skywalking/blob/2f629ab1c9b96a77ecf6cca1e1dc20def0d20f1f/apm-sniffer/apm-sdk-plugin/mongodb-3.x-plugin/src/main/java/org/skywalking/apm/plugin/mongodb/v3/MongoDBMethodInterceptor.java#L206) 方法，拼接拼接集群地址，设置集群地址到 Mongo 对象的私有变量( SkyWalking 自动生成 )。

- `#beforeMethod(...)`

   

  方法，创建 ExitSpan 对象。代码如下：

  - 第 171 行：调用 `ContextManager#createExitSpan(...)` 创建 ExitSpan 对象。其中，操作名使用 `MONGO_DB_OP_PREFIX` + `Method#getName()` 。

  - 第 174 行：设置 EntrySpan 对象的组件类型。

  - 第 177 行：设置 EntrySpan 对象的 `db.type` 标签键值对。

  - 第 180 行：设置 EntrySpan 对象的分层。

  - 第 183 至 185 行：当

     

    ```
    Config.Plugin.MongoDB.TRACE_PARAM = true
    ```

     

    开启时，设置

    操作语句

    到 EntrySpan 对象的

     

    ```
    db.statement
    ```

     

    标签键值对。其中，操作语句通过

     

    `#getTraceParam(Object)`

     

    方法生成。

    - [`#getFilter(List)`](https://github.com/YunaiV/skywalking/blob/2f629ab1c9b96a77ecf6cca1e1dc20def0d20f1f/apm-sniffer/apm-sdk-plugin/mongodb-3.x-plugin/src/main/java/org/skywalking/apm/plugin/mongodb/v3/MongoDBMethodInterceptor.java#L136)
    - [`#limitFilter(String)`](https://github.com/YunaiV/skywalking/blob/2f629ab1c9b96a77ecf6cca1e1dc20def0d20f1f/apm-sniffer/apm-sdk-plugin/mongodb-3.x-plugin/src/main/java/org/skywalking/apm/plugin/mongodb/v3/MongoDBMethodInterceptor.java#L154)

- [`#afterMethod(...)`](https://github.com/YunaiV/skywalking/blob/2f629ab1c9b96a77ecf6cca1e1dc20def0d20f1f/apm-sniffer/apm-sdk-plugin/mongodb-3.x-plugin/src/main/java/org/skywalking/apm/plugin/mongodb/v3/MongoDBMethodInterceptor.java#L188) 方法，调用 `ContextManager#stopSpan()` 方法，完成 ExitSpan 对象。

- `#handleMethodException(...)`

   

  方法，处理异常。代码如下：

  - 第 199 行：标记 ExitSpan 发生异常。
  - 第 202 行：设置 EntrySpan 的日志。

# 666. 彩蛋