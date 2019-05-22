---
layout: post
category : OpenStack
tagline: "regexisart"
tags : [nova, host, node, service]
---
{% include JB/setup %}

**如需转载，请标明原文出处以及作者**

*陈锐 RuiChen @kiwik*

*2015/2/4 22:44:22*

----------

*写在最前面：*

*这段时间连续改了几个scheduler和resource\_tracker相关的bug，仔细的读了一下相关的代码，一点感受分享给大家。*

**代码基线：Kilo**

## 混乱中求秩序 ##

compute\_node在nova中是一个很神奇的东西，在不同的上下文中有不同的名字，本身还和Service存在复杂的关系。

数据库中有两张表services和compute\_nodes，分别代表了以下两个概念：

1. services: 一个nova的基本进程单元，例如：nova-compute, nova-conductor, nova-scheduler等等。大规模部署的时候，可能在不同主机(hostname)存在多个相同名称(binary)的进程存在，为了标识不同的进程，使用了(CONF.host, binary)作为唯一键。

2. compute\_nodes：从nova-scheduler的角度看，是最小的调度(schedule)单元，从nova-compute的角度看，是计算资源的拥有者。

## Host API ##

先从API说起，有一组host相关的API，host-list, host-update等等。

- host-list：听名字以为是列出了所有的compute\_node，其实是列出了所有部署了nova进程的host，在这里host代表了`service`，功能和service-list一致，包括数据都是从Services表中获取的，可以完全用service-list取代。

- host-describe：相对来说比较独特的host API了，显示了计算节点上资源的使用情况，包括：总量(total)，当前使用量(used\_now)和每个租户的使用量(project)。在这里host又代表的是`compute_node`。

- host-update：包括两个功能，enable/disable host和设置host维护状态。可以认为是Xen driver的特有API，因为其他的driver都没有实现。enable/disable host功能和enable/disable service功能一致。需要提到的是Xen driver的设置host维护状态，触发维护态之后，Xen driver会将这个host上的所有instance，通过XenAPI迁移到相同资源池(aggregate)的其他host上。

- host-action：设置host的power状态，可以输入的参数有startup/reboot/shutdown，KVM/VMware一个都没有实现，Xen/HyperV实现了reboot和shutdown，没有实现startup，嘿嘿，原因你懂的。

## 对象模型 ##

再来看一下nova内部的对象模型。

### nova-compute ###

- ResourceTracker：每一个ResourceTracker对象是DB中一个compute_node的内存映射，保存了计算资源的使用情况，包括：cpu，ram，local\_gb，pci，numa\_topology等。在定时任务中和DB数据同步。**在nova-compute中compute\_node名叫ResourceTracker。**

- ComputeManager：通过self.\_resource\_tracker\_dict缓存了所有被当前nova-compute进程管理的ResourceTracker对象，Libvirt、Xen、HyperV这类driver每一个nova-compute管理一个compute\_node，VMmare和Ironic可以一个nova-compute管理多个compute\_node，VMware driver下一个compute\_node对应一个VMware的集群，Ironic driver下表示一个物理服务器。

> 所以我们经常说到的“主机(host)有多少剩余资源？”，其实更准确的描述应该是“compute\_node有多少剩余资源？”。因为在VMware的场景下，一个compute\_node是包括多台host的cluster。

### nova-scheduler ###

- HostState：compute\_node在nova-scheduler进程中的表示形式，从DB的compute\_nodes中构造，表示了compute\_node的资源使用情况，以(hostname, hostname)为key缓存在HostManager的host\_state\_map中，并在合适的时候从DB中同步。**在nova-scheduler中compute\_node名叫HostState。**

- HostManager：在host\_state\_map保存所有的HostState。

> 同理，“nova-scheduler没有为instance选到合适的主机(host)”，应该是“nova-scheduler没有为instance选择到合适的compute\_node”。

说到这里，自己都觉得自己像是孔乙己，在给大家讲“茴字的四种写法”，呵呵。但是仔细想想，用host指代compute\_node，在KVM driver下可能是适用的，但是在VMware driver下呢？如果一定要给nova中的host和node做一个明确的区分的话，host可以认为是nova-compute进程运行的主机，而node是instance运行的主机。

## 和Service的关系 ##

数据库中的compute\_nodes表存在一个service\_id列指向对应的nova-compute的services表记录。Kilo版本的BP [detach-service-from-computenode](http://specs.openstack.org/openstack/nova-specs/specs/kilo/approved/detach-service-from-computenode.html "http://specs.openstack.org/openstack/nova-specs/specs/kilo/approved/detach-service-from-computenode.html") 解除了两者之间的外键关联，更加明确了service和compute\_node的定位，解耦之后两者通过CONF.host关联。

数据库services表中的记录是在进程启动的时候插入的，并以(CONF.host, binary)为唯一键，而compute\_nodes表是在ResourceTracker.update\_available\_resource时插入的，以(CONF.host, nodename)为唯一键。