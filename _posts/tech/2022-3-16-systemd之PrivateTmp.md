---
layout: post
title: systemd之PrivateTmp
category: VMM
tags: VMM
description: systemd之PrivateTmp
---
#  systemd之PrivateTmp

主机上根目录占用满，发现是/tmp目录下某个文件夹（mariadb）占用空间大，但是文件夹命名比较奇怪：

```sh
ll
...
rwx------  3 root root    4096 Mar 15 20:26 systemd-private-e153918a94a74296867771449bc394a8-mariadb.service-mPLdEr
drwx------  3 root root    4096 Mar 15 21:29 systemd-private-e153918a94a74296867771449bc394a8-neutron-openvswitch-agent.service-Z6DIf6
drwx------  3 root root    4096 Mar 15 23:55 systemd-private-e153918a94a74296867771449bc394a8-neutron-server.service-jNN2Cp
...
```

### 问题环境

主机：CentOS Linux release 7.7.1908 (Core)

### 问题分析

文件夹命令遵循一定的规则，因此不像是人为创建，由于是systemd开头，所以还是从systemd入手查询相关的资料，得知：

**托管于systemd的服务启用了安全Tmp系统功能。**

```
 vim /usr/lib/systemd/system/mariadb.service
 ...
 UMask=007

# Give a reasonable amount of time for the server to start up/shut down
TimeoutSec=600

# Place temp files in a secure directory, not /tmp
PrivateTmp=true
LimitNOFILE=65535
LimitNPROC=65535
...
```

在service配置文件中可以看到PrivateTmp=true此选项，表示开启了安全Tmp系统功能。

### PrivateTmp

用英文看比较明了

```
--PrivateTmp
Takes a boolean argument. If true, sets up a new file system namespace for the executed processes and mounts private /tmp/ and /var/tmp/ directories inside it that are not shared by processes outside of the namespace.This is useful to secure access to temporary files of the process, but makes sharing between processes via /tmp/ or /var/tmp/ impossible. If true, all temporary files created by a service in these directories will be removed after the service is stopped.
```

#### 作用

1、/tmp目录以及/var/tmp目录所有进程都在公用，不够安全，使用PrivateTmp后，进程用于自己的独立的目录以及相应的权限

2、关于目录的管理托管于systemd，即当systemd进程启动时会建立相应的目录（目录会在两个地方建立，/tmp以及/var/tmp/下建立两个目录），当通用systemd进程关闭时会删除相应的目录，不用程序单独处理

#### 原理

在上述说明中也可以看到，开启选项后，会为进程创造单独的namespace，然后会把/tmp/以及/var/tmp/挂载到新建的目录中。即开启mnt命名空间，然后执行挂载。

```sh
cd /proc/41252（主进程号）
cat mountinfo |grep tmp
...
165 131 8:2 /tmp/systemd-private-e153918a94a74296867771449bc394a8-mariadb.service-mPLdEr/tmp /tmp rw,relatime shared:145 master:1 - ext4 /dev/sda2 rw,data=ordered
166 131 8:2 /var/tmp/systemd-private-e153918a94a74296867771449bc394a8-mariadb.service-k8LOj8/tmp /var/tmp rw,relatime shared:146 master:1 - ext4 /dev/sda2 rw,data=ordered
...
```

当然也可以用nsenter登陆验证

```
nsenter -t 41252 -m
//可以看到此处只能看到自己的挂载点的文件，而不是主机上的/tmp，/var/tmp同样如此
cd /tmp
```

#### 程序使用

由于使用了命名空间挂载，所以对于程序本身来说么不需要更改任何东西，但是对于其他进程来说，如果要读取改进程的临时文件，则需要注意临时文件的路径，而不是直接读取/tmp或者/var/tmp下目录，而是主机上systemd新建的目录

### tmpfiles.d

其实/tmp下目录并不是永久存储的，有可能会被删除，那么PrivateTmp会不会被系统自动删除（非进程本身），从而影响正在运行的进程？这牵扯到tmpfiles.d（临时文件管理系统）。

`tmpfiles.d` 配置文件定义了一套临时文件管理机制： *创建* 文件、目录、管道、设备节点， *调整* 访问模式、所有者、属性、限额、内容， *删除* 过期文件。 主要用于管理易变的临时文件与目录，例如 `/run`, `/tmp`, `/var/tmp`, `/sys`, `/proc`, 以及 `/var` 下面的某些目录。

**systemd-tmpfiles** 根据这些配置， 在系统启动过程中创建易变的临时文件与目录，并在系统运行过程中进行周期性的清理。牵扯 `systemd-tmpfiles-setup.service`, `systemd-tmpfiles-cleanup.service` 等相关服务。

#### **systemd-tmpfiles-clean.service服务**

服务何时被执行呢？

Linux下该服务的执行可以根据systemd-tmpfiles-clean.timer进行管理

```
[root@sam01 ~]# cat /usr/lib/systemd/system/systemd-tmpfiles-clean.timer
#  This file is part of systemd.
#
#  systemd is free software; you can redistribute it and/or modify it
#  under the terms of the GNU Lesser General Public License as published by
#  the Free Software Foundation; either version 2.1 of the License, or
#  (at your option) any later version.

[Unit]
Description=Daily Cleanup of Temporary Directories
Documentation=man:tmpfiles.d(5) man:systemd-tmpfiles(8)

[Timer]
OnBootSec=15min
OnUnitActiveSec=1d

# OnBootSec 表示相对于机器被启动的时间点
# 表示相对于匹配单元(本标签下Unit=指定的单元)最后一次被启动的时间点
```

上述配置文件表示两种情况会执行该服务

1. 开机15分钟执行服务
2. 距离上次执行该服务1天后执行服务

 

服务如何执行呢？

```
[root@sam01 ~]# cat /usr/lib/systemd/system/systemd-tmpfiles-clean.service
#  This file is part of systemd.
#
#  systemd is free software; you can redistribute it and/or modify it
#  under the terms of the GNU Lesser General Public License as published by
#  the Free Software Foundation; either version 2.1 of the License, or
#  (at your option) any later version.

[Unit]
Description=Cleanup of Temporary Directories
Documentation=man:tmpfiles.d(5) man:systemd-tmpfiles(8)
DefaultDependencies=no
Conflicts=shutdown.target
After=systemd-readahead-collect.service systemd-readahead-replay.service local-fs.target time-sync.target
Before=shutdown.target

[Service]
Type=oneshot
ExecStart=/usr/bin/systemd-tmpfiles --clean
IOSchedulingClass=idle

# Type=oneshot 这一选项适用于只执行一项任务、随后立即退出的服务
# 命令文件 /usr/bin/systemd-tmpfiles
# 命令参数 --clean
# 通过定期执行 /usr/bin/systemd-tmpfiles --clean 完成清理
```

命令： /usr/bin/systemd-tmpfiles

```
[root@cdpm03 41252]# /usr/bin/systemd-tmpfiles --help
systemd-tmpfiles [OPTIONS...] [CONFIGURATION FILE...]

Creates, deletes and cleans up volatile and temporary files and directories.

  -h --help                 Show this help
     --version              Show package version
     --create               Create marked files/directories
     --clean                Clean up marked directories
     --remove               Remove marked files/directories
     --boot                 Execute actions only safe at boot
     --prefix=PATH          Only apply rules with the specified prefix
     --exclude-prefix=PATH  Ignore rules with the specified prefix
     --root=PATH            Operate on an alternate filesystem root

```

哪些目录被标记，又是什么样的标记呢？

这些都在配置文件中定义：对于不同目录下的同名配置文件，仅以优先级最高的目录中的那一个为准。具体说来就是： `/etc/tmpfiles.d` 的优先级最高、 `/run/tmpfiles.d` 的优先级居中、 `/usr/lib/tmpfiles.d` 的优先级最低。 软件包应该将自带的配置文件安装在 `/usr/lib/tmpfiles.d` 目录中， 而 `/etc/tmpfiles.d` 目录仅供系统管理员使用。

#### /tmp以及/var/tmp管理

/tmp以及/var/tmp的目录规则定义在 /usr/lib/tmpfiles.d/tmp.conf中：

```
vim  /usr/lib/tmpfiles.d/tmp.conf
#  This file is part of systemd.
#
#  systemd is free software; you can redistribute it and/or modify it
#  under the terms of the GNU Lesser General Public License as published by
#  the Free Software Foundation; either version 2.1 of the License, or
#  (at your option) any later version.

# See tmpfiles.d(5) for details

# Clear tmp directories separately, to make them easier to override
# 类型, 路径, 权限, 属主, 属组, 寿命, 参数
v /tmp 1777 root root 10d
v /var/tmp 1777 root root 30d

# Exclude namespace mountpoints created with PrivateTmp=yes
x /tmp/systemd-private-%b-*
X /tmp/systemd-private-%b-*/tmp
x /var/tmp/systemd-private-%b-*
X /var/tmp/systemd-private-%b-*/tmp


```

上述配置表示：

1. 清理/tmp目录超过10天的内容，但是匹配/tmp/systemd-private-%b-*的目录及其路径下的全部内容会被保留
2. 清理/var/tmp目录超过30天的内容，但是匹配/var/tmp/systemd-private-%b-*的目录及其路径下的全部内容被保留

即排除了systemd进程创建的目录，由systemd自己来管理，从而避免了误删除正在运行程序的临时文件夹。

#### 过期时间

"寿命"是根据对象的最后修改时间(mtime)、 最后访问时间(atime)、 最后状态变化时间(ctime)(目录除外) 计算的。 如果三者(或两者)中最晚的时间与当前系统时间之差大于"寿命"字段的值， 那么该对象就会被删除， 否则该对象将会被保留。

#### 权限问题

清理临时文件中文件其实有个小小的bug，即所有清理的文件root用户必须有w权限，否则不能删除

```sh
cd /tmp
ll
-r-xrwxrwx 1 root root  0 Mar  8 04:00 1.txt
/usr/bin/systemd-tmpfiles --clean
ll
-r-xrwxrwx 1 root root  0 Mar  8 04:00 1.txt
```



参考文档

[tmpfiles.d 中文手册](http://www.jinbuguo.com/systemd/tmpfiles.d.html)

[Linux下关于/tmp目录的清理规则](https://www.cnblogs.com/samtech/p/9490166.html)



