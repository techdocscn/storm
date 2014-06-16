---
layout: documentation
---
本地模式在进程中模拟一个 Storm 集群，可以 用来开发和测试 topologies。在本地模式里面运行 topologies 和 [在集群上](Running-topologies-on-a-production-cluster.html)非常接近。
产生一个进程中集群，只要简单地使用 `LocalCluster` 类。例如：

```java
import backtype.storm.LocalCluster;

LocalCluster cluster = new LocalCluster();
```

然后你可以在 `LocalCluster` 对象上用 `submitTopology` 方法来提交 topologies。 就像在 [StormSubmitter](/apidocs/backtype/storm/StormSubmitter.html) 上面的对应方法一样，`submitTopology` 需要一个名字，一个 topology 配置，和 topology 对象。接下来你可以用  `killTopology` 方法，及用 topology 名字做参数来杀死一个 topology。

停止一个本地集群，只需要调用：

```java
cluster.shutdown();
```

### 本地模式的通用配置

你可以在[这里](/apidocs/backtype/storm/Config.html)查看完整配置列表。

1. **Config.TOPOLOGY_MAX_TASK_PARALLELISM**: 这个配置给一个单独 component 可以产生的线程数目加了一个上限。产品环境下的 topologies 经常会有很多并行处理（数百个线程），这在本地测试 topology 的情况下会产生不合理的负载。你可以用这个设置来很容易地控制并行处理。
2. **Config.TOPOLOGY_DEBUG**: 当这个设置被设成 true 的时候, Storm 将会把每一个 tuple 从 spout 或者 bolt 发出的消息记录在日志里面。这些记录在调试的时候非常有用。