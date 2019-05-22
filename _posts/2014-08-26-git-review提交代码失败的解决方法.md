---
layout: post
category : OpenStack
tagline: "regexisart"
tags : [git, review, gerrit, ssh, http, vpn, socks]
---
{% include JB/setup %}

**如需转载，请标明原文出处以及作者**

*陈锐 RuiChen @kiwik*

*2014/8/26 21:05:44*

----------

*写在最前面：*

*这段时间OpenStack社区gerrit的29418端口被墙了，小伙伴们都十分着急，写好了patch半天提交不了，那种感觉你懂的。还好团队里能人辈出，翻墙都翻出了花儿，在这里给大家Sharing一下。*

## 土豪型

土豪们就是有钱，花个百来块，搞个vpn，代码唰唰的就上去了，看的我们这些技术屌丝十分的羡慕。

vpn就像花钱租了个梯子，钱到墙翻，不解释。

## 技术屌丝型

技术屌丝的特点就是喜欢研究个所谓的“底层”技术，在这里配置一个ssh的socks代理，很能体现技术屌丝的特质。

在~/.ssh/目录创建一个config文件，内容如下，在网上找个socks的代理，用ip和port替换`socks-proxy-ip:port`，配好别忘了用你自己的gerrit用户测试一下，`ssh -p 29418 your-user-name@review.openstack.org`

```
stack@devstack:/opt/stack/nova$  [bp/io-ops-weight]$ cat ~/.ssh/config
Host review.openstack.org
    ProxyCommand connect -S socks-proxy-ip:port %h %p
    IdentityFile ~/.ssh/id_rsa
    TCPKeepAlive yes
    IdentitiesOnly yes
    PreferredAuthentications publickey
stack@devstack:/opt/stack/nova$  [bp/io-ops-weight]$ ssh -p 29418 kiwik@review.openstack.org

  ****    Welcome to Gerrit Code Review    ****

  Hi Rui Chen, you have successfully connected over SSH.

  Unfortunately, interactive shells are disabled.
  To clone a hosted Git repository, use:

  git clone ssh://kiwik@review.openstack.org:29418/REPOSITORY_NAME.git

Connection to review.openstack.org closed.
```

> 注意：这种方法要用到一个connect命令，windows下安装git之后，就直接安装了这个命令，但需要把git安装目录的bin目录配置到%PATH%下；linux环境，需要通过包管理器安装一下，例如：apt-get install connect-proxy

附送一个查找socks代理的网址：

[http://letushide.com/filter/socks5,all,all/list_of_free_SOCKS5_proxy_servers](http://letushide.com/filter/socks5,all,all/list_of_free_SOCKS5_proxy_servers)

技术屌丝洋洋得意的通过这种方法提交上了一个patch，很高兴，但是第二天发现代理不行了，一顿手忙脚乱的找免费代理，又是改配置，又是测试的忙活。

配置ssh socks免费代理这种方式就像是一群人都要翻墙，大家都去搬砖垫脚，但砖就那么几块，今天翻过去了，明天说不定砖就被别人搬走了，太折腾。

## 技术大牛型

是牛人就不折腾，也不靠人民币，挥挥手问题解决。git review默认通过ssh的方式，http和https的方式是不是也支持能？答案是肯定的，为了就是避免你在Firewall之后不能通过29418端口提交代码，但是需要一点小技巧。

- 首先，需要登录review.openstack.org，然后在Settings -> HTTP Password里，生成一个HTTP密码，应该是一个大小写加数字的随机字符串。

- 然后通过`git remote set-url gerrit https://username:http-password@review.openstack.org/openstack/nova.git`命令把ssh修改成https方式，当然也可以用http，经过我的实验都是可以的。别忘了把上面字符串中的用户名/密码改成你的gerrit用户名和上一步生成的HTTP密码。

修改前：

```
chenrui@CHENRUI-PC /d/dev/PyProjects/nova (master)
$ git remote -v
gerrit  ssh://kiwik@review.openstack.org:29418/openstack/nova.git (fetch)
gerrit  ssh://kiwik@review.openstack.org:29418/openstack/nova.git (push)
origin  https://github.com/openstack/nova.git (fetch)
origin  https://github.com/openstack/nova.git (push)
```

修改后：

```
chenrui@CHENRUI-PC /d/dev/PyProjects/nova (master)
$ git remote -v
gerrit  https://kiwik:your-gerrit-http-password@review.openstack.org/openstack/nova.git (fetch)
gerrit  https://kiwik:your-gerrit-http-password@review.openstack.org/openstack/nova.git (push)
origin  https://github.com/openstack/nova.git (fetch)
origin  https://github.com/openstack/nova.git (push)
```

测试一下：

```
chenrui@CHENRUI-PC /d/dev/PyProjects/nova (master)
$ git review -n -v
2014-08-26 20:39:42.736000 Running: git log --color=never --oneline HEAD^1..HEAD
2014-08-26 20:39:42.779000 Running: git remote
2014-08-26 20:39:42.808000 Running: git branch -a --color=never
2014-08-26 20:39:42.844000 Running: git rev-parse --show-toplevel --git-dir
2014-08-26 20:39:42.873000 Running: git remote update gerrit
Fetching gerrit
From https://review.openstack.org/openstack/nova
   0701dcc..7fb2a03  master     -> gerrit/master
   e6dd7da..f8af69a  stable/havana -> gerrit/stable/havana
   9805f3d..9ada982  stable/icehouse -> gerrit/stable/icehouse
2014-08-26 20:40:00.831000 Running: git rev-parse HEAD
2014-08-26 20:40:00.869000 Running: git show-ref --quiet --verify refs/remotes/gerrit/master
2014-08-26 20:40:00.899000 Running: git rebase -p -i remotes/gerrit/master
2014-08-26 20:40:13.004000 Running: git reset --hard 4fcab31f5160f90f01e9b11017445a8bd9de8f90
2014-08-26 20:40:13.277000 Running: git config --get color.ui
2014-08-26 20:40:13.308000 Running: git log --color=always --decorate --oneline HEAD --not --remotes=gerrit
No changes between HEAD and gerrit/master. Submitting for review would
be pointless.
```

直接配置gerrit使用http协议这种方式就像是技术屌丝搬了半天砖，发现旁边的技术大牛在墙上开了个暗道直接过去了，呵呵，傻眼~~

> 放这样一篇文章在技术blog里，也算是有中国特色的技术blog了吧。