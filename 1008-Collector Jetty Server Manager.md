# SkyWalking 源码分析 —— Collector Jetty Server Manager

**本文主要基于 SkyWalking 3.2.6 正式版**

- [1. 概述](http://www.iocoder.cn/SkyWalking/collector-jetty-server-module/)
- [2. JettyManagerModule](http://www.iocoder.cn/SkyWalking/collector-jetty-server-module/)
- [3. JettyManagerProvider](http://www.iocoder.cn/SkyWalking/collector-jetty-server-module/)
- [4. JettyManagerService](http://www.iocoder.cn/SkyWalking/collector-jetty-server-module/)
- [666. 彩蛋](http://www.iocoder.cn/SkyWalking/collector-jetty-server-module/)

------

------

# 1. 概述

本文主要分享 **Collector Jetty Server Manager**。Collector 通过该管理器，管理启动的多个 Jetty Server，例如 Agent Jetty Server、Naming Jetty Server、UI Jetty Server。![img](https://static.iocoder.cn/images/SkyWalking/2020_08_01/02.png)

> 友情提示：建议胖友已经读过 [《SkyWalking 源码分析 —— Collector Server Component 服务器组件》](http://www.iocoder.cn/SkyWalking/collector-server-component/?self)

Jetty Server Manager 在 SkyWalking 架构图处于如下位置( **红框** ) ：

> FROM https://github.com/apache/incubating-skywalking
> ![img](https://static.iocoder.cn/images/SkyWalking/2020_08_01/01.jpeg)

下面我们来看看整体的项目结构，如下图所示 ：

![img](https://static.iocoder.cn/images/SkyWalking/2020_08_01/03.png)

😈 代码量非常少，考虑到这是个单独的项目，所以单独成文。

# 2. JettyManagerModule

`org.skywalking.apm.collector.jetty.manager.JettyManagerModule` ，实现 [Module](https://github.com/YunaiV/skywalking/blob/40823179d7228207b06b603b9a1c09dfc4f78593/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/Module.java) 抽象类，Jetty 管理器 Module 。

[`#name()`](https://github.com/YunaiV/skywalking/blob/39ee2260b8463316550c52276109a5af917e34e8/apm-collector/apm-collector-jetty-manager/collector-jetty-manager-define/src/main/java/org/skywalking/apm/collector/jetty/manager/JettyManagerModule.java#L33) **实现**方法，返回模块名为 `"jetty_manager"` 。

[`#services()`](https://github.com/YunaiV/skywalking/blob/39ee2260b8463316550c52276109a5af917e34e8/apm-collector/apm-collector-jetty-manager/collector-jetty-manager-define/src/main/java/org/skywalking/apm/collector/jetty/manager/JettyManagerModule.java#L37) **实现**方法，返回 Service 类名：JettyManagerService 。

# 3. JettyManagerProvider

`org.skywalking.apm.collector.jetty.manager.JettyManagerProvider` ，实现 [ModuleProvider](https://github.com/YunaiV/skywalking/blob/40823179d7228207b06b603b9a1c09dfc4f78593/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/ModuleProvider.java) **抽象类**，Jetty 管理器组件服务提供者。

[`#name()`](https://github.com/YunaiV/skywalking/blob/39ee2260b8463316550c52276109a5af917e34e8/apm-collector/apm-collector-jetty-manager/collector-jetty-manager-provider/src/main/java/org/skywalking/apm/collector/jetty/manager/JettyManagerProvider.java#L46) **实现**方法，返回组件服务提供者名为 `"jetty"` 。

[`module()`](https://github.com/YunaiV/skywalking/blob/39ee2260b8463316550c52276109a5af917e34e8/apm-collector/apm-collector-jetty-manager/collector-jetty-manager-provider/src/main/java/org/skywalking/apm/collector/jetty/manager/JettyManagerProvider.java#L50) **实现**方法，返回组件类为 JettyManagerModule 。

[`#requiredModules()`](https://github.com/YunaiV/skywalking/blob/39ee2260b8463316550c52276109a5af917e34e8/apm-collector/apm-collector-jetty-manager/collector-jetty-manager-provider/src/main/java/org/skywalking/apm/collector/jetty/manager/JettyManagerProvider.java#L72) **实现**方法，返回依赖组件为空。

------

[`#prepare(Properties)`](https://github.com/YunaiV/skywalking/blob/39ee2260b8463316550c52276109a5af917e34e8/apm-collector/apm-collector-jetty-manager/collector-jetty-manager-provider/src/main/java/org/skywalking/apm/collector/jetty/manager/JettyManagerProvider.java#L54) **实现**方法，执行准备阶段逻辑。

- 第 55 行 ：创建 JettyManagerServiceImpl 对象，并调用 `#registerServiceImplementation(...)` **父类**方法，注册到 `services` 。

[`#start()`](https://github.com/YunaiV/skywalking/blob/39ee2260b8463316550c52276109a5af917e34e8/apm-collector/apm-collector-jetty-manager/collector-jetty-manager-provider/src/main/java/org/skywalking/apm/collector/jetty/manager/JettyManagerProvider.java#L58) **实现**方法，执行启动阶段逻辑。目前是个**空方法**。

[`#notifyAfterCompleted()`](https://github.com/YunaiV/skywalking/blob/39ee2260b8463316550c52276109a5af917e34e8/apm-collector/apm-collector-jetty-manager/collector-jetty-manager-provider/src/main/java/org/skywalking/apm/collector/jetty/manager/JettyManagerProvider.java#L61) **实现**方法，执行启动完成逻辑。

- 第 63 至 69 行 ：遍历注册的服务器列表，逐个调用 `JettyServer#start()` 方法，进行启动。

# 4. JettyManagerService

`org.skywalking.apm.collector.jetty.manager.service.JettyManagerService` ，继承 [Service](https://github.com/YunaiV/skywalking/blob/40823179d7228207b06b603b9a1c09dfc4f78593/apm-collector/apm-collector-core/src/main/java/org/skywalking/apm/collector/core/module/Service.java) **接口**，Jetty 管理器服务**接口**。

[`#createIfAbsent(host, port, contextPath)`](https://github.com/YunaiV/skywalking/blob/71993b1790b29df66644334dcfe1796e889ddc9b/apm-collector/apm-collector-jetty-manager/collector-jetty-manager-define/src/main/java/org/skywalking/apm/collector/jetty/manager/service/JettyManagerService.java#L40) **接口**方法，创建 Jetty Server ，若不存在。

[`#addHandler(host, port, serverHandler)`](https://github.com/YunaiV/skywalking/blob/71993b1790b29df66644334dcfe1796e889ddc9b/apm-collector/apm-collector-jetty-manager/collector-jetty-manager-define/src/main/java/org/skywalking/apm/collector/jetty/manager/service/JettyManagerService.java#L49) **接口**方法，添加 Jetty Server 请求处理器。

## 4.1 JettyManagerServiceImpl

`org.skywalking.apm.collector.jetty.manager.service.JettyManagerServiceImpl` ，Jetty 管理器服务**实现类**。

[**构造方法**](https://github.com/YunaiV/skywalking/blob/21a84f89165f74c7fe702650ff4d12db5a9613e4/apm-collector/apm-collector-jetty-manager/collector-jetty-manager-provider/src/main/java/org/skywalking/apm/collector/jetty/manager/service/JettyManagerServiceImpl.java#L46) ，使用来自 JettyManagerProvider 的 `servers` 服务器数组。**这是为什么 JettyManagerProvider 没有对 `servers` 做新增操作，结果里面有数据的原因**。

[`#createIfAbsent(host, port, contextPath)`](https://github.com/YunaiV/skywalking/blob/21a84f89165f74c7fe702650ff4d12db5a9613e4/apm-collector/apm-collector-jetty-manager/collector-jetty-manager-provider/src/main/java/org/skywalking/apm/collector/jetty/manager/service/JettyManagerServiceImpl.java#L50) **实现**方法，创建 Jetty Server ，若不存在。判断方式为 `host + port` 为唯一。

[`#addHandler(host, port, serverHandler)`](https://github.com/YunaiV/skywalking/blob/21a84f89165f74c7fe702650ff4d12db5a9613e4/apm-collector/apm-collector-jetty-manager/collector-jetty-manager-provider/src/main/java/org/skywalking/apm/collector/jetty/manager/service/JettyManagerServiceImpl.java#L66) **实现**方法，添加 Jetty Server 请求处理器。判断方式为 `host + port` 为唯一。

# 666. 彩蛋