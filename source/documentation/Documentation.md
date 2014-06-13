---
layout: documentation
---
### Storm 基础

* [Javadoc](http://nathanmarz.github.com/storm)
* [Concepts](Concepts.html)
* [Configuration](Configuration.html)
* [Guaranteeing message processing](Guaranteeing-message-processing.html)
* [Fault-tolerance](Fault-tolerance.html)
* [Command line client](Command-line-client.html)
* [Understanding the parallelism of a Storm topology](Understanding-the-parallelism-of-a-Storm-topology.html)
* [FAQ](FAQ.html)

### Trident

Trident 是一个 Storm 的替换接口。它提供了 exactly-once 处理，“事务性（transactional）”的数据持久化，和一套通用流分析操作。

* [Trident Tutorial](Trident-tutorial.html)     -- 基本概念和流程
* [Trident API Overview](Trident-API-Overview.html) -- 变换和协调操作
* [Trident State](Trident-state.html)        -- exactly-once 处理和快速持久聚合
* [Trident spouts](Trident-spouts.html)       -- 事务性和非事务性数据输入

### 设置和部署

* [Setting up a Storm cluster](Setting-up-a-Storm-cluster.html)
* [Local mode](Local-mode.html)
* [Troubleshooting](Troubleshooting.html)
* [Running topologies on a production cluster](Running-topologies-on-a-production-cluster.html)
* 通过 Maven [Building Storm](Maven.html) 

### 中级

* [Serialization](Serialization.html)
* [Common patterns](Common-patterns.html)
* [Clojure DSL](Clojure-DSL.html)
* [Using non-JVM languages with Storm](Using-non-JVM-languages-with-Storm.html)
* [Distributed RPC](Distributed-RPC.html)
* [Transactional topologies](Transactional-topologies.html)
* [Kestrel and Storm](Kestrel-and-Storm.html)
* [Direct groupings](Direct-groupings.html)
* [Hooks](Hooks.html)
* [Metrics](Metrics.html)
* [Lifecycle of a trident tuple]()

### 高级

* [Defining a non-JVM language DSL for Storm](Defining-a-non-jvm-language-dsl-for-storm.html)
* [Multilang protocol](Multilang-protocol.html) (如何支持另外一种语言)
* [Implementation docs](Implementation-docs.html)