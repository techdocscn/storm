---
layout: about
---

Storm 从设计上就是支持多语言的。Storm 的核心是一个 [Thrift](http://thrift.apache.org/) [定义](https://github.com/apache/incubator-storm/blob/master/storm-core/src/storm.thrift) 文件，该文件定义并提交 topology。由于 Thrift 本身可以用于多种语言，topology 也可以以任何 Thrift 支持的语言定义和提交。

类似的，spout 和 bolt 亦可以以任何 Thrift 支持的语言定义。非 JVM 的 spout 和 bolt 通过一个[基于 JSON 的协议](/documentation/Multilang-protocol.html) 通过标准输入/输出和 Storm 进行通信。我们已经有基于不同语言的对该协议的实现， 如 [Ruby](https://github.com/apache/incubator-storm/blob/master/storm-core/src/multilang/rb/storm.rb), [Python](https://github.com/apache/incubator-storm/blob/master/storm-core/src/multilang/py/storm.py), [Javascript](https://github.com/Lazyshot/storm-node), [Perl](https://github.com/gphat/io-storm), 还有 [PHP](https://github.com/lazyshot/storm-php).

*storm-starter* 有一个 [样本 topology](https://github.com/nathanmarz/storm-starter/blob/master/src/jvm/storm/starter/WordCountTopology.java)，它用 Python 实现了其中一个 bolt。
