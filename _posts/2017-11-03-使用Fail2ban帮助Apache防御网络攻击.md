---
layout: post
category : OpenStack
tagline: "regexisart"
tags : [apache, fail2ban, mod-security, mod-evasive, cicd]
---
{% include JB/setup %}

**如需转载，请标明原文出处以及作者**

*陈锐 RuiChen @kiwik*

*2017/11/3 19:23:06*

----------

*写在最前面：*

*网络安全的东西距离我现在的工作比较遥远，说实话没有想到会被攻击，但还是遇上了，也许攻击者并非具有针对性，让我有一天的时间将漏洞补上，算是侥幸。在公有云上使用公有 IP 暴露 HTTP 服务，还是要多加小心。*

## 背景 ##

这段时间在做 [OpenLab](http://openlab.website/ "http://openlab.website/") 相关的工作，目标是构建一套 CI 系统，将与 OpenStack 相关的 SDK 项目（OpenStack社区之外的）与 OpenStack 集成并进行自动化测试，从而发现 SDK 的问题，帮助 SDK 修复改进，促进 OpenStack 北向 API 生态发展。我们选用的是与 OpenStack 社区一致的方案，使用 `Zuul v3` + `Nodepool`，配合 Github 的 Pull Request 作为触发条件，自动触发测试任务，并使用华为公有云自动发放虚拟机作为测试环境，感兴趣的可以[在此](https://github.com/theopenlab/cicd-howto#reference-deployment-view "https://github.com/theopenlab/cicd-howto#reference-deployment-view")了解。

我们提供了一个测试任务的[状态页面](http://80.158.20.68/ "OpenLab Job Status")，实时展示任务执行情况，类似于 OpenStack 社区的 [zuulv3.openstack.org](http://zuulv3.openstack.org/ "http://zuulv3.openstack.org/") ，它由 Apache 提供服务，运行在公有云虚拟机上，虚拟机绑定了公网 IP ，对外可以访问。

## Apache 异常 ##

由于我们的 HTTP 服务直接监听 `80` 端口，所以公网上可以直接通过 IP 地址访问。有一天同事反映页面打开速度变得很慢，不像前两天流畅，感觉不对劲儿，登录看了一下 Apache 的 access 日志有好几个G，用 tail 跟了一下日志文件，发现每秒都有十几条请求，日志刷屏瞬间搞得我头很晕。分析了一下 access 日志的内容，其中 99.9% 都是访问一些国外网站的奇怪的请求，返回码都是 404 。重启 Apache 之后，瞬间又有大量的请求进入。没想到这么偏远的角落都有人来，看起来是要摆上几个防御塔了 :)

## mod_evasive ##

Apache 有一个 evasive 模块可以帮助识别并阻止 DoS/DDos 攻击，原理是跟踪 HTTP 请求，如果同一请求在一段配置的时间内访问超过阈值，就报警，然后在***应用级别***阻止后续的请求，返回 `403 Forbidden`。配置文件 `/etc/apache2/mods-enabled/evasive.conf` 可以设置具体的识别阈值和识别窗口大小。mod\_evasive 的安装启用方式如下：

{% highlight bash %}

sudo apt-get install libapache2-mod-evasive

sudo mkdir /var/log/mod_evasive

sudo a2enmod evasive

sudo systemctl reload apache2 

{% endhighlight %}

下面就是系统日志中的一条报警：

> Nov 02 06:59:23 openlab-cicd-misc mod_evasive[32968]: Blacklisting address 54.36.188.35: possible DoS attack.

## mod_security ##

mod\_security 相比 mod\_evasive 要更强大一些，作为一个 Web 应用防火墙（WAF），它使用规则引擎识别多种网络攻击行为模式，例如：SQL 注入，跨站攻击等，并可以改变 Request 和 Response 中的内容做出防御，它也工作在***应用级别***。安装和启用 mod\_security 的方式如下：

{% highlight bash %}

sudo apt-get install libapache2-modsecurity

sudo mv /etc/modsecurity/modsecurity.conf{-recommended,}

sudo a2enmod security2

sudo systemctl reload apache2

{% endhighlight %}

mod\_security 默认是启用在侦测模式下 `SecRuleEngine DetectionOnly`，只发现问题，不解决问题，球队中的“眼神防守者”，我们需要在配置文件 `/etc/modsecurity/modsecurity.conf` 中改为 `SecRuleEngine On`，使它在发现攻击行为后作出相应的防守动作， mod\_security 功能强大配置也比较多，在这我就不过多叙述。

启用 mod\_security 之后，默认在 Apache2 日志目录下会生成审计日志 `/var/log/apache2/modsec_audit.log`，这个日志和其他的 Apache 日志都是 `fail2ban` 日志数据源。内容如下：

{% highlight txt %}

--d0f2d400-A--
[02/Nov/2017:09:31:54 +0000] WfrmCsCoAMcAAS3qeMUAAAAJ 118.68.247.170 55708 192.168.0.199 80
--d0f2d400-B--
CONNECT 45.33.54.195:80 HTTP/1.0

--d0f2d400-F--
HTTP/1.1 405 Method Not Allowed
Allow: GET,HEAD,POST,OPTIONS
Content-Length: 313
Connection: close
Content-Type: text/html; charset=iso-8859-1

--d0f2d400-E--
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>405 Method Not Allowed</title>
</head><body>
<h1>Method Not Allowed</h1>
<p>The requested method CONNECT is not allowed for the URL /index.html.</p>
<hr>
<address>Apache/2.4.18 (Ubuntu) Server at 45.33.54.195 Port 80</address>
</body></html>

--d0f2d400-H--
Stopwatch: 1509615114235762 483 (- - -)
Stopwatch2: 1509615114235762 483; combined=19, p1=5, p2=10, p3=1, p4=0, p5=3, sr=0, sw=0, l=0, gc=0
Response-Body-Transformed: Dechunked
Producer: ModSecurity for Apache/2.9.0 (http://www.modsecurity.org/).
Server: Apache/2.4.18 (Ubuntu)
Engine-Mode: "ENABLED"

--d0f2d400-Z--

{% endhighlight %}

## fail2ban ##

fail2ban 与上面两个有本质的不同，它是一个独立软件，并不是 Apache 的扩展模块，更重要的是它工作在***网络级别***，上面两个工作在***应用级别***。fail2ban 通过扫描日志文件，可以是 Apache 也可以是 sshd 或其它应用的日志，发现符合模式的日志内容之后，阻止具有恶意迹象的 IP 继续对系统的访问。安装和启用 fail2ban 的方式如下：

{% highlight bash %}

sudo apt-get install fail2ban

sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local

sudo systemctl restart fail2ban

sudo fail2ban-client status

{% endhighlight %}

fail2ban 默认只启用了对于 sshd 防护，需要修改配置文件 `/etc/fail2ban/jail.local` 启用其他的能力，下面就是我配置的一段通过分析 apache mod_security 的审计日志，启动防御的配置段：

{% highlight ini %}

[apache-modsecurity]
enabled = true
port     = http,https
logpath = /var/log/apache2/modsec_audit.log
maxretry = 2

{% endhighlight %}

fail2ban 内置了识别很多种网络攻击的 filter 文件，在 `/etc/fail2ban/filter.d/` 中，我对 `apache-modsecurity.conf` 稍微修改了一下，满足我的要求。其中最重要的就是参数 `failregex`，通过正则表达式抓取日志中的攻击方 IP，然后进行阻止。

{% highlight ini %}

[INCLUDES]
before = apache-common.conf

[Definition]
#failregex = ^%(_apache_error_client)s ModSecurity:  (\[.*?\] )*Access denied with code [45]\d\d.*$
failregex = \[.*?\]\s[\w-]*\s<HOST>\s

ignoreregex = 

{% endhighlight %}

> 认真学习过正则表达式，让我一直觉得受益匪浅。

我一开始并不太理解 mod\_security 和 mod\_evasive 所谓***应用级别***的阻止与 fail2ban ***网络级别***的阻止有什么不同，对比启用 `fail2ban` 之后的现象发现，启用了 mod\_security 和 mod\_evasive 之后，网络请求还是会继续进入 Apache 处理，access 日志中还是存在大量异常的请求，只不过响应码从 404 变成了其他 403 之类，日志还是会暴涨好几个G，但是启用了 fail2ban 之后，进入 Apache 的请求明显变少了，看了下 fail2ban 的文档，发现一个老朋友 `iptables`，原来 fail2ban 在获取到恶意 IP 之后，直接通过 iptables 规则，将网络请求 reject 掉了，请求压根就不会到应用层级处理，当然就不会轮到 Apache 处理，了然，就如 OpenStack 中的安全组。

{% highlight bash %}

# iptables -L -n
Chain INPUT (policy ACCEPT)
target     prot opt source               destination         
f2b-apache-shellshock  tcp  --  0.0.0.0/0            0.0.0.0/0            multiport dports 80,443
f2b-apache-modsecurity  tcp  --  0.0.0.0/0            0.0.0.0/0            multiport dports 80,443
f2b-apache-fakegooglebot  tcp  --  0.0.0.0/0            0.0.0.0/0            multiport dports 80,443
f2b-apache-botsearch  tcp  --  0.0.0.0/0            0.0.0.0/0            multiport dports 80,443
f2b-apache-nohome  tcp  --  0.0.0.0/0            0.0.0.0/0            multiport dports 80,443
f2b-apache-overflows  tcp  --  0.0.0.0/0            0.0.0.0/0            multiport dports 80,443
f2b-apache-noscript  tcp  --  0.0.0.0/0            0.0.0.0/0            multiport dports 80,443
f2b-apache-badbots  tcp  --  0.0.0.0/0            0.0.0.0/0            multiport dports 80,443
f2b-apache-auth  tcp  --  0.0.0.0/0            0.0.0.0/0            multiport dports 80,443
f2b-sshd   tcp  --  0.0.0.0/0            0.0.0.0/0            multiport dports 22
DROP       all  --  93.188.246.105       0.0.0.0/0           

Chain f2b-apache-auth (1 references)
target     prot opt source               destination         
REJECT     all  --  54.36.188.35         0.0.0.0/0            reject-with icmp-port-unreachable
RETURN     all  --  0.0.0.0/0            0.0.0.0/0           

Chain f2b-apache-modsecurity (1 references)
target     prot opt source               destination         
REJECT     all  --  114.37.234.98        0.0.0.0/0            reject-with icmp-port-unreachable
REJECT     all  --  106.11.157.172       0.0.0.0/0            reject-with icmp-port-unreachable
REJECT     all  --  106.11.157.177       0.0.0.0/0            reject-with icmp-port-unreachable
REJECT     all  --  206.16.132.62        0.0.0.0/0            reject-with icmp-port-unreachable
REJECT     all  --  106.11.155.162       0.0.0.0/0            reject-with icmp-port-unreachable
REJECT     all  --  106.11.156.176       0.0.0.0/0            reject-with icmp-port-unreachable
REJECT     all  --  106.11.158.190       0.0.0.0/0            reject-with icmp-port-unreachable

{% endhighlight %}

