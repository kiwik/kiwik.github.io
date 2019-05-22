---
layout: post
category : OpenStack
tagline: "regexisart"
tags : [OpenStack, keystone, token, authentication, non-bearer tokens, Kerberos]
---
{% include JB/setup %}

**如需转载，请标明原文出处以及作者**

*陈锐 RuiChen @kiwik*

*2014/1/19 14:43:05*

----------

## 起因

曾经有一个同事问我，为什么一个用户可以使用另一个用户的token？当时没有细想，觉得token就像密码，如果知道了就能用，后来想想这里面确实是有一些安全的问题的，专门翻了一下Keystone Havana版本的BluePrint，里面还专门有个BluePrint是解决这个问题的，这就是[Token-Binding](https://blueprints.launchpad.net/keystone/+spec/authentication-tied-to-token)。

## Bearer tokens

当前的Keystone的token安全模型引申一个OAuth的概念，就是所谓的`bearer tokens`（携带token），意思就是知道token的人就能使用这个token的所有权限，就像生成token的人的权限一样。

token是在http消息头部明文传递的，攻击者很容易就能直接获取到，SSL可以保证token在传输过程中不被获取，但是这种解决方式，不能保证SSL管道两端的token不被获取。特别是很多log中也会将token打印出来。

Token-Binding被设计的目的就是绑定token的生成者和使用者，使UserA生成的token，不能被UserB使用。也就是`non-bearer tokens`。

## 工作流程

当前Keystone的token-binding只支持Kerberos一种方式，X509计划在Icehouse中支持。Kerberos我理解的不是很深，只懂基本流程，以及和Keystone的交互过程。Kerberos介绍可以查看[这里](http://en.wikipedia.org/wiki/Kerberos_%28protocol%29)。

1. 假设一个场景，我们使用HostA作为Client端，要向HostB上运行的Keystone获取token，使用user/password的方式获取。同时Kerberos的server端也运行在HostB上。

2. 首先要配置Keystone运行在Apache-httpd下，同时要配置Apache-httpd的Kerberos模块。在Keystone的`keystone.conf`中external认证需要enabled。

3. 在HostB上为Kerberos创建UserA，并设置UserA的密码PasswordA。同时为Keystone创建UserA，并设置密码PasswordB，Kerberos和Keytone中的用户名必须相同。

4. 首先要在Client端，通过Kerberos的`kinit`命令认证Client，`kinit UserA`输入之后，会要求输入UserA的Kerberos密码PasswordA。

5. 然后向Keystone发起auth请求，在POST消息体中，输入UserA/PasswordB，此时Keystone认证时，其实是同时使用了password和external两种认证方式。此处需要RestClient工具支持Kerberos，例如：curl。

6. Client输入的Keystone的用户名和密码（UserA/PasswordB）还是走password认证，获取token。

7. Apache-httpd端的Kerberos认证通过之后，会将Kerberos的用户名（UserA）写入auth请求的环境变量`REMOTE_USER`传递给Keystone，Keystone的external认证从数据库中通过REMOTE_USER查询用户信息。

8. Keystone创建token时，将UserA的bind信息（REMOTE_USER）写入token记录中，和创建者的用户信息绑定，也就是和REMOTE_USER绑定。我们查看token的SQL数据库记录时，会在token信息中看到bind的记录。

9. 如果这个UserA的tokenA被另一个UserB窃取，UserB使用tokenA，再次向Keystone发起修改UserA密码的请求，因为UserB需要使用kinit认证Kerberos，Kerberos会将UserB的用户信息放入`REMOTE_USER`，Keystone在认证tokenA的bind信息时，会发现tokenA中的bind-user是UserA而不是当前的UserB，验证失败，防止tokenA被窃取后重用。

## 现有问题

- Eventlet不支持Kerberos，所以除了Keystone和Swift（运行在httpd上），其他组件都不支持。
- 所有的OpenStack模块的client都不支持Kerberos。
- Kerberos证书过期之后，要重新生成，所以需要一个自动的方式，使host上Kerberos证书被刷新。

## 配置Token-Binding

配置项很简单，可以直接参考OpenStack的[官方文档](http://docs.openstack.org/developer/keystone/configuration.html#token-binding)。

参考文献：

[http://my.safaribooksonline.com/book/networking/linux/0596003919/authentication-techniques-and-infrastructures/linuxsckbk-chp-4-sect-12#X2ludGVybmFsX0h0bWxWaWV3P3htbGlkPTAtNTk2LTAwMzkxLTklMkZsaW51eHNja2JrLWNocC00LXNlY3QtMTImcXVlcnk9](http://my.safaribooksonline.com/book/networking/linux/0596003919/authentication-techniques-and-infrastructures/linuxsckbk-chp-4-sect-12#X2ludGVybmFsX0h0bWxWaWV3P3htbGlkPTAtNTk2LTAwMzkxLTklMkZsaW51eHNja2JrLWNocC00LXNlY3QtMTImcXVlcnk9)

[http://www.jamielennox.net/blog/2013/10/22/keystone-token-binding/](http://www.jamielennox.net/blog/2013/10/22/keystone-token-binding/)