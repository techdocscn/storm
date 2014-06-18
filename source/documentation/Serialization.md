---
layout: documentation
---
本页面是关于 0.6.0 版之后的 Storm 中的序列化系统如何工作。Storm 在 0.6.0 版之前使用一个不同的序列化系统，在 [序列化（0.6.0 版之前](Serialization-\(prior-to-0.6.0\).html) 有文档介绍。

Tuple 可以包含任何类型的对象。因为 Storm 是一个分布式系统，它需要知道对象在任务间传递的时候如何序列化和反序列化。

Storm 使用  [Kryo](http://code.google.com/p/kryo/)  来序列化。Kryo 是一个灵活、快速的序列化库，生成的序列化结果也很小。

默认地，Storm 已经能够序列化原始类型、字符串、字节数组、ArrayList、HashMap、HashSet，以及 Clojure collection 类型。如果你想在你的 tuple 中使用其它类型，你需要注册一个自定义序列化器。

### 动态类型

Tuple 中的字段没有类型声明。你只需要把对象放到 tuple 中 Storm 会自动发现如何动态序列化。在我们讨论序列化接口之前，让我们先花点时间搞清楚为什么 Storm 的 tuple 是动态类型的。

为 tuple 的字段添加静态类型会给 Storm API 带来大量复杂性。Hadoop，比如说，就对其键（key）和值（value）使用静态类型，但是需要用户大量地标注。Hadoop API 用起来颇为劳累，而且“类型安全”也得不偿失。动态类型用起来就要简单些。

更远来说，也不可能静态地为 Storm 的 tuple 决定类型。比如一个 bolt 订阅了多个流。来自那些流的 tuple 可能在字段上有不同的类型。当一个 bolt 在 `execute` 里收到一个 `Tuple` 的时候，tuple 可能是来自任何一个流，所以可能有任何类型组合。也许通过一些反射（reflection）的魔术你可以为 bolt 订阅的每个 tuple 流声明一个不同的方法，但是 Storm 倾向于使用动态类型这种更能简单、直接的方法。

最后，另一个使用动态类型的理由是这样 Storm 才在诸如 Clojure 和 JRuby 这样的动态类型语言里面流畅地使用。

### 自定义序列化

如前所述，Storm 使用 Kyro 来进行序列化。要实现一个自定义序列化器，你需要在 Kyro 上注册一个新的序列化器。我们强烈推荐你先阅读  [Kryo 的主页](http://code.google.com/p/kryo/)  理解一下它是如何处理自定义序列化的。

添加一个自定义序列化器是通过你的 topology 的配置里的 “topology.kryo.register” 属性来实现的。它也可以接受一个列表的注册项，每个项可以使下面两种形式之一：

1. 要注册的类名。这种情况下，Storm 会使用 Kyro 的 `FieldsSerializer` 来序列化这个类。这可能也可能不是这个类的最佳选择 —— 见 Kyro 文档以知更多详情。
2. 一个从类名映射到  [com.esotericsoftware.kryo.Serializer](http://code.google.com/p/kryo/source/browse/trunk/src/com/esotericsoftware/kryo/Serializer.java)  的一个实现的 map。

让我们看一个例子。


```
topology.kryo.register:
  - com.mycompany.CustomType1
  - com.mycompany.CustomType2: com.mycompany.serializer.CustomType2Serializer
  - com.mycompany.CustomType3
```

`com.mycompany.CustomType1` 和 `com.mycompany.CustomType3` 会用 `FieldsSerializer`，而 `com.mycompany.CustomType2` 则会用 `com.mycompany.serializer.CustomType2Serializer` 来序列化。

Storm 提供了一些帮助函数来在一个 topology 的 config 里面注册序列化器。 [Config](/apidocs/backtype/storm/Config.html)  类有一个方法叫做 `registerSerialization`，它可以接受一个注册项并把它添加到 config 中。

还有一个高级的配置叫做 `Config.TOPOLOGY_SKIP_MISSING_KRYO_REGISTRATIONS`。如果你把这个设置为 true，Storm 就会忽略任何注册了的但是代码不在 classpath 中的序列化。不然，Storm 就会在没法找到序列化的时候抛出异常。你在一个集群上运行多个各有不同的序列化的 topology 时很方便，但是你应该在`storm.yaml`  文件中声明在所有 topology 用到的所有序列化。

### Java 序列化

如果 Storm 遇到了它没有序列化信息的类型，如果可能，它会使用 Java 序列化。如果对象不能通过 Java 序列化器来序列化，Storm 会抛出一个错误。

请注意 Java 序列化非常昂贵，不论是从 CPU 消耗来说还是序列化生成对象尺寸来说。我们强烈推荐你在把 topology 放到产品集群运行的时候注册一个自定义序列化器。支持 Java 序列化仅仅是为了方便快速开发 topology 原型。

你也可以通过把 `Config.TOPOLOGY_FALL_BACK_ON_JAVA_SERIALIZATION` 设成 false 来防止默认退回到 Java 序列化。

### 组件特定的序列化注册

Storm 0.7.0 允许你按组件来设置（关于此点你可以在  [设置](Configuration.html)  一文读到更多）。当然，如果一个组件定义了一个序列化，那么序列化也需要在其它 bolt 能用，不然它们就无法接受来自那个组件的消息了！

当一个 topology 被提交后，仅有单一集合的序列化被选中用于 topology 中的所有组件用来传递消息。这是把各组件特有的序列化器注册项和一个常规的序列化注册项集合合并得到的。如果有两个组件都为同一个类定义了序列化器，那么系统会任意选择一个。

要在两个组件的特定的注册冲突的的情况下为一个特定的类强行选择一个序列化器，你只需要在 topology 特定的配置中定义你要用的那个序列化器。Topology 特定的配置相对于组件特定的序列化注册配置有优先权。
