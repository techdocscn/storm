---
layout: documentation
---

### 开始参与

Storm 的某些［问题库］(https://issues.apache.org/jira/browse/STORM) 中的问题有 "Newbie" 标识。如果你有兴趣参与到 Storm 的开发并做出贡献，你可以从这些简单问题开始。它们将有助于你迅速热身并逐渐熟悉整个代码库。这些问题多数都是一些局部问题，往往只需要一些小小的改动，即可完成。

### 学习代码库

维基的 [实现文档](Implementation-docs.html) 部分对于整个代码库有一个详细的介绍。这个将会对代码库的理解有巨大的帮助。

### 参与过程

用户可以通过 Github 的 pull 请求发送参与请求。如果对于这个 pull 请求有任何问题，大家可以用 GitHub 的评论功能进行进一步的讨论和互动。

对于小的补丁（patch），用户可以直接提交 pull 请求。对于大的提交，用户应当按照如下步骤进行。该过程是为了能够在设计早期就发现问题从而避免不必要的工作的浪费。

1. 在［问题库］(https://issues.apache.org/jira/browse/STORM) 中如果没有这个问题，创建一个新的问题（issue）
2. 在你创建的问题下进行评论，解释你要做的事情，以及你可能会改动的代码，以及这些计划的改动会如何融入到整个项目中。
3. Storm 委员会将会和你进行沟通和交互，确保你的计划和设计是正确的
4. 实现和解决问题，发送一个 pull 请求，开始你的工作。

### 基于 Storm 的模块

基于 Storm 的一些模块（比如 spout，bolt 等）不适合于 Storm 核心包裹的提交，这种情况下你可以创建你自己的项目，或者将你的项目作为 [@stormprocessor](https://github.com/stormprocessor) 的一个部分。你必须将你自己的项目放到你自己的 Github，并给 email 组发信请求将你的项目并入到 @stormprocessor 中。社区会对于这个请求进行讨论，从而决定你的项目是否有用。如果可以，你将会被加入到 @stormprocessor 组织，于是你可以在里面管理你的项目。将你的项目加入 @stormprocessor 将有利于使你的项目被更多其他用户发现并使用。

### 改进文档

对于文档的改进的贡献也是非常重要的。最好的参与方法是通过 email list 发送 email。
