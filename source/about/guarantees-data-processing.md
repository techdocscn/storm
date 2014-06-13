---
layout: about
---

Storm的一个核心机制就是能够高效跟踪每一个tuple在topology中的lineage，从而保证完整的处理每一个tuple。可以到[这里](/documentation/Guaranteeing-message-processing.html)进一步了解它是如何工作的。

Storm的基本抽象保障对于每个 tuple 至少处理一次，这跟常用的队列系统是一样的。消息（messages）仅仅在失败的情况下会被重新处理。

[Trident](/documentation/Trident-tutorial.html)是基于 Storm 的高级抽象。通过使用Trident，你可以在语义上保证对于tuple处理一次且仅一次。
