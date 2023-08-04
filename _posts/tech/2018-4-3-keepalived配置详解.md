---
layout: post
title: keepalived 配置详解
category: HA
description: keepalived 配置详解
---
# keepalived 配置详解#

### 1.**keepalived 简介**

​Keepalived软件起初是专为LVS负载均衡软件设计的，用来管理并监控LVS集群系统中各个服务节点的状态，后来又加入了可以实现高可用的VRRP功能。因此，Keepalived除了能够管理LVS软件外，还可以作为其他服务（例如：Nginx、Haproxy、MySQL等）的高可用解决方案软件。

Keepalived软件主要是通过VRRP协议实现高可用功能的。VRRP出现的目的就是为了解决静态路由单点故障问题的，它能够保证当个别节点宕机时，整个网络可以不间断地运行。

因此Keepalived主要包含两个功能：

​	管理LVS负载均衡软件

​	作为系统网络服务的高可用性（failover）

本文重点节点高可用功能相关的配置项（监控LVS集群系统没有用到，暂时挂起）。

### 2.**keepalived 高可用性原理**

keepalived是以VRRP协议为实现基础的，VRRP全称Virtual Router Redundancy Protocol，即虚拟路由冗余协议。

Keepalived高可用对之间是通过VRRP通信的，因此，我们从 VRRP开始了解起：

　　　　1) VRRP,全称 Virtual Router Redundancy Protocol,中文名为虚拟路由冗余协议，VRRP的出现是为了解决静态路由的单点故障。

　　　　2) VRRP是通过一种竟选协议机制来将路由任务交给某台 VRRP路由器的。

　　　　3) VRRP用 IP多播的方式（默认多播地址（224.0_0.18))实现高可用对之间通信。

　　　　4) 工作时主节点发包，备节点接包，当备节点接收不到主节点发的数据包的时候，就启动接管程序接管主节点的开源。备节点可以有多个，通过优先级竞选，但一般 Keepalived系统运维工作中都是一对。

　　　　5) VRRP使用了加密协议加密数据，但Keepalived官方目前还是推荐用明文的方式配置认证类型和密码。

Keepalived服务的工作原理：

Keepalived高可用对之间是通过 VRRP进行通信的， VRRP通过竞选机制来确定主备的，主的优先级高于备，因此，工作时主会优先获得所有的资源，备节点处于等待状态，当主挂了的时候，备节点就会接管主节点的资源，然后顶替主节点对外提供服务。

在 Keepalived服务对之间，只有作为主的服务器会一直发送 VRRP广播包,告诉备它还活着，此时备不会枪占主，当主不可用时，即备监听不到主发送的广播包时，就会启动相关服务接管资源，保证业务的连续性.接管速度最快可以小于1秒。

### 3.**keepalived安装**

keepalived的安装比较简单。

```
yum install keepalived -y
```

或者下载源代码进行编译安装

```
curl --progress http://keepalived.org/software/keepalived-1.2.15.tar.gz | tar xz
cd keepalived-1.2.15
./configure --prefix=/usr/local/keepalived-1.2.15
make
sudo make install
```

### 4.**keepalived配置文件**

keepalived有三类配置区域，是一个配置文件里面三种不同类别的配置区域，分别是：
    1. 全局配置(Global Configuration)
    2. VRRPD配置
    3. LVS配置

- 全局配置

  ```
  global_defs {
     notification_email {
       acassen@firewall.loc
     } #定义报警邮件地址
     notification_email_from Alexandre.Cassen@firewall.loc #定义发送邮件的地址
     smtp_server 192.168.200.1 #邮箱服务器
     smtp_connect_timeout 30 #定义超时时间
     router_id LVS_DEVEL #定义路由标识信息，相同局域网唯一
     vrrp_skip_check_adv_addr # 检查vrrp报文中的所有地址比较耗时，设置此标志的意思是如果接收的到报文和上一个报文来至同一个路由器，则不执行检查。默认是跳过检查
     vrrp_strict #严格遵守vrrp协议，下面这些功能将会禁止：1.   0 VIP   2. unicast(单播) peers    3. vrrp 版本2的ipv6功能
     vrrp_garp_interval 0 #小数类型，单位秒，在一个网卡上每组gratuitous arp消息之间的延迟时间，默认为0，一个发送的消息=n组 arp报文
     vrrp_gna_interval 0 #小数类型，单位秒， 在一个网卡上每组na消息之间的延迟时间，默认为0
     vrrp_garp_master_delay 10 #设置当keepalived转变为master后，延迟多少秒发送第二组gratuitous arp。时间单位为秒，默认5秒，0表示不发送第二组gratuitous arp发送。
     vrrp_garp_master_repeat 1 #keepalived状态转变为master后，每次发送多少组grntuitous APR 信息的数量，默认为5个
     vrrp_garp_lower_prio_delay 10 #当MASTER收到更低优先级的通告时，延迟多少秒发送第二组的免费ARP。
     vrrp_garp_lower_prio_repeat 1 #当master keepalived接收到一个较低优先级的广播后，一次发送gratuitous apr的数量组
     vrrp_garp_master_refresh 60 #master keepalived 每次发送gratuitous arp的最小时间间隔。默认是0，没有
     vrrp_garp_master_refresh_repeat 2 #master keepalived 每次发送gratuitous arp消息的组数量。
  }
  ```
  全局配置中有些配置可以省略，示例为：

  ```
  global_defs {
    vrrp_garp_master_delay 10
    vrrp_garp_master_repeat 1
    vrrp_garp_lower_prio_delay 10
    vrrp_garp_lower_prio_repeat 1
    vrrp_garp_master_refresh 60
    vrrp_garp_master_refresh_repeat 2
    vrrp_garp_interval 0.001
    vrrp_gna_interval 0.000001
  }
  ```


- VRRPD配置

  VRRPD的配置介绍如下子块：

  1.vrrp_script

  2.vrrp_instance

   下面依次做介绍：

  - vrrp_script

     首先说明为什么需要script，之前一直不理解，既然keepalived保证了master与slave的存活选举，在某一时段master vip肯定只在某个节点上，为什么还需要自己来编写脚本控制相关的服务那？

     举个例子，你的软件服务在两台节点上提供服务，如果在某一时间master所在节点的软件服务挂掉了，由于所有的请求都是到master vip上，所以导致你的服务完全不可用。

     keepalived只能做到对网络故障和keepalived本身的监控，即当出现网络故障或者keepalived本身出现问题时，进行切换。但是这些还不够，我们还需要监控keepalived所在服务器上的其他业务进程，根据业务进程的运行状态决定是否需要进行主备切换。这个时候，我们可以通过编写脚本对业务进程进行检测监控。

    ```
    作用：添加一个周期性执行的脚本。脚本的退出状态码会被调用它的所有的VRRP Instance记录。
    注意：至少有一个VRRP实例调用它并且优先级不能为0.优先级范围是1-254.
    vrrp_script <SCRIPT_NAME> {
              ...
        }

    选项说明：
    scrip "/path/to/somewhere"：指定要执行的脚本的路径。
    interval <INTEGER>：指定脚本执行的间隔。单位是秒。默认为1s。
    timeout <INTEGER>：指定在多少秒后，脚本被认为执行失败。
    weight <-254 --- 254>：调整优先级。默认为2.
    rise <INTEGER>：执行成功多少次才认为是成功。
    fall <INTEGER>：执行失败多少次才认为失败。
    user <USERNAME> [GROUPNAME]：运行脚本的用户和组。
    init_fail：假设脚本初始状态是失败状态。

    解释：
    weight： 
    1. 如果脚本执行成功(退出状态码为0)，weight大于0，则priority增加。
    2. 如果脚本执行失败(退出状态码为非0)，weight小于0，则priority减少。
    3. 其他情况下，priority不变
    ```

  - vrrp_instance

    需要注意的是，组播与单播，在有些场景下，例如防火墙不需要组播的情况下，请使用单播，否则节点直接不能通信会造成多主vip。

    ```
    命令说明：
    state MASTER|BACKUP：指定该keepalived节点的初始状态。
    interface eth0：vrrp实例绑定的接口，用于发送VRRP包。
    use_vmac [<VMAC_INTERFACE>]：在指定的接口产生一个子接口，如vrrp.51，该接口的MAC地址为组播地址，通过该接口向外发送和接收VRRP包。
    vmac_xmit_base：通过基本接口向外发送和接收VRRP数据包，而不是通过VMAC接口。
    native_ipv6：强制VRRP实例使用IPV6.(当同时配置了IPV4和IPV6的时候)
    dont_track_primary：忽略VRRP接口的错误，默认是没有配置的。

    track_interface {
      eth0
      eth1 weight <-254-254>
      ...
    }：如果track的接口有任何一个出现故障，都会进入FAULT状态。

    track_script {
      <SCRIPT_NAME>
      <SCRIPT_NAME> weight <-254-254>
    }：添加一个track脚本(vrrp_script配置的脚本。)

    mcast_src_ip <IPADDR>：指定发送组播数据包的源IP地址。默认是绑定VRRP实例的接口的主IP地址。
    unicast_src_ip <IPADDR>：指定发送单播数据包的源IP地址。默认是绑定VRRP实例的接口的主IP地址。
    version 2|3：指定该实例所使用的VRRP版本。

    unicast_peer {
       <IPADDR>
       ...
    }：采用单播的方式发送VRRP通告，指定单播邻居的IP地址。

    virtual_router_id 51：指定VRRP实例ID，范围是0-255.
    priority 100：指定优先级，优先级高的将成为MASTER。
    advert_int 1：指定发送VRRP通告的间隔。单位是秒。
    authentication {
      auth_type PASS|AH：指定认证方式。PASS简单密码认证(推荐),AH:IPSEC认证(不推荐)。
      auth_pass 1234：指定认证所使用的密码。最多8位。
    }

    virtual_ipaddress {
       <IPADDR>/<MASK> brd <IPADDR> dev <STRING> scope <SCOPE> label <LABEL>
       192.168.200.17/24 dev eth1
       192.168.200.18/24 dev eth2 label eth2:1
    }：指定VIP地址。

    nopreempt：设置为不抢占。默认是抢占的，当高优先级的机器恢复后，会抢占低优先级的机器成为MASTER，而不抢占，则允许低优先级的机器继续成为MASTER，即使高优先级的机器已经上线。如果要使用这个功能，则初始化状态必须为BACKUP。使用该功能，最好是节点初始化是都设置为BACKUP

    preempt_delay：设置抢占延迟。单位是秒，范围是0---1000，默认是0.发现低优先级的MASTER后多少秒开始抢占。
    通知脚本：
    notify_master <STRING>|<QUOTED-STRING> [username [groupname]]
    notify_backup <STRING>|<QUOTED-STRING> [username [groupname]]
    notify_fault <STRING>|<QUOTED-STRING> [username [groupname]]
    notify <STRING>|<QUOTED-STRING> [username [groupname]]

    # 当停止VRRP时执行的脚本。
    notify_stop <STRING>|<QUOTED-STRING> [username [groupname]]
    ```
    示例为：

    ```
    vrrp_script check_grafana {
      script "/etc/keepalived/script/check_grafana_service.sh"
      interval 30
      weight 2
    }

    vrrp_instance grafana_internal {
      state BACKUP
      interface mgnt
      virtual_router_id 241
      unicast_src_ip 192.168.1.31
      unicast_peer {
        192.168.1.32
        192.168.1.33
                }
      priority 100
      nopreempt
      advert_int 1
      virtual_ipaddress {
        192.168.1.30
      }
      track_script {
        check_grafana
      }
    }
    ```
    脚本示例：
    ```
    #!/bin/bash
    ###############################################################
    ## Running by root, Install on keepalived server
    ## Function : keepalived script of checking vip services
    ## For All of Linux
    ###############################################################

    DATE=`date '+%Y-%m-%d %T'`
    SERVICE=grafana
    CONF_FILE="/etc/keepalived/config/keepalived_grafana.conf"
    VIP_EXIST=`ip a|grep 192.168.1.30|wc -l`
    GRAFANA_SERVICE=`ps ax|grep grafana-server|grep -v grep|wc -l`
    ZOMBIE=`ps ax|grep grafana-server|grep -v 'grep'|awk '{print $3}'`

    CHECK_VIP()
    {
        if [ $VIP_EXIST -eq 1 ] && [ $GRAFANA_SERVICE -eq 1 ]; then 
            echo "## $DATE : [DEBUG] 11 VIP and grafana are running"
            #if [ $ZOMBIE != 'Ssl' ] || [[ ! $ZOMBIE =~ 'D|R' ]]; then
            #    systemctl restart grafana-server
            #fi
        elif [ $VIP_EXIST -eq 1 ] && [ $GRAFANA_SERVICE -eq 0 ]; then
            echo "## $DATE : [WARN] 10 VIP is running, but grafana-server is down , restart grafana-server..."
            systemctl restart grafana-server
        elif [ $VIP_EXIST -eq 0 ] && [ $GRAFANA_SERVICE -eq 1 ]; then
            echo "## $DATE : [WARN] 01 VIP is down, but grafana is running, shutdown grafana-server..."
            systemctl stop grafana-server
        elif [ $VIP_EXIST -eq 0 ] && [ $GRAFANA_SERVICE -eq 0 ]; then
            echo "## $DATE : [DEBUG] 00 VIP and grafana are not running"
        else
            echo "what the fuck....."
        fi
           
    } >> /var/log/keepalived_grafana.log

    CHECK_VIP

    ```

- LVS配置

  //todo 

  暂未使用...





参考文档：

[keepalived](https://blog.csdn.net/wos1002/article/details/56483325)

[keepalived实现服务高可用](https://www.cnblogs.com/clsn/p/8052649.html) 