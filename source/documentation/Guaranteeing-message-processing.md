---
layout: documentation
---
Storm 保证每一条从 spout 出来的消息都会被完全处理。本文档描述了 Storm 如何实现这个保证以及你作为一个用户如何能够从 Storm 的可靠处理能力中获益。

### “完全处理”到底是什么意思？

从一个 spout 出来的 tuple 能够导致上千个其它 tuple 的产生。比如在流中数单词的 topology 里：

```java
TopologyBuilder builder = new TopologyBuilder();
builder.setSpout("sentences", new KestrelSpout("kestrel.backtype.com",
                                               22133,
                                               "sentence_queue",
                                               new StringScheme()));
builder.setBolt("split", new SplitSentence(), 10)
        .shuffleGrouping("sentences");
builder.setBolt("count", new WordCount(), 20)
        .fieldsGrouping("split", new Fields("word"));
```

这个 topology 从一个 Kestrel 队列中读取句子，把句子分割成里面的词，然后对每个词输出我们之前看见过它的次数。从这个 spout 出来的一个 tuple 触发了基于它的很多其它 tuple 产生：句子里面的每个词有一个 tuple，每个单词的更新的计数也有一个 tuple。消息树看起来会是这个样子：

![Tuple 树](images/tuple_tree.png)

当一棵 tuple 树被访问结束而且树里面每条消息都被处理了，Storm 就认为从 spout 里面出来的一个 tuple 被处理了。当一个 tuple 的消息树在特定时限内没能完全被处理，这个 tuple 就被认为是失败了。这个时限能够使用 [Config.TOPOLOGY_MESSAGE_TIMEOUT_SECS](/apidocs/backtype/storm/Config.html#TOPOLOGY_MESSAGE_TIMEOUT_SECS) 设置按 topology 设定，默认时间为 30 秒。

### 一条消息完全处理或者没能完全处理后会发生什么？

要理解这个问题，让我们先来看看从一个 spout 出来的 tuple 的生命周期。这是 spout 实现的界面（请参考 [Javadoc](/apidocs/backtype/storm/spout/ISpout.html) 以获得更多信息）供您参考：

```java
public interface ISpout extends Serializable {
    void open(Map conf, TopologyContext context, SpoutOutputCollector collector);
    void close();
    void nextTuple();
    void ack(Object msgId);
    void fail(Object msgId);
}
```

首先，Storm 通过调用 `nextTuple` 方法从 `Spout` 请求一个 tuple。`Spout` 使用 `open` 方法提供的 `SpoutOutputCollector` 来向它的输出流之一输出一个 tuple。在输出 tuple 的时候，`Spout` 会提供一个用于在以后识别 tuple 的 “message id”。比如，`KestrelSpout` 就从一个 Kestrel 队列读取一个消息并把 Kestrel 为消息提供的 id 作为 “message id” 输出。向 `SpoutOutputCollector` 输出一条消息看起来是这样的：

```java
_collector.emit(new Values("field1", "field2", 3) , msgId);
```

然后，tuple 会被送到消费其的 bolt，Storm 会处理好跟踪它创建的消息树的任务。如果 Storm 检测到一个 tuple 已经被完全处理，Storm 会调用其源始 `Spout` 任务的 `ack` （确认）方法并告知 `Spout` 提供给 Storm 的 message id。同样地，如果 tuple 超时了 Storm 就会调用 `Spout` 的 `fail` 方法。注意一个 tuple 总会被同一个 `Spout` 来 ack 或者 fail。所以如果一个 `Spout` 是作为多个任务在集群中执行的，一个 tuple 是不会被创建它的任务以外的任务来 ack 或者 fail 的。

让我们再用 `KestrelSpout` 来看看 `Spout` 需要做什么来保证消息被处理。当 `KestrelSpout` 从 Kestrel 队列中取出一条消息的时候，它会“打开”这条消息。这意味着实际上还没有从队列中被取走，而是被放到了一个 “pending” （等待）状态，等待消息处理结束的确认。此外，如果客户端断开了，那么所有等待状态的消息都会被放回队列中去。当一条消息被打开后，Kestrel 会向客户端提供那条消息的数据以及一个独一无二的 id。`KestrelSpout` 在把 tuple 输出到 `SpoutOutputCollector` 的时候使用完全相同的 id 来作为 tuple 的 “message id”。之后的某个时刻，当 `ack` 或 `fail` 在 `KestrelSpout` 上被调用的时候，`KestrelSpout` 会发送一个 ack 或者 fail 消息到 Kestrel 并带上 message id，以便能从队列中去掉或者加回该消息。

### 什么是 Storm 的可靠性 API？

作为用户你要做两件事才能获益于 Storm 的可靠性功能。首先，你必须要在任何在 tuple 树里面创建新链接的时候告诉 Storm。其次，你要在完成处理一条 tuple 的时候告诉 Storm。做到了这些，Storm 才能够检测什么时候一棵 tuple 树被完整地处理了，才能适当地 ack 或者 fail 一个 spout 的 tuple。Storm 的 API 提供了简洁的实现方式。

在一棵 tuple 树里面指定一个连接叫做 _锚定_。你在省城一个新的 tuple 的时候锚定就会发生。让我们用下面的 bolt 做例子。这个 bolt 把一个含有句子的 tuple 分拆成每个词一个的 tuple。

```java
public class SplitSentence extends BaseRichBolt {
        OutputCollector _collector;
        
        public void prepare(Map conf, TopologyContext context, OutputCollector collector) {
            _collector = collector;
        }

        public void execute(Tuple tuple) {
            String sentence = tuple.getString(0);
            for(String word: sentence.split(" ")) {
                _collector.emit(tuple, new Values(word));
            }
            _collector.ack(tuple);
        }

        public void declareOutputFields(OutputFieldsDeclarer declarer) {
            declarer.declare(new Fields("word"));
        }        
    }
```

程序通过在 `emit` 的第一个参数提供输入 tuple 来  _锚定_  每个单词 tuple。 单词 tuple 被定位后，一旦单词 tuple 在下游处理失败，在树根的 spout tuple 就能够被重新发出。为了对比，让我们来看看如果单词 tuple 是被这样输出的会发生什么：

```java
_collector.emit(new Values(word));
```

这样输出一个单词 tuple 会导致其 _未锚定_ 。如果这个 tuple 在下游处理失败，根上的 tuple 就不会被重发。不过根据你在你的 topology 里面需要的容错性保证，有时也需要输出一个未锚定的 tuple。

一个输出 tuple 能够被锚定到超过一个输入 tuple 上。在做流连接（join）和流聚集（aggregation）的时候很有用。一个拥有多个锚点的 tuple 处理失败会导致多个 tuple 从 spout 被重新发出。多锚点可以通过指定一个列表的 tuple 而不是一个 tuple 来指定。比如：


```java
List<Tuple> anchors = new ArrayList<Tuple>();
anchors.add(tuple1);
anchors.add(tuple2);
_collector.emit(anchors, new Values(1, 2, 3));
```

多锚点把输出 tuple 加到了多个 tuple 树。注意多锚点也可能打破一棵树结构并创建一个 tuple DAG 图，像这样：

![Tuple DAG 图](images/tuple-dag.png)

Storm 的实现既支持 DAG 图也支持树（pre-release 版只支持树，所以 “tuple 树” 这个名字就留下来了。）。

锚点是你指定 tuple 树的方法 —— Storm 的可靠性 API 的下一个也是最后一个部分，就是如何指示你处理完了 tuple 树里面单独的一条 tuple。这是通过 `OutputCollector` 上的 `ack` 和 `fail` 方法来实现的。如果你回头看看 `SplitSentence` 例子，你会发现输入 tuple 是在所有的单词 tuple 都被输出之后被 ack 的。

你可以使用 `OutputCollector` 的 `fail` 方法来使 spout tuple 在 tuple 树的根上立即失败。比如，你的应用程序可能会捕捉到一个数据库客户端抛出的异常，并明确地 fail 掉输入 tuple。明确的 fail 和等待 tuple 超时 相比，能够使得 spout tuple 更快地被重发，

你处理的每个 tuple 都必须被 ack 或者 fail。Storm 使用内存来跟踪每个 tuple，所以任务最终会耗尽内存。

很多 bolt 都依循一个常见的模式来读一个输入 tuple，基于它输出一个 tuple，并在 `execute` 方法的结尾 ack 这个 tuple。这些 bolt 都可以被归到 filter （过滤器）或者和简单函数一类。Storm 有一个叫做 `IBasicBolt` 的接口就为你包装了这些模式。`SplitSentence` 例子也可以用 `IBasicBolt` 来像这样重写：

```java
public class SplitSentence extends BaseBasicBolt {
        public void execute(Tuple tuple, BasicOutputCollector collector) {
            String sentence = tuple.getString(0);
            for(String word: sentence.split(" ")) {
                collector.emit(new Values(word));
            }
        }

        public void declareOutputFields(OutputFieldsDeclarer declarer) {
            declarer.declare(new Fields("word"));
        }        
    }
```

这个实现比之前的简单，但是语义上完全相同。输出到 `BasicOutputCollector` 的 tuple 都被自动地锚定到了输入 tuple 上，在方法执行结束的时候，它也帮你自动 ack 了输入 tuple。

相比之下，进行连接（join）和聚集（aggregation）的 bolt 就会把 ack 推迟到它基于一些 tuple 算出来结果之后。聚集和连接通常都会对其输出 tuple 使用多个锚点。这些都超出了 `IBasicBolt` 的简单模式。

### 怎样让我的应用程序在 tuple 可能重发的情况下正确工作？

如同在软件设计中常说的，答案是“依情况而定”。Storm 0.7.0 引入了 “事务 topology” 功能，允许你在大部分计算上使用完全容错的”刚好一次“的消息传递语义。在 [这里](Transactional-topologies.html).  可以读到关于事务 topology 的更多信息。

### Storm 如何高效地实现可靠性？

一个 Storm topology 有一系列特殊的 “acker” 任务，它们为每一个 spout tuple 跟踪 tuple DAG 图。当一个 acker 发现一个 DAG 图处理完毕后，它会发一个消息到创建 spout tuple 的任务来 ack 消息。你可以通过 [Config.TOPOLOGY_ACKERS](/apidocs/backtype/storm/Config.html#TOPOLOGY_ACKERS) 来设置 acker 任务的数量。Storm 给 TOPOLOGY_ACKERS 缺省值是一个任务。对于处理很多信息的 topology 你需要增大此数字。

The best way to understand Storm's reliability implementation is to look at th理解 Storm 的可靠性实现的最佳方法是观看 tuple 和 tuple DAG 图的生命周期。当一个 tuple 在 topology 中被创建的时候，不管是在 spout 里还是在 bolt 里，它都会有一个随机的 64 位 id。Acker 就使用这些 id 来对每个 spout tuple 跟踪 tuple DAG 图的情况。

每个 tuple 都知道其所为之存在的所有 spout tuple 的 tuple 树里的 id。当你在 bolt 里输出一个新 tuple的时候，来自 tuple 的锚点的 spout tuple id 都被复制进了新的 tuple。当一个 tuple 被 ack 的时候，它会送一条消息到合适的 acker 任务，带上该 tuple 树如何变化了的信息。特别地它还告诉 acker “现在我处理完这个 spout tuple 的树，这里是树里面锚定在我身上的新 tuple”。

比如，如果 tuple “D” 和 “E” 是基于 tuple “C” 创建的，这就是 tuple 树在 “C” 被 ack 的时候会发生的变化：

![ack 的时候发生的事情](images/ack_tree.png)

因为 “C” 已经在 “D“ 和 “E” 被加到树的同时被删除了，树永远都不会过早地结束处理。

关于 Storm 如何跟踪 tuple 树还有另外一些细节。如前所述，你可以在 topology 中有任何数量的 acker 任务。这也就带来了下面这个问题：当一个 tuple 在 topology 中被 ack 的时候，它怎么知道向哪个 acker 任务发送那些信息呢？
  
Storm 使用 mod hashing（余数散列）来吧一个 spout tuple id 映射到一个 acker 任务上。每个 tuple 都带有它所在的每棵树的 spout tuple id，它们也就知道该和哪些 acker 任务通信。

另一个关于 Storm 的细节是 acker 任务是如何跟踪哪个 spout 任务负责哪个 spout tuple 的。当一个 spout 任务输出一个新 tuple 的时候，它只是向合适的 acker 发一条消息告诉它自己的任务 id 是负责那个 spout tuple 的。当 acker 发现一棵树处理结束后，它就知道改向哪个任务 id 发送完成消息。

Acker 任务并不明确地跟踪 tuple 树。对于那些有上万结点（乃至更多）的 tuple 树，跟踪所有的 tuple 树可能会耗尽 acker 的内存。取而代之，acker 使用另一种每个 spout tuple 只需要固定内存（大概 20 字节）的策略。该跟踪算法是 Storm 工作的关键也是其主要技术突破。

一个 acker 任务保存了一个从 spout tuple id 到一对数值的 map。第一个值是创建了 spout tuple 的任务 id，这被用于之后发送完成消息。第二个值是一个叫做 “ack val” 的 64 位数字。Ack val 代表了整个 tuple 树的状态，不管它有多大或者多小。它就是树里所有曾被创建的或被 ack 的 tuple id 的异或值（xor）。

当一个 acker 任务发现一个 “ack val” 变成 0 了，它就知道整个 tuple 树 处理结束了。因为 tuple id 是 64 位随机数，一个 “ack val” 碰巧变成 0 的几率是非常低的。如果你算一算，以每秒 10K 个 ack 的速度，也需要 50,000,000 年才发生一个错误。即使到了那时，如果一个 tuple 在 topology 里面失败了，这也只会造成数据丢失。

既然你理解了可靠性算法，让我们来看看所有的失败情况，看看 Storm 如何在每一种情况下避免数据丢失：

- **因为任务死了一个 tuple 没有被 ack** ：这种情况下在失败 tuple 的树的树根处的 spout tuple id 会超时并被重发。
- **Acker 任务死了** ：这种情况下 acker 跟踪的所有 spout tuple 都会超时并被重发。
- **Spout 任务死了** ：这种情况下 spout 使用的源会负责重发消息。比如，像 Kestrel 和 RabbitMQ 一类的队列会在客户端断掉的时候把所有等待中（pending）的消息都重新放回队列。

如你所见，Storm 的可靠性机制是完全分布式的，可扩展的，并且能够容错。

### 可靠性的调试

Acker 任务都很轻量级，所以在一个 topology 中你并不需要用很多。你可以通过 Storm UI （部件 id 叫做 “__acker”）来跟踪他们的性能。如果吞吐量并不理想，你可以尝试添加更多的 acker 任务。

如果可靠性对你并不重要 —— 也就是说，你不在乎在失败的情况下丢失 tuple —— 那么你就可以通过不跟踪 spout tuple 的 tuple 树来提高性能。不跟踪 tuple 树使得传递的消息数量减半，因为一般而言对于 tuple 树里面的每个 tuple 都有一个 ack 消息。此外，对每个下游的 tuple 我们也只需保留更少的 id，减少带宽使用。

有三种方式可以关掉可靠性。第一种是把 Config.TOPOLOGY_ACKERS 设为 0。这种情况下 Storm 会在 spout 输出 tuple 之后 立刻调用 `ack` 方法。Tuple 树不会被跟踪。

第二种方法是按消息关掉可靠性支持。你可以通过在 `SpoutOutputCollector.emit` 方法中略去消息 id 的方式关掉对于单个 spout tuple 的跟踪。

最后，如果你无所谓 topology 的下游的某个子集的 tuple 处理失败，你可以把它们输出成未锚定的 tuple。因为它们没有被锚定到任何 spout tuple，它们也不会在自己未被 ack 的情况下导致任何 spout tuple 失败。
