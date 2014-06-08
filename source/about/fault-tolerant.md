---
layout: 关于
---

Storm 的可容错性：当workers实效或者出现故障的时候，storm 会自动重启它们；如果一个节点（node）出现故障，storm会选择另外一个健康的节点（node）并重新运行这些worker。

Storm 守护程序（daemons），Nimbus 和 Supervisors，是无状态并且能迅速故障转移（fail-fast）。如果这些守护程序实效，它们会自动重新启动，而绝不会对系统有任何影响。这也就是说你可以运行*kill -9*杀死 Storm 守护程序，Nimbus和Supervisors，这对你的集群（cluster）或者topologies的健康运行毫无影响。

如果你有兴趣，可以进一步阅读 Storm 的可容错性[手册](/documentation/Fault-tolerance.html).

