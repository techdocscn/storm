---
layout: documentation
---
本页说明了建立和运行一个 Storm 集群的步骤。如果你使用 AWS， 你应该看一下 [storm-deploy](https://github.com/nathanmarz/storm-deploy/wiki) 项目。[storm-deploy](https://github.com/nathanmarz/storm-deploy/wiki) 完全自动化了基于 EC2 的 Storm 集群的准备，配置和安装。它还给你建立了 Ganglia 让你来查看 CPU, 磁盘和网络的用量。

如果你遇到有关 Storm 集群的困难，首先尝试在 [Troubleshooting](Troubleshooting.html) 页里面寻找解决方法。如果找不到的话，可以给邮件组发电子邮件。

这里是设置一个 Storm 集群的过程摘要：

1. 建立一个 Zookeeper 集群
2. 在 Nimbus 和 worker 机器上安装依赖程序
3. 在 Nimbus 和 worker 机器上下载和解压一个 Storm 的发行包
4. 在 storm.yaml 文件中填写必须的配置
5. 使用 "storm" 脚本和你选择的 supervisor 启动监控下的守护进程


### 建立一个 Zookeeper 集群

Storm 使用 Zookeep 来协调集群。Zookeeper **不是**用于消息传递，所以 Storm 交给 Zookeeper 的负载相当低。单独节点的 Zookeeper 集群应该对于大多数情况都是足够的，但是如果你需要 failover 或者在部署大型的 Storm 集群，你可能需要一个大一些的 Zookeeper 集群。部署 Zookeeper 的指令可以在[这里](http://zookeeper.apache.org/doc/r3.3.3/zookeeperAdmin.html)找到。

一些有关部署 Zookeeper 的要点：

1. 因为 Zookeeper 是一个快速报错的设计，在遇到未知情况的时候会结束运行，在监控下运行 Zookeeper 非常重要。请查看[这里](http://zookeeper.apache.org/doc/r3.3.3/zookeeperAdmin.html#sc_supervision)以获得更多细节。
2. 建立一个 cron 来压缩 Zookeeper 的数据和事务记录非常重要。Zookeeper 守护进程自己不会做这个工作，如果你不建立 cron 的话，Zookeeper 会很快占满磁盘空间。请查看[这里](http://zookeeper.apache.org/doc/r3.3.3/zookeeperAdmin.html#sc_maintenance)来获得更多细节。


### 在 Nimbus 和 worker 机器上安装依赖程序

接下来你需要在 Nimbus 和 workder 机器上面安装依赖程序。它们是：

1. Java 6
2. Python 2.6.6

这些是通过了和 Storm 一起测试的版本。Storm 可能会也可能不会和其它版本的 Java 和／或者 Python 一起工作。

### 在 Nimbus 和 worker 机器上下载和解压一个 Storm 的发行包

接下来，在 Nimbus 和每一台 worker 机器上面下载和解压一个 Storm 的发行包。Storm 发型包可以[从这里](http://github.com/apache/incubator-storm/downloads)下载。

### 在 storm.yaml 文件中填写必须的配置
Storm 发行包包含一个用来配置 Storm 守护进程的 `conf/storm.yaml` 文件。你可以在[这里](https://github.com/apache/incubator-storm/blob/master/conf/defaults.yaml)查看缺省的配置值。storm.yaml 覆盖在 defaults.yaml 的缺省值。有几个配置是运行集群所必须的：

1) **storm.zookeeper.servers**: 这是一个在 Zookeeper 集群中的主机列表。它看起来应该象：

```yaml
storm.zookeeper.servers:
  - "111.222.333.444"
  - "555.666.777.888"
```

如果你的 Zookeeper 集群用的端口号不同，你应该同时设置 **storm.zookeeper.port** 。

2) **storm.local.dir**: Nimbus 和 Supervisor 守护进程需要一个本地磁盘上的目录来存放一些小的状态数据（比如 jars, confs 等）。你应该在每个机器上产生这个目录，设置好合适的全县，然后用它配置这个值。例如：

```yaml
storm.local.dir: "/mnt/storm"
```

3) **nimbus.host**: worker 节点需要知道哪台机器是 master 以便下载 topology jars 和 confs。例如：

```yaml
nimbus.host: "111.222.333.44"
```

4) **supervisor.slots.ports**: 对于每个 worker 机器，你需要配置多少个 workers 在这台机器上用这个配置来运行。每个 worker 使用一个单独的端口来接受消息，这个设置定义了哪些端口可以使用。如果你在这里定义5个端口，Storm 将只会运行最多5个 workers。如果你定义3个端口，Storm 最多运行3个 workers。缺省情况下，这个设置被设成4个 workers 运行在端口 6700, 6701, 6702, 6703。例如：

```yaml
supervisor.slots.ports:
    - 6700
    - 6701
    - 6702
    - 6703
```

### 使用 "storm" 脚本和你选择的 supervisor 启动监控下的守护进程

最后一步是启动所有的 Storm 守护进程。在监控下启动这些守护进程非常重要。 Storm 是一个__快速报错__系统，也就是说任何时候当它遇到一个未知错误的时候就会停止运行。Storm 被设计成可以在任何一点停止然后在进程重新启动以后正确恢复。这就是为什么 Storm 不保留运行状态 -- 如果 Nimbus 或者 Supervisors 重新启动，运行中的 topologies 不会受到影响。下面是如果运行 Storm 守护进程：

1. **Nimbus**: 在 master 机器上面在监控下执行命令 "bin/storm nimbus"。
2. **Supervisor**: 在每个 worker 机器上面在监控下运行命令 "bin/storm supervisor"。supervisor 守护进程负责在该机器上启动和终止 worker 进程。
3. **UI**: 运行 "bin/storm ui" 命令来启动 Storm UI （一个你可以从浏览器访问的提供集群和 topologies 调试信息的网站）。这个 UI 可以在浏览器里用 http://{nimbus host}:8080 访问。

正如你看到的那样，运行守护程序是很清楚的。守护程序会在你解压 Storm 发行包的 logs/ 目录下记录日志。