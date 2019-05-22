---
layout: post
category : OpenStack
tagline: "regexisart"
tags : [OpenStack, Vagrant, develop, win-sshfs, DevStack]
---
{% include JB/setup %}

**如需转载，请标明原文出处以及作者**

*陈锐 RuiChen @kiwik*

*2013/12/22 19:16:36*

----------

没事看了一下OpenStack的开发者指导，发现只有Linux下的开发环境搭建，那对我们这些Windows下的开发者应该怎么办呢？最近琢磨了一下，刚好运气不错，接连发现了几个很好用的工具可以支持这件事，下面来介绍一下我的方法。

## Vagrant

首先需要在PC上装一个虚拟机，我选的是[Vagrant](http://www.vagrantup.com/)，偶然的机会发现Pycharm中有对于Vagrant的支持，上网查了一下，Vagrant宣传可以快速搭建虚拟开发环境，用了一下发现也就是一个傻瓜式的虚拟机管理软件，底层的虚拟化用的是*VirtualBox*

1. 先安装 *VirtualBox* [下载](https://www.virtualbox.org/wiki/Downloads)，就安装Windows x86的就可以，小巧好用。

2. 然后安装 *Vagrant* [下载](http://downloads.vagrantup.com/)，选择最新版本的msi文件

3. 下载要使用的镜像文件，我使用的是 *Ubuntu 12.04 x64* [下载](http://files.vagrantup.com/precise64.box)，其他的镜像可以在这里找[here](http://www.vagrantbox.es/)，其实Vagrant也可以直接从网络地址下载镜像并导入，但是网速不好的话可能会比较慢，直接下载好，从本地导入会更快一些

4. 下载镜像到本地之后，直接在镜像所在目录执行`python -m SimpleHTTPServer`，就在镜像所在的目录启动了一个http server，直接访问 *http://127.0.0.1:8000/* 就能访问

5. 导入镜像`vagrant box add Ubuntu12.04x64 http://127.0.0.1:8000/precise64.box`，安装Vagrant之后，Vagrant的 *bin* 目录就会被自动加入%PATH%，如果执行vagrant命令没有成功，可以手动加一下%PATH%路径

6. 为Vagrant创建一个工作目录，我创建的是 *E:\VirtualBoxVMs* ，然后执行`vagrant init Ubuntu12.04x64`初始化

7. 初始化之后，在当前目录会生成一个Vagrant的配置文件 *Vagrantfile*，用一般的编辑器打开，里面就是init的Vagrant虚拟机的配置文件，详细配置可以查看[这里](http://docs.vagrantup.com/v2/vagrantfile/index.html)，我在这里就介绍几个有用的配置

8. 虚拟机的内存大小，默认的Vagrant虚拟机的内存貌似是300M左右，建议将值调整到1G，刚开始没有调整，导致我安装DevStack有些慢，剩余内存只有8M左右，很简单，将下面一段去掉默认的注释就可以了，`# vb.gui = true`最好还是保持注释

    ```ruby
    config.vm.provider :virtualbox do |vb|
      # Don't boot with headless mode
      # vb.gui = true
      
      # Use VBoxManage to customize the VM. For example to change memory:
      vb.customize ["modifyvm", :id, "--memory", "1024"]
    end
    ```

9. 端口映射，Vagrant默认会将Host OS的2222端口映射到guest OS的22端口，所以通过一般的ssh客户端登陆，需要指定127.0.0.1的2222端口ssh登陆，通过 *vagrant ssh*  是更加方便的方式，但是如果想访问OpenStack的Horizon怎么办？需要改`Vagrantfile`，将下面的一行去掉注释，这会将Host OS的8080端口映射到guest OS的80端口，访问Horizon就直接在浏览器里输入 *http://127.0.0.1:8080*  就可以了，很方便

    ```ruby
    config.vm.network :forwarded_port, guest: 80, host: 8080
    ```

10. 共享文件目录，Vagrant默认会将Host OS的init的目录映射到VirtualBox guest OS的`/vagrant`目录，使用的是将Host OS的init目录通过vboxfs的方式mount到guest OS上，在guest OS上可以通过 *mount* 命令查看，如果还想将其他的目录同步到guest OS，可以用如下的方法，第一个目录代表Host OS的路径，第二个目录代表guest OS的路径，最后一个参数 *disabled: false* ，是我个人习惯加的，如果改成true，就表示这个配置不生效

    ```ruby
    config.vm.synced_folder "D:/dev/PyProjects/remote", "/vagrant_data", disabled: false
    ```

11. 所有对于`Vagrantfile`的修改，都需要执行 `vagrant reload`重启guest OS生效

12. 如果虚拟机创建成功，DevStack安装好了之后，就可以通过`vagrant package`命令将这个已配置好DevStack的Ubuntu虚拟机打包成box文件，用于以后的环境恢复，或者直接分发给项目组的同事使用，关于DevStack的安装配置可以参考的我的另一篇[blog](http://kiwik.github.io/openstack/2013/12/21/DevStack-install-in-China/)

![][1]

*以上的所有的Vagrant的功能，帮助我们方便的管理了个人的OpenStack开发环境，完成了Host OS向guest OS同步文件，但是还有一个问题，就是怎么方便的修改和调试guest OS上的代码？也就是从guest OS向Host OS的文件同步，例如：想直接在DevStack环境的nova代码上加上调试日志，并查看代码的调用逻辑，这就用到了`win-sshfs`*

在这里心中默默的羡慕了一下vi流的大神们，你们压根不用这么麻烦。:)

## win-sshfs

win-sshfs的功能很强大，可以将Linux的目录直接mount到Windows系统上作为一个根盘符，详细文档请查看[here](http://code.google.com/p/win-sshfs/)

具体的安装，跟着官方文档一步一步来就可以了，先需要把 *.Net*  和 *Dokan*  安装好，win-sshfs安装的时候会检查依赖的库是否已经安装。

根据下图所示，输入本机ip 127.0.0.1和用户名密码，默认是*vagrant/vagrant*，记得要把端口改为*2222*，点击Mount，就开始挂载了，当右下角图标变为Unmount，就表示挂在成功，可以直接去**我的电脑**里看，应该多了一个根盘符，点开就可以直接访问Vagrant guest OS上的文件了，可以选择你喜欢的IDE直接编辑，我使用的Pycharm直接将根盘符中nova文件夹导入，就可以开始编辑nova的源代码了

![][2]
![][3]

[1]: https://raw.github.com/kiwik/kiwik.github.io/master/_posts_images/2013-12-22/1.JPG
[2]: https://raw.github.com/kiwik/kiwik.github.io/master/_posts_images/2013-12-22/2.JPG
[3]: https://raw.github.com/kiwik/kiwik.github.io/master/_posts_images/2013-12-22/3.JPG