---
layout: documentation
---
在这个教程里，你会学会如何创建一个 Storm Topology 以及把它们部署到一个 Storm 集群上。我们主要使用 Java 语言，但是有些例子也会用 Python 来展示 Storm 对多种语言的支持。

## 准备工作

这个教程会使用 [storm-starter](http://github.com/nathanmarz/storm-starter) 项目里面的例子。我们建议你 clone 一个项目并跟着我们的例子走。你可以阅读 [设置开发环境](Setting-up-development-environment.html) 以及 [创建新的 Storm 项目](Creating-a-new-Storm-project.html) 来设置你的机器。

## Storm 集群的部件

一个 Storm 集群表面上和一个 Hadoop 集群非常类似。在 Hadoop 里面你运行的是 “MapReduce Job”，在 Storm 你运行的是 “Topology”。“Job” 和 “Toplogy” 它们本身是非常不一样的——最重要的差别就是 MapReduce job 最终会运行结束，而 topology 永远都在处理消息（直到你杀掉它）。

Storm 集群有两种不同的结点：主节点（master）和工人节点（worker）。主节点运行了一个叫做 ”Nimbus“ 的守护进程（daemon），和 Hadoop 的 “JobTracker” 很像。Nimbus 负责把代码发布到集群中去，给机器指定任务，以及监视失败。

每个工人结点运行了一个叫做 “Supervisor” 的守护进程。Supervisor 会监听指定给该机器的任务，按照需求根据 Nimbus 指定给它的任务启动和停止工人进程。每个工人进程执行 topology 的一个子集；一个运行的 topology 包含了分布在若干机器上的多个工人进程。

![Storm 集群](images/storm-cluster.png)


Nimbus 和 Supervisor 之间的所有协作都通过 [Zookeeper](http://zookeeper.apache.org/) 集群来完成。此外，Nimbus 守护进程和 Supervisor 守护进程都是快速报错（fail-fast）且无状态的；所有的状态都存在 Zookeeper 里或者本地磁盘上。这就是说你可以用 kill -9 来杀掉 Nimbus 或者 Supervisor，然后它们能像什么都没发生过一样重新启动。这个设计使得 Storm 集群具有非常的稳定性。

## Topologies （拓扑）

要在 Storm 进行实时计算，你需要创建 “topology”。一个 topology 是一个计算图（graph）。一个 topology 中得每一个结点都有一些处理逻辑，结点之间的链接则指明了数据会如何在其间流动。

运行一个 topology 非常简单。首先你把你所有的代码和依赖库都打包到一个 jar 里面。然后，你运行像这样一个命令：

```
storm jar all-my-code.jar backtype.storm.MyTopology arg1 arg2
```

这就会使用参数 `arg1` 和 `arg2` 来执行类 `backtype.storm.MyTopology`。类的主函数（main）定义了 topology 并把它提交给 Nimbus。而 `storm jar` 部分则处理了连接到 Nimbus 并上传 jar 等问题。

因为 topology 的定义就是 Thrift 结构，Nimbus 就是一个 Thrift service，你可以通过任何编程语言创建和提交 topology。上面的例子是基于 JVM 的语言最简单的提交方法。请参考  [在产品集群上运行 topology](Running-topologies-on-a-production-cluster.html)] 来获取如何启动和停止 topology 的信息。

## 流

Storm 的核心抽象就是“流”。流是一个没有边界的 tuple （元组）序列。Storm 提供了一些分布式的且可靠地把一个流转换为另一个流的原语。比如，你可以把你一个 tweets （推文）的流转换为一个热门话题的流。

Storm 提供来进行流转换的基本原语叫做 ”spout“ （喷口）和 “bolt” （螺栓）。Spout 和 bolt 都有你可以自己实现用来运行你应用程序的特定逻辑的接口。

Spout 是一个流的来源。比如，一个 spout 可以从 [Kestrel](http://github.com/nathanmarz/storm-kestrel) 队列读一些 tuple 出来，把它们输出到一个流里面。或者一个 spout 也可以连接到 Twitter API 并输出一个推文的流。

Bolt 则可以消费任何数量的流，进行处理，也许还可以生成新的流。一些复杂的流转换，比如从一个推文的流来计算一个热门话题的流，就需要多个步骤，也就是说多个 bolt。Bolt 可以做任何事情，比如运行函数，过滤 tuple，流聚集（aggregation），流连接（join），读写数据库等等。

Spout 和 bolt 组成的网络被打包成了一个 “topology”，这是你提交给 Storm 集群执行的顶级抽象元素。一个 topology 就是一个流转换的图，每个结点就是一个 spout 或者 bolt。图的则边表明了哪个 bolt 订阅了哪个流。当一个 spout 或者 bolt 输出一个 tuple 到流里面的时候，它会把这个 tuple 送到每一个订阅了这个流的 bolt。

![一个 Storm topology](images/topology.png)

你的 topolgy 里面的结点的链接方式表明了 tuple 会如何被传播。比如，如果有一个 Spout A 和 Bolt B 之间的链接，一个 Spout A 到 Bolt C 的链接，以及一个 Bolt B 到 Bolt C 的链接，那么每次 Spout A 输出一个 tuple 的时候，它就会把 tuple 送到 Bolt B 和 Bolt C。所有 Bolt B 输出的 tuple 也都会跑到 Bolt C。

Storm topology 中的每个结点都是并行运行的。在你的 topology 中，你可以指定每个结点有多少的并行量，Storm 会在集群中生成相应数量的线程来执行运算。

一个 Topology 永远都在执行，除非你杀掉它。Storm 会自动把失败的任务重新指定运行地点。此外，Storm 保证数据绝对不会丢失，即使机器关机，消息丢失。

## 数据模型

Storm 使用 tuple 作为其数据模型。一个 tuple 就是一串命名的值，tuple 里面的一个域可以是任何类型的对象。Storm 已经支持在 tuple 的字段里面使用任何原始数据类型、字符串、以及字节数组（byte array）。要使用其他类型的对象的话，你只需要为那个类型实现一个 [serializer （序列化器）](Serialization.html)。

Topology 里面的每个结点必须声明它输出地 tuple 的输出字段。比如，下面这个 bolt 就声明了它输出一个二元 tuple，字段叫做 “double” 和 ”triple“：

```java
public class DoubleAndTripleBolt extends BaseRichBolt {
    private OutputCollectorBase _collector;

    @Override
    public void prepare(Map conf, TopologyContext context, OutputCollectorBase collector) {
        _collector = collector;
    }

    @Override
    public void execute(Tuple input) {
        int val = input.getInteger(0);        
        _collector.emit(input, new Values(val*2, val*3));
        _collector.ack(input);
    }

    @Override
    public void declareOutputFields(OutputFieldsDeclarer declarer) {
        declarer.declare(new Fields("double", "triple"));
    }    
}
```

这里，`declareOutputFields` 函数声明了该组件使用的输出字段 `["double", "triple"]`。后面的章节会解释 Bolt 的其它部分。

## 一个简单的 topology

现在我们来看一个简单的 topology，深入探索一下基本概念，看看代码是什么样子。我们来看看 storm-starter 里面的 `ExclamationTopology` 的定义：

```java
TopologyBuilder builder = new TopologyBuilder();        
builder.setSpout("words", new TestWordSpout(), 10);        
builder.setBolt("exclaim1", new ExclamationBolt(), 3)
        .shuffleGrouping("words");
builder.setBolt("exclaim2", new ExclamationBolt(), 2)
        .shuffleGrouping("exclaim1");
```

这个 topology 包含一个 spout 和两个 bolt。Spout 输出单词，每个 bolt 则在自己的输入后面添加”!!!“。三个结点在一条直线上：spout 输出到第一个 bolt，然后它又输出到第二个 bolt。如果 spout 输出了两个 tuple [“bob”] 和 [“john”]，那么第二个 bolt 就会输出单词 [“bob!!!!!!”] 和 [“john!!!!!!”]。

这段代码使用 `setSpout` 和 `setBolt` 方法定义了结点。这些方法使用一个用户指定的 id、一个包含了处理逻辑的对象、以及你希望结点拥有的并行量作为输入。在这个例子中，spout 得到了 ”words“ 这个 id，bolt 得到了 ”exclaim1“ 和 ”exclaim2” 这两个 id。

包含了处理逻辑的对象则为 spout 实现了 [IRichSpout](/apidocs/backtype/storm/topology/IRichSpout.html) 接口，为 bolt 实现了 [IRichBolt](/apidocs/backtype/storm/topology/IRichBolt.html) 接口。

最后一个参数，你希望给结点的并行量，是可选的。它表明了在整个集群应该有多少线程运行这个部件。如果你省略它，Storm 只会给那个结点指派一个线程。

`setBolt` 返回一个 [InputDeclarer](/apidocs/backtype/storm/topology/InputDeclarer.html) 对象用来定义 bolt 的输入。这里 “exclaim1” 部件声明它要使用乱序分组（shuffle grouping）读 “words” 部件输出的所有 tuple，“exclaim2” 组件声明它想用乱序分组读 ”exclaim1” 所输出的所有 tuple。“乱序分组”是说输入任务的 tuple 会被随机的分配到 bolt 任务中去。在组件之间分组数据的方式有很多。我们会用几节来解释它们。

如果你想 “exclaim2“ 组件读取 ”words“ 组件和 “exclaim1” 组件输出地所有 tuple，你可以这么写 “exclaim2” 组件的定义：

```java
builder.setBolt("exclaim2", new ExclamationBolt(), 5)
            .shuffleGrouping("words")
            .shuffleGrouping("exclaim1");
```

如你所见，输入声明可以连起来以便制定 bolt 的多个源。

让我们再深挖一下这个 topology 里面的 spout 和 bolt 的实现。Spout 负责把新的消息输出到 topology 中。这个 topology 里面的 `TestWordSpout` 从一个列表 [“nathan”, “mike”, “jackson”, “golda”, “bertels”] 中每 100ms 输出一个随机单词作为一元 tuple。TestWordSpout 中的 `nextTuple()` 的实现是这个样子的：

```java
public void nextTuple() {
    Utils.sleep(100);
    final String[] words = new String[] {"nathan", "mike", "jackson", "golda", "bertels"};
    final Random rand = new Random();
    final String word = words[rand.nextInt(words.length)];
    _collector.emit(new Values(word));
}
```

如你所见，实现得很直接。

`ExclamationBolt` 在它的输入后面添加了 “!!!” 字符串。我们来看看 `ExclamationBolt` 的完整实现：

```java
public static class ExclamationBolt implements IRichBolt {
    OutputCollector _collector;

    public void prepare(Map conf, TopologyContext context, OutputCollector collector) {
        _collector = collector;
    }

    public void execute(Tuple tuple) {
        _collector.emit(tuple, new Values(tuple.getString(0) + "!!!"));
        _collector.ack(tuple);
    }

    public void cleanup() {
    }

    public void declareOutputFields(OutputFieldsDeclarer declarer) {
        declarer.declare(new Fields("word"));
    }
    
    public Map getComponentConfiguration() {
        return null;
    }
}
```

`prepare` 方法给 bolt 提供了一个 `OutputCollector` 用来输出 tuple。Tuples 可能在任何时候从 bolt 输出 —— 使用 `prepare`、`execute`、或者 `cleanup` 方法，甚至从别的线程里异步输出。这里的 `prepare` 实现只是简单地把 `OutputCollector` 作为一个实例成员变量保留下来以便一会儿在 `execute` 方法中使用。


`execute` 方法从 bolt 的某个输入中收到一个 tuple。`ExclamationBolt` 则把 tuple 的第一个字段取出来，加上 "!!!" 并作为一个新的 tuple 输出。如果你实现了一个订阅多个输入源的 bolt，你可以用 `Tuple#getSourceComponent` 方法来找到这个  [Tuple](/apidocs/backtype/storm/tuple/Tuple.html) 是从哪个组件来的。

在 `execute` 方法里面还发生了一些别的事情，输入的 tuple 作为第一个参数传给了 `emit`，输入 tuple 在最后一行被 ack （应答）了。这些都是 Storm 的用来保证无数据丢失的可靠性 API 的一部分，会在本教程后面讲到。

在 bolt 被关闭的时候，`cleanup` 方法会被调用，它应该关闭任何打开了的资源。但是这个方法并不保证一定会被调用。比如任务运行的机器爆炸了，那么这方法就不可能被调用。`cleanup` 方法是给你在 [本地模式](Local-mode.html) （Storm 集群是在一个进程里模拟的）运行 topology 的时候用的，你可能会启动和杀掉很多 topology 而又不想受任何资源泄露之苦。

`declareOutputFields` 方法声明 `ExclamationBolt` 会输出一元 tuple，带有一个叫做 “word” 的字段。

`getComponentConfiguration` 方法则允许你设置这个组件如何运行的若干方面。这是一个高级话题，在 [设置](Configuration.html) 一文中会有深入解释。

像 `cleanup` 和 `getComponentConfiguration` 这样的方法一般在 bolt 实现中都不需要。如果合适，你可以用带有默认实现的基类来更简洁地定义 bolt。通过继承 `BaseRichBolt`，`ExclamationBolt` 可以更简单的实现为：


```java
public static class ExclamationBolt extends BaseRichBolt {
    OutputCollector _collector;

    public void prepare(Map conf, TopologyContext context, OutputCollector collector) {
        _collector = collector;
    }

    public void execute(Tuple tuple) {
        _collector.emit(tuple, new Values(tuple.getString(0) + "!!!"));
        _collector.ack(tuple);
    }

    public void declareOutputFields(OutputFieldsDeclarer declarer) {
        declarer.declare(new Fields("word"));
    }    
}
```

## 在本地模式运行 ExclamationTopology

我们来看看怎么在本地模式运行 `ExclamationTopology` 并验证它可以工作。

Storm 有两种操作方式：本地模式和分布式模式。本地模式中，Storm 使用线程来模拟工人，完全在一个进程内运行。本地模式对于测试和开发 topology 非常有用。在你运行 storm-starter 里面的 topology 的时候，它们都会在本地模式运行，你能看到每个组件生成的消息。你可以在 [本地模式](Local-mode.html)读到更多关于在本地模式运行 topology 的内容。

在分布式模式中，Storm 在一个机器的集群中工作。当你把一个 topology 提交给主结点的时候，你也提交了所有需要用来运行这个 topology 的代码。主结点会负责发布你的代码，分配工人来运行你的 topology。如果工人宕机了，主结点会把它们重新分派到另一个地方。你可以到 [阅读更多关于在集群上运行 topology 的内容。 ](Running-topologies-on-a-production-cluster.html)] 阅读更多关于在集群上运行 topology 的内容。 

下面是在本地模式运行 `ExclamationTopology` 的代码：

```java
Config conf = new Config();
conf.setDebug(true);
conf.setNumWorkers(2);

LocalCluster cluster = new LocalCluster();
cluster.submitTopology("test", conf, builder.createTopology());
Utils.sleep(10000);
cluster.killTopology("test");
cluster.shutdown();
```

首先，通过创建一个 `LocalCluster` 对象，代码定义了一个进程内的集群。把 topology 提交到这个虚拟集群和提交 topology 到分布式集群是一样的。它通过调用 `submitTopology` 把一个 topology 提交到 `LocalCluster`，参数包括一个用来运行 topology 的名称，一个 topology 的配置对象，以及 topology 本身。

名称是用来识别 topology 的，以便你将来可以杀掉它。一个 topology 会无尽地运行下去直到你杀死它。

配置对象是用来调整运行的 topology 的若干方面。在这里指定的两项配置都很常见：

1.  **TOPOLOGY_WORKERS**  （通过 `setNumWorkers` 方法设置）指定你想要在集群上分配多少个 _进程_ 来执行 topology。Topology 上的每个组件都会执行同样多的 _线程_ 。分配给某个组件的线程是通过 `setBolt` 和 `setSpout` 方法来设置的。那些 _线程_ 位于在工人 _进程_ 中。每个工人 _进程_ 包含有对于一定数量组件的一定数量的 _线程_ 。比如，你可以在 config 中为你所有组件一共指定 300 个线程和 50 个工人进程。每个工人进程会执行 6 个线程，每个线程都可以用于不同的组件。你通过调整每个组件的并行量和用于这些线程运行的工人进程数量来调试 Storm topology 的性能。
2. **TOPOLOGY_DEBUG**  （通过 `setDebug` 方法设置），当设置为 true 的时候，它告诉 Storm 记录（log）一个组件发出的每条消息。这在本地模式测试 topology 的时候很有用，但是在集群上运行 topology 的时候你可能需要让它保持关闭。

Topology 有很多参数可以设置。各种设置都在 [Config 的 Javadoc](/apidocs/backtype/storm/Config.html) 有详细介绍。

要知道如何设置你的开发环境以便在本地模式（比如在 Eclipse 中）运行你的 topology，请参看 [建立一个新的 Storm 项目](Creating-a-new-Storm-project.html)。

## 流分组

流分组告知 topology 如何在组件之间传递消息。请记住，spout 和 bolt 是在整个集群中作为多个任务（task）执行的。如果你看看 topology 是如何在任务一层执行的，它看起来会是这个样子：

![topology 中的任务](images/topology-tasks.png)

当 Bolt A 的一个任务输出一个 tuple 到 Bolt B 的时候，它会送到那个任务里去呢？

“流分组”回答了这个问题，它告诉 Storm 如何在任务之间传送 tuple。在我们了解不同类型的流分组之前，让我们先来看看来自 [storm-starter](http://github.com/nathanmarz/storm-starter)的另一个 topology。这个 [WordCountTopology](https://github.com/nathanmarz/storm-starter/blob/master/src/jvm/storm/starter/WordCountTopology.java) 从一个 spout 读入句子并通过 `WordCountBolt` 输出它此前看到每个词的数量：

```java
TopologyBuilder builder = new TopologyBuilder();
        
builder.setSpout("sentences", new RandomSentenceSpout(), 5);        
builder.setBolt("split", new SplitSentence(), 8)
        .shuffleGrouping("sentences");
builder.setBolt("count", new WordCount(), 12)
        .fieldsGrouping("split", new Fields("word"));
```

`SplitSentence` 对它收到的每个句子的每个单词输出一个 tuple，然后 `WordCount` 在内存里面保存了一个单词到计数的映射 map。每次 `WordCount` 收到一个词，它就更新它就更新它的状态，输出一个新的单词计数。

流分组有几种不同的方式。

最简单的分组方式叫做“乱序分组”，它把 tuple 发给随机的任务。`WordCountTopology` 使用了乱序分组来把 tuples 从 `RandomSentenceSpout` 发送到 `SplitSentence` bolt。它能够在整个 `SplitSentence` bolt 的任务中均匀地分配 tuple 的处理工作。

一个更有趣的分组方式叫做“字段分组”。`SplitSentence` bolt 和 `WordCount` bolt 就使用了字段分组。对于 `WordCount` bolt 的功能来说，同一个词总是被分配到同一个任务很关键。不然，多个任务会看见同一个词，它们每个都会输出一个错误的计数因为没有谁拥有完整的信息。字段分组让你能够把流按照其字段某个子集分组。这使得该字段子集相同的值都会前往同一个任务。因为 `WordCount` 使用在 “word” 字段上的字段分组订阅了 `SplitSentence` 的输出流，同样的单词总会前往相同的任务，bolt 才能产生正确的输出。

字段分组也是实现流连接（join）、流聚集（aggregation）以及很多其他用例的基础。在内部，流分组使用 mod hashing （余数散列）实现的：

还有其他几种分组方式。你可以在 [概念](Concepts.html)　一文中了解。

## 使用其他语言定义 bolt

Bolt 可以使用任何语言定义。使用其它语言写的 bolt 是作为子进程执行的，Storm 通过 stdin/stdout 上的 JSON 消息和它们通信。通信协议只需要大概 100 行左右的转接库，Storm 已经带有 Ruby、Python 和 Fancy 的转接库。

这里是 `WordCountTopology` 里面的 `SplitSentence` bolt：

```java
public static class SplitSentence extends ShellBolt implements IRichBolt {
    public SplitSentence() {
        super("python", "splitsentence.py");
    }

    public void declareOutputFields(OutputFieldsDeclarer declarer) {
        declarer.declare(new Fields("word"));
    }
}
```

`SplitSentence` 重载了 `ShellBolt` 并通过 `splitsentence.py` 参数来声明它运行 `python`。这里是 `splitsentence.py` 的实现：

```python
import storm

class SplitSentenceBolt(storm.BasicBolt):
    def process(self, tup):
        words = tup.values[0].split(" ")
        for word in words:
          storm.emit([word])

SplitSentenceBolt().run()
```

要得到更多关于使用使用别的语言写 spout 和 bolt 的信息，以及学习如何使用别的语言创建 topology （并完全避免 JVM），请参看 [在 Storm 中使用非 JVM 语言](Using-non-JVM-languages-with-Storm.html).

## 消息处理保证

在本教程前面，我们跳过了关于 tuple 是如何输出的一些方面的问题。那些方面是 Storm 的可靠性 API 的一部分：Storm 如何保证 spout 出来的每一条消息都会被完全处理。请参看 [消息处理保证](Guaranteeing-message-processing.html) 来了解它如何工作，以及你可以怎么利用 Storm 的可靠性功能。

## 事务 topology

Storm 保证每条消息都至少能够在整个 topology 走完一次。一个常见的问题就是，”你怎么在 Storm 上面计数？你不会数多了吗？“ Storm 有一个叫做事务 topology 的功能使你可以对绝大部分计算实现”刚好一次“的消息语义。在 [这里](Transactional-topologies.html) 可以读到更多关于事务 topology 的内容。

## 分布式 RPC

本教程展示了如何在 Storm 上进行基本的流处理。你还可以用 Storm 的原语做很多事情。Storm 最有趣的一个应用是分布式 RPC，你可以把高强度的运算实时并行化。请阅读 [here](Distributed-RPC.html) 了解更多关于分布式 RPC 的内容。

## 结论

本教程基本介绍了 Storm topology 的开发、测试、和部署。本文档的其它部分深入介绍了使用 Storm 的各个方面。
