---
layout: documentation
---
## 排除故障

本页列举一些当人们在他们的解决方案中应用 Storm 时遇到的问题。

### Worker 进程在启动时崩溃，并且没有堆栈跟踪（stack trace）

可能的表现：
 
 * Topologies 在只有一个结点时正常工作，但 worker 在多结点时崩溃

解决方案：

 * 你可能有某个子网络没有正确设置，以至于在这个子网中，结点不能根据服务器名称（hostname)找到其他结点。ZeroMQ 在不能解析服务器时会崩溃。有2个解决方案：
  * 在 /etc/hosts 中建立服务器名称到 IP 的映射
  * 配置一个内部 DNS，来让结点能根据服务器名称找到其他结点。
  
### 结点间不能互相通信

可能的表现：

 * 每个 spout 的 tuple 都失败
 * 系统无法处理

解决方案：

 * Storm 不能和 ipv6 一起使用. 你可以在 supervisor child 选项中加入 `-Djava.net.preferIPv4Stack=true` 来强制使用 ipv4。
 * 你可能有某个子网络没有正确设置。参见 `Worker 进程在启动时崩溃，并且没有堆栈跟踪（stack trace）`

### Topology 每过一段时间就停止处理数据

可能的表现：

 * 正常处理一段时间，但是忽然停止工作并且 spout tuples 开始全部失败。
 
解决方案：

 * ZeroMQ 2.1.10有一个 bug. 降级到 ZeroMQ 2.1.7。
 
### 有些 supervisors 不在 Storm 用户界面中显示

可能的表现：
 
 * Storm 用户界面中缺少某些 supervisor 进程
 * Storm 用户界面中的 supervisor 列表在刷新的时候变化

解决方案：

 * 确保 supervisor 的本地目录相互独立，（比如不可以通过 NFS 公用一个本地目录）
 * 尝试删除 supervisor 的本地目录，然后重启守护进程（daemons）。Supervisors 会创建一个唯一的 ID 并本地保存。如果把这个 ID 拷贝给了其他结点，Storm 会出现异常。

### “Multiple defaults.yaml found” 错误

可能的表现：

 * 当使用“Storm jar” 部署 topology 时，出现上述错误

解决方案：

 * 最可能的情况是，你在 topology jar 里面包含了 Storm jar。当你打包你的 topology jar时，不要包括 Storm 的jar。Storm 会自动把 Storm jar 加入你的 classpath。

### 运行 Storm jar 时出现“NoSuchMethodError”

可能的表现：

 * 当你运行 Storm jar 时，出现 “NoSuchMethodError”

解决方案：

 * 你部署topology 时用了和编译 topology 时不同的版本的 Storm。你需要确保使用来自你编译 topology 时用的 Storm 相同版本的客户端。


### Kryo ConcurrentModificationException

可能的表现：

 * 运行时程序跑出下列堆栈跟踪：

```
java.lang.RuntimeException: java.util.ConcurrentModificationException
	at backtype.storm.utils.DisruptorQueue.consumeBatchToCursor(DisruptorQueue.java:84)
	at backtype.storm.utils.DisruptorQueue.consumeBatchWhenAvailable(DisruptorQueue.java:55)
	at backtype.storm.disruptor$consume_batch_when_available.invoke(disruptor.clj:56)
	at backtype.storm.disruptor$consume_loop_STAR_$fn__1597.invoke(disruptor.clj:67)
	at backtype.storm.util$async_loop$fn__465.invoke(util.clj:377)
	at clojure.lang.AFn.run(AFn.java:24)
	at java.lang.Thread.run(Thread.java:679)
Caused by: java.util.ConcurrentModificationException
	at java.util.LinkedHashMap$LinkedHashIterator.nextEntry(LinkedHashMap.java:390)
	at java.util.LinkedHashMap$EntryIterator.next(LinkedHashMap.java:409)
	at java.util.LinkedHashMap$EntryIterator.next(LinkedHashMap.java:408)
	at java.util.HashMap.writeObject(HashMap.java:1016)
	at sun.reflect.GeneratedMethodAccessor17.invoke(Unknown Source)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:616)
	at java.io.ObjectStreamClass.invokeWriteObject(ObjectStreamClass.java:959)
	at java.io.ObjectOutputStream.writeSerialData(ObjectOutputStream.java:1480)
	at java.io.ObjectOutputStream.writeOrdinaryObject(ObjectOutputStream.java:1416)
	at java.io.ObjectOutputStream.writeObject0(ObjectOutputStream.java:1174)
	at java.io.ObjectOutputStream.writeObject(ObjectOutputStream.java:346)
	at backtype.storm.serialization.SerializableSerializer.write(SerializableSerializer.java:21)
	at com.esotericsoftware.kryo.Kryo.writeClassAndObject(Kryo.java:554)
	at com.esotericsoftware.kryo.serializers.CollectionSerializer.write(CollectionSerializer.java:77)
	at com.esotericsoftware.kryo.serializers.CollectionSerializer.write(CollectionSerializer.java:18)
	at com.esotericsoftware.kryo.Kryo.writeObject(Kryo.java:472)
	at backtype.storm.serialization.KryoValuesSerializer.serializeInto(KryoValuesSerializer.java:27)
```

解决方案： 

 * 这个情况意味着你用了一个可变的对象（mutable object）来作为输出 tuple。你必须只输出不可变的对象。这里发生的事情是在这个对象被序列化，以便在网络上传输的同时，你的 bolt 试图改变这个对象。


### 从 Storm 深层抛出的 NullPointerException

可能的表现：

 * 有一个如下所示的 NullPointerException：

```
java.lang.RuntimeException: java.lang.NullPointerException
    at backtype.storm.utils.DisruptorQueue.consumeBatchToCursor(DisruptorQueue.java:84)
    at backtype.storm.utils.DisruptorQueue.consumeBatchWhenAvailable(DisruptorQueue.java:55)
    at backtype.storm.disruptor$consume_batch_when_available.invoke(disruptor.clj:56)
    at backtype.storm.disruptor$consume_loop_STAR_$fn__1596.invoke(disruptor.clj:67)
    at backtype.storm.util$async_loop$fn__465.invoke(util.clj:377)
    at clojure.lang.AFn.run(AFn.java:24)
    at java.lang.Thread.run(Thread.java:662)
Caused by: java.lang.NullPointerException
    at backtype.storm.serialization.KryoTupleSerializer.serialize(KryoTupleSerializer.java:24)
    at backtype.storm.daemon.worker$mk_transfer_fn$fn__4126$fn__4130.invoke(worker.clj:99)
    at backtype.storm.util$fast_list_map.invoke(util.clj:771)
    at backtype.storm.daemon.worker$mk_transfer_fn$fn__4126.invoke(worker.clj:99)
    at backtype.storm.daemon.executor$start_batch_transfer__GT_worker_handler_BANG_$fn__3904.invoke(executor.clj:205)
    at backtype.storm.disruptor$clojure_handler$reify__1584.onEvent(disruptor.clj:43)
    at backtype.storm.utils.DisruptorQueue.consumeBatchToCursor(DisruptorQueue.java:81)
    ... 6 more
```

解决方案：

 * 这是由多线程同时调用`OutputCollector`的方法引起的. 所有的输出，确认，和失败都必须由同一个线程来完成。你可能不小心创建了一个用另一个线程输出的`IBasicBolt`。 当执行被调用时，`IBasicBolt` 会自动确认，以至于不同的线程在使用 `OutputCollector` ，导致了这个异常。当使用基础的 bolt 时，所有的输出必须由运行`execute`的线程运行。