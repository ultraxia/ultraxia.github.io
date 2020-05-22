---
layout:     post
title:      OpenStack 实现 GPU 与 CPU、内存的 NUMA 绑定
subtitle:    "\"这也太难了吧\""
date:       2020-05-22
author:     奥特虾
header-img: img/post-bg-cloud.jpg
catalog: false
tags:
    - OpenStack
---

> “负责安卓云底层开发的小彭请了假，这个需求有点紧，你来做吧 ”



## 前言

你好，我是小傅，这是我入职新公司的第二周。

我所接到的需求，就是这篇文章的标题，老实说，尽管我~~身经百战见得多了~~，但是看到这个需求时，我是谁我在哪，这是什么意思？

这个需求着实让我一时间无从下手，唉...接都接了，哪有怂的道理，那就开始做吧。



## 正文

解决这个问题的前提是，得先弄明白 NUMA 是什么，为此，在查阅了很多资料后我终于有些似懂非懂了，相关的资料我会放在文章的结尾，感兴趣的同学可以直接阅读，在这篇文章中我就不再赘述了，网上的大牛们写的很专业，我就不做复读机了。



不过呢为了方便你的理解，我这儿有一个不太准确的比喻，你可以将 NUMA 理解为**对单台物理机计算资源的划分**，举个栗子，假设我有一台 8 核 16G 的服务器，服务器上有 2 个 8G 内存条的插槽。那么，当你设置了两个 NUMA 后，NUMA0 用于管理 CPU 的 1-4 个核，以及第一根内存条，那 NUMA1 则用于管理 CPU 的 5-8个 核，以及第二根内存条。



![img](https://ss2.bdstatic.com/70cFvnSh_Q1YnxGkpoWK1HF6hhy/it/u=1272062901,807942533&fm=26&gp=0.jpg)





我们都知道，虚拟化是云计算的基础。在一台云主机上，对于用户而言貌似是在独享这个云主机的硬件资源，可实际却并非如此，你所用的云主机的 CPU 和内存（我们一般称为 vCPU 和 vRAM ，即虚拟CPU 和虚拟内存）实际上都是在物理机硬件上虚拟出来的一部分，每一台云主机在物理服务器上其实都是一个进程，这些进程之间共享底层的计算资源（禁止套娃）。



那么，在云计算中为什么要做 NUMA 绑定呢？



我们继续来举个栗子，假设我有一台 2 核 4G 的云主机（虚拟机），其中 vCPU0 在物理 CPU0 上，vCPU1 在物理 CPU1上（内存同理），而物理 CPU0 属于 NUMA0 ，物理CPU1 属 于NUMA1 。这就是一个典型的跨 NUMA 的栗子，为了方便你的理解，我找到了一张图。



![NUMA ](http://static.open-open.com/lib/uploadImg/20150323/20150323105252_596.png)





看了张图后，你应该就很清楚了，虽然 NUMA 的出现是为了提升计算机的性能，但是想要将性能发挥到极致，还得将资源绑定在同一个 NUMA 中，尽可能地不要出现跨 NUMA 。



之所以会产生这样的需求是因为我们公司有一个叫做云手机的产品，云手机的本质其实就是一个跑着 Android 系统的虚拟机。



![云手机 ](https://s1.ax1x.com/2020/05/22/YjEAKK.png)



在过去的性能测试中，测试团队发现云手机的 GPU 性能存在着一些瓶颈，在和美国团队沟通后，决定将 GPU、CPU、内存绑定到同一个 NUMA 上。请注意，这里指的是云手机的 vGPU、vCPU、vRAM 最终所对应的物理硬件资源。



CPU和内存绑定到同一个 NUMA 不算复杂，Libvirt 有对应的命令可以实现，所以只需要阅读这部分的源码，就不难找到解决方案，难的是如何绑定 GPU，numactl 命令可以查看 CPU 和 内存的 NUMA 绑定，但是对 GPU 似乎有些心有余而力不足。



那就直接说结论吧，通过虚拟机的 XML 文件可以看到，每块显卡都对应了一块 Linux 上的设备文件，可能是 renderD128，也有可能是 renderD129，这两个设备分别对应了 NUMA0 和 NUMA1。



![device](https://s1.ax1x.com/2020/05/23/YjeSUg.png)



知道了这个之后，逻辑上就变得简单多了，只需要获取到CPU或者内存所在的 NUMA 后去指定 GPU 对应的设备，或者知道 GPU 对应设备后去指定CPU和内存所在的 NUMA 就就行了。



在我们的业务逻辑中，程序会根据 GPU 对应的设备的负载情况，动态地选择负载较低的设备，以实现一个负载均衡的效果，所以，知道 GPU 所在 NUMA 后便可以去指定CPU和内存的 NUMA 了。



这段伪代码展示了是我们的实际业务代码，由于这部分属于商业机密，我省略了中间的部分，但是不影响我们的理解，这段代码用于为虚拟机添加一块 GPU 的设备，我在这仅做了一行代码的改动，就是最后返回一个设备名，有了这个设备名后仅需定义一个字典做  GPU device 和 NUMA id 的 mapping 即可。



![code](https://s1.ax1x.com/2020/05/23/YjmT1A.png)





熟悉云计算的朋友们肯定知道，OpenStack 最终是生成了一个 XML 文件，最终 libvirt 通过 XML 里定义的参数去新建一台虚拟机。



所以，为了生成这样的 XML 文件，需要我们通过 libvirt 的 driver 去获取到相应的信息。找啊找，我们最终在 `/nova/virt/libvirt/driver.py` 中找到了这个函数。



![](https://s1.ax1x.com/2020/05/23/YjnVNF.png)



这部分的代码写的很清楚，如果` guest_cpu_numa_config `不存在，即用户没有配置 NUMA 信息，则系统会自动为它分配一个默认值，显然这是我们不愿看到的，我们希望自己去配置 NUMA 的信息，即代码走到 else 的分支里，想要走到 else 分支，则必须满足两个条件，`topology `和` guest_cpu_numa_config `不为空值。



这里播报一个小插曲，在我调试的过程中，`topology `始终为 None，这意味着我的机器不支持NUMA，然而这显然是不可能的，到底是哪里出了问题，没办法，继续啃源码吧，点进 `self._get_host_numa_topology `函数后，会发现有一个函数会去检查你的机器是否支持 NUMA，注意看 `support_matrix`，这里定义了硬件的架构，在初始的源码中只有 `arch.I686 `和` arch.X86_64` 这两个值，然而通过 virsh 命令发现，我的机器是 `AARCH64` 架构的，自己填上，解决。



![](https://s1.ax1x.com/2020/05/23/Yjnn39.png)



那么第二个问题来了，`guest_cpu_numa_config `是 None 怎么办，可以从之前的代码中看到，`guest_cpu_numa_config `是向某个函数传递了` instance_numa_topology `后解析出来的，搞清楚先有鸡还是先有蛋后，问题就迎刃而解了。`instance_numa_topology ` 依旧是一个空值，那有句话怎么说来着....没有枪没有炮，那就自己写吧。



可是 Python 不像 Go 一样，一个数据由哪些属性组成通过结构体的变量便一目了然，`instance_numa_topology`  中到底应该写哪些字段。其实我也不知道，不过瞎猫碰上死耗子，[OpenStack官方文档](https://docs.openstack.org/nova/rocky/contributor/testing/libvirt-numa.html
) 给了我很大的灵感，接着就是通过 DEBUG 去倒推它的字段了。



![](https://s1.ax1x.com/2020/05/23/YjnJ4e.png)



cell_id 其实就是 NUMA id，这里设为和 GPU device 对应的 NUMA id 即可，其他的参数，根据当前虚拟机的flavor填写即可。



这样便大功告成了，重启 nova-compute 服务，创建一台虚拟机，查看 XML 文件，如预期一样，GPU、CPU、内存都被绑定在了 NUMA 0 上。



![](https://s1.ax1x.com/2020/05/23/YjZqgI.png)



![](https://s1.ax1x.com/2020/05/23/YjZb8A.png)



BTW，我创建的虚拟机有2个core，分别落在了物理GPU的0-127上，在我的机器中，0-127是 NUMA0 的势力范围。



这样一来，需求就解决了。





## 后记

关于 NUMA ，我为你精选了下面这些资料

[OpenStack Nova 高性能虚拟机之 NUMA 架构亲和](https://www.cnblogs.com/jmilkfan-fanguiju/p/10589768.html)

[Openstack NUMA 分析 (入门)](https://www.iteye.com/blog/hangdong-zhang-2294888)

[浅析NUMA机制](https://www.jianshu.com/p/0607c5f62c51)

最后，非常感谢我的同事们在此过程中对我的指点与帮助，正是他们不厌其烦地为我解答问题，我才得以在短时间内实现这个需求，也谢谢耐心看到了这里的你。

那么，下篇再会。



