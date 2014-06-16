---
layout: documentation
---
# 运行中的 topology 有工人进程(worker processes)，执行线程(exectuor)和任务(task)

Storm 分布于以下三个主要的实体，用于运行一个topology在一个 Storm 的集群：

1. 工人进程
2. 执行线程
3. 任务

下面是它们的关系的一个简单图示：


![The relationships of worker processes, executors (threads) and tasks in Storm](images/relationships-worker-processes-executors-tasks.png)

一个 _工人进程_ 执行一个 topology 的子集。一个工人进程属于某一个 topology，可以为一个或者多个组件（spouts 或者 bolts）运行一个或者多个执行线程。一个运行中的 topology 包括许多这样的进程，在一个 Storm cluster的许多机器上运行。

一个执行线程是由工人进程派生出来的线程。它可以为同一个组件（spouts 或者 bolt）运行一个或者多个任务。

一个任务执行实质上的数据处理－ cluster 上所有每一个被实现的 spout 或者 bolt 执行同样多的任务。在一个 topology 的生命周期内，一个组件的任务数量总是一样的。而一个组件的线程数量可以随着时间而改变。也就是说，“线程数量 <= 任务数量”的条件总是成立的。默认情况下，任务数量与执行线程数量是相同的，即 Storm 将在一个线程中运行一个任务。

# 配置 topology 的并行量

要注意到，在 Storm 的专业术语中，“并行量”是专门用来描述所谓的“并行量暗示”的，它表示一个组件的线程初始数量。但在这个文档里，我们用“并行量”宽泛的用法，来描述应该如何配置一个 Storm topology 的线程数量，工人进程数量以及任务数量。当“并行量”被用于 Storm 里面正常的狭隘的定义的时候，我们将具体地说明。

下面的章节介绍了各种配置选项的概况以及如何在代码中设置它们。但是，设置这些选项有多种方法，而这个表哥之罗列了一些。Storm 母线有如下[设置配置的优先顺序s](Configuration.html): ``defaults.yaml`` < ``storm.yaml`` < topology 的专属配置 < 内部组件的专属配置 < 外部组件的专属配置。

## 工人进程的数量

* 描述: 为 cluster 里跨机器的 topology，创建多少工人进程
* 配置选项: [TOPOLOGY_WORKERS](/apidocs/backtype/storm/Config.html#TOPOLOGY_WORKERS)
* 如何在代码中设置 (示例):
    * [Config#setNumWorkers](/apidocs/backtype/storm/Config.html)

## 执行线程的数量

* 描述: _每一个组件_派生多少个执行线程.
* 配置选项: ?
* 如何在代码中设置 (示例):
    * [TopologyBuilder#setSpout()](/apidocs/backtype/storm/topology/TopologyBuilder.html)
    * [TopologyBuilder#setBolt()](/apidocs/backtype/storm/topology/TopologyBuilder.html)
    * 注意，Storm 0.8 中，“并行量暗示”这个参数表示对那个 bolt 的执行线程的初始数量

## 任务的数量

* 描述: 对_每一个组件_创建多少个任务 _per component_.
* 配置选项: [TOPOLOGY_TASKS](/apidocs/backtype/storm/Config.html#TOPOLOGY_TASKS)
* 如何在代码中设置 (示例):
    * [ComponentConfigurationDeclarer#setNumTasks()](/apidocs/backtype/storm/topology/ComponentConfigurationDeclarer.html)


下面是一段示例如何在实际中设置的代码:

```java
topologyBuilder.setBolt("green-bolt", new GreenBolt(), 2)
               .setNumTasks(4)
               .shuffleGrouping("blue-spout);
```

在上述代码中，我们配置 Storm 来运行 "GreenBolt" bolt, 它的初始设置是两个执行线程何四个关联的任务。Storm 将在每个线程上运行两个任务。如果你不明确地设置任务数量，Storm 将默认地在每个线程上运行一个任务。

# 一个运行中的 topology 的示例

下面的图示展示了一个简单的 topology 在操作中的样子。这个 topology 包括三个组件：一个 spout 叫做 "BlueSpout" 和 两个 bolts 叫做 “GreenBolt" 和 “YellowBolt"。 这些组件是这样被连接起来的："BlueSpout“ 把它的输出发给 "GreenBolt", “GreenBolt" 再将它自己的输出发给 "YellowBolt"

![Storm 上运行的 topology 的示例](images/example-of-a-running-topology.png)

``GreenBolt`` 就在上述代码中配置好了，而 ``BlueSpout``和d ``YellowBolt`` 只设置了并行量暗示(执行线程的数量)。相关的代码如下:

```java
Config conf = new Config();
conf.setNumWorkers(2); // use two worker processes

topologyBuilder.setSpout("blue-spout", new BlueSpout(), 2); // set parallelism hint to 2

topologyBuilder.setBolt("green-bolt", new GreenBolt(), 2)
               .setNumTasks(4)
               .shuffleGrouping("blue-spout");

topologyBuilder.setBolt("yellow-bolt", new YellowBolt(), 6)
               .shuffleGrouping("green-bolt");

StormSubmitter.submitTopology(
        "mytopology",
        conf,
        topologyBuilder.createTopology()
    );
```

当然，Storm 还自带额外的配置设置来控制一个 topology 的并行量，包括：

* [TOPOLOGY_MAX_TASK_PARALLELISM](/apidocs/backtype/storm/Config.html#TOPOLOGY_MAX_TASK_PARALLELISM): 这个设置为一个组件所能派生出来的执行线程设了一个上限。这往往被用于测试一个 topology 在本地状态下现在所派生的执行组件数量。你可以设置这个选项，例如. [Config#setMaxTaskParallelism()](/apidocs/backtype/storm/Config.html).

# 如何来改变一个运行中的 topology 的并行量

Storm 的一个实用的功能是可以增加或者减少工人进程以及执行线程的数量，而不需要重启这个 cluster 或者 topology。这样的做法叫重新平衡。

你有两种选项来重新平衡一个 topology:

1. 用 Storm 网页来重新平衡 topology 
2. 用 CLI 工具，如下。

这是一个用 CLI 工具的例子:

```
# Reconfigure the topology "mytopology" to use 5 worker processes,
# the spout "blue-spout" to use 3 executors and
# the bolt "yellow-bolt" to use 10 executors.

$ storm rebalance mytopology -n 5 -e blue-spout=3 -e yellow-bolt=10
```

# 参考

* [Concepts](Concepts.html)
* [Configuration](Configuration.html)
* [Running topologies on a production cluster](Running-topologies-on-a-production-cluster.html)]
* [Local mode](Local-mode.html)
* [Tutorial](Tutorial.html)
* [Storm API documentation](/apidocs/), most notably the class ``Config``

