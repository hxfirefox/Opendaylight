TTP源码分析
===========

版本来源: [Github上的opendaylight TTP项目](https://github.com/opendaylight/ttp)

参考资料来源: [Table Type Patterns](https://wiki.opendaylight.org/view/Table_Type_Patterns)

#概述

TTP是ONF转发抽象工作组(FAWG)的第一个成果输出，其目的是使OF控制器与交换机在一些功能上达成一致，方便管理不断增长的OF设备。当前TTP的关注点是OF，这只是NDM抽象中的一部分。

#源码分析

项目代码结构如图所示:

![project code structure](https://github.com/hxfirefox/Opendaylight/blob/master/Lithium/resource/ttp_code_struct.jpg)

该项目主要有两个子项目组成，分别是ttp-model和parser

##ttp-model

定义了一种TTP的yang模型，此TTP与ONF标准中定义的TTP基本一致，运用该模型可为交换机关联"acitve TTPs"和"supported TTPs"信息。

##parser

解析器提供处理TTP的命令行工具。
