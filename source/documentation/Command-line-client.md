---
layout: documentation
---
此页包括所有在 “storm” 命令行客户端下可用得命令。请学习[搭建开发环境](Setting-up-a-development-environment.html)中的指令来配置你的 “storm” 客户端与远程集群交流。

这些命令是:

1. jar
1. kill
1. activate
1. deactivate
1. rebalance
1. repl
1. classpath
1. localconfvalue
1. remoteconfvalue
1. nimbus
1. supervisor
1. ui
1. drpc

### jar

语法: `storm jar topology-jar-path class ...`

以提供的参数运行`class`中的 main 方法。这些在 `~/.storm` 中的 storm jars 和配置需要包含在 classpath 中。这个配置过程是为了当 topology 被提交的时候 [StormSubmitter](/apidocs/backtype/storm/StormSubmitter.html) 会上载在 `topology-jar-path` 路径下的 jar。

### kill

语法: `storm kill topology-name [-w wait-time-secs]`

杀死名字是 `topology-name` 的 topology 。Storm 会首先在 topology 的消息 timeout 时间内 停止 topology 的 spouts，以便让所有正在处理中的消息可以完成。然后 Storm 会关闭 workers 和清除他们的状态。你可以用 -w 参数来改变 Storm 在停止和关闭之间的等待时间。

### activate

语法: `storm activate topology-name`

激活指定的 topology 的 spouts.

### deactivate

语法: `storm deactivate topology-name`

停止指定的 topology 的 spouts.

### rebalance

语法: `storm rebalance topology-name [-w wait-time-secs]`

有的时候你可能希望分散开运行中的topology 的 worker。例如，假定你有一个10节点的集群，每个节点运行4个 worker，然后假定你给这个集群增加了另外的10个节点。你可能会希望让 Storm 分散开运行中的 topology 的 workers 让每个 node 运行2个 worker。一个可用的方法是杀死这个 topology 然后重新提交它，但是 Storm 提供了一个更加简单的 “rebalance” 命令来完成这个任务。

Rebalance will first deactivate the topology for the duration of the message timeout (overridable with the -w flag) and then redistribute the workers evenly around the cluster. The topology will then return to its previous state of activation (so a deactivated topology will still be deactivated and an activated topology will go back to being activated).

### repl

语法: `storm repl`

打开一个包含 storm jars 和配置的 classpath 的 Clojure REPL 以便于调试。

### classpath

语法: `storm classpath`

打印出用在 storm 客户端运行这条命令时的 classpath。

### localconfvalue

语法: `storm localconfvalue conf-name`

打印出 `conf-name` 在本地 Storm 配置中的值。这个本地 Storm 配置是由 `~/.storm/storm.yaml` 和 `defaults.yaml` 合并而成。

### remoteconfvalue

语法: `storm remoteconfvalue conf-name`

打印出 `conf-name` 在集群 Storm 配置中的值。 这个集群 Storm 配置是由 `$STORM-PATH/conf/storm.yaml` 和 `defaults.yaml` 合并而成。这个命令必须运行在集群机器上面。

### nimbus

语法: `storm nimbus`

启动 nimbus 守护进程。这个命令必须在类似 [daemontools](http://cr.yp.to/daemontools.html) 或者 [monit](http://mmonit.com/monit/) 的工具的监控下运行。请查看 [建立 Storm 集群](Setting-up-a-Storm-cluster.html) 以获得更多信息。

### supervisor

语法: `storm supervisor`

启动监控守护进程。这个命令必须在类似 [daemontools](http://cr.yp.to/daemontools.html)  或者 [monit](http://mmonit.com/monit/) 的工具的监控下运行。请查看 [建立 Storm 集群](Setting-up-a-Storm-cluster.html) 以获得更多信息。

### ui

语法: `storm ui`

启动 UI 守护进程。这个 UI 为 Storm 集群提供了一个 web 界面来显示运行中的 topologies 的具体状态。这个命令必须在类似 [daemontools](http://cr.yp.to/daemontools.html)  或者 [monit](http://mmonit.com/monit/) 的工具的监控下运行。请查看 [建立 Storm 集群](Setting-up-a-Storm-cluster.html) 以获得更多信息。

### drpc

语法: `storm drpc`

Launches a DRPC daemon. 这个命令必须在类似 [daemontools](http://cr.yp.to/daemontools.html)  或者 [monit](http://mmonit.com/monit/) 的工具的监控下运行。请查看 [建立 Storm 集群](Setting-up-a-Storm-cluster.html) 以获得更多信息。
