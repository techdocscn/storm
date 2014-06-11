---
layout: 文档
---

在这篇文章里，我们将列举一些关于Storm的基本概念以及到相关资源的链接。我们将讨论以下一些概念：

1. Topologies
2. 流（Streams）
3. Spouts
4. Bolts
5. 流组（Stream groupings）
6. 可靠信（Reliability）
7. Tasks
8. Workers

### Topologies

实时应用程序的逻辑在 Storm 中被包装为topology。Storm topology 类似于 MapReduce 的 job。Storm 的 topology 和 MapReduce 的 job 的最大的区别在于 MapReduce job 是会终止的；然而 Storm 的 topology 不会，它会一直运行直到被用户主动终止。Topology 是一个将 spout 和 bolts 用 stream group 连接而成的图。我们将在后面逐个解释这些概念。

**资源：**

* [TopologyBuilder](/apidocs/backtype/storm/topology/TopologyBuilder.html): 在 Java 中，使用这个类创建 topology
* [在 production 集群上运行 topology](Running-topologies-on-a-production-cluster.html)
* [局部模式](Local-mode.html): 学习如何在局部模式下开发并测试 topology。

### 流（Streams）

流是 Storm 的核心概念。流是一个不间断无界连续的tuples，这些tuples在分布式环境下被并行的创建并处理。流的定义是通过对流中的 tuple 序列中每个字段的命名完成的。在默认情况下，tuples 的字段可以包括：integers，longs，shorts，bytes，strings，doubles，floats，booleans，以及 byte arrays。你甚至可以在 tuple 中使用自定义数据类型，但是你必须提供自己的序列化器。

我们在定义每一个流的时候会提供一个id。由于单流（singl-stream） spouts 和 bolts 的广泛使用，[OutputFieldsDeclarer](/apidocs/backtype/storm/topology/OutputFieldsDeclarer.html )提供一些方法（methods）让用户可以定义一个流而不需要指定id。这种情况下，该流会被分配一个值为'default'的缺省 id。

**资源：**

* [Tuple](/apidocs/backtype/storm/tuple/Tuple.html): 流是由 tuple 组成的
* [OutputFieldsDeclarer](/apidocs/backtype/storm/topology/OutputFieldsDeclarer.html): 用于定义流及其架构
* [Serialization](Serialization.html): 关于 Storm 的 tuples 的动态类型以及定义自定义序列化器
* [ISerialization](/apidocs/backtype/storm/serialization/ISerialization.html): 自定义序列化器必须实现这个接口
* [CONFIG.TOPOLOGY_SERIALIZATIONS](/apidocs/backtype/storm/Config.html#TOPOLOGY_SERIALIZATIONS): 用该配置注册自定义序列化器

### Spouts

在一个 topology 里面，Spout 是流的 tuple 源，即 tuple 的产生者。通常来说，spout 从一个外部源读取 tuples，并向 topology 发送 tuples（比如 Kestrel 队列，或者 Twitter API）。Spout可以是__可靠的__也可以是__不可靠的__。可靠的 spout 可以在 tuple 未被成功处理的情况下重新发送；不可靠 spout 在该情况下则不会重发。

Spouts 可以向多个流发送 tuples。我们可以使用 [OutputFieldsDeclarer](/apidocs/backtype/storm/topology/OutputFieldsDeclarer.html) 的 `declareStream` 方法来定义多个流，然后用  [SpoutOutputCollector](/apidocs/backtype/storm/spout/SpoutOutputCollector.html) 的 `emit` 方法指定希望被发送到的流。

Spout 类的最重要的方法是 `nextTuple`. `nextTuple` 或者发送一个 tuple 到 topology当中去，或者当没有新的 tuple 的时候返回。要注意的是任何 spout 的实现一定不要阻塞（block），因为 Storm 是在同一个线程上调用所有 spout 的方法。

Spout 类的另外一些重要的方法就是 `ack` 和 `fail`. 当 Storm 发现一个 tuple 发送到 topology 并被成功处理的时候会调用 `ack` 方法，失败的时候会调用 `fail`。Storm 仅仅对可靠的 Spout 调用这两个方法。 更多细节请参阅[the Javadoc](/apidocs/backtype/storm/spout/ISpout.html)。

**资源：**

* [IRichSpout](/apidocs/backtype/storm/topology/IRichSpout.html): Spouts 必须实现的接口。
* [保障消息处理](Guaranteeing-message-processing.html)

### Bolts

Topologies 里面所有的数据处理都是在 bolts 中完成的。Bolts 可以做的事情包括但不局限于：过滤（filtering），functions, 整合（aggregation）, 联合（joins）, 以及和数据库结合等。

Bolts 可以做简单的流的转换（transformations）. 如果需要做更复杂的流的转换，一般我们需要多个步骤，也就是多个 bolts 来完成。比如，将一个 tweets 流转换成 trending images 的流需要至少两步：第一步需要一个 bolt 对于每一个 image 的 retweet 进行计数，然后我们需要至少一个 bolt 来对于转发 (retweet) 最多的图片进行输出。如果我们需要这个系统有更好的可扩展性，使用三个 bolt 将会比两个 bolts 更优。 

Bolts 可以向多个流发送 tuples。我们可以使用 [OutputFieldsDeclarer](/apidocs/backtype/storm/topology/OutputFieldsDeclarer.html) 的 `declareStream` 方法来定义多个流，然后使用 [OutputCollector](/apidocs/backtype/storm/task/OutputCollector.html) 的 `emit` 方法指定希望被发送到的流。

当你定义一个 bolt 的输入流，你需要订阅（subscribe）其他部件（component）的一些流。如果你想订阅某个其他部件的所有相关的流，你必须一个一个的来。[InputDeclarer](/apidocs/backtype/storm/topology/InputDeclarer.html) 提供了一些语法上的便利（syntactic sugar）使得订阅所有流 id 为 default 的流更容易一些。用 `declarer.shuffleGrouping("1")` 可以订阅组件1的所有缺省的流。这个等同与 `declarer.shuffleGrouping("1", DEFAULT_STREAM_ID)`。

Bolts 的主要方法是 `execute`。它以一个新的 tuple 为输入，使用 [OutputCollector](/apidocs/backtype/storm/task/OutputCollector.html) 对象来发送新的 tuples。Bolts 必须为它处理的每一个 tuple 调用 `OutputCollector` 的 `ack` 方法，使 Storm 知道 tuple 的处理结束了，从而通知这个 tuple 的发送者 spount。通常情况流程是， bolts 处理一个输入 tuple，发送 0 或者多个 tuples，然后调用 ack 通知 Storm 对于该 tuple 处理完毕。Storm 提供  [IBasicBolt](/apidocs/backtype/storm/topology/IBasicBolt.html) 接口会自动调用 ack。

在 bolts 中启用多线程以异步方式处理是完全允许的。[OutputCollector](/apidocs/backtype/storm/task/OutputCollector.html) 是一个多线程安全（thread-safe）的，可以在任何时候被同时调用。

**资源：**

* [IRichBolt](/apidocs/backtype/storm/topology/IRichBolt.html): bolts 的通用接口。
* [IBasicBolt](/apidocs/backtype/storm/topology/IBasicBolt.html): 对于一些常用的功能比如过滤（filtering），可以使用这个接口。
* [OutputCollector](/apidocs/backtype/storm/task/OutputCollector.html): bolts 使用这个类向输出流发送 tuples。
* [保障消息处理](Guaranteeing-message-processing.html)

### Stream groupings

要定义一个 Topology，一个重要的环节就是要指定对于每一个 bolt 接受那些流作为输入。Stream grouping 就是用来定义一个流应该如何分配给各个 bolt 上的 tasks。

Storm有七种基本的 Stream groupings。另外你可以通过实现 [CustomStreamGrouping](/apidocs/backtype/storm/grouping/CustomStreamGrouping.html) 接口来实现自定义的 Stream grouping。

1. **Shuffle grouping**: Tuples 被随机的分配到各个 bolt 的 tasks，以保证每个 bolt 接收到同等数量的 tuples。
2. **Fields grouping**: 按照在 grouping 中指定的流的字段来分组。比如，如果一个流是指定 "user-id" 字段来分组，那么具有同样 "user-id“ 的 tuples 将会被分配到同一个 task；而具有不同 "user-id" 的 tuples将有可能被分配到不同的 tasks。
3. **All grouping**: 流的每一个 tuples 都会被发送到所有的 bolt 的 tasks。对于这个类型的使用要小心。
4. **Global grouping**: 整个流的所有 tuples 都会被发送到 bolt 的某一个 task：通常就是分配给id值最低的那个 task。
5. **None grouping**: 这种 grouping 方法表明用户不在乎 stream 是如何被 grouped 的。目前这种分组方法等同于 Shuffle grouping。但最终如果可能的话，Storm 会将这些 bolts 放到该 bolt 的订阅者（bolt 或者 spout）同一线程当中去执行。
6. **Direct grouping**: 这是一种特殊类型的分组。按照这种方式分组的流是由 tuple 的__产生者__来决定哪一个消费者 task 将接收到这个 tuple。只有被声明为 direct stream 的流才可以使用这种分组方法。而且这种 tuples 必须使用 [emitDirect](/apidocs/backtype/storm/task/OutputCollector.html#emitDirect(int, int, java.util.List)) 方法来发送. 而 bolt 也可以使用 [TopologyContext](/apidocs/backtype/storm/task/TopologyContext.html) 或者通过跟踪 [OutputCollector](/apidocs/backtype/storm/task/OutputCollector.html) 的 emit 方法(返回tuples的消费者的task id)的输出来获取消费者的 task ids。
7. **Local or shuffle grouping**: 如果目标 bolt 有一个或者多个 tasks 运行在同一个 worker process 中，那么 tuples 将会被随机发送到这些同一进程当中的 tasks。否则，tuples 将会被随机发送。

**资源：**

* [TopologyBuilder](/apidocs/backtype/storm/topology/TopologyBuilder.html): 使用该类定义 topology
* [InputDeclarer](/apidocs/backtype/storm/topology/InputDeclarer.html): 当在 `TopologyBuilder` 中调用 `setBolt` 方法时，该对象会被返回给调用者，它将被用于指定一个 bolt 的输入流以及指定这些流如何被分组（grouping）。
* [CoordinatedBolt](/apidocs/backtype/storm/task/CoordinatedBolt.html): 这个 bolt 通常被用于分布式远程调用（RPC）topology，并且大量使用 direct streams 和 direct groupings。

### 可靠性
Storm 保证每个 spout tuple 都会被 topology 完整的处理。Storm 会跟踪每个 spout tuple 形成的 tuple tree，并监测何时整个 tuple tree 被成功的处理。每个 topology 都设置有"消息超时（message timeout）"，如果在这个超时设置内监测不到某个 tuple tree 有没有被成功的处理，Storm 会把这个 tuple 的执行标记为失败，然后会对该 tuple 重新处理。

为了充分利用 Storm 的可靠性特性，每当你在 tuple tree 中创建一条新的边（edge）的时候，或者当你完成处理一个 tuple 的时候，你必须通知 Storm。你可以使用[OutputCollector](/apidocs/backtype/storm/task/OutputCollector.html) 对象来通知 Storm，该对象也被 bolts 用于发送 tuples：通过它的 `emit` 方法来表明新的 edge 的产生，通过 `ack` 方法表明对于一个 tuple 的成功处理。

更多的细节在 [Guaranteeing message processing](Guaranteeing-message-processing.html) 中有更为详细的介绍。

### Tasks

每个 spout 或者 bolt 会在整个集群（cluster）中执行很多 tasks。每一个 task 对应与一个线程，而 stream grouping 定义如何将 tuples 从一组 tasks 发送到另一组 tasks。 你可以调用 [TopologyBuilder](/apidocs/backtype/storm/topology/TopologyBuilder.html) 的 `setSpout` 和 `setBolt` 方法设置每个 spout 或者 bolt 的并行度。

### Workers

Topology 可能被分配在一个或者多个 worker 上执行。每一个 worker 是一个独立的 JVM 并负责执行 topology 的所有 tasks 的一个子集。比如，如果一个 topology 的并行度是 300，并且被分配 50个 workers，那么每个 worker 会执行 6 个 tasks（worker JVM 内的线程）。Storm 尽力将 tasks 平均分配到每一个 workers。

**资源：**

* [Config.TOPOLOGY_WORKERS](/apidocs/backtype/storm/Config.html#TOPOLOGY_WORKERS): 该设置定义执行 topology 所分配的 workers 的数量。
