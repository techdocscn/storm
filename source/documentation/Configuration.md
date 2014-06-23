---
layout: documentation
---
Storm 有大量的配置项可以优化 nimbus, supervisors, 以及运行中的 topology 的行为. 一些 topology 基础配置是系统配置, 无法在 topology 上修改, 其他配置是可以针对每一个 topology 修改的.

在 Storm 的代码库中, 所有配置项的默认值都定义在 [defaults.yaml](https://github.com/apache/incubator-storm/blob/master/conf/defaults.yaml)  文件中. 你可以在 Nimbus 和 supervisors 的 classpath 中定义 storm.yaml 文件, 在该文件中重写这些配置. 最后, 当你使用 [StormSubmitter](/apidocs/backtype/storm/StormSubmitter.html) 提交 topology 时, 还可以定义 topology 特定的配置. 不过, 它只会重写那些以 "TOPOLOGY" 为前缀的配置项.

Storm 0.7.0 以及之后的版本需要针对每一个 bolt 以及 spout 重写配置项. 仅有以下几个配置项可以用这种方式重写:

1. "topology.debug"
2. "topology.max.spout.pending"
3. "topology.max.task.parallelism"
4. "topology.kryo.register": 这个配置项和其他的稍有不同, 在这个 topology 中所有的组建的序列化功能将被开启. 关于序列化的更多细节, 请看[这里](Serialization.html). 


使用 Java API 指定特定组件的配置有以下两种方式:

1. *内部的(Internally):* 在 spout 或 bolt 中重写 `getComponentConfiguration` 并返回组件特定的配置.
2. *外部的(Externally):* 在 `TopologyBuilder` 类中的 `setSpout` 和 `setBolt` 方法会返回一个含有 `addConfiguration` 和 `addConfigurations` 方法的对象, 它可以用来重写组件的配置.

配置项读取顺序的优先级如下, defaults.yaml < storm.yaml < topology 特定的配置 < 内部组件特定的配置 < 外部组件特定的配置.


**资源列表:**

* [Config](/apidocs/backtype/storm/Config.html): 所有配置项列表以及一个创建 topology 特定配置的辅助类.
* [defaults.yaml](https://github.com/apache/incubator-storm/blob/master/conf/defaults.yaml): 所有配置项的默认值
* [Setting up a Storm cluster](Setting-up-a-Storm-cluster.html): 如何创建以及配置一个 Storm 集群
* [Running topologies on a production cluster](Running-topologies-on-a-production-cluster.html): 在集群中运行 topology 时有用的配置项列表
* [Local mode](Local-mode.html): 在本地模式中有用的配置项列表.
