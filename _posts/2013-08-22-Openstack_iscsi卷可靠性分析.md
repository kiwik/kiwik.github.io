---
layout: post
category : OpenStack
tagline: "regexisart"
tags : [OpenStack, cinder, nova, iscsi, 存储, 可靠性]
---
{% include JB/setup %}

**如需转载，请标明原文出处以及作者**

*陈锐 RuiChen @kiwik*

*2013/08/22 20:30:29*

----------

*写在最前面：*

*以OpenStack G版 2013.1的nova和cinder代码验证，cinder配置 LVM + iscsi*

这两天抽时间验证了一下Openstack环境下，通过iscsi暴露的卷的可靠性问题

部署模型为：1计算节点(nova-compute)，1存储节点(cinder-volume)，1控制节点(其他进程)

如下：

----------

|              | iscsi initiator | iscsi target | 虚拟机   | 恢复手段                                              |
|--------------|----------------------|-------------------|---------------|-------------------------------------------------------|
| 重启iscsid   | 中断                 | 正常              | iscsi卷不可用 | active虚拟机：reboot shutdown虚拟机：卸卷，挂卷，启动 |
| 重启计算节点 | 中断                 | 正常              | iscsi卷不可用 | active虚拟机：reboot shutdown虚拟机：卸卷，挂卷，启动 |
| 重启存储节点 | 正常                 | 中断              | iscsi卷不可用 | 启动存储节点，恢复网络连接                            |

----------

总结：

- iscsi initiator端重启之后，计算节点上的iscsi块设备映射会消失，会导致已挂载iscsi卷的shutdown的虚拟机不能启动，社区官方文档中(以下连接volume部分)，给的解决方案是卸卷，挂卷，再启动虚拟机；

- iscsi target端重启，等同于网络断链，target端重新启动之后，iscs initiator端会自动重新建立tcp连接，恢复iscsi卷。

对于重启计算节点的情况，还有另一个解决办法，将**nova.conf**的`resume_guests_state_on_host_boot=true`，当计算节点重启之后，会自动拉起节点上的本来是Active的虚拟机，并建立iscsi连接，但是shutdown的虚拟机还需要手动恢复。

恢复方法参考openstack官方文档[here](http://docs.openstack.org/trunk/openstack-ops/content/maintenance.html)

