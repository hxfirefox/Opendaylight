Lithium中关键特性更新
====================

#Controller

**更新：**

- MD-SAL集群额外提升，API拓展

**交付：**

- BUG-2351 Performance improvements for MD-SAL 
通过API与MD-SAL实现变化提升性能
- BUG-2348 Improve operations & monitoring
简化MD-SAL的监控并提升可除错性
- BUG-2262 Clustering: NormalizedNode serialization improvements
提升NormalizedNode序列化性能并降低内存使用

|Bug|Relationship|Description|Deliverable|
|:----:|:----|:----|:----|
|2622|parent|Utilize NormalizedNode streaming classes in Clustering code||
|2664||Use Normalized Node streaming code when persisting snapshots and replicating snapshots[1]|https://git.opendaylight.org/gerrit/#/c/14620/|
|2265||Migrate messages that contain NormalizedNode into java serializable so that they can use the NormalizedNodeStreamWriter[2]|https://git.opendaylight.org/gerrit/#/c/12448/ https://git.opendaylight.org/gerrit/#/c/14476/ https://git.opendaylight.org/gerrit/#/c/14510/|
|2266||Add more unit tests for the NormalizedNodeStreamWriter|https://git.opendaylight.org/gerrit/#/c/13038 - stable/helium https://git.opendaylight.org/gerrit/#/c/13275 - master|
|2267||Optimize the stream writer to reuse the builder|https://git.opendaylight.org/gerrit/#/c/12406/|
|2268||Convert Raft messages that will carry NormalizedNode into java serializables|https://git.opendaylight.org/gerrit/#/c/14483/|


>注
>[1] 以下情景使用NormalizedNode stream writer：

>1. 快照持久化

> - 从存储中读出数据的事务不应序列化NormalizedNode
> - CaptureSnapshotReply应实现Serializable来传递NormalizedNode至RaftActor 
> - 捕获的快照将不存储在内存中

>2. 快照安装
> - 将消息按java序列化转化并通过akka写入流
> - 安装快照时创建快照
>
>[2] 迁移以下消息

> - WriteData
> - MergeData
> - ReadDataReply
> - DataChanged

>当转换上述消息时应确保WriteData/MergeData中的实例标识按stream wirter紧凑格式

#AAA

**更新：**

- 增加数据持久化，Federation，SSO，额外拓展和提升

#ALTO

**新增：**
 - 实现应用层数据优化（ALTO）协议，向应用提供网络信息
 - 为ODL实现ALTO的北向API
 
 >ODL提供的拓扑服务主要关注网络，对于应用而言暴露过多的网络细节，ALTO提供简化网络视角指导应用使用网络资源，详见RFC7285
 >大部分项目参与者为国内人员，包括清华、同济以及华为

#BGP/LS PCEP

**更新：**

- 增加BGP Flowspec，graceful restart，段路由，PCEP安全传输
 - 涉及如下RFC：5886，4486，5492，6286，5004，5575，7311，4724

#CAPWAP

**新增：**
 - 实现ODL管理WTP网络设备

#Device Identification and Drive Management

**新增：**

 - 应对特定设备功能的需求，所谓特定设备功能是指设备执行某一特性时的性能及限制，例如，配置VLAN和调整流表是设备的特性，不同设备对于上述特性采用不同的实现方式
 - DIDM支持以下功能：
  - **Discovery** - 检测设备是否位于控制器管理域，并建立连接。对于不支持OpenFlow的设备，可以通过手动注入设备信息方式发现，如GUI或REST API
  - **Identification** – 判断设备类型
  - **Driver Registration** – 注册设备
  - **Synchronization** – 收集设备信息、配置和链路信息
  - **Define Data Models for Common “Features”** – 数据模型定义为执行通用特性，如VLAN配置
  - **Define RPCs for Common “Features”** – 为上述特性定义RPC，Drivers实现特定设备的特性及RPC
 - DIDM使用SNMP南向插件并依赖AAA

#DLUX

**更新：**

- 支持Topology Framework、Topology Extension Points、Enhanced visualization capabilities

#Group Based Policy

**更新：**

- 增加对OpenStack Neutron的支持，支持Service Function Chaining，OfOverlay对NAT的支持，table offsets

#IoTDM

**新增：**

- 以数据为中心的中间件作为兼容oneM2M的IoTDM，并授权应用获取IoT数据

#LACP

**新增：**

- 以MD-SAL服务方式实现链路聚合控制协议（LACP），用于自动发现和聚合控制器与交换机之间的多条链路 

#Lisp Flow Mapping Service

**更新：**

- 改进ELP处理，北向API转为MD-SAL，Service Function Chaining的持续集成，Neutron提升

#Network Intent Composition

**新增：**

- 基于网络行为和策略的“意图”管理和指导网络服务及资源
- 使用新的北向接口，提供通用抽象的策略语义，而不是类似OpenFlow的流规则
- 面向SDN应用，如OpenStack Neutron，Service Function Chaining，Group Based Policy
- 可以使用控制协议有Openflow，OVSDB，I2RS，Netconf，SNMP

#Neutron & OVSDB Services

**更新：**

- API转为MD-SAL，增加功能校验支持LBaaS，增加Distributed Virtual Router、SNAT、External Gateway及Floating IP支持

#OpenFlow Plugin

**更新：**

- Lithium版本的OF Plugin进行了重新架构，支持基于MD-SAL的OF1.0、OF1.3，增加对TTP的支持
