---
layout: post
category : OpenStack
tagline: "regexisart"
tags : [OpenStack, DevStack, install, pip, apt-get, Ubuntu]
---
{% include JB/setup %}

**如需转载，请标明原文出处以及作者**

*陈锐 RuiChen @kiwik*

*2013/12/21 16:28:59*

----------

标题取得有点大，不过我觉得中国的开发者应该能够理解，在中国用DevStack搭建一套OpenStack环境究竟有多麻烦，中间会出多少不能预知的问题，在这里分享一下我的做法。

[DevStack](http://devstack.org/guides/single-vm.html)可是号称的5分钟安装好OpenStack的

> Use your cloud to launch new versions of OpenStack in about 5 minutes.

## step0

创建一个虚拟机，我的开发环境是Windows 7，所以使用了`Vagrant`快速创建了一个Ubuntu 12.04 x64的虚拟机，在Windows上使用Vagrant可以参考我的另一篇[blog](http://kiwik.github.io/openstack/2013/12/22/%E4%BD%BF%E7%94%A8Vagrant%E5%92%8Cwin-sshfs%E6%94%AF%E6%8C%81OpenStack%E5%BC%80%E5%8F%91/)，在这里就不在多说了。

## step1

ssh登录虚拟机，使用的是Vagrant，所以默认使用vagrant用户登录，在`/home/vagrant`目录下，先安装git

```bash
sudo apt-get install git
```

这里提醒一下使用Vagrant的时候ssh登录到虚拟机上，很多命令都要使用sudo

## step2

将DevStack git clone到虚拟机上

```bash
git clone https://github.com/openstack-dev/devstack.git
```

## step3

执行`./stack.sh`？等等，现在还不是时候，在国内的网络情况下，一定会有各种超时，先想一下DevStack安装的过程中有哪些命令需要连接网络，`apt-get` `curl` `pip`，先来搞定apt-get

## step4

将Ubuntu默认的源改成国内的源地址，大家一定很熟悉了，这里贴一个例子，是我这里最快的搜狐的源

```bash
cp /etc/apt/sources.list /etc/apt/sources.list.bak
vim /etc/apt/sources.list
```

```
deb http://mirrors.sohu.com/ubuntu/ precise-updates main restricted
deb-src http://mirrors.sohu.com/ubuntu/ precise-updates main restricted
deb http://mirrors.sohu.com/ubuntu/ precise universe
deb-src http://mirrors.sohu.com/ubuntu/ precise universe
deb http://mirrors.sohu.com/ubuntu/ precise-updates universe
deb-src http://mirrors.sohu.com/ubuntu/ precise-updates universe
deb http://mirrors.sohu.com/ubuntu/ precise multiverse
deb-src http://mirrors.sohu.com/ubuntu/ precise multiverse
deb http://mirrors.sohu.com/ubuntu/ precise-updates multiverse
deb-src http://mirrors.sohu.com/ubuntu/ precise-updates multiverse
deb http://mirrors.sohu.com/ubuntu/ precise-backports main restricted universe multiverse
deb-src http://mirrors.sohu.com/ubuntu/ precise-backports main restricted universe multiverse
```

## step5

当前的DevStack脚本，首先要安装pip，pip的安装是通过curl在pypi的官方源上下载的，一开始我就是卡在这步，pip半天不能下载，总是卡在40%，直接找到安装pip的脚本`vi devstack/tools/install_pip.sh`，把curl的地址也改成国内的地址

from

`PIP_TAR_URL=https://pypi.python.org/packages/source/p/pip/pip-$INSTALL_PIP_VERSION.tar.gz`

to

`PIP_TAR_URL=http://e.pypi.python.org/packages/source/p/pip/pip-$INSTALL_PIP_VERSION.tar.gz`

##  step6

然后搞定pip的安装源

创建`~/.pip/pip.conf`文件，将pip的源配置成国内的源，超时时间也调长

```ini
[global]
timeout = 6000
index-url = http://e.pypi.python.org/simple
[install]
use-mirrors = true
mirrors = http://e.pypi.python.org
```

## step7

可以执行`./stack.sh`了，速度刷刷的

PS：另外还有一个方法可以试试，如果你有很强大的代理，可以直接在localrc中设置代理`http_proxy` `https_proxy` `no_proxy`三个常量，stack.sh脚本会将这个三个设置应用到安装过程中，不过我这里没有成功，在curl下载https的pip时，报了ssl证书错误。

使用Vagrant的时候，如果虚拟机需要使用主机代理，首先找到默认网关，默认网关加上代理端口就可以了，我用的是goagent，所以配置就是这样了

```ini
http_proxy="10.0.2.2:8087"
https_proxy="10.0.2.2:8087"
no_proxy="127.0.0.1"
```

![][1]

![][2]

[1]: https://raw.github.com/kiwik/kiwik.github.io/master/_posts_images/2013-12-21/1.JPG
[2]: https://raw.github.com/kiwik/kiwik.github.io/master/_posts_images/2013-12-21/2.JPG
