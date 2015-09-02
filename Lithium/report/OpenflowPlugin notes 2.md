OpenDaylight OpenFlow Plugin 过载保护
=====================================

# 过载保护

OF Plugin中的过载保护按如下流程工作： 

ConnectionConductor将消息送入队列，是最靠近OFJava的部分，其中的on*Message方法监听协议栈入消息。ConnectionConductor实现类(ConnectionConductorImpl)将消息送入QueueKeeper，每个ConnectionConductor拥有一个本地的QueueKeeper实例。

QueueKeeper拥有两条队列，均为有限大小及阻塞式：

- Unordered (用于packetIn消息)
- Ordered (用于除packetIn以外的消息)

当某条queue充满后，继续送往该队列的消息将被丢弃。一旦队列能够重新送入消息，harverster将从休眠中被唤醒

QueueZipper包装了2条队列，提供poll方法定期轮询队列，当轮询的队列为空时，则轮询下一队列。(详见QueueKeeperFairImpl)

每个QueueKeeper在QueueKeeperHarvester中注册。Harvester运行一个线程，轮询所有注册的QueueKeeper。轮询消息排队送往QueueProcessor。当所有注册的QueueKeeper为空，Harvester进入休眠

QueueProcessor中多个线程负责将OFJava-API消息转译为MD-SAL消息(保序)。QueueProcessor使用2个线程池：

- 一个线程池处理队列事务
- 另一个线程池将消息发布到MD-SAL

队列充满可能有如下原因：

- MD-SAL过载
- 节点发生泛洪，或其他原因造成处理流程减速

如QueueProcessor中的队列充满，将阻塞Harvester。一旦Harvester阻塞，则QueueKeeper中的队列将无法排空。

>Note: The current implementation of the feature offers no checking of the memory or cpu load to actively throttle messages.

![ofplugin_overload_protection](https://wiki.opendaylight.org/images/7/71/OverloadProtectionBrief.png)

# 过载保护效果

- 当某个节点发生泛洪，来自其他节点的消息不会被阻塞
- 平等消息处理，发生泛洪节点的消息不会优先处理
- 内存不会耗尽，因为消息无法送入本地的队列，一旦失败就会丢弃
- 此功能不会对netty层产生压力
