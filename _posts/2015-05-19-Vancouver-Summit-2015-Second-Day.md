---
layout: post
category : OpenStack
tagline: "regexisart"
tags : [Vancouver, summit, nova]
---
{% include JB/setup %}

**如需转载，请标明原文出处以及作者**

*陈锐 RuiChen @kiwik*

*2015/05/19 23:35:50*

----------

*写在最前面：*

*今天开始写一个系列，记录我在OpenStack Vancouver Summit上的所见所闻所感，以下为个人立场，不代表官方观点。个人能力有限，如有不正确之处，请随时指出，谢谢*

## Topics ##

### Taking Risks: How Experimentation Leads to Breakthroughs ###

今天开场的是Mark，给我们的感觉依然是清楚的口语发音和幽默淡定的台风，看来这几点是OpenStack基金会的必要条件 :)

演讲中提到了一个很有意思的新概念：“OpenStack is an integration engine.” 

希望通过OpenStack整合云计算相关的一切，不管是容器，虚拟化，还是Bare Metal。

整个演讲给我的感觉就是：“容器，容器，容器！”

Mark提到每一个开源项目都会经过试验阶段，然后到达成功，Nova和虚拟化已经比较成熟，到达成功阶段，而现在以Docker为代表的容器技术，正处于试验阶段，需要我们不断的尝试，突破和冒险。

Rackspace的Adrian Otto作为Magnum的PTL，讲到了容器技术和OpenStack技术的融合，Magnum社区的现状，并现场做demo演示Magnum的功能。并与Google的Sandeep Parikh一起演示了一个demo，将应用同时部署在Racespace和Goolge的容器集群上，并通过LB在两个集群做负载均衡，然后挂掉一个应用集群，验证应用是否受影响。

Mark还强调了在East Building的Container day。

Miratis的Craig Peters介绍了Murano作为App Catalog的功能, 并现场演示了demo，请记住http://apps.openstack.org/ 

从这个topic可以看出OpenStack社区对于新兴技术项目的支持，希望通过新项目不断的丰富自己的生态环境，扩展社区的范围，并和上下层做到很好的整合。

### OpenStack Update from eBay and PayPal ###

eBay是OpenStack的深度用户，Subbu Allamaraju接下来做了演讲介绍eBay和PayPal当前使用OpenStack的情况，这里有一组2015年的数据，我觉得非常给力：

-  Havana
-  \> 12k Hypervisors
-  \> 300k cores
-  10+ availability zones
-  15+ virtual private cloud
-  1.6 pb block storage
-  100% KVM
-  100% OVS

当前PayPal的100%的web和中间件应用都已经迁到了OpenStack之上，eBay完成了20%，eBay和PayPal所有的开发和测试机都是用OpenStack。

我觉得每一个搞OpenStack的同学都应该低头想想，我们是不是没有把OpenStack用好？

同时，eBay也在会场高呼：“We need help！”

1. 增加core reviewer的数量，换句话说，就是提高bug merge的速度
2. 产品化和易操作性，也就是提高OpenStack成熟度
3. 标准化创新，也可以认为是丰富OpenStack生态环境

### Superuser Award ###

今年的超级用户奖给了Comcast，对比了候选人的数据，我发现这个奖不是给有最大规模的OpenStack用户，像是eBay和Warmart用的很好但都没有得奖，社区更偏向授予那些将使用中的问题反馈社区，推动OpenStack前进的公司，像Comcast，竟然向社区提交了36000行代码，让我很惊讶。

### VMware Integrated OpenStack: Turn your existing VMware Environment into Production-Grade Openstack in Under an Hour (Seriously!) ###

Vmware的这个topic感觉是上午最有料的一个，演示了通过VIO怎么快速的部署基于VMware技术的OpenStack环境，整个操作在vSphere web client上完成，通过导入一个VIO的包，经过简单的界面操作，将整个的OpenStack环境部署起来，操作不超过10步。

整个部署模型分为两个部分：管理集群和计算集群，所有的OpenStack组件(Nova, Cinder, Neutron等)都做为虚拟机，部署在管理集群，通过OpenStack创建的虚拟机都运行在计算集群。有几个我觉得很抢眼的功能在这里高亮一下：

1. VMware演示了给OpenStack升级打patch的过程，patch包是一个deb的包，选择patch之后一键打patch，看起来应该是把所有的OpenStack项目的patch统一到了一个文件里，一次全部打上去，并且支持revert，同时可以通过UI和cli查询已打patch列表。

2. 扩容计算和存储也很简单，丝毫看不到OpenStack的痕迹，整个操作都在vSphere的web client上完成，add nova cluster，然后添加datastore和glance使用的datastore，

3. 设置Host维护模式，设置之后，vMotion会自动将节点上的虚拟机迁空，用户不用感知，看起来应该直接支持live-migration.

4. vRealize，可以管理整个OpenStack部署，健康检查，监控告警，并可以通知OpenStack运维人员，或者是受影响的租户，vRealize还可以生用户每个月的云费用清单，并预测后续的使用情况，并可以与AWS的花费对比。

很高能吧，我当时坐在前排，仔细的看了下vRealize的演示chrome浏览器地址栏，赫然是 file:/// 开头，原来如此，看起来应该还处于内部试验阶段，并不成熟。

当有人问到这个OpenStack发行版，是否还支持其他的计算/网络/存储driver时，演讲者含糊其辞说了半天我们VMware和OpenStack在一起work的很好，是OpenStack社区认证的标准API，完全不起支持其他driver的事。在我看来，也没必要支持，VMware参加OpenStack社区就不是为了支持其他的driver。

演讲结束之后，我私下的找了VMware的开发讨论了一个[bug](https://bugs.launchpad.net/nova/+bug/1420662 "https://bugs.launchpad.net/nova/+bug/1420662")，因为我觉得这个bug有点敏感，就没有直接在topic中提问。

> 背景是这样：
> 
> Nova的VMware driver由于一个nova-compute管理一个Cluster的VMware虚拟机，所以在Cluster规模很大(3000+ instances)的时候，会导致nova-compute进程初始化失败，并会影响虚拟机同步周期任务，原因是一次从数据库中fetch太多虚拟机记录，导致nova-compute和nova-conductor rpc timeout。

我提到RPC超时的时候，他一下子明白了我在说那个bug。我说这个问题，nova的好几个core都认为是VMware的实现和nova的架构设计有冲突导致，不应该是nova框架修改。我问，那么VMware方面有没有考虑如何修复这个问题，或者有什么改进方案，他笑笑说，虽然VMware的文档，提到了一个集群下可以管理很多虚拟机，但是那只是一个上限值，在实际应用过程中，并不建议在一个集群中塞那么多虚拟机，解决这个问题的最好的办法就是，不要让集群规模那么大，多搞几个集群将虚拟机分开。我追问，那么你建议的进群规模应该是多少呢？他有点为难，也许是说一个和VMware官方文档不一致的数字对他来说并不合适，就推辞说，以后他们可能会有一个文档来给出建议值，但是还需要试验，来确定这个值得大小。

### Backport or Upgrade? OpenStack Releases Unchained  ###

OpenStack半年一次的release节奏对于很多人来说都是一场赛跑，升级是唯一的选择么？也许将需要的bug-fix或者feature反合，是另一种更低成本的解决问题的方式。

先来看看升级的成本：

1. 升级计划和方案
2. 移植定制化特性
3. 测试
4. 配置脚本和部署
5. 计划中断时间

演讲者的观点是，选择升级还是反合应该以满足基本需要为依据，而且需要持续的关注OpenStack社区是否有现成的patch可以解决遇到的问题。

这里有一个流程图可以帮助选择升级还是反合，因为画图时间太久，我简单描述一下他的判断依据：

1. 是否马上需要这个feature或者fix？
    - yes，转2
    - no， 等待后续版本升级
2. 社区有patch在解决这个问题么？
    - no，转3
    - yes，转4
3. 这部分代码是否模块化，可以通过hook机制解决么？
    - yes，私有patch，临时通过hook解决
    - no，私有patch，在版本升级时移植
4. 这个patch已经合入社区主干了么？
    - yes，转5
    - no，私有patch，在升级时移植
5. API版本是否兼容?
    - yes，升级
    - no，反合社区patch

这里涉及到了私有patch，所以演讲者提到了内部CI/CD环境的重要性，需要通过内部CI/CD环境不断的验证私有补丁，并持续内部测试，并在将来用社区patch代替私有patch。

将每个OpenStack组件(Nova, Cinder, Neutron)分别运行在不同的节点，有助于简化升级过程，从演讲者的话中，听出他们的解决方案，是把不同的组件运行在不同的虚拟机中，突然想到也许Kolla项目可以帮助演化出更加简单的升级方案，容器镜像更新，重新部署？我不太懂容器，一个突然的想法。

### Using Rally for OpenStack certification at Scale! ###

开始介绍Rally的时候，Boris Pavlovic问了一下多少人知道Rally，用过Rally，整个房间很多人，但是举手的非常的少，他明显有点失望，后来讲到Rally也支持通过Murano安装，Boris顺便问了一下谁了解Murano，几乎没人举手，Boris很释怀的笑了，Rally要比Murano知名度更高一些，呵呵，从这个小细节也能看出这两个项目在OpenStack社区的现状。

Boris介绍了Rally目标，架构和运行方式，并在现场演示了一下Rally运行的demo，真实环境运行的demo很流畅，说明当前Rally已经比较成熟。

Rally当前在BlueBox的应用:

1. 新云环境的部署
    - 验证新环境的SLA
    - 整个系统的功能验证，包括：不属于OpenStack的部分的验证
2. 新的部署工具的release
    - 部署工具的原有功能的回归测试
    - 部署工具新feature的验证
3. 新feature和配置的验证
    - 验证OpenStack新特性和配置是否生效
    - 新特性是否影响SLA

Rally对于OpenStack其实可以做到以下几个层次：

Everything works -> expected perf -> under expected load(上限) -> under expected scale(上限) -> high availability

Rally的介绍很多人问问题，看起来大家都对它很感兴趣，并且希望Rally能使用在自己的产品或者环境中。

后续Rally还会做HA相关的测试，例如：关闭某些服务然后看系统的反应，具体的方式也就是在测试中，使用ssh或其他方式登陆到目标主机，然后停掉某些服务，再验证环境是否ok。

### Hold my beer and watch this! Upgrading from Havana to Juno in one fell swoop with live customers. ###

另一个关于Upgrade的topic，BlueBox的Jesse Keating有深入和精彩的讲解。

每一个模块的升级模式：

1. 应用新代码和新配置
2. 停止旧版本服务
3. 升级数据库schema
4. 启动新版本服务

BlueBox的升级顺序：

1. glance
2. cinder
3. nova
4. neutron
5. swift
6. keystone
7. horizon

有人问到类似Racespace的OpenStack公有云部署场景下，有很多的nova-compute节点的情况又应该怎么升级？Jesse的回答和我心里的考虑是一致的，首先，升级nova中除了compute的所有服务，其实主要是需要升级nova db schema和nova-conductor，然后，nova-compute再一个一个升级，在我看来其实应该将nova-api分离出来，等所有nova-compute升级完成，再升级nova-api，避免新接口走入老代码。当然最好配合update_level配置。详见我的blog [“Nova如何支持live Upgrade”](http://kiwik.github.io/openstack/2015/04/04/Nova%E5%A6%82%E4%BD%95%E6%94%AF%E6%8C%81live-upgrade/ "http://kiwik.github.io/openstack/2015/04/04/Nova%E5%A6%82%E4%BD%95%E6%94%AF%E6%8C%81live-upgrade/")

----------

以下议题强烈感谢 huangtianhua (Heat core)提供：

### The Good and the Bad of OpenStack REST APIs ###

首先给出REST的定义：REST是一种基于Http协议实现资源操作的一种软件风格。然后分析了OpenStack REST APIs目前的优缺点。

优点：

1. 模块化：服务类型分类
2. 服务目录：每种服务有对应的endpoints的定义
3. 认证流程：携带用户名，密码，租户到keystone认证获取token和对应的服务目录， 然后携带token访问对应服务的endpoints

缺点：

1. 版本混乱：用户不知道该用哪个版本，v1?v2?v2.1?v3? 提出API的设计应该具备高度可扩展性，仅应在api有重大变化时引入新的版本号。
2. 绕过服务目录：不在服务目录里面的服务不应该使用，很多模块，比如heat，会绕过服务目录通过配置文件或者其他方式构建该服务的endpoint。
3. 客户端-服务端耦合：
    - 某些特性客户端和服务端相互依赖
    - API大多是通过客户端间接测试
4. 客户端和服务端缺乏中间件，客户端直接对服务端URL硬编码
5. 包含RPC的风格：滥用‘action’的概念，很多模块在url里面直接携带actions，比如/v2/image/{image_id}/actions/deactive，各个模块可以大排查，将action从url去除，因为REST是基于资源操作的，actions可以通过body携带。
6. 模块间不一致：
    - 分页查询的接口：
        - nova，glance，cinder等使用‘limit’和‘marker’， 而另外一些模块根本就没有分页查询或者使用其他比如‘page’和‘end_marker’
        - 有的模块会有count的统计，大部分模块没有这个功能
        - 在过滤时有全匹配和通配的不同支持
    - http响应的返回码

目前API Working Group正在分析，后续会交付OpenStack REST APIs的guideline，希望有兴趣的同学可以参加讨论。

## See you later ##

今天blog完成的晚了，已经凌晨2点。呵呵，关键是晚上的活动太给力 `HP & Scality’s Supernatural Evening`, 一个在旧火车站举行的OpenStack的大party，BBQ，啤酒，特色小吃，魔术表演，让我现在都没有困意。说实话比会场提供的午餐好吃太多。