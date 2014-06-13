---
layout: about
---

Storm 可以和任何队列系统和数据库系统集成。Storm 的 [spout](/apidocs/backtype/storm/spout/ISpout.html) 抽象使得和任何队列系统的集成都非常容易。队列系统集成包括：

1. [Kestrel](https://github.com/nathanmarz/storm-kestrel)
2. [RabbitMQ / AMQP](https://github.com/Xorlev/storm-amqp-spout)
3. [Kafka](https://github.com/nathanmarz/storm-contrib/tree/master/storm-kafka)
4. [JMS](https://github.com/ptgoetz/storm-jms)
5. [Amazon Kinesis](https://github.com/awslabs/kinesis-storm-spout)

同样的，把 Storm 和数据库系统集成也非常容易。你只需要简单地打开一个到你数据库的连接，像平时一样读写即可。Storm 会按需处理并行化、数据分割、出错重试等问题。