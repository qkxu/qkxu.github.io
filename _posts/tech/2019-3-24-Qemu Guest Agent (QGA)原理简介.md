---
layout: post
title: Qemu Guest Agent (QGA)原理简介
category: VMM
description: Qemu Guest Agent (QGA)原理简介
---
#  Qemu Guest Agent (QGA)原理简介

### 1.**简介**

经常使用vmWare的同学都知道有vmware-tools这个工具，这个安装在vm内部的工具，可以实现宿主机与虚拟机的通讯，大大增强了虚拟机的性能与功能，

如vmware现在的Unity mode下可以让应用程序无缝地与宿主机交互，更不用提直接复制粘帖文件及内容的小功能了。

对于KVM而言，其实也有一款这样的工具叫做 Qemu Guest Agent(以下称qga)。

### 2.**原理**

qga是一个运行在虚拟机内部的普通应用程序（可执行文件名称默认为qemu-ga，服务名称默认为qemu-guest-agent），其目的是实现一种宿主机和虚拟机进行交互的方式，这种方式不依赖于网络，而是依赖于virtio-serial（默认首选方式）或者isa-serial，而QEMU则提供了串口设备的模拟及数据交换的通道，最终呈现出来的是一个串口设备（虚拟机内部）和一个unix socket文件（宿主机上）。

qga通过读写串口设备与宿主机上的socket通道进行交互，宿主机上可以使用普通的unix socket读写方式对socket文件进行读写，最终实现与qga的交互，交互的协议与qmp相同（简单来说就是使用JSON格式进行数据交换），串口设备的速率通常都较低，所以比较适合小数据量的交换。

### 3.实例

宿主机上libvirt的虚拟机xml配置channel：

```xml
<channel type='unix'>
<source mode='bind' path='/var/lib/libvirt/qemu/org.qemu.test_agent'/>
<target type='virtio' name='org.qemu.guest_agent.0'/>
</channel>
```
其中：

source中的path路径就是再宿主机上生成的socket通道，而tartget中的name则是虚拟机中模拟出的串口设备。

target中name的名称并不是唯一的，但是这里最好采用这个，后面会说明。



虚拟机中需要安装下qemu-guest-agent：

```sh
yum install qemu-guest-agent
setenforce 0
systemctl start qemu-guest-agent
```

setenforce是临时关闭SELinux，下文有说明。

在宿主机上测试功能：

```sh
virsh  qemu-agent-command vm_name '{"execute":"guest-info"}'|python -m json.tool
```

能够返回正确的json则说明功能OK。
```
{
    "return": {
        "supported_commands": [
            {
                "enabled": true,
                "name": "guest-set-watchdog-timeout",
                "success-response": true
            },
            {
                "enabled": true,
                "name": "guest-get-osinfo",
                "success-response": true
            },
            {
                "enabled": true,
                "name": "guest-get-timezone",
                "success-response": true
            },

        ...
}

```
### 4.注意点

1. 更改虚拟机中xml中的name 

   在虚拟机中可以看到qemu模拟的设备

   ```sh
   ll /dev/virtio-ports/
   org.qemu.guest_agent.0 -> ../vport0p1
   ```

   查看qemu-guest-agent服务的配置文件可以看到：

   ```
   [Unit]
   Description=QEMU Guest Agent
   BindsTo=dev-virtio\x2dports-org.qemu.guest_agent.0.device
   After=dev-virtio\x2dports-org.qemu.guest_agent.0.device
   IgnoreOnIsolate=True
   
   [Service]
   UMask=0077
   EnvironmentFile=/etc/sysconfig/qemu-ga
   ExecStart=/usr/bin/qemu-ga \
     --method=virtio-serial \
     --path=/dev/virtio-ports/org.qemu.guest_agent.0 \
     --blacklist=${BLACKLIST_RPC} \
     -F${FSFREEZE_HOOK_PATHNAME}
   StandardError=syslog
   Restart=always
   RestartSec=0
   
   [Install]
   WantedBy=dev-virtio\x2dports-org.qemu.guest_agent.0.device
   ```

   此服务默认的device都是org.qemu.guest_agent.0，当在xml使用的name为其他值时，应当修改此服务配置，然后重启服务。

   

2. 关于qemu-guest-agent的文件系统权限问题

   在SELinux中，程序对于文件的读写要求较为严格，qemu-guest-agent中的有些操作或者自己开发的接口牵扯到写文件，查看

   /var/log/messages以及/var/log/audit/audit.log可以看到进程的权限缺失问题。

   需要暂时关闭seliunx或者针对此程序放开权限。下面是永久关闭SELinux的方式（其他方式可参考底部链接）：

    vim  /etc/sysconfig/selinux

   SELINUX=disabled

   重启生效

   


参考文档：

[qemu-guest-agent简介](https://www.cnblogs.com/nineep/p/9469942.html)

[linux中的selinux](https://blog.csdn.net/yanjun821126/article/details/80828908) 