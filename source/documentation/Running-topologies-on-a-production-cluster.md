---
layout: documentation
---
在产品集群上运行 topology 和在 [本地模式](Local-mode.html) 运行很相似。有以下步骤：

1) 定义 topology（如果使用 Java 定义可用  [TopologyBuilder](/apidocs/backtype/storm/topology/TopologyBuilder.html) ）

2) 使用  [StormSubmitter](/apidocs/backtype/storm/StormSubmitter.html)  来把 topology 提交到集群。`StormSubmitter` 的输入包括 topology 的名字、topology 的配置对象，以及 topology 本身。比如：

```java
Config conf = new Config();
conf.setNumWorkers(20);
conf.setMaxSpoutPending(5000);
StormSubmitter.submitTopology("mytopology", conf, topology);
```

3) 创建一个包含了你的代码和所有依赖库的 jar（除了 Storm 本身 —— Storm 的 jar 会被自动加到工人结点的 classpath 中去）。

如果你使用 Maven， [Maven Assembly Plugin](http://maven.apache.org/plugins/maven-assembly-plugin/)  可以帮你打包。把下面一段加到你的 pom.xml 即可：

```xml
  <plugin>
    <artifactId>maven-assembly-plugin</artifactId>
    <configuration>
      <descriptorRefs>  
        <descriptorRef>jar-with-dependencies</descriptorRef>
      </descriptorRefs>
      <archive>
        <manifest>
          <mainClass>com.path.to.main.Class</mainClass>
        </manifest>
      </archive>
    </configuration>
  </plugin>
```
然后运行 mvn assembly:assembly 来得到打包就绪的 jar。请确保你把 Storm 的 jar 都  [exclude](http://maven.apache.org/plugins/maven-assembly-plugin/examples/single/including-and-excluding-artifacts.html)  出去了，因为集群已经在它的 classpath 上有 Storm 了。

4) 使用 `storm` 客户端把 topology 提交到集群，指定你的 jar 的路径，要运行的类名，以及任何需要的参数：

`storm jar path/to/allmycode.jar org.me.MyTopology arg1 arg2 arg3`

`storm jar` 会把 jar 提交到集群并配置 `StormSubmitter` 类来和正确的集群通信。在这个例子里，在上传了之后，`storm jar` 会在 `org.me.MyTopology` 上用参数 “arg1”、“arg2” 和 “arg3” 来调用主函数。

你可以在  [设置开发环境](Setting-up-development-environment.html)  找到如何配置你的 `storm` 客户端使其和特定的 Storm 集群通信的介绍。

### 常见配置

有一些配置你可以按 topology 设定。你可以在  [这里](/apidocs/backtype/storm/Config.html)  找到所有这些配置。那些 “TOPOLOGY” 开头的都能够按 topology 重载（其它的是集群配置，不能被重载）。这里是一些常见的 topology 设置：

1. **Config.TOPOLOGY_WORKERS** ：这设置用于执行 topology 的工人进程的数量。比如，如果你把这个设置为 25，整个集群就会有 25 个 Java 进程处理所有的任务。如果你的 topology 的所有组件合起来有 150 的并行量，每个工人进程会有 6 个任务以线程方式执行。
2. **Config.TOPOLOGY_ACKERS** ：这个设置用于跟踪 tuple 树并检测一个 spout tuple 是否完整处理的任务数量。Acker 是 Storm 可靠性模型的必要组成部分，你可以在  [消息处理保证](Guaranteeing-message-processing.html) 一文中读到更多。
3. **Config.TOPOLOGY_MAX_SPOUT_PENDING** ：这个设置了能够的同时等待单个 spout 任务的 spout tuple 的最大数量（等待则意味着 tuple 还没有被 ack 或者 fail）。我们强烈建议你设置此参数以防队列爆炸。
4. **Config.TOPOLOGY_MESSAGE_TIMEOUT_SECS** ：这是一个 spout tuple 能够在被认定为失败前被完全处理的时间长度。默认值为 30 秒，对于大多数 topology 都足够了了。见  [消息处理保证](Guaranteeing-message-processing.html)  一文获取关于 Storm 可靠性模型如何工作的更多信息。
5. **Config.TOPOLOGY_SERIALIZATIONS** ：你可以使用这个设置来把更多 serializer （序列化器）注册到 Storm 以便在 tuple 中使用自定义类型。


### 杀死 topology

要杀掉一个 topology，只需运行：

`storm kill {stormname}`

把你在提交 topology 的时候用的名字告诉 `storm kill` 即可。

Storm 不会马上杀死一个 topology。取而代之，它会停用 spout 所以它们不会再输出更多的 tuple，然后 Storm 在毁掉所有工人之前先等待 Config.TOPOLOGY_MESSAGE_TIMEOUT_SECS 秒。这使得 topology 有足够的时间结束处理它被杀时正在处理的 tuple。

### 更新一个运行中的 topology

要更新一个运行中的 topology，现在唯一的选项是先杀掉当前 topology 再提交一个新的。我们计划实现一个 `storm swap` 新功能，它可以把一个运行中的 topology 置换成一个新的，并保证停机时间尽可能少，且两个 topology 不会同时处理在 tuple。

### 监视 topology

监视一个 topology 的最佳地点是 Storm UI。 Storm UI 提供了关于任务中发生的错误以及关于当前运行的 topology 中的每个组件的吞吐量和延迟的详细统计数据的信息. 

你也可以查看集群的机器上工人进程的日志。
