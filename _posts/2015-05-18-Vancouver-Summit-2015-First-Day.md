---
layout: post
category : OpenStack
tagline: "regexisart"
tags : [Vancouver, summit, nova]
---
{% include JB/setup %}

**如需转载，请标明原文出处以及作者**

*陈锐 RuiChen @kiwik*

*2015/05/19 00:35:50*

----------

*写在最前面：*

*今天开始写一个系列，记录我在OpenStack Vancouver Summit上的所见所闻所感，以下为个人立场，不代表官方观点。个人能力有限，如有不正确之处，请随时指出，谢谢*

## Topics ##

### Cloud Unlocked: Connecting the Clouds to Create Huge Value for Users ###

今年仍然是Jonathan Bryce开场，一开始介绍了一下会议的注意事项，会场布局，特色的活动等。今年有6000+人参加温哥华峰会，当Jonathan提议Kilo版本的贡献者起立，接受6000多人的鼓掌时，那感觉还是非常爽的。一转眼Kilo也已经是第11个版本了，照例回顾了一下Kilo版本的代码规模，贡献人数等，给我的一个最直接的感觉是，东方面孔非常的多，其中最多的就是中国人和日本人。我国参加峰会的公司里，Huawei，IBM，Intel都是组团前来，经常在找会场的路上就能碰到聊两句。

Jonathon演讲里还有一个数据也很有意思，公有云主要集中在美国东海岸，而亚洲主要是私有云市场，但都是基于OpenStack技术。这从另一方面也能体现东西方云计算的技术差距，还有两个地区客户对于云计算的认识和接受程度。

这两年基于OpenStack技术的TV和电影相关的产品也有很大的成长，OpenStack官方提供的数据是北美占38%，欧洲占14%，没有想到亚洲都占了64%。

Guillaume Aubuchon提到了他们在电视行业中，使用云计算技术遇到的问题：

1. 多个区域的云数据都在不断增长
2. 多个云部署环境不能互联
3. 物理数据迁移的过程中会有数据损坏和丢失

随后Keystone的PTL Morgan Fainberg上台介绍了一下Keystone新特性Federate Identity，通过Federate我们可以使多个区域的公有云，私有云互相连接，打破云和云之间的隔阂。Morgan特别强调了Keystone在混合云场景的重要性，还推荐了下午的议题“Keystone feduration Dive deep”。

Guillaume Aubuchon随后介绍了一下，OpenStack在影视制作中的应用，并在现场录制了一段高清诗视频，上传到云中，并通知洛杉矶的同事进行后期处理，十几分钟之后，通过手机播放了经过后期处理的视频短片。

### OpenStack – An Enterprise Force Awakens ###

在Kilo版本中全球有1500多名开发者为OpenStack贡献代码，历史统计总共有3500多名的开发者曾为OpenStack贡献代码，OpenStack有15000多的单元测试和集成测试用例。那么下一步OpenStack需要做什么？

当前OpenStack已经涵盖的范围包括：

- Scalable
- Secure
- Supportive

HP的高级副总裁Mark Interrante认为下一步还需要做的是：

- Unobtrusive (应该更加容易支持混合云应用场景)
- Simple
- Stable
- Collaborative (解决跨模块/跨领域的问题)

### 7 Habits of Highly Effective Contributors ###

Keynote结束之后本来想去听这个议题“What's Next in OpenStack: A Glimpse at the Roadmap”，无奈人太多没有挤进房间，换成了这个。

Rackspace架构师Adrian Otto阐述了高效的OpenStack开发者应该具有的习惯：

1. Use IRC，需要帮助，需要注意，需要review patch
2. attend meetings，正式，积极，讨论
3. Use mailing list，阐述想法，提问，并回答问题，通过topic tag过滤自己感兴趣的邮件
4. Do code reviews, 坚持每天review，高质量review，阐述想法，建设性的建议，推进patch合入
5. use bug tickets，subscribe，claim/work，Open before patch，加tag区分
6. use blueprints，propose, discuss, claim/work，
7. contriblute, code, docs, testing, consistent, small patches(<400), link to bugs/BPs，将patch分成小块，便于review

*里面有一些词语没有想到很准确的翻译方法，害怕误导大家，偷懒贴了原文 :)*

Adrian有个观点我觉得很值得推荐，在此高亮：

**提交代码是阐述idea的最快的方式。**

### 午餐小插曲 ###

中午吃饭和一个日本朋友同桌，看起来像是OpenStack User，问了一下，原来他的公司是做IT从业人员资格认证工作的，日本对于从业人员有认证要求。

他很好奇，在中国，对于CloudStack和OpenStack的接受程度，哪种数据库产品比较流行(Mysql, Postgresql，Oracle)，HTML5是否流行。

我从他那里了解到，日本CloudStack占主流，主要是NEC/富士通/日立都在使用CloudStack，大约占了7成市场，当初选择CloudStack的原因是稳定性比OpenStack要好，我告诉他，就我所知NEC和日立分别在Nova和Cinder都有很多贡献，OpenStack的成熟度也在不断的提高，他也认为在日本市场未来两年OpenStack会超越CloudStack。

最后我说到中国日本韩国的开发都有timezone的问题，工作时间和OpenStack社区正相反，他也无奈的笑了。

### Introduction of a new Nova REST API: Why we need to use Nova v2.1 API ###

下午连着听了两个Nova相关的议题。

Nova的v2版本的API已经冻结，不允许修改，新增特性需要增加到v2.1 API中，在Liberty版本，默认endpoint将改成v2.1。

Nova的v2.1 API主要有三个方面的改变：

1. 兼容性，不使用v2.1 API的客户，还可以一直使用v2 API，不受影响
2. 参数验证，使用JSON schema框架验证API的输入参数，v2的参数是在代码逻辑中校验
3. micorversion, 后续对于API的修改都需要以microversion的方式导出，v2 API中的一些issue可以在v2.1API中通过microversion修复。

遗留问题：

1. v2和v2.1的行为差异，对于不能识别的输入参数，v2忽略，但是v2.1会抛出错误，这是一个不兼容的问题，tempest就有这样的问题，所以有可能客户应用也有同样的问题。Nova PTL建议如果不指定版本就不抛出异常，指定时再抛出异常。
2. 最后我问了一下，对于没有merge到社区的自定义API使用的microversion如果和社区后续使用的有冲突，怎么解决？ Ghanshyam Mann建议最好把API提交到upstream，不要保留私有的API实现，至少在当前没有计划使用placeholder，为厂商提供自定义API的microversion空间。API最好还是标准化，因为不同的厂商实现的2.4 API可能有所不同，这会给用户带来很多的困惑，例如：为什么Huawei实现的2.4 API和Intel的2.4 API不一样？

### Enhancing OpenStack Projects with Advanced SLA and Scheduling ###

听这个议题的时候，可谓是激动和悲伤各占一半，激动的是我在Kilo版本里实现的BP [IoOpsWeigher](https://blueprints.launchpad.net/nova/+spec/io-ops-weight "https://blueprints.launchpad.net/nova/+spec/io-ops-weight") 在演讲中被提及，悲伤的是对于multiple scheduler的想法，再次的被否定，原因是nova-scheduler之间不能共享数据，scheduler数目增多，会增加竞争资源的可能性。

这个议题介绍了FilerScheduler的机制，也介绍了resource tracking如何与scheduler配合，在这里不再详述。

还简单介绍了Kilo版本中scheduler部分的主要工作：

1. Service和ComputeNode分离，[BP](https://blueprints.launchpad.net/nova/+spec/detach-service-from-computenode "https://blueprints.launchpad.net/nova/+spec/detach-service-from-computenode")
2. 修改filters从HostManager中获取信息的方式，[BP](https://blueprints.launchpad.net/nova/+spec/scheduler-optimization "https://blueprints.launchpad.net/nova/+spec/scheduler-optimization")
3. 重构scheduler测试用例
4. 使用内部client区分scheduler和Nova的其他部分

Liberty中希望进行的scheduler优化如下：

1. 清理Nova和scheduler之间的接口
2. 持久化scheduler request信息，包括hints
3. 更具针对性的scheduler错误日志，操作人员希望通过日志，知道scheduler究竟为什么失败？
4. 解决availability zone相关的问题
5. 多scheduler之间共享状态
6. ratios值移至resource trace计算
7. 迁移前检查目标节点是否可以满足虚拟机要求，Sylvain在我之后提交了[BP](https://blueprints.launchpad.net/nova/+spec/check-destination-on-migrations "https://blueprints.launchpad.net/nova/+spec/check-destination-on-migrations")，但是两个[BP](https://blueprints.launchpad.net/nova/+spec/verifiable-force-hosts "https://blueprints.launchpad.net/nova/+spec/verifiable-force-hosts")是解决同一个问题，应该最终只有一个会被接纳。
8. scheduler从nova拆分，不确定？

## See you later ##

一天的会议之后，大家都很累了，还好有展厅活动调节心情，在展台之间挑选了满满一袋的纪念品之后，回到酒店写下了这篇blog，现在已是当地时间的晚上12:30了，期待明天的议题，No！是今天 :)

