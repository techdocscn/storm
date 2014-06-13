---
layout: documentation
---
此页是来解释 Storm 容错系统的设计细节。

## 当一个 worker 死亡的时候会发生什么？

当一个 work 死亡的时候，supervisor 会重启它。如果它持续启动失败，不能向 Nimbus 发送 heartbeat 的话，Nimbus 会把这个 work 重新分配给其它的机器。

## 当一个 node 死亡的时候会发生什么？

分配给该机器的 tasks 会超时然后 Nimbus 会把这些 tasks 重新分配给其它的机器。

## 当 Nimbus 或者 supervisor 守护进程死亡的时候会发生什么？

Nimbus 和 supervisor 守护进程被设计成快速报错（当任何非期望情况发生的时候，程序自我终止）和无状态（所有的状态都保存在 Zookeeper 中或者在磁盘上）。就像在 [Setting up a Storm cluster](Setting-up-a-Storm-cluster.html) 中说明的一样，Nimbus 和 supervisor 守护进程必须在类似 daemontools 或者 monit 的工具的监控下运行。所以如果 Nimbus 和 supervisor 守护进程死亡的话，他们会重新启动，就像没有任何事情发生一样。

值得注意的是，没有 worker 程序会受 Nimbus 或者 Supervisors 死亡影响。这和 Hadoop 中如果 JobTracker 死亡，所有的 jobs 会丢失相反，

## Nimbus 是一个单点故障吗？

如果你丢失了一个 Nimbus 节点，workers 仍然会继续工作。另外，supervisors 会继续在 workers 死亡的时候重新启动他们。然后，没有 Nimbus 的时候，workders 不会在需要的时候（比如你失去了一个 worker 机器）被重新分配到其它的机器上。

所以答案是 Nimbus 可以算是一种单点故障。但实际上这应该不是一个大问题，因为当 Nimbus 守护进程死亡的时候没什么灾难会发生。有一些让 Nimbus 高可用性的未来计划。

## Storm 如何保证数据处理？

Storm 提供了保证在 nodes 死亡或者 messages 丢失的情况下保证数据处理的机制。细节请查看 [Guaranteeing message processing](Guaranteeing-message-processing.html) 。
