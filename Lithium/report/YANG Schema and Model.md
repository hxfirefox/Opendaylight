YANG Schema and Model
=====================

#Schema 定义

Binding Indpendent Data Schema 描述数据结构、方法及通告消息，Schema基于YANG，不过其中的一些术语和定义更加适应Java系统并更能更好地支持控制器使用场景，Schema用于定义:

- Modules – 即一组对外暴露的功能、特性、类型、RPC和数据结构
- Features – 即一种通过schema中被标识为条件的部分的机制
- Types – 可供本模块或引用此模块的其他模块使用的数据类型
- Data Structures – 定义树中的数据节点
- RPCs – 即模块操作
- Notifications - 通告
- 数据关系
- 有效性约束
- 其他模块或功能拓展(augmentation)
- YANG拓展

#针对ODL的YANG修订及拓展

**处理多个模块版本**

引入模块版本来独立数据，RPCs及notifications，此隔离层使得客户端可以在同一时刻使用多个不同版本的YANG，从而避免了数据冲突,基于此目的，针对可能出现多个版本并存，架构必须支持翻译不同版本数据。

**QNames**

常规XML QName包含本地元素名及XML域名，YANG Schema中加入了模块版本，在YANG中，QName是定义节点，类型，方法或通告的全名，包含XML命名空间，版本和本地命名，如下:

```
QName = (XMLNamespace,Revision,LocalName)
```

其中:

- XMLNamespace – 命名空间
- Revision – 版本号
- LocalName – YANG中定义节点名

例如：

```
module sample-yang{
	yang-version 1;
	namespace "urn:opendaylight:yang:sample-yang";
    prefix "sy";
    
    import ietf-inet-types {prefix "inet"; revision-date "2010-09-24";}
    import ietf-yang-types {prefix "yang"; revision-date "2010-09-24";}
    import opendaylight-inventory {prefix "inv"; revision-date "2013-08-19";}
    import opendaylight-l2-types {prefix "l2-types"; revision-date "2013-08-27";}

    organization "ODL";

    contact "WILL-BE-DEFINED-LATER";

    revision 2015-04-01 {
        description "yang sample definitions";
    }
    
    typedef sample-id {
        type uint32;
    }
    
    typedef sample-name {
        type string;
    }

    grouping sample-group {
	    leaf id {
		    type sample-id;
	    }
		
		leaf name {
			type sample-name
		}
    }
}
```

则当编译产生相应的Java代码时，如下：

```
public static final QName QNAME = org.opendaylight.yangtools.yang.common.QName.cachedReference(org.opendaylight.yangtools.yang.common.QName.create("urn:opendaylight:yang:sample-yang","2015-04-01","sample-group"));
```

**RPCs**

在常规Netconf/YANG场景中，RPC用于功能或API的建模，在Controller SAL中，RPC用于对Provider提供的功能建模
