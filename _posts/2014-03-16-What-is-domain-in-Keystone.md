---
layout: post
category : OpenStack
tagline: "regexisart"
tags : [OpenStack, keystone, domain, policy]
---
{% include JB/setup %}

**如需转载，请标明原文出处以及作者**

*陈锐 RuiChen @kiwik*

*2014/3/16 17:51:36 *

----------

Keystone的domain概念刚出现的时候，看了一下当时的blue print，初衷也就是构造一个namespace，解决一套OpenStack系统中的用户的同名问题，使用domain_id和name作为联合唯一键，使不同的domain下的用户可以使用相同的名称。

随着OpenStack的发展，domain又被赋予了新的职责，而不仅仅作为一个namespace使用，让我们来看看它到底还能做些什么？

##相关对象

Keystone中当前和domain相关的对象有user，project，group和token，其中user，project和group从sql driver看，这三个模型的name字段都和domain_id一起作为联合唯一键在数据库层面增加了约束。

token比较特殊，获取V3 token的时候有一个可选参数`scope`，来指定需要获取什么样scope的token，可以参考[keystone V3 API](http://api.openstack.org/api-ref-identity.html)。

获取token的时候，scope分为三种：

- domain
- project
- trust

其实最终keystone内部处理的时候，也会把trust转化为project。又因为获取token的时候，scope是可选参数，可以不填，所以还可以获取一种特殊的不关联scope的token，这种token的唯一的目的也就是确认一个唯一的用户。

token实际的scope最终转化为两种domain和project。

##授权

keystone的授权（`create_grant`）过程，其实是一个三方关联的过程，三方就是：actor（被授权的主体），role（权限），target（被授权的范围）。实际授权过程可以这样理解：

> 为actor在target的范围内赋予role的权利。

- actor -> user & group
- targe -> project & domain
- role

再说回来，domain scope token包含的roles，就是这个domain范围内的所有这个user相关的role，其中包括user本身和user所属的group的role的一个并集。而project scope token包含的roles，同上，就是范围变成了project相关的role。

##怎么用domain

keystone默认的**policy.json**，其实是拥有admin角色的用户可以做任何的事情，可以在domainA中创建project，也可以在domainB中创建project，这样就无法做到，限制domain内的admin只可以在所属domain内创建project，也就是**domain admin**的概念，为了弥补这一问题，keystone引入了`policy.v3cloudsample.json`。

我们来分析一下是怎么做到的，来看其中的一段例子，摘自policy.v3cloudsample.json

{% highlight bash linenos %}

"admin_required": "role:admin",

"identity:create_project": "rule:admin_required and domain_id:%(project.domain_id)s",

"identity:get_project": "rule:admin_required and domain_id:%(target.project.domain_id)s",

"identity:list_projects": "rule:admin_required and domain_id:%(domain_id)s",

{% endhighlight %}

先来看create\_project，首先要求admin角色，需要注意的是and的后半句**domain\_id:%(project.domain\_id)s**，这条规则的意思就是create\_project时，使用的token的domain\_id必须等于project所在的domain的domain\_id。

也就是如下场景：

1. 为userA在domainA的范围内赋予admin的权限
2. userA指定domainA作为scope，申请一个domainA scope的tokenA
3. userA使用tokenA，去创建project，创建project时domain_id参数必须为domainA的id
4. 创建project成功

这里有几个关键点需要注意，首先userA在domainA内必须要有admin权限，这样才能满足**rule:admin\_required**，其次，token需要是一个domain scope的token，这样token中才会有**domain\_id** ，再次，创建project的时候，domain\_id参数必须等于domainA的id，这样才能满足**domain\_id:%(project.domain\_id)s**。

*这样一条规则的意义在于，可以限制只有在domainA内有权限的用户才能在domainA创建project。*

get\_project比较好理解，就是查询的project的domain\_id必须和token的domain\_id相同，也就是只能查询token所在范围内的project。

list\_project是查询出所有的project，但是会根据token的domain_id过滤，然后剩下所有和token的domain\_id相同的project。

keystone增加了domain这样一个概念之后，其实也就把keystone本身的资源user、project、group按照domain给做了一个划分，可以做到在domain范围内的对于user、project和group的管理。

policy.v3cloudsample.json中又增加了几种的权限规则，例如：cloud\_admin、domain\_admin、project\_domain。大家可以自己结合policy.v3cloudsample.json来看一下它们各自的作用。

{% highlight bash linenos %}

"admin_required": "role:admin",

"cloud_admin": "rule:admin_required and domain_id:admin_domain_id",

"service_role": "role:service",

"service_or_admin": "rule:admin_required or rule:service_role",

"owner" : "user_id:%(user_id)s or user_id:%(target.token.user_id)s",

"admin_or_owner": "(rule:admin_required and domain_id:%(target.token.user.domain.id)s) or rule:owner",

"admin_or_cloud_admin": "rule:admin_required or rule:cloud_admin",

"user_domain_id": "domain_id:%(target.user.domain_id)s or domain_id:%(user.domain_id)s",

"project_domain_id": "domain_id:%(target.project.domain_id)s or domain_id:%(project.domain_id)s",

"groups_domain_id": "domain_id:%(group.domain_id)s or domain_id:%(target.group.domain_id)s",

"same_domain_id": "domain_id:%(domain_id)s or domain_id:%(target.domain.id)s",

"match_domain_id": "rule:same_domain_id or rule:user_domain_id or rule:project_domain_id or rule:groups_domain_id",

"domain_admin": "rule:admin_required and rule:match_domain_id",

"project_admin": "rule:admin_required and project_id:%(target.project.id)s",

{% endhighlight %}

具体policy规则的配置可以参考keystone的[官方文档](http://docs.openstack.org/developer/keystone/configuration.html#keystone-api-protection-with-role-based-access-control-rbac)

