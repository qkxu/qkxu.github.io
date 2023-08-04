---
layout: post
title: linux下的watchdog
category: VMM
description: linux下的watchdog
---
#  linux下的watchdog

### 1.**Linux对Watchdog的支持**

1. watchdog的基本工作原理

   在Linux 内核下, watchdog的基本工作原理是：当watchdog启动后(即/dev/watchdog 设备被打开后)，如果在某一设定的时间间隔内/dev/watchdog没有被执行写操作, 硬件watchdog电路或软件定时器就会重新启动系统。

   /dev/watchdog 是一个主设备号为10， 从设备号130的字符设备节点。 Linux内核不仅为各种不同类型的watchdog硬件电路提供了驱动，还提供了一个基于定时器的纯软件watchdog驱动。 驱动源码位于内核源码树drivers/char/watchdog/目录下。

2. 硬件与软件watchdog的区别

   1. 通常情况下，watchdog需要硬件支持，但是如果确实没有相应的硬件，还想使用watchdog功能，则可以使用liunx模拟的watchdog，即软件watchdog。

   2. 硬件watchdog必须有硬件电路支持, 设备节点/dev/watchdog对应着真实的物理设备， 不同类型的硬件watchdog设备由相应的硬件驱动管理。软件watchdog由一内核模块softdog.ko 通过定时器机制实现，/dev/watchdog并不对应着真实的物理设备，只是为应用提供了一个与操作硬件watchdog相同的接口。

   3. 硬件watchdog比软件watchdog有更好的可靠性。 软件watchdog基于内核的定时器实现，当内核或中断出现异常时，软件watchdog将会失效。而硬件watchdog由自身的硬件电路控制, 独立于内核。无论当前系统状态如何，硬件watchdog在设定的时间间隔内没有被执行写操作，仍会重新启动系统。

   4. 一些硬件watchdog卡如WDT501P 以及一些Berkshire卡还可以监测系统温度，提供了 /dev/temperature接口。

   5. 对于应用程序而言, 操作软件、硬件watchdog的方式基本相同：打开设备/dev/watchdog, 在重启时间间隔内对/dev/watchdog执行写操作。即软件、硬件watchdog对应用程序而言基本是透明的。

      在任一时刻， 只能有一个watchdog驱动模块被加载，管理/dev/watchdog 设备节点。如果系统没有硬件watchdog电路，则可以加载软件watchdog驱动softdog.ko。

3. 驱动配置

   不同的硬件watdog需要相应的内核驱动模块，在加载模块时有相应的参数，以i6300esb设备为例，驱动模块为i6300esb。

   ```
   [root@localhost ~]# lsmod |grep esb
   i6300esb               13566  0 
   [root@localhost ~]# modinfo i6300esb
   filename:       /lib/modules/3.10.0-862.el7.x86_64/kernel/drivers/watchdog/i6300esb.ko.xz
   alias:          char-major-10-130
   license:        GPL
   description:    Watchdog driver for Intel 6300ESB chipsets
   author:         Ross Biro and David Härdeman
   retpoline:      Y
   rhelversion:    7.5
   srcversion:     8296B12AC239294E2E67E67
   alias:          pci:v00008086d000025ABsv*sd*bc*sc*i*
   depends:        
   intree:         Y
   vermagic:       3.10.0-862.el7.x86_64 SMP mod_unload modversions 
   signer:         CentOS Linux kernel signing key
   sig_key:        3A:F3:CE:8A:74:69:6E:F1:BD:0F:37:E5:52:62:7B:71:09:E3:2B:96
   sig_hashalgo:   sha256
   parm:           heartbeat:Watchdog heartbeat in seconds. (1<heartbeat<2046, default=30) (int)
   parm:           nowayout:Watchdog cannot be stopped once started (default=0) (bool)
   ```

   heartbeat，该设备的心跳时间，即watchdog打开时的默认重启时间。
   nowayout，当为1时，表明该设备打开后，不能够停止，即不能被喂狗。

   当有硬件设备时，系统启动会加载相应的驱动模块，并采取默认值，如果想修改默认参数有两种方式:

   1. 重新加载model
      ```sh
      modprobe -r i6300esb
      modprobe i6300esb heartbeat=120
      ```
      这种方式时候是重新加载model，并设置新的参数，立即生效，重启失效

   2. 在/etc/modprobe.d/watchdog.conf（新增文件，名称只供参考）中，添加
      options  i6300esb heartbeat=120
      每次重启会加载自定义配置的model参数设置。

      


### 2.Watchdog的管理

watchdog管理牵扯到的基本操作包括，开启watchdog，喂狗（即写/dev/watchdog防止狗自动重启系统），设置重启时间间隔，关闭狗等。

不通的硬件驱动对于watchdog管理细节稍微不同，但是对编程而言基本提供了一致接口，细节部分可参考/drivers/watchdog/下的源代码。

```c
#include <linux/watchdog.h>

#define WDT_DEVICE_FILE "/dev/watchdog"

int main(void)
{
  int g_watchdog_fd = -1;
  int timeout = 0;
  int timeout_reset = 120;

  //开启watchdog
  g_watchdog_fd = open(WDT_DEVICE_FILE, O_RDWR);
  if (g_watchdog_fd == -1)
    {
        printf("Error in file open WDT device file(%s)...\n", WDT_DEVICE_FILE);

        return 0;
    }
  //获取watchdog的超时时间（heartbeat）
  ioctl(g_watchdog_fd, WDIOC_GETTIMEOUT, &timeout);
  printf("default timeout %d sec.\n", timeout);
  //设置watchdog的超时时间（heartbeat）
  ioctl(g_watchdog_fd, WDIOC_SETTIMEOUT, &timeout_reset);
  printf("We reset timeout as %d sec.\n", timeout_reset);
  //喂狗
  while(1){
      //喂狗
      ret = ioctl(g_watchdog_fd, WDIOC_KEEPALIVE, 0);
      //喂狗也通过写文件的方式，向/dev/watchdog写入字符或者数字等
      // static unsigned char food = 0;
      //write(g_watchdog_fd, &food, 1);
      if (ret != 0) {
          printf("Feed watchdog failed. \n");
          close(fd);
          return -1;
      } else {
          printf("Feed watchdog every %d seconds.\n", sleep_time);
      }
      //feed_watchdog_time是喂狗的时间间隔，要小于watchdog的超时时间
      sleep(feed_watchdog_time);
    }
  //关闭watchdog
  write(g_watchdog_fd, "V", 1);
  //以下方式实测并不能关闭watchdog
  //ioctl(g_watchdog_fd, WDIOC_SETOPTIONS, WDIOS_DISABLECARD)
  close(g_watchdog_fd);
}
```

注意点：

​    开启watchdog也可以用非正常方式，例如cat /dev/watchdog（会报错，只是开启，没有关闭。）

​    ioctl(g_watchdog_fd, WDIOC_SETOPTIONS, WDIOS_DISABLECARD)操作并不能关闭watchdog，或许是当前内核的bug。

​    写"V"的方式关闭watchdog，对于不同的驱动或许方式不同，本文实测i6300esb以及softdog均可，其他的硬件未测试，具体方式可以查看源代码。



### 3.**watchdog应用**

在liunx中有个watchdog RPM包，此软件既不是硬件狗也不是用来模拟硬件的软件狗，它是一个watchdog应用，用来喂狗以及其他的功能。

安装watchdog软件包：

```sh
yum install watchdog
```

修改watchdog的配置文件：

```sh
# With enforcing SELinux policy please use the /usr/libexec/watchdog/scripts/
# or /etc/watchdog.d/ for your test-binary and repair-binary configuration.
#repair-binary          = /usr/sbin/repair
#repair-timeout         = 
#test-binary            = 
#test-timeout           = 

#watchdog-device = /dev/watchdog

# Defaults compiled into the binary
#temperature-device     =
#max-temperature        = 120

# Defaults compiled into the binary
#admin                  = root
#interval               = 1
#logtick                = 1
#log-dir                = /var/log/watchdog
```



说明：

- watchdog-device：表示是否使用系统watchdog设备，启用后watchdog软件会管理硬件狗（开启、喂狗、停狗）

- watchdog-timeout：指定硬件狗（或者软件狗）的超时时间，默认情况下，watchdog应用会改变系统中狗的超时时间为60s

  此属性在默认配置中没有写，可以手动添加。

- interval：喂狗的默认时间周期

- test-binary：检测程序，与repair-binary配合使用，编写检测系统是否正常的程序，例如可以写一个检测网卡是否正常的程序。

- repair-binary ：与test-binary配合使用，当检测系统返回-1时，执行修复程序（默认是重启系统），例如当网卡down之后将网卡拉起来。

需要注意的是，实测watchdog-5.13-12，binary均不支持sh脚本，只支持二进制程序。

启动喂狗程序：

```sh
systemctl start watchdog
```

正常关闭喂狗程序时，也会关闭狗。

### 4. 虚拟化场景下的watchdog

libvirt支持为kvm/qemu客户机创建watchdog，用于当客户机内部crash时，自动会触发相应的action。

libvirt支持模拟以下几种watchdog：

- i6300esb  - 推荐的watchdog，模拟为一种pci设备，openstack层面只支持这一种（nova拼写xml中写死）。
- ib700  - 模拟为platform设备，xml中请勿分配pci设备号(不需要拼写<address\>)。
- diag288 - 模拟S390中的diag288设备，需要S390硬件支持。

action支持以下几种方式：

- disabled：不使用watchdog设备
- reset：强行重置虚拟机
- poweroff：强行关闭虚拟机
- pause：暂停虚拟机
- none：只是启用watchdog，在虚拟机hang住时不执行任何操作

创建相应的虚拟机，qemu会为虚拟机虚拟出i6300esb硬件狗设备，虚拟机内部可以看到相应的设备。

在虚拟机中安装watchdog软件，修改watchdog应用配置，启用watchdog软件（会打开watchdog），则当虚拟机内部crash时则会触发相应的动作。

当watchdog动作触发后，libvirt还会触发event事件来告知上层应用（例如nova），进而可以将此事件反馈给用户（例如发告警）。

参考文档：

[使用 watchdog 构建高可用性的 Linux 系统及应用](<https://www.ibm.com/developerworks/cn/linux/l-cn-watchdog/>)

[watchdog软件配置项说明](<http://www.sat.dundee.ac.uk/psc/watchdog/watchdog-configure.html>)

[LibvirtWatchdog](<https://wiki.openstack.org/wiki/LibvirtWatchdog>)