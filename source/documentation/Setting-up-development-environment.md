---
layout: documentation
---
这篇文章将告诉你如何设置一个 Storm 的开发环境。简而言之，我们需要如下步骤：

1. 下载一个 [Storm 的发布](..//downloads.html)，解包，然后将解包后的目录下 `bin/` 子目录放到你的环境变量 PATH 中
2. 为了使你能够在一个远程的集群（cluster）启动和停止 topology，你需要将该集群的信息放到 `~/.storm/storm.yaml` 文件中。 

请详细阅读以下信息，以更多了解每一步走的细节。

### 什么是开发环境（Development Environment）

Storm 有两种操作模式：本机模式和远程模式。在本机模式下，你可以完全在你的本机上开发和测试你的 topology；而在远程模式下，有必须将你的 topology 提交到一个机器的集群，等待执行。

Storm 开发环境包括安装所有需要的系统使得用户可以在本机模式下开发和测试 topology，能够将 topology 打包，并能远程提交／终止 topology 的运行。

我们可以迅速的了解一下你的本机和远程集群之间的关系。Storm 集群都是由一个叫做 "Nimbus" 的节点来进行管理。用户的本机通过和 Nimbus 通信来提交代码（打包为 jar）和 topology，然后由 Nimbus 来将这些工作分配给集群中的 workers 执行。用户在本机上调用 `storm` 的命令和 Nimbus 通信。该 `storm` 客户端仅在远程模式下使用，而不能用于本机模式下 topology 的开发和测试。

### 本机安装 Storm 发布

如果用户希望能够从本机向远程集群提交 topology，用户必须在本机安装一个 Storm 的发布版本。该安装会给用户 `storm` 客户端程序，用户可以通过这个客户端程序和远程集群交互。用户可以从［这里］(https://github.com/apache/incubator-storm/downloads) 下载，解包到本地机器上，然后将解包的目录下 `bin/` 子目录放到 `PATH` 环境变量中即可（最好确认 `bin/storm` 文件是可执行的）。

在本机安装 Storm 发布只是提供用户和远程集群的交互能力。对于本机的开发以及测试，我们推荐使用 Maven 并将 Storm 作为你的项目（project）的一个 dependency。用户可以到 [Maven](Maven.html) 阅读更多关于 Maven 的细节。 

### 远程集群启动和停止 topology

先前的步骤仅仅是在本机安装 `storm` 客户端以便于建立本机到远程集群的通讯。然后你需要配置客户端，使得它可以和指定的 Storm 集群通讯。你只需要将 master 的地址加入到 `~/.storm/storm.yaml` 文件中即可。比如：

```
nimbus.host: "123.45.678.890"
```

或者，如果你使用 [storm-deploy](https://github.com/nathanmarz/storm-deploy) 项目 在 AWS 上管理你的 Storm 集群，你可以使用 `attach` 命令，手动指定 Storm 集群，也可以在多个集群间转换。

```
lein run :deploy --attach --name mystormcluster
```
更多信息请参阅 Storm 发布 [wiki](https://github.com/nathanmarz/storm-deploy/wiki)