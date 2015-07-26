TTP源码分析
===========

版本来源: [Github上的opendaylight TTP项目](https://github.com/opendaylight/ttp)

参考资料来源: [Table Type Patterns](https://wiki.opendaylight.org/view/Table_Type_Patterns)

#概述

TTP是ONF转发抽象工作组(FAWG)的第一个成果输出，其目的是使OF控制器与交换机在一些功能上达成一致，方便管理不断增长的OF设备。当前TTP的关注点是OF，这只是NDM抽象中的一部分。

#规划

In the long run, we’d like for a TTP (or other NDM) to be negotiated as part of connecting to the switch or out of band, e.g., via OFCONFIG or OVSDB. The TTP would then either be self-describing or provide a link (either explicitly via a URI or implicitly by looking up the name in a naming-authority database) to a description of what features the TTP provides. For example something saying it had 3 tables: one flexible OF1.0-style table, one L2 forwarding table and one L3 forwarding table.

Until vendors start to support this, the most useful thing seems to be supporting something we’d like to call “empirical TTPs”. The general idea is that as OpenFlow 1.3 hardware starts to be available, we will need a way to understand what capabilities it supports and what capabilities it doesn’t. There are just too many variables and optional features to be able to discover this at runtime and do anything with the information.

To do this in a sane manner, we will need to maintain a (static, offline) database of devices and their features. For example, knowing that Huawei 8500 series support only a single table with OF1.0-style features. (I’m making the vendor and model number up, not basing it on any real system.) I also want to note the the description of supported features need not be complete or static over time. We can start off by describing some of the features we’ve found that work and then move on from there adding more “known good” features as we discover/need them.

Fortunately, TTPs provide a relatively straightforward way to do this. They allow for an unambiguous description of the features a switch supports. If we have tooling in OpenDaylight to read and parse these TTPs we can have semi-progammatic support for OpenFlow 1.3 hardware in a way that makes it actually usable. First we start to store the features of switches in the offline database and then we fake the negotiation by reading the vendor ID and whatever else we need to identify the switch (Do we have to fall back to just Vendor + DPID or is there other info we can use?) and then just inferring the right TTP.

Along these lines, the OpenDaylight NDM project/effort would like to start by working on basic tooling to read, parse and write TTPs as well as maintain a database of the TTPs keyed off of vendors and device models (with hopefully enough information to suss that out during OF1.3 switch connection). Once we have this, hopefully we can start to build up some information around current OpenFlow 1.3 devices and make those much more useful.

Also, as we do this, we can start to make tools that let us get real things done with OpenFlow 1.3 hardware in a non-device specific way. For example, writing test suites that profile OF1.3 switches and produce a TTP by testing their features directly. Also, we’d like to be able to do things like discover isomorphisms between TTPs and/or find the commonality between multiple TTPs.

Really, this is all just in the spirit of enabling developers to take as much advantage of OpenFlow 1.3 hardware as we can in the least device-specific manner we can as fast as we can.
Hardware Targets

    Broadcom OF-DPA: Provides true OpenFlow 1.3 multiple table support with binaries and source code to work on most recent Broadcom-based switches assuming firmware access.
    Intel Flexpipe: ???
    IBM Switches: Have support for L2 forwarding in OF1.0 and adds MPLS support in OF1.3.

Software Targets

Glue code that shouldn't have to do much to put software switches into the appropriate mode. Mostly, this involves finding currently unused table numbers and allocating them to supporting a profile. Also preventing existing apps from violating the notion of how those tables should work established with NDMs. 

#源码分析

项目代码结构如图所示:

![project code structure](https://github.com/hxfirefox/Opendaylight/blob/master/Lithium/resource/ttp_code_struct.jpg)

该项目主要有两个子项目组成，分别是ttp-model和parser

##ttp-model

定义了一种TTP的yang模型，此TTP与ONF标准中定义的TTP基本一致，运用该模型可为交换机关联"acitve TTPs"和"supported TTPs"信息。

##parser

解析器提供处理TTP的命令行工具。
