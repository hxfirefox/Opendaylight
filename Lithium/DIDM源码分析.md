DIDM源码分析
============

版本来源：[GitHub上Opendaylight DIDM项目](https://github.com/opendaylight/didm)

参考资料来源：[DIDM:Developer Guide](https://wiki.opendaylight.org/view/DIDM:Developer_Guide)

#概述
DIDM是设备标识与驱动管理(Device Identification and Driver Management)的缩写，其设计初衷是解决ODL控制器中设备相关功能的统一处理。所谓设备相关功能，是指不同设备执行相同特性功能时所存在的限制以及表现出的性能。例如，在ODL控制器中，最常见的特性需求是配置VLAN和调整流表，即使是这样的特性需求，每个设备厂商的实现也是有所区别的。对于上述厂商实现的特性需求，通常称之为设备驱动，设备驱动是与设备相关功能绑定的，使用这种绑定的前提是识别设备类型，即了解设备是何厂商制造。

#架构
DIDM框架提供了以下功能：

- *发现(Discovery)* - 判断设备是否属于控制器控制，以及是否可以建立与控制器的连接，发现功能提供两种机制：1）使用OpenFlow，2）通过手动方式，如GUI或REST API
- *识别(Identification)* – 判断设备类型
- *驱动注册(Driver Registration)* – 采用routed RPCs方式注册设备驱动
- *同步(Synchronization)* – 设备信息、配置、链接搜集
- *通用特性的数据模型(Data Models for Common Features)* – 定义通用特性(例如，VLAN配置)的数据模型，一旦定义所述数据模型，则对通用特性的操作可以通过将数据写入模型的方式进行
- *通用特性的RPC(RPCs for Common Features)* – 通用特性的API采用RPC方式，驱动实现RPC以支持相关特性
