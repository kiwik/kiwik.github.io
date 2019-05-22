---
layout: post
category : OpenStack
tagline: "regexisart"
tags : [OpenStack, develop, win-sshfs, git, DevStack]
---
{% include JB/setup %}

**如需转载，请标明原文出处以及作者**

*陈锐 RuiChen @kiwik*

*2014/1/18 21:41:58*

----------

*写在最前面：*

*此文是我在开发OpenStack的过程中的一点小小经验，习惯于Windows+Pycharm，如果你的开发环境是Linux+vim，恭喜你，你正走在正轨上 。*

可能大多数中国的OpenStack开发者都是在Windows平台上开发代码，那么OpenStack的[开发者指导](http://docs.openstack.org/developer/nova/devref/development.environment.html)对我们的用处就不大。

## 目标

大家一般的开发流程是在Windows上开发完代码，然后用win-scp或者Filezilla拷贝到Linux环境上做pep8，UT或者DevStack的验证，如果有问题再修改，再重新同步，中间可能还涉及到dos2unix的文件转换，想想就头大。

我们的目标是在Windows平台上用自己最熟悉的IDE开发代码，然后迅速的做pep8，UT和DevStack的验证。向Windows和Linux之间的同步文件说“不”！

开发过程中的几个重点：

- IDE
- pep8
- UT
- DevStack
- git review

## IDE

Windows平台上推荐Pycharm，功能很强大，3.0还提供了社区免费版，当然还有eclipse+pydev，也不错。

## pep8 + UT + DevStack + git review

好戏来了，Windows上怎么跑pep8、UT、DevStack？

pep8和UT在Windows上跑还有一点的可能，不过要经过一堆的安装依赖包、编译、workaround，我试过很麻烦，这点上Python确实不如Java。

DevStack完全没有可能性。

换一个思路，还是让他们跑在Linux上，在最合适的环境，做最合适的事。不管你是用ESX还是[Vagrant](http://kiwik.github.io/openstack/2013/12/22/%E4%BD%BF%E7%94%A8Vagrant%E5%92%8Cwin-sshfs%E6%94%AF%E6%8C%81OpenStack%E5%BC%80%E5%8F%91/)，都无所谓，创建一个Ubuntu，然后把`DevStack`装好，安装DevStack可以参考我以前的[blog](http://kiwik.github.io/openstack/2013/12/21/DevStack-install-in-China/)。

DevStack装完，再把vim，git，pip，tox都装好，这些后面都会用到，tox的使用可以参考我的[blog](http://kiwik.github.io/openstack/2013/07/15/Openstack%E5%B7%A5%E7%A8%8B%E7%9A%84%E6%8C%81%E7%BB%AD%E9%9B%86%E6%88%90%E5%AE%9E%E8%B7%B5-tox/)。这里是几个Linux上的git配色和基本设置。

```bash
git config --global core.excludesfile "/home/ruichen/.gitignore"
git config --global color.status auto
git config --global color.diff auto
git config --global color.branch auto
git config --global color.interactive auto
git config --global core.editor vim
git config --global core.autocrlf input
```

## win-sshfs

最后一个东西`win-sshfs`，这个是将Windows上的IDE环境和Linux上的git，pep8，UT和DevStack连接的关键。win-sshfs可以参考的我的另一篇[blog](http://kiwik.github.io/openstack/2013/12/22/%E4%BD%BF%E7%94%A8Vagrant%E5%92%8Cwin-sshfs%E6%94%AF%E6%8C%81OpenStack%E5%BC%80%E5%8F%91/)。

*win-sshfs说的简单一点就是通过ssh，将Linux的一个目录，挂载到Windows上作为一个盘符。*

我的设置是这样的。

![](https://raw.github.com/kiwik/kiwik.github.io/master/_posts_images/2014-01-19/1.PNG)

需要注意的是，ssh的登录用户一定要和DevStack的安装用户一致，这样可以保证在Windows上打开OpenStack源文件时，文件的用户权限不被修改。我使用vagrant安装的DevStack，所以登录用户是vagrant。由于DevStack也是在github上git clone的OpenStack的最新代码，所以直接使用/opt/stack/下已经git clone好的代码就可以了。直接映射Linux路径`/opt/stack/`。挂载成功之后就是H：盘。

然后用Pycharm直接打开H：盘上的目录，导入工程。

![](https://raw.github.com/kiwik/kiwik.github.io/master/_posts_images/2014-01-19/2.PNG)

打开文件就可以直接修改了。

![](https://raw.github.com/kiwik/kiwik.github.io/master/_posts_images/2014-01-19/3.PNG)

这里需要注意两点

- Pycharm或者Eclipse中的换行，要改成Linux兼容模式
- git config要设置成`core.autocrlf=input`

原因是我们直接修改的是Linux Guest OS中的文件，Linux git config中`core.autocrlf=false`，所以要避免把Windows上的CRLF提交到git仓库中。

![](https://raw.github.com/kiwik/kiwik.github.io/master/_posts_images/2014-01-19/4.PNG)

在Linux Guest OS中配置用户、邮箱和git review。修改完代码，检查了pep8和UT，还可以顺便在DevStack上进行一下全流程测试，一切ok，直接运行git review提交到Jenkins，是不是很方便？

- IDE (on windows)
- pep8 (on linux)
- UT (on linux)
- DevStack (on linux)
- git review (on linux)

这种方法的好处是，从头到尾，我们修改的就是一份代码，不用担心版本不一致，同时，还能使用习惯的Windows开发环境。Windows环境上连git都不用装。