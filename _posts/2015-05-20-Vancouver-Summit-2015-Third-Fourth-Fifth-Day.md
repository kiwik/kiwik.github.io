---
layout: post
category : OpenStack
tagline: "regexisart"
tags : [Vancouver, summit, nova, upgrade, api, heat]
---
{% include JB/setup %}

**如需转载，请标明原文出处以及作者**

*陈锐 RuiChen @kiwik*

*2015/05/20 22:35:50*

----------

*写在最前面：*

*今天开始写一个系列，记录我在OpenStack Vancouver Summit上的所见所闻所感，以下为个人立场，不代表官方观点。个人能力有限，如有不正确之处，请随时指出，谢谢*

> 两天白天听topic，晚上写blog，实在是有点疲劳，后三天的内容整理在这一个blog里，不好意思有所延期。

## Topics ##

### APIs Matter ###

在这里先补一篇昨天的topic，我觉得还是很重要的，有必要在这里提一下。

就像昨天blog的结尾提到的，OpenStack当前的REST API有一些问题有待解决，资源名称，URI结构，返回码，文档中有一些不合理的地方，project之间的实现也存在一些不一致，对于用户来说使用起来会很困难，就像我们之前就发现API文档远远落后与版本release，一些边缘/新增特性完全没有文档可以参考，所以我们就在每个版本发布的时候，测试并整理一篇内部的API文档，提供给产品化团队参考。

这个topic中提到有时候在开发API的时候，发现有多种API组织方式，最终的实现有时候是看自己的喜好，有时候是根据参考的API风格，没有一个标准的API开发指导，来定义什么是对的，什么是错的，没有一个团队来检查这些不一致的地方。

为了解决这个问题成立了 **API working group** ，这个跨project的working group的作用就是：

1. 维护一个[OpenStack REST APIs向导](https://github.com/openstack/api-wg "https://github.com/openstack/api-wg")

	- 结构化API资源
	- 标准化分页格式
	- 定义每种HTTP响应码的正确使用场景
	- 管理API的演进

2. 审核添加和修改REST API的patch和spec
3. 和每一个project team一起努力，对于API定义提出建议，调整有问题的API，并标准化到API向导文档。
4. 如果有不清晰的问题，作为一个case提交到API向导文档，指导后续的API开发。

会下和Alex Xu了解了一下API working group的现状，当前还处于刚开始的阶段，很多东西需要完善，当然最缺的是在每个模块review API变更的开发人员。API guidelines也需要根据不同使用场景在继续完善，是一个长期的工作，方向已经很明确了，需要一个标准的一致的OpenStack API，所以以后要是开发API相关的feature第一件事，就是要读一遍 **API guidelines**。

### Dive into VM Live Migration ###

作为Nova最为复杂的流程，不管是开发还是用户，对于Live Migration都会感到头疼。我自己总结了一个Nova Live-migration的流程图（Kilo），大家自己感受一下，[点这里](http://kiwik.github.io/openstack/2015/05/23/Nova-Live-Migration-Workflow/ "http://kiwik.github.io/openstack/2015/05/23/Nova-Live-Migration-Workflow/")。峰会上能有一个深入解析，听得人当然不会少。我准点过去，都只能坐在地上听完，但确实很值得。

从QEMU/KVM，libvirt层开始，讲解了影响虚拟机live-migration的几个参数：热迁移带宽，*TUNNELLED*，*DOWNTIME*，*COMPRESSED*，并图示了几个参数不同设置情况下的底层实现。

热迁移当前的两个问题：

1. 源节点的CPU flags必须是目的节点CPU flags的一个子集，也就是说，当Hosts的CPU型号不同时，热迁移只能从低到高，不能从高到低。（但我认为这是Hypervisor的限制，而不是Nova的限制。）
2. 当CONF.reserved\_host\_memory\_mb设置为负值时，会导致可用内存计算出的值比实际值大，可能导致热迁移失败。（Emm...有配置成负值的场景么？我怀疑）

倒是想到有一种场景可能发生问题，当CONF.ram\_allocation\_ratio配置为大于1的值，scheduler有可能选到一个实际可用内存小于虚拟机使用内存的节点，导致热迁移失败。考虑，CONF.ram\_allocation\_ratio=2，instance.ram=4，dest\_host.total\_ram=4，dest\_host.free=1。

`dest_host.total_ram * CONF.ram_allocation_ratio - (dest_host.total_ram - dest_host.free = 5)`

还讨论了热迁移安全方面的考虑，由于虚拟机的数据是需要在两台host之间通过网络传递的，而虚拟机可能包含敏感信息，所以有可能被网络嗅探，而一些情况下，有可能导致虚拟机数据在未加密的情况下通过网络传输。

可能的修复方案：

1. Hypervisor原生加密
    - QEMU不支持
2. libvirt隧道传输
    - 将libvirt\_migration\_url从qemu+tcp://改为qemu+ssh://
    - libvirt\_migration\_flag开启VIR\_MIGRATE\_TUNNELLED
    - Uses only one core （没搞明白，等高人解答）
3. 在hosts之间建立IPSec隧道

在QEMU/KVM层的热迁移优化：

1. 多线程内存页压缩
    - 压缩在热迁移过程中的所有页面
    - 使用zlib
    - 可配置线程数量和压缩比率
2. [Post-copy memory migration](http://en.wikipedia.org/wiki/Live_migration#Post-copy_memory_migration "http://en.wikipedia.org/wiki/Live_migration#Post-copy_memory_migration")
    - 优点：一种更高效的方式，可以在有限时间内完成热迁移。Pre-copy如果脏页产生速度超过传输速度，热迁移可能进行很长时间，甚至永远不能结束。
    - 缺点：如果热迁移失败，需要重启虚拟机。
    - 缺点：对于host有很大的性能冲击，network faults引起性能消耗。

当前已实现的OpenStack部分的热迁移优化：

1. 热迁移进度跟踪
2. 热迁移job的状态监控和失败处理

详见patch：[https://review.openstack.org/#/c/151664/](https://review.openstack.org/#/c/151664/ "https://review.openstack.org/#/c/151664/")

后续热迁移的优化：

1. 取消热迁移
2. 热迁移进度查询
3. 迁移带宽和max\_down\_time可以在迁移时指定

### Congress Hands on lab ###

今天最大的惊喜就是这个topic，形式我很喜欢，在对Congress做了一段基本功能介绍之后，提供一个Google Doc的地址，让所有人根据doc中指导步骤，连接到一套真实环境中（Horizon & SSH），动手尝试一下Congress的几个典型使用场景。相当于 **Quick Start**，如果有问题可以马上举手，Congress团队的人会有专业解答，，是不是很贴心 :)

所以想快速的体验一个新到的project，就参加 `Hands-on Labs`

如果想快速的宣传一个你的新project，那么也请准备 `Hands-on Labs`，不要给我看PPT，让我试试它能做什么！

Hands-on Labs是一个非常有说服力里的demo，实打实的使用体验，对于使用者能得到迅速的问题解答，对于project owner能得到第一手的用户反馈。

简单介绍一下Congress，它是一个开放的policy框架，可以用来描述期望的数据中心行为方式，用我讨厌的说法就是 **Policy as a Service** （一时之间什么都成了service，Ewwww...），很多人应该还是被概念绕的云里雾里（概念太多，所以云计算行业有很好的环境，培养大忽悠，呵呵）

先说一下什么是policy，举个列子：

> 所有虚拟机不能具有一个可以被访问的80端口。

admin也许定义这样的policy，保证所有对外提供web服务的虚拟机，使用HTTPS代替HTTP。从OpenStack的概念分解，就是虚拟机的port应用安全组，同时安全组中不能有一条80的入规则。

来看Congress的几个典型场景，自己体会一下：

1. 检查OpenStack环境中的冲突，与policy定义不一致的地方。虚拟机使用的安全组是否有一条80入规则。
2. 避免冲突发生。禁止向已有虚拟机的安全组中添加80入规则，或是在创建虚拟机时，禁止使用有80入规则的安全组。
3. 修复环境中的冲突。当虚拟机使用的安全组有80入规则时，删除这条规则。

当云的规模大了，运维的时间长了之后，不在期望中的问题一定会出现，这些事一定是需要一个系统保证，而不是admin人肉运维，Congress就是救星。

现在你应该理解了所谓的，“描述期望的数据中心行为方式。”，是什么意思了。

来看一下前文提到的policy的定义：

	error(user_id,email,vm_name,id) :-
	    connected_to_internet(id,device_id),
	    neutronv2:security_group_port_bindings (port_id=id, security_group_id=group_id),
	    neutronv2:security_group_rules(security_group_id=group_id, direction="ingress",
	                                   protocol="tcp", ethertype="IPv4",
	                                   port_range_min=port_min , port_range_max=port_max),
	    lteq(port_min,80),
	    gteq(port_max,80),
	    nova:servers(id=device_id, name = vm_name, user_id=user_id),
	    keystone:users(id=user_id, email=email)


[点这里](http://congress.readthedocs.org/en/latest/policy.html "http://congress.readthedocs.org/en/latest/policy.html")，看一下Congress policy的详细介绍。

峰会结束之后，我准备研究一下Congress，到时候补一篇blog，欢迎讨论。

### 10 Minutes OpenStack Upgrades? Done! ###

另一个比较好的升级方案。

*升级前的方案设计和操作原则：*

- 升级的好处和意义 （首要考虑，不使用新特性或是解决bug，没必要升级）
- 容易回滚 （升级失败，之后要快速恢复）
- 升级的最小业务中断时间 （要让用户知道有什么影响）
- 方案要能重用 （多套部署环境可以使用）

*基本步骤：*

- 代码和配置变更
- 升级数据库schema
- 功能测试和集成测试

这套方案的最大特点就是，首先部署一套镜像的控制节点，数据库和少量的测试计算节点，然后同步所有的**真实数据**到镜像数据库，升级镜像的控制和计算节点，验证镜像环境，可以是几天，也可以几个月的完整测试，通过浮动ip切换到镜像环境，断开源环境的控制面，同时打通镜像控制和源计算节点的网络，大规模升级原计算节点。

优势：

1. 有机会可以进行操作系统和数据库系统升级。（控制节点直接升级操作系统，计算节点较多，可以在整体的升级流程完成之后，逐个进行，通过热迁移迁空主机，不会导致整体业务中断，所以整体的业务中断时间应该能保证在10分钟完成。）
2. 对于虚拟部署和物理部署都可用。
3. 可以使用真实环境的数据在升级后的镜像环境验证。（这点我觉得非常重要，真实数据才能发现问题。）

劣势：

1. 物理部署镜像环境需要额外的服务器，虚拟机部署需要预留资源。
2. 镜像环境测试完成之后，升级时需要重新同步数据库数据，超大规模情况下，同步数据花费的时间不好预估，应该会超过10分钟。
3. 需要预留/迁空几台计算节点，作为下次升级的测试计算节点使用，最好划分为专用AZ。

### Nova Design Summit ###

温馨提示： Design Summit的schedule在Google Play的OpenStack APP里是没有的，需要安装一个叫 Liberty Design Summit 的APP，注意都是Sched出品的，而且因为在sched.org被分为了两个Event，所以两个APP之间不能同步数据。

*真搞不懂为什么要分成两个？*

分享一下这次峰会所有Design Summit的地址：

[https://wiki.openstack.org/wiki/Design_Summit/Liberty/Etherpads](https://wiki.openstack.org/wiki/Design_Summit/Liberty/Etherpads "https://wiki.openstack.org/wiki/Design_Summit/Liberty/Etherpads")

Design Summit都有etherpad记录会议的议程和结论，所以大家可以在上面的链接里查看所有的会议详情。

参加了几场Nova的Design Summit。

1. Nova: Scheduler in Liberty
    
    - [https://etherpad.openstack.org/p/YVR-nova-scheduler-in-liberty](https://etherpad.openstack.org/p/YVR-nova-scheduler-in-liberty "https://etherpad.openstack.org/p/YVR-nova-scheduler-in-liberty")

2. Nova: Resource Tracker, Clustered Hypervisors, and NFV
    
    - [https://etherpad.openstack.org/p/YVR-nova-resource-tracker](https://etherpad.openstack.org/p/YVR-nova-resource-tracker "https://etherpad.openstack.org/p/YVR-nova-resource-tracker")

3. Nova: Future of Nova API v2.0 and 3rd Party APIs
    
    - [https://etherpad.openstack.org/p/YVR-nova-api-2.0-3rd-party](https://etherpad.openstack.org/p/YVR-nova-api-2.0-3rd-party "https://etherpad.openstack.org/p/YVR-nova-api-2.0-3rd-party")

    - v2.1成为默认的endpoint，所以在v2.1上做全面的测试，发现和v2的不兼容，并修复，是欢迎的。应该向service catalog的使用上靠近，逐渐取消配置文件中的url硬编码。

4. Nova: Spec/Blueprint Unconference
    
    - [https://etherpad.openstack.org/p/YVR-nova-spec-blueprint-unconference](https://etherpad.openstack.org/p/YVR-nova-spec-blueprint-unconference "https://etherpad.openstack.org/p/YVR-nova-spec-blueprint-unconference")

5. Nova: Liberty Priorities - Part 1/2

    - [https://etherpad.openstack.org/p/YVR-nova-liberty-priorities](https://etherpad.openstack.org/p/YVR-nova-liberty-priorities "https://etherpad.openstack.org/p/YVR-nova-liberty-priorities")


----------

以下议题强烈感谢 huangtianhua (Heat core)提供：

### Heat Design Summit ###

1. Heat client usability

	- resource-type-list（查询系统支持的资源类型）：增加support_status支持情况，如什么时候开始支持的，什么时候要deprecation。
	- stack-list（查询stack信息）：用目录树给用户呈现stack及其资源的状态：

			MyStack CREATE_FAILED
			| - MyServer OS::Heat-foo CREATE_FAILED
		   		| - server OS::Nova::Server CREATE_COMPLETE
			   	| - volume OS::Cinder::Volume CREATE_FAILED
			| - Net OS::Neutron::Port CREATE_COMPLETE

	- deployment-list支持只返回部署失败的deployment信息
	- stack-show 查询返回stack的详细信息包括其下的resource的信息，一旦出现问题，用户便于查看

    回顾下Kilo版本设计会上的topic，了解的同学就会发现，Heat持续的关注可用性、易用性的，个人感觉Neutron客户端的易用性是有待提高的。

2. Heat template format improvements

	主要讨论了关于模板需要新增几个function内嵌功能：

	- split：HOT模板需要支持split功能，类同CFN模板的Fn::Split，解析某些string为list
	- suffix：给列表的每个元素都附加一个统一后缀的功能，举例：suffix(['a','b','c'], 'p') returns ['ap','bp','cp']
	- list\_flatten： 扁平化多个列表及其元素，举例list\_flatten: [[a, b, c], [d, e, [f]]] return[a, b, c, d, e, f]

3. Senlin Autoscaling Project - Deep Dive

    IBM的QimingTeng给大家介绍了一个stackforge的项目senlin。主要功能是集群化OpenStack的服务。介绍下背景：Heat目前有Autoscaling功能，来源于AWS，但是仅仅是一个很小的子集，Autoscaling应该是一个服务，而Heat的职责是协调服务而不应该提供新服务。所以Heat在很早以前就想把Autoscaling服务独立出来，senlin就这样孕育而生。用户依旧可以使用heat模板来定义Autoscaling，但是扩缩（包括创建删除）交由senlin来处理。
    
    整个介绍heat core还是很感兴趣的，对目前正在实现的扩缩的策略，heat core team建议不要再使用alarm来发signal，因为涉及认证等问题，希望不要沿用heat走过的老路，建议使用zaqar(zaqar要火起来了：）)。
    
    听众对于这个新项目也很感兴趣，整个过程很多人提问，无意间听到坐在旁边的两个老外在私语，大意是这个项目挺有意思，而且很容易冲core，原来老外也喜欢当core呀... 

    附上介绍胶片：http://www.slideshare.net/cliffton75/senlin-deep-dive-2015-0520

## See you later ##

整个后三天的Design Summit对于开发人员的英语沟通能力要求非常的高，很多技术问题，我们用国语讲的时候，还需要和听众不断的澄清，用英语来沟通，难度可想而知，所以各位小伙伴都要努力死磕英语了，呵呵。

坐在三楼阳台，看着水上飞机和海鸥飞来飞去，很享受，很惬意~~

下次东京峰会再见了。
