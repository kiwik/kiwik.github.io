---
layout: post
category : OpenStack
tagline: "regexisart"
tags : [OpenStack, KVM, QEMU, libvirt]
---
{% include JB/setup %}

**如需转载，请标明原文出处以及作者**

*陈锐 RuiChen @kiwik*

*2014/5/4 17:53:39 *

----------

*写在最前面：*

*这段时间一直在墨西哥出差，其中遇到了各种糟心的事儿，关注我微博的同学可能都知道，但是要说的是，也有一些收获，一个就是终于在30岁的时候在墨西哥找到了一点点学英语的小窍门；另一个就是这段时间一直在想办法实现一个Ceilometer的blueprint，由于用到了libvirt，QEMU和KVM，对虚拟化的理解有了一点点进步，总结了一下，写下这篇blog。*

> 这篇文章是对KVM, QEMU，libvirt中的一些概念的解释和澄清，如有错误之处，请大家随时指出。

## KVM

KVM是`Kernel-based Virtual Machine`的缩写。从Linux kernel 2.6.20开始就包含在Linux内核代码之中，使用这个或者更高版本内核的Linux发行版，就直接可以使用KVM。KVM依赖于host CPU的虚拟化功能的支持(Intel-VT & AMD-V)，类似于Xen的HVM，对于guest OS的内核没有任何的要求，可以直接支持创建Linux和Windows的虚拟机。

这里有个图详细的解释了KVM的运行原理：

![][1]

KVM支持用户态(Userspace)进程通过KVM内核(Kernel)模块利用CPU的虚拟化技术创建虚拟机。虚拟机的vCPU映射到进程中的线程，虚拟机的RAM映射到进程的内存地址空间，IO和外围设备通过进程进行虚拟化，也就是通过QEMU。所以我们看到在OpenStack中创建一个虚拟机就对应一个计算节点的QEMU进程。

再来看一下KVM的内存模型，这和我正在做的那个blueprint相关(Memory Balloon stats)。

![][2]

刚才说到guest OS RAM的地址空间映射到qemu-kvm进程的内存地址空间，这样进程就可以很容易的对于guest OS的RAM进行控制，当guest需要使用RAM时，qemu-kvm就在自己的进程内存空间中划分一段给guest用。对于guest OS设置了MaxMemory和CurrentMemory之后，guest OS的RAM上限也就有了，就是MaxMemory，如果当前的guest实际使用不了那么多RAM，就可以将CurrentMemory调小，将多余的内存还给host，guest中看到的内存大小就是CurrentMemory，这就是`Memory Balloon`特性。

## QEMU

刚才讲KVM的时候一直说进程可以通过KVM模块创建虚拟机，那个所谓的进程其实就是QEMU。KVM团队维护了一个QEMU的分支版本(qemu-kvm)，使QEMU可以利用KVM在x86架构上加速，提供更好的性能。qeum-kvm版本的目标是将所有的特性都合入上游的QEMU版本，之后这个版本会废弃，直接使用QEMU主干版本。

其实，之前还有Xen维护的QEMU版本，叫做qemu-xen，由于所有的特性都已经合入QEMU1.0，所以Xen现在直接使用了QEMU。

> 之前在安装OpenStack的时候，都需要安装一个包kvm，我个人认为这个包其实就是QEMU，KVM是已经包含在Linux Kernel里了，所以不需要再安装。包名称使用kvm有点混淆概念。这点在Ubuntu的包描述上可以看出来。

{% highlight bash %}

stack@devstack:~$  [master]$ dpkg -l | grep qemu
ii  kvm                              1:84+dfsg-0ubuntu16+1.0+noroms+0ubuntu14.13 dummy transitional package from kvm to qemu-kvm
ii  qemu                             1.0+noroms-0ubuntu14.13                     dummy transitional package from qemu to qemu-kvm
ii  qemu-kvm                         1.0+noroms-0ubuntu14.13                     Full virtualization on i386 and amd64 hardware

{% endhighlight %}

在Nova的配置文件中有一个`virt_type`配置项，可以配置成kvm也可以是qemu，这里的kvm其实指的也是qemu-kvm。如果要配置成kvm，需要host的CPU支持Intel-VT和AMD-V技术，并且要载入kvm内核模块，由于有硬件加速，创建的guest OS的性能要优于qemu；qemu配置项指的就是完全的QEMU虚拟化，没有硬件加速，主要用于比较老式的CPU和操作系统环境，或者是在虚拟机中创建虚拟机的情况，当然完全的QEMU虚拟化性能要比qemu-kvm差一些。

*QEMU的功能大致分为两类：*

- 模拟(emulator)
- 虚拟化(virtualizer)

模拟：就是在一种CPU架构上模拟另一种CPU架构，运行程序。例如：在x86环境上模拟ARM的运行环境，执行ARM程序，或者在PowerPC环境上模拟x86指令集。

虚拟化：就是在host OS上运行guest OS的指令，并为guest OS提供虚拟的CPU、RAM、IO和外围设备。

我们在OpenStack中使用的就是QEMU的虚拟化功能。

QEMU进程直接使用了KVM的接口`/dev/kvm`，向KVM发送创建虚拟机和运行虚拟机的命令。框架代码如下：

{% highlight c linenos %}

open("/dev/kvm")
ioctl(KVM_CREATE_VM)
ioctl(KVM_CREATE_VCPU)
for (;;) {
     ioctl(KVM_RUN)
     switch (exit_reason) {
     case KVM_EXIT_IO:  /* ... */
     case KVM_EXIT_HLT: /* ... */
     }
}

{% endhighlight %}

虚拟机运行起来之后，当guest OS发出硬件中断或者其它的特殊操作时，KVM退出，QEMU可以继续执行，这时QEMU根据KVM的退出类型进行模拟IO操作响应guest OS。

由于QEMU是一个普通的用户态进程，而guest OS的运行又在QEMU当中，所以host不能绕过QEMU直接观察guest OS。但是QEMU提供了一系列的接口可以导出guest OS的运行情况，内存状态和IO状态等。

以下就是一个QEMU进程：

{% highlight bash %}

109       1673     1  1 May04 ?        00:26:24 /usr/bin/qemu-system-x86_64 -name instance-00000002 -S -machine pc-i440fx-trusty,accel=tcg,usb=off -m 2048 -realtime mlock=off -smp 1,sockets=1,cores=1,threads=1 -uuid f3fdf038-ffad-4d66-a1a9-4cd2b83021c8 -smbios type=1,manufacturer=OpenStack Foundation,product=OpenStack Nova,version=2014.2,serial=564d2353-c165-6238-8f82-bfdb977e31fe,uuid=f3fdf038-ffad-4d66-a1a9-4cd2b83021c8 -no-user-config -nodefaults -chardev socket,id=charmonitor,path=/var/lib/libvirt/qemu/instance-00000002.monitor,server,nowait -mon chardev=charmonitor,id=monitor,mode=control -rtc base=utc -no-shutdown -boot strict=on -device piix3-usb-uhci,id=usb,bus=pci.0,addr=0x1.0x2 -drive file=/opt/stack/data/nova/instances/f3fdf038-ffad-4d66-a1a9-4cd2b83021c8/disk,if=none,id=drive-virtio-disk0,format=qcow2,cache=none -device virtio-blk-pci,scsi=off,bus=pci.0,addr=0x4,drive=drive-virtio-disk0,id=virtio-disk0,bootindex=1 -drive file=/opt/stack/data/nova/instances/f3fdf038-ffad-4d66-a1a9-4cd2b83021c8/disk.config,if=none,id=drive-ide0-1-1,readonly=on,format=raw,cache=none -device ide-cd,bus=ide.1,unit=1,drive=drive-ide0-1-1,id=ide0-1-1 -netdev tap,fd=26,id=hostnet0 -device virtio-net-pci,netdev=hostnet0,id=net0,mac=fa:16:3e:db:86:d4,bus=pci.0,addr=0x3 -chardev file,id=charserial0,path=/opt/stack/data/nova/instances/f3fdf038-ffad-4d66-a1a9-4cd2b83021c8/console.log -device isa-serial,chardev=charserial0,id=serial0 -chardev pty,id=charserial1 -device isa-serial,chardev=charserial1,id=serial1 -vnc 127.0.0.1:1 -k en-us -device cirrus-vga,id=video0,bus=pci.0,addr=0x2 -device virtio-balloon-pci,id=balloon0,bus=pci.0,addr=0x5

{% endhighlight %}

呵呵，可能因为是Java程序员出身，越看QEMU越像Java的沙箱模型，只不过Java的沙箱运行的是Java程序，QEMU运行是虚拟机。

## libvirt

最后再来看一下libvirt。

libvirt相对的简单，就是一个统一的虚拟化管理接口，当前支持的虚拟化实现如下：

- The KVM/QEMU Linux hypervisor
- The Xen hypervisor on Linux and Solaris hosts.
- The LXC Linux container system
- The OpenVZ Linux container system
- The User Mode Linux paravirtualized kernel
- The VirtualBox hypervisor
- The VMware ESX and GSX hypervisors
- The VMware Workstation and Player hypervisors
- The Microsoft Hyper-V hypervisor
- The IBM PowerVM hypervisor
- The Parallels hypervisor
- The Bhyve hypervisor

第一个就是QEMU，libvirt的virsh是可以传递QEMU的监控命令的，我希望实现的那个blueprint就是用libvirt去传递QEMU的监控命令，开启QEMU的内存使用统计，然后再通过libvirt获取虚拟机的内存使用情况。

{% highlight bash %}

stack@devstack:~$  [master]$ virsh version
Compiled against library: libvirt 1.2.2
Using library: libvirt 1.2.2
Using API: QEMU 1.2.2
Running hypervisor: QEMU 2.0.0

stack@devstack:~$  [master]$ virsh help  qemu-monitor-command
  NAME
    qemu-monitor-command - QEMU Monitor Command

  SYNOPSIS
    qemu-monitor-command <domain> [--hmp] [--pretty] {[--cmd] <string>}...

  DESCRIPTION
    QEMU Monitor Command

  OPTIONS
    [--domain] <string>  domain name, id or uuid
    --hmp            command is in human monitor protocol
    --pretty         pretty-print any qemu monitor protocol output
    [--cmd] <string>  command

{% endhighlight %}


*好了，今天也就写到这儿，多谢大家耐心看完，希望对你有所帮助。*

[1]: https://raw.github.com/kiwik/kiwik.github.io/master/_posts_images/2014-05-04/1.png
[2]: https://raw.github.com/kiwik/kiwik.github.io/master/_posts_images/2014-05-04/2.png