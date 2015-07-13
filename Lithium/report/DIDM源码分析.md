DIDM源码分析
============

版本来源：[GitHub上Opendaylight DIDM项目](https://github.com/opendaylight/didm)

参考资料来源：[DIDM:Developer Guide](https://wiki.opendaylight.org/view/DIDM:Developer_Guide)

#概述
DIDM是设备标识与驱动管理(Device Identification and Driver Management)的缩写，其设计初衷是解决ODL控制器中设备相关功能的统一处理。所谓设备相关功能，是指不同设备执行相同特性功能时所存在的限制以及表现出的性能。例如，在ODL控制器中，最常见的特性需求是配置VLAN和调整流表，即使是这样的特性需求，每个设备厂商的实现也是有所区别的。对于上述厂商实现的特性需求，通常称之为设备驱动，设备驱动是与设备相关功能绑定的，使用这种绑定的前提是识别设备类型，即了解设备是何厂商制造。

#架构
DIDM框架提供了以下功能：

- **发现(Discovery)** - 判断设备是否属于控制器控制，以及是否可以建立与控制器的连接，发现功能提供两种机制：1）使用OpenFlow协议，2）非OpenFlow设备通过手动方式，如GUI或REST API
- **识别(Identification)** – 判断设备类型
- **驱动注册(Driver Registration)** – 采用routed RPCs方式注册设备驱动
- **同步(Synchronization)** – 设备信息、配置、链接搜集
- **通用特性的数据模型(Data Models for Common Features)** – 定义通用特性(例如，VLAN配置)的数据模型，一旦定义所述数据模型，则对通用特性的操作可以通过将数据写入模型的方式进行
- **通用特性的RPC(RPCs for Common Features)** – 通用特性的API采用RPC方式，驱动实现RPC以支持相关特性

#处理流程

>用于手动发现的API将在下个版本中提供

1. 设备与控制器建立连接
2. 在config和operational存储中创建资产节点
3. 完成operational树中的创建后，识别管理(Identification Manager)将收到数据变更通知
4. 识别管理将按照(Openflow，SNMP，若均不是则标识为unknown)的顺序从设备获取信息
5. 完成识别后，将在operational树中增加设备类型信息
6. 当应用设备类型信息时，设备驱动将收到相应的通知
7. 驱动负责收集同步数据
8. 驱动注册routed RPC或数据变更通知

基本过程如下图所示：

![ident mrg](https://wiki.opendaylight.org/images/d/d2/Ident_mrg.jpg)

#接口参考文档

访问http://${CONTROLLER-IP}:8181/apidoc/explorer/index.html，其中的DIDM部分可查阅REST

#源码分析

项目代码结构如图所示：

![project code structure](https://github.com/hxfirefox/Opendaylight/blob/master/Lithium/resource/code_struct.jpg)

整个代码结构可以按加载内容分为两部分，第一部分是DIDM框架代码(除vendor外的目录)，第二部分是DIDM驱动范例(包含了独立的features)。DIDM框架按照features下features.xml中的描述，如下所示，主要包含didm-identification-api、didm-identification-impl、didm-drivers-api上述3个bundle，分别对应识别管理与驱动接口两个部分功能。

```xml
<feature name='odl-didm-identification-api' version='${project.version}' description='OpenDaylight :: didm identification :: api'>
  <feature version='${ofplugin.version}'>odl-openflowplugin-nsf-model</feature>

  <bundle>mvn:org.opendaylight.didm/didm-identification-api/${project.version}</bundle>
</feature>
<feature name='odl-didm-identification' version='${project.version}' description='OpenDaylight :: didm identification'>
  <feature version='${ofplugin.version}'>odl-openflowplugin-nsf-services</feature>
  <feature version='${snmp.version}'>odl-snmp-plugin</feature>
  <feature version='${project.version}'>odl-didm-identification-api</feature>

  <bundle>mvn:org.opendaylight.didm/didm-identification-impl/${project.version}</bundle>
  <configfile finalname="${config.configfile.directory}/didm-identification.xml">mvn:org.opendaylight.didm/didm-identification-impl/${project.version}/xml/config</configfile>
</feature>
<feature name='odl-didm-drivers-api' version='${project.version}' description='OpenDaylight :: didm drivers :: api'>
  <feature version='${ofplugin.version}'>odl-openflowplugin-nsf-model</feature>

  <bundle>mvn:org.opendaylight.didm/didm-drivers-api/${project.version}</bundle>
</feature>
```

##identification

包含identification/api与identification/impl两个模块，其中api为identification的数据模型定义，如DeviceTyps的定义、inventory node的扩展等，采用yang实现；impl为identification的处理流程，包含**处理流程**章节所述的步骤3～5，采用Java及yang实现。

在api中定义了DIDM使用的DeviceTypes基本数据类型，在*didm-identification.yang*中有如下定义：

```
identity device-type-base {
    description "base identity for all device type identifiers";
}

identity unknown-device-type {
    description "Indicates the device type could not be identified.";
    base device-type-base;
}

augment "/inv:nodes/inv:node" {
    ext:augment-identifier "device-type";
    leaf device-type {
        type identityref {
            base device-type-base;
        }
    }
}
```

如上述定义，基本数据类型为device-type-base，用于处理被成功识别的设备；而unkonwn-device-type在此基础上继承得到，处理无法识别的设备，同时使用augment，拓展了inventory node，为其增加了一个叶节点device-type，其中存储设备类型，也即可以从inventory中获得关于设备类型的信息。
在inventory node增加设备类型信息的同时，DIDM还设计了一个存储完整设备信息的datastore，可以通过外部配置设备信息，在*didm-device-types.yang*中进行了定义，如下所示：

```
container device-types {
    list device-type-info {
        key "device-type";

        leaf device-type {
            type identityref {
                base id:device-type-base;
            }
            description "identifier for a list entry, the device type name.";
        }

        leaf openflow-manufacturer {
            type string;
            description "The openflow manufacturer of the device.";
        }

        leaf-list openflow-hardware {
            type string;
            description "The openflow hardware of the device.";
        }

        leaf-list sysoid {
            type string;
            description "The SNMP sysObjectId that uniquely identifies the device";
        }
    }
}
```

此datastore中定义包含信息有device-type(设备类型)，同时也是该datastore的键值；openflow-manufacturer(设备厂商)；openflow-hardware(硬件信息，通常应当是硬件地址信息，即MAC)，注意硬件信息以列表呈现，即设备可能包含多个硬件；sysoid(SNMP相关信息)，同样以列表呈现。

在impl中，*DeviceIdentificationManageMoudle.java*是didm-identification-impl bundle的入口文件，主要作用在于其中的createInstance方法，如下所示，将创建DeviceIdentificationManager实例，此实例将负责处理设备的识别流程。

```java
@Override
public java.lang.AutoCloseable createInstance() {
    LOG.trace("Creating DeviceIdentificationManager instance");
    return new org.opendaylight.didm.identification.impl.DeviceIdentificationManager(getDataBrokerDependency(), getRpcRegistryDependency());
}
```

按照**处理流程**章节中流程图，DeviceIdentificationManager应当以inventory中数据变更事件驱动，因此在创建其实例过程中主要是注册数据变化监听，如下所示：

```java
dataChangeListenerRegistration = dataBroker.registerDataChangeListener(LogicalDatastoreType.OPERATIONAL, NODE_IID, this, AsyncDataBroker.DataChangeScope.BASE);
if (dataChangeListenerRegistration == null) {
    LOG.error("Failed to register onDataChanged Listener");
}
```

当有数据变化消息时，会触发onDataChanged方法的调用，由onDataChanged调用handleDataCreated方法，handleDataCreated方法为接受的每个变化数据，启动一个线程，该线程的任务是等待250ms，再次读取inventory中的FlowCapableNode数据(从其注释看，这么做的原因是需要的信息附着到node上的速度较慢，需要等待250ms)，使用键值为更新数据中的node。数据获得成功后，进入设备识别流程，调用方法identifyDevice。

方法identifiyDevice实现了流程图中描述的步骤3～4，包括

- 监听inventory node变化
- 填充inventory中的DeviceType

首先，从datastore DeviceTypes中读取已经配置的设备信息，包括类型，厂商，硬件信息，对OpenFlow信息进行匹配，匹配源来自node中的FlowCapableNode，如下所示：

```java
List<DeviceTypeInfo> dtiInfoList = readDeviceTypeInfoFromMdsalDataStore();

// 1) check for OF match
FlowCapableNode flowCapableNode = node.getAugmentation(FlowCapableNode.class);
checkOFMatch(path,node,flowCapableNode,dtiInfoList);
```

其中的匹配流程调用checkOFMatch方法，具体执行过程是将厂商及硬件信息进行比较，如厂商信息相同，且硬件信息包含在配置的信息中，则认为设备类型属于，如下所示：

```java
private void checkOFMatch(final InstanceIdentifier<Node> path, Node node, FlowCapableNode flowCapableNode, List<DeviceTypeInfo> dtiInfoList ){
	 if (flowCapableNode != null) {
         String hardware = flowCapableNode.getHardware();
         String manufacturer = flowCapableNode.getManufacturer();
         String serialNumber = flowCapableNode.getSerialNumber();
         String software = flowCapableNode.getSoftware();

         LOG.debug("Node '{}' is FlowCapable (\"{}\", \"{}\", \"{}\", \"{}\")",
                 node.getId().getValue(), hardware, manufacturer, serialNumber, software);

         // TODO: is there a more efficient way to do this?
         for(DeviceTypeInfo dti: dtiInfoList) {
             // if the manufacturer matches and there is a h/w match
             if (manufacturer != null && (manufacturer.equals(dti.getOpenflowManufacturer()))) {
                 List<String> hardwareValues = dti.getOpenflowHardware();
                 if(hardwareValues != null && hardwareValues.contains(hardware)) {
                         setDeviceType(dti.getDeviceType(), path);
                         return;
                 }
             }
         }
     }

}
```

设备匹配后，则调用setDeviceType对inventory node中的DeviceType进行填充。如下所示：

```java
final InstanceIdentifier<DeviceType> deviceTypePath = path.augmentation(DeviceType.class);
final WriteTransaction tx = dataBroker.newWriteOnlyTransaction();

tx.merge(LogicalDatastoreType.OPERATIONAL, deviceTypePath, new DeviceTypeBuilder().setDeviceType(deviceType).build());
```

接下来，验证sysoid，调用SNMPSevice的RPC方法获取了设备的sysoid，如下所示：

```java
FetchSysOid fetchSysOid = new FetchSysOid(rpcProviderRegistry);
String sysOid = fetchSysOid.fetch(ipStr);
```

获取sysoid后与配置信息进行比较，最终也会对inventory node中的DeviceType进行填充，上述流程保证了OpenFlow设备与SNMP设备均可以正确地添加DeviceType。

当OpenFlow与SNMP方式均匹配失败，则设备将被填充未知设备类型，如下所示：

```java
setDeviceType(UNKNOWN_DEVICE_TYPE, path);
```

##drivers
