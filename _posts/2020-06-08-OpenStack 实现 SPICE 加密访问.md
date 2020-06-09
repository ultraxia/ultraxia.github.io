---
layout:     post
title:      OpenStack 实现 SPICE 加密访问
subtitle:    ""
date:       2020-06-09
author:     奥特虾
header-img: img/post-bg1-security.jpg
catalog: false
tags:
    - OpenStack
---

> “这个需求有点紧，你调研一下，最迟周一就要联调”，”可是现在已经周五下午了啊....“



## 前言

原本，这是一个愉快的周五下午，手头的活基本都结束了，准备摸摸鱼，等待下班后去车站前往扬州，参加朋友的婚礼。此时，发生了上面的这段对话。



我们的安全团队通过各种普通用户意想不到的姿势，在不登陆控制台的情况下直接嗅探出了用户远程连接桌面的地址和端口。即便登录服务器还有一道密码认证关卡，但是毫无疑问这已经构成了一定的风险。



”中央已经决定了，就由你来修复这个安全漏洞，你去调研一下 SPICE 怎么做加密，赶紧开始开发吧“。



虽然嘴上满口答应，但是此时还是担心这个任务会影响我原本的出行计划，算了...先上手干着吧。









## 正文

### 设置 OpenStack 支持 SPICE 协议

OpenStack 默认采用 VNC 作为远程桌面协议浅析，既然需求是在 SPICE 协议的基础上实现加密，那么第一步就是得让 OpenStack  支持 SPICE 协议了，这一点其实很容易实现。



打开 `/etc/nova/nova.conf` ,将下面的这段配置贴上即可，这里的 IP 地址请替换成 OpenStack 的管理网地址。

```yaml
[spice]
enabled=true
server_listen=10.130.176.14
server_proxyclient_address=10.130.176.14
```



需要注意的是，网上有些教程会让你设置 SPICE 的同时 disable VNC 协议。我觉得可以，但没必要，VNC 和 SPICE 是可以同时开启的。



配置修改完成后，重启 Nova 服务就大功告成了。



### 如何将用户设置的密码传递到底层？

万里长征的第一步走完了，接下来就是支持加密了。我们的出发点是由用户设置一个远程连接密码，就像阿里云一样，通过网页进入控制台需要先验证连接密码，再验证服务器的登陆密码。要实现这样的功能，就意味着我们需要将用户定义的密码作为参数传递到底层，并根据相应的配置新建一台虚拟机。



因为是新手的缘故，一开始我走了一些弯路，我最初的设想是在 nova-api 的 body 中新增一个 spice_passwd 的参数，一层一层传递到 libvirt 层。但是我尝试修改了一下源码后发现实在是太复杂了，如果参数从 nova-api 传递进来，需要在 nova-api，nova-scheduler，nova-conductor，nova-compute 之间传来传去，需要改动的源码太多了，且容易出错。



后来拍拍脑袋才意识到，我可以把这个字段放到 metadata 里呀！



在 `nova\virt\libvirt\driver.py` 中，可以通过 `instance.metadata.get()` 方法直接获取到 metadata ，这样一来，参数传递的问题便迎刃而解了。



那么最重要的一点来了，如何才能设置 SPICE 密码呢。通过查询资料后发现，QEMU 是支持 SPICE 设置密码的, 如下方实例的 XML 文件所示，添加 passwd 的参数就可以设置连接密码，在这个例子中连接密码为 ”ptqhxx“。 

```
<graphics type='spice' autoport='yes' listen='0.0.0.0' keymap='en-us' passwd='ptqhxx'>
      <listen type='address' address='0.0.0.0'/>
 </graphics>
```



### 修改源码

接下来，就开始改源码吧！



先说一下思路，在 metadata 中我们新增一个 spice_passwd 的键，值为用户设置的密码。libvirt 获取到 metadata 中的 SPICE 密码后，将 XML 文件中的 passwd 值设置为接收到的密码。



在这里主要需要修改两个源码文件

1. `nova\virt\libvirt\config.py::LibvirtConfigGuestGraphics`
```python
class LibvirtConfigGuestGraphics(LibvirtConfigGuestDevice):

    def __init__(self, **kwargs):
        super(LibvirtConfigGuestGraphics, self).__init__(root_name="graphics",
                                                         **kwargs)

        self.type = "vnc"
        self.autoport = True
        self.keymap = None
        self.listen = None
        self.passwd = None                        <=== 新增

    def format_dom(self):
        dev = super(LibvirtConfigGuestGraphics, self).format_dom()

        dev.set("type", self.type)
        if self.autoport:
            dev.set("autoport", "yes")
        else:
            dev.set("autoport", "no")
        if self.keymap:
            dev.set("keymap", self.keymap)
        if self.listen:
            dev.set("listen", self.listen)
        if self.passwd:                           <=== 新增
            dev.set("passwd", self.passwd)        <=== 新增
        return dev
```



这一部分主要是在原先的数据结构中新增一个 passwd 的字段



2. `nova\virt\libvirt\driver.py::_get_guest_config`

```python
spice_passwd = instance.metadata.get('spice_passwd')
if (CONF.spice.enabled and
        virt_type not in ('lxc', 'uml', 'xen')):
    graphics = vconfig.LibvirtConfigGuestGraphics()
    graphics.type = "spice"
    graphics.keymap = CONF.spice.keymap
    graphics.listen = CONF.spice.server_listen
    graphics.passwd = spice_passwd
    
    guest.add_device(graphics)
    add_video_driver = True
```



这一部分是将获取到的 spice_passwd 字段后期写入到 XML 文件中



### 重启 Nova 服务

配置修改完成，源码修改完成，重启服务！



### 验证环节

接下来我们创建一台 spice_passwd 为 ”ptqhxx” 的虚拟机

```
nova --debug boot \
  --min-count 1 \
  --max-count 1 \
  --flavor  1  \
  --block-device id=$system_image_id,source=image,device=vda,dest=volume,size=40,bootindex=0,shutdown=remove \
  --nic net-id=cbcdbe0b-7591-468c-bec3-0a023adf005d \
  --meta 'spice_passwd'='ptqhxx'  \
  test
```



![instance](https://s1.ax1x.com/2020/06/09/t4sCX6.png)





使用工具远程连接，弹出密码验证框，输入密码，成功



![](https://s1.ax1x.com/2020/06/09/t4sXKP.png)



## 后记

至此，终于在下班前搞定了，很幸运，没有耽误当天的行程。



写下这篇文章的时候，已经是联调结束的周二了，回想起刚刚度过的周末真是令人难忘。



最后，祝我这位朋友新婚快乐，百年好合 。



![](https://wx1.sinaimg.cn/mw1024/d1797d2fgy1gfk2ccz5o3j215y0u0afs.jpg)



