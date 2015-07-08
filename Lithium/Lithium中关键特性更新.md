Lithium中关键特性更新
====================

#1. Lithium特性更新概述

Lithium相对于Helium更新特性共27项，其中原有特性提升或增强13项，新增特性14项，如下表所示

|特性类型|相对于Helium|数量|说明|
|:----:|:----|:----|:----|
|协议类|新增|6|ALTO/Opflex/SNMP Plugin/LACP/Source-Group Tag Exchange Protocol/CAPWAP|
|协议类|更新|2|Openflow Plugin/BGP/LS PCEP|
|服务类|新增|1|Persistence Store Service|
|服务类|更新|4|Lisp Flow Mapping Service/Neutron & OVSDB Services/VPN Service/AAA|
|组件平台框架类|新增|5|Device Identification and Drive Management/Network Intent Composition/Reservation/Topology Processing Framework/Time Series Data Repository|
|组件平台框架类|更新|3|Controller/DLUX/VTN|
|接口策略类|新增|2|IoTDM/Unified Secure Channel|
|接口策略类|更新|4|Group Based Policy/SNMP4SDN/Secure Network Bootstrapping Interface/SFC|

#2. Lithium更新特性分析

##Controller

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

##AAA

**更新：**

- 增加数据持久化，Federation，SSO，额外拓展和提升

##ALTO

**新增：**
 - 实现应用层数据优化（ALTO）协议，向应用提供网络信息
 - 为ODL实现ALTO的北向API
 
 >ODL提供的拓扑服务主要关注网络，对于应用而言暴露过多的网络细节，ALTO提供简化网络视角指导应用使用网络资源，详见RFC7285
 >大部分项目参与者为国内人员，包括清华、同济以及华为

##BGP/LS PCEP

**更新：**

- 增加BGP Flowspec，graceful restart，段路由，PCEP安全传输
 - 涉及如下RFC：5886，4486，5492，6286，5004，5575，7311，4724

##CAPWAP

**新增：**
 - 实现ODL管理WTP网络设备

##Device Identification and Drive Management

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

##DLUX

**更新：**

- 支持Topology Framework、Topology Extension Points、Enhanced visualization capabilities

##Group Based Policy

**更新：**

- 增加对OpenStack Neutron的支持，支持Service Function Chaining，OfOverlay对NAT的支持，table offsets

##IoTDM

**新增：**

- 以数据为中心的中间件作为兼容oneM2M的IoTDM，并授权应用获取IoT数据

##LACP

**新增：**

- 以MD-SAL服务方式实现链路聚合控制协议（LACP），用于自动发现和聚合控制器与交换机之间的多条链路 

##Lisp Flow Mapping Service

**更新：**

- 改进ELP处理，北向API转为MD-SAL，Service Function Chaining的持续集成，Neutron提升

##Network Intent Composition

**新增：**

- 基于网络行为和策略的“意图”管理和指导网络服务及资源
- 使用新的北向接口，提供通用抽象的策略语义，而不是类似OpenFlow的流规则
- 面向SDN应用，如OpenStack Neutron，Service Function Chaining，Group Based Policy
- 可以使用控制协议有Openflow，OVSDB，I2RS，Netconf，SNMP

##Neutron & OVSDB Services

**更新：**

- API转为MD-SAL，增加功能校验支持LBaaS，增加Distributed Virtual Router、SNAT、External Gateway及Floating IP支持

##OpenFlow Plugin

**更新：**

- Lithium版本的OF Plugin进行了重新架构，支持基于MD-SAL的OF1.0、OF1.3，增加对TTP的支持
- 修复Helium存在问题：**统计搜集性能提升**
 - Helium版本中统计采用同一时刻向所有节点发送统计请求，引发统计响应风暴，导致CPU冲高及对MD-SAL存储的压力
 - 按照连接数及各自表(Flow tables、Group、Meter)中的条目数进行动态间隔统计轮询
 - 每个节点在统计间隔中进行统计轮询
 - 只在前一次统计完成后才启动下一次统计
 - 不同统计类型使用不同的统计周期，保持低优先级统计的初始周期间隔，并根据响应数量进行调整
- 新增特性：
  - 处理协议保留端口
  - 拓扑能够在初始阶段频繁变化，到达稳态后平滑变动，Helium版本拓扑发送过多的LLDP，可考虑稳态时使用节点通告来发现拓扑变化
  - 线程模型及报文处理优先级
  - 端口配置
  - 队列配置
  - 角色请求
  - OpenDaylight GUI感知OF1.3，例如：能够配置1.3风格的流表、meter、组表
  - [多控制器和仲裁](https://wiki.opendaylight.org/view/OpenDaylight_OpenFlow_Plugin:Backlog:MultiControllerAndArbiterDesign)
![enter image description here](https://wiki.opendaylight.org/images/d/dc/OFP-Arbiter.jpg)

##Opflex

**新增：**

- Opflex策略代理协调OVS工作，执行基于组的策略模型

##[Persistence Store Service](https://wiki.opendaylight.org/view/Project_Proposals:Persistence)

**新增：**

- 提供应用持久化服务，针对非模型数据，例如AAA、AADS、TSDR等，此框架统一持久化任务，以便应用可遵循系统化的方法实现持久化
- 提供可拓展、可重用的软件平台，包括以下内容：
 - API
 - 抽象实现允许插入多种对象存储引擎，例如JPA、Cassandra、Mongo、HBase等，目前已支持JPA实现
 - 一系列集成测试工具
- SDN应用以下列方法与持久化数据进行交互，包含：
 - 数据定义：定义新数据结构，删除数据结构，修改数据结构
 - 数据维护：插入、更新、删除数据
 - 数据检索：查询数据
- 架构如下，图中RDBMS和NoSQL仅为强调多种不同的数据库(SQL vs NoSQL)，以及在该框架下可以运行多个数据库实例

![enter image description here](https://wiki.opendaylight.org/images/3/37/HighLevelArchitecture.jpg)

- 软件架构如下图所示

![enter image description here](https://wiki.opendaylight.org/images/a/a2/SoftwareArchitecture.png)

##Reservation

**新增：**

- 提供动态低级别的资源预留，用户可以将网络作为一个服务、连接或资源池（端口，带宽），其初衷是提供端到端的多层配置

##Secure Network Bootstrapping Interface

**更新：**

- 为转发元素增加Linux副抽象，并提升特性抽象和引导基础架构

##SFC

**更新：**

- 增加Yang模型来表示SFC；接收来自REST和RPC的需求表达式；构建SFC的界面；LISP支持；负载均衡功能分组；NSH、MPLS及VLAN的OF渲染器；南向REST接口；SFC-OVS for SFF bridges, tunnels, and ports CRUD via OVSDB; 报文分发器；与ODL_GBP集成

##SNMP4SDN

**更新：**

- 增加关于ACL的流配置，通过REST API激活流配置及多厂商支持

##SNMP Plugin

**新增：**

- 针对应用及控制通过SNMP与交换设备进行交互的需求，通过SNMP南向插件，应用变为SNMP管理与设备进行交互，该项目期望实现通用的SNMP实现

##Source-Group Tag Exchange Protocol

**新增：**

- SXP用于传递IP与源组标签绑定信息，源组使用唯一的SGT值标识。防火墙可使用SGT创建独立于拓扑的ACL(源和目的的IP/SGT信息都可被发送至防火墙)。当新的终端接入网络，SXP能够自动地更新防火墙中终端信息。在ODL中，通常使用GBP操作组策略。SXP中的源组与ODL GBP作用相同，因此GBP也可使用SXP SGT

##Topology Processing Framework

**新增：**

- 提供一个框架用于简化拓扑数据的聚合和查询，以便于提供一个统一的拓扑视图，包括多协议、底层及叠加资源

##Time Series Data Repository

**新增：**

- 用于搜集、存储、查询和维护ODL控制器时间序列数据，利用TSDR提供的功能，网络管理员可以利用这些数据驱动应用进行安全风险探测，性能分析，操作配置优化，流量工程，以及自动化智能网络分析

##VPN Service

**更新：**

- L3 VPN服务API，与开源路有套件集成，如Quagga、RYU，OpenStack Neutron集成

##VTN

**更新：**

- 迁移至MD-SAL，显著提升VTN中的策略管理，支持微软SVCMM

##Unified Secure Channel

**新增：**

- USC框架提供中央服务器来协调终端之间的加密通信。USC提供了一个客户端代理，通知控制器对其加密功能，可以指示选择基于业务策略需要加密的流，潜在场景是为控制器之间通信加密
