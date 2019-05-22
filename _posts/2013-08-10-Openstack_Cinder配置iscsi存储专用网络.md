---
layout: post
category : OpenStack
tagline: "regexisart"
tags : [OpenStack, cinder, nova, iscsi, volume]
---
{% include JB/setup %}

**如需转载，请标明原文出处以及作者**

*陈锐 RuiChen @kiwik*

*2013/08/10 20:23:29*

----------

为了提高网络的可靠性和安全方面的要求，我们也许想将openstack的控制面网络和存储面网络分离，也就是配置*iscsi*的专用网络。

以kvm+lvm+iscsi的ubuntu环境为例，考虑一个简单的部署模型，一个计算节点(nova-compute)，一个存储节点(cinder-volume)，一个控制节点(其他所有进程)。

我们需要按照以下步骤配置：

1. 计算节点和存储节点都需要两块网卡eth0和eth1，控制节点需要一块网卡eth0。网络规划如下，其中`192.*`为控制面网络，`10.*`为存储面网络

    计算节点eth0: 192.168.1.3
    
    计算节点eth1: 10.144.144.11
    
    存储节点eth0: 192.168.1.4
    
    存储节点eth1: 10.144.144.12
    
    控制节点eth0: 192.168.1.5

2. 修改cinder的配置文件`cinder.conf`，将iscsi_ip_address改为10.144.144.12，默认配置为$my_ip，也就是和控制面同一个ip，然后重启cinder-volume

3. 修改存储节点上的iscsi target tgtd进程的启动方式，使其只监听存储网络，修改`/etc/init/tgt.conf`，将`exec tgtd`改为`exec tgtd --iscsi portal=10.144.144.12:3260`然后重启tgtd进程，这样我们通过`netstat -naop | grep 3260`，就能看到tgtd只监听了10.144.144.12的3260端口


以上3步就配置好了，openstack cinder和nova的源代码根据配置就能自动适配，nova在给虚拟机挂卷的时候，调用cinder的`initialize_connection`方法，将需要的iscsi信息取回来，其中就包括了我们在`cinder.conf`中配置的*iscsi_ip_address=10.144.144.12*和iscsi的端口，然后nova执行`iscsiadm`的login等命令，都是根据这个ip构造参数，以后虚拟机和存储卷之间的网络通信都是通过存储面网络进行。
