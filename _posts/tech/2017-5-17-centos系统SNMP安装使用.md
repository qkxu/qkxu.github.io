---
layout: post
title: centos系统SNMP安装使用
category: telemetry
description: centos系统SNMP安装使用
---
# centos系统SNMP安装使用#

### 1.**SNMP介绍**

​        SNMP(Simple Network Management Protocol,简单网络管理协议)的前身是简单网关监控协议(SGMP)，用来对通信线路进行管理。随后，人们对SGMP进行了很大的修改，特别是加入了符合Internet定义的SMI和MIB：体系结构，改进后的协议就是著名的SNMP。SNMP的目标是管理互联网Internet上众多厂家生产的软硬件平台，因此SNMP受Internet标准网络管理框架的影响也很大。现在SNMP已经出到第三个版本的协议，其功能较以前已经大大地加强和改进了。安装的snmp都是支持3个版本的协议，但是默认使用的是v1与v2协议，v3协议需要配置相应的user以及秘钥等

### 2.**安装SNMP**
  ```
  centos、redhat命令：yum install net-snmp* -y
  ```
  当前版本用的是5.7.2。
### 3.**配置SNMP**
- 设置开机启动

    ```
    centos命令： systemctl enable snmpd.service
    ```

- 配置snmp文件

  以上安装完成后，使用的是snmp的默认配置，通过这些默认配置，我们只能获取主机的部分信息。但一些其他的重要信息，无法获取。如主机的CPU使用情况，内存使用情况等。

  配置方法：修改snmp的配置文件snmpd.conf（不同的操作系统，可能所在的配置文件不同）
  ```
  vi /etc/snmp/snmpd.conf
  ```
  找到以下信息

  ```
  # Third, create a view for us to let the group have rights to:

  # Make at least  snmpwalk -v 1 localhost -c public system fast again.
  #       name           incl/excl     subtree         mask(optional)
  view    systemview    included   .1.3.6.1.2.1.1
  view    systemview    included   .1.3.6.1.2.1.25.1.1
  ```
  添加一行
  ```
  ####
  # Third, create a view for us to let the group have rights to:

  # Make at least  snmpwalk -v 1 localhost -c public system fast again.
  #       name           incl/excl     subtree         mask(optional)
  view    systemview    included   .1
  view    systemview    included   .1.3.6.1.2.1.1
  view    systemview    included   .1.3.6.1.2.1.25.1.1
  ####
  ```
  view：定义了可以查看哪些节点设备的信息。snmp默认配置只能查看.1.3.6.1.2.1.1和.1.3.6.1.2.1.25.1.1节点下的设备信息，而主机CPU和内存等设备都不在这些节点下，所以无法获取这些数据。 view    systemview    included   .1 表示可以查看.1节点下的所有设备信息。

  另外如果要得到磁盘设备数据要修改配置：
  ```
  # disk PATH [MIN=100000]
  #
  # PATH:  mount path to the disk in question.
  # MIN:   Disks with space below this value will have the Mib's errorFlag set.
  #        Default value = 100000.

  # Check the / partition and make sure it contains at least 10 megs.

  #disk / 10000  将此行的注释去掉即可

  #注意这个配置只是个示例，用于标注采集磁盘的PATH，如果有多个挂载盘，可以配置多个
  #例如 disk /dev 10000等
  #includeAllDisks 10% 表示采集所有路径，其中10%是表示至少有10%的空余空间
  ```

- 启动snmp服务

  ```
  centos命令： systemctl start snmpd.service
  ```
- 测试snmp服务
  ```
  本机执行：
  snmpwalk -v 2c -c public 127.0.0.1 .1.3.6.1.4.1.2021.4.5.0
  其他主机：
  snmpwalk -v 2c -c public snmp主机ip .1.3.6.1.4.1.2021.4.5.0
  查看是否有数据即可
  如果出现 No more variables left in this MIB View类似错误，
  说明view    systemview    included   .1没有添加正确，仔细查看是否配置正确
  ```

### 3.**配置SNMP（V3协议）**
经过上述配置，snmp就可以使用v2协议提供服务了，如果对于v3协议没有需求，后面的内容就可以略过。

SNMP V3协议由RFC 3411-RFC 3418定义，主要增加SNMP在安全性和远端配置方面的强化。

SNMP第三版提供重要的安全性功能：
- 信息完整性：保证封包在传送中没有被篡改
- 认证：检验信息来自正确的来源。
- 封包加密：避免被未授权的来源窥探。


SNMPv3中引入了下列三个安全级别：
- noAuthNoPriv：不需要认证，不提供隐私性（加密）。
- authNoPriv：基于HMAC-MD5或HMAC-SHA的认证，不提供加密。
- authPriv：除了认证之外，还将CBC-DES加密算法用作隐私性协议。


对于V3协议需要创建用户，有两种方式：
- net-snmp-config工具来创建（也是推荐方式）

  注：这个工具并不在net-snmp与 net-snmp-utils中，而是在net-snmp-devel中，如果安装时执行的是

  yum install net-snmp* -y则已经包含，无需额外安装，否则要单独安装下

  执行以下命令（需要关闭snmp服务）：

  ```
  net-snmp-config --create-snmpv3-user --help
  ```
  可以看到（其中authpass为认证密码，与-a配合使用，-X为加密密码，与-x配置使用，密码都必须大于8位）
  ```
  Usage:
    net-snmp-create-v3-user [-ro] [-A authpass] [-X privpass]
                            [-a MD5|SHA] [-x DES|AES] [username]
  ```
  例如创建AuthNoPriv级别的用户
  ```
  net-snmp-config --create-snmpv3-user -ro -A test123456 -a MD5 userAuthNoPriv
  回显为：
  adding the following line to /var/lib/net-snmp/snmpd.conf:
     createUser userAuthNoPriv MD5 "test123456" DES
  adding the following line to /etc/snmp/snmpd.conf:
     rouser userAuthNoPriv
  ```
  例如创建AuthPriv级别的用户
  ```
  net-snmp-config --create-snmpv3-user -ro -A test123456 -X test123456 -a MD5 -x DES userAuthPriv
  ```

- 直接修改snmp.conf文件（/var/lib/net-snmp/snmpd.conf以及/etc/snmp/snmpd.conf），在文件末尾追加即可

  上述两种用户则对应的是：

  ```
  /var/lib/net-snmp/snmpd.conf:
  createUser userAuthNoPriv MD5 "test123456"
  /etc/snmp/snmpd.conf:
  rouser userAuthNoPriv
  /var/lib/net-snmp/snmpd.conf:
  createUser userAuthPriv MD5 "test123456" DES "test123456"
  /etc/snmp/snmpd.conf：
  rouser userAuthPriv
  ```

  创建noAuthNoPriv级别的用户（net-snmp-config工具并没有提供创建此级别的用户）
  ```
  /var/lib/net-snmp/snmpd.conf:
  createUser usernoAuthNoPriv
  /etc/snmp/snmpd.conf：
  rouser testsnmpuser noauth
  ```
  更多的参数详情以及配置可以用 man snmpd.conf看到。

以上两种方式都需要完成配置后重启snmp服务。

测试snmpV3协议：
```
AuthNoPriv级别：
snmpwalk -v 3 -a MD5 -u userAuthNoPriv -A test123456 -l authNoPriv 127.0.0.1 .1.3.6.1.4.1.2021.4.5.0
AuthPriv级别：
snmpwalk -v 3 -a MD5 -u userAuthPriv -A test123456 -l authPriv -x DES -X test123456 127.0.0.1 .1.3.6.1.4.1.2021.4.5.0
其中参数含义如下：
SNMP Version 3 specific
  -a PROTOCOL		set authentication protocol (MD5|SHA)
  -A PASSPHRASE		set authentication protocol pass phrase
  -l LEVEL		set security level (noAuthNoPriv|authNoPriv|authPriv)
  -u USER-NAME		set security name (e.g. bert)
  -x PROTOCOL		set privacy protocol (DES|AES)
  -X PASSPHRASE		set privacy protocol pass phrase
```

### 4.**关闭v1，v2协议版本**

如果使用了v3用户，安全起见则可以关闭v1，v2协议版本，关闭方法：

```
####
# First, map the community name "public" into a "security name"

#       sec.name  source          community
com2sec notConfigUser  default       public 

####
# Second, map the security name into a group name:

#       groupName      securityModel securityName
group   notConfigGroup v1           notConfigUser
group   notConfigGroup v2c           notConfigUser

####
# Third, create a view for us to let the group have rights to:
```

修改为（即将某几行修改为注释）
```
####
# First, map the community name "public" into a "security name"

#       sec.name  source          community
#com2sec notConfigUser  default       public

####
# Second, map the security name into a group name:

#       groupName      securityModel securityName
#group   notConfigGroup v1           notConfigUser
#group   notConfigGroup v2c           notConfigUser

####
# Third, create a view for us to let the group have rights to:

```
重启服务生效

### 5.**放开161端口**

snmp协议使用的是UDP协议，占用的端口为161，如果防火墙没有放行此端口，会导致远端节点通过snmp协议获取不到数据（远程节点没有响应）。

命令行方式：

1.开放端口命令： /sbin/iptables -I INPUT -p udp --dport 161 -j ACCEPT

2.保存：/etc/rc.d/init.d/iptables save

3.重启服务：/etc/init.d/iptables restart

4.查看端口是否开放：/sbin/iptables -L -n

