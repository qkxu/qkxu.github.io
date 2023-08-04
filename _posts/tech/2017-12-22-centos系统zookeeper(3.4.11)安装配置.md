---
layout: post
title: centos系统zookeeper安装配置
category: telemetry
description: centos系统zookeeper安装配置
---
# centos系统zookeeper安装配置#

### 1.**安装与配置**

zookeeper的安装网上有一堆资料，其实比较简单，zookeeper的安装分为三种模式：单机模式、集群模式和伪集群模式。

- **单机模式**

  从官网上下载安装包，或者手动下载

  ```
  wget https://mirrors.tuna.tsinghua.edu.cn/apache/zookeeper/zookeeper-3.4.11/zookeeper-3.4.11.tar.gz
  ```

  将安装包解压即可用，但是zookeeper要求java运行环境，并且需要jdk版本1.6以上，必须在环境上安装java。

  ```
  tar -zxvf zookeepre-3.4.11.tar.gz
  ```

  默认配置文件路径为 Zookeeper-3.4.11/conf/目录下，文件名为zoo.cfg。进入conf/目录下可以看到一个zoo_sample.cfg文件，可供参考。可以直接复制为zoo.cfg，然后做修改。

  ```
  copy zoo_sample.cfg zoo.cfg
  ```

  默认配置文件的内容为:

  ```
  # The number of milliseconds of each tick
  # 服务器与客户端之间交互的基本时间单元（ms）
  tickTime=2000
  # The number of ticks that the initial 
  # synchronization phase can take
  # 此配置表示允许follower连接并同步到leader的初始化时间，它以tickTime的倍数来表示。当超过设置倍数的tickTime时间，则连接失败。
  initLimit=10
  # The number of ticks that can pass between 
  # sending a request and getting an acknowledgement
  # Leader服务器与follower服务器之间信息同步允许的最大时间间隔，如果超过次间隔，默认follower服务器与leader服务器之间断开链接
  syncLimit=5
  # the directory where the snapshot is stored.
  # do not use /tmp for storage, /tmp here is just 
  # example sakes.
  # 保存zookeeper数据
  dataDir=/tmp/zookeeper
  # the port at which the clients will connect
  # 客户端访问zookeeper时经过服务器端时的端口号
  clientPort=2181
  # the maximum number of client connections.
  # increase this if you need to handle more clients
  # 支持的最大连接数
  #maxClientCnxns=60
  #
  # Be sure to read the maintenance section of the 
  # administrator guide before turning on autopurge.
  #
  # http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_maintenance
  #
  # The number of snapshots to retain in dataDir
  #autopurge.snapRetainCount=3
  # Purge task interval in hours
  # Set to "0" to disable auto purge feature
  #autopurge.purgeInterval=1
  ```

  基本的配置说明英文都有了，重点说下dataDir与dataLogDir这两个配置

  dataDir：保存zookeeper数据路径

  dataLogDir：这个配置在3.4.11中是没有的，看下zkEnv.sh的脚本发现这个配置是根据环境变量中设定读取的，因为不需要在conf中配置，即使配置也不起作用。可以在环境变量中设定起效，默认情况下会在运行zookeeper的当前目录生成zookeeper.out。

  配置文件更改完毕后，就可以启动zookeeper了（注意下端口不要被占用）

  启动脚本在Zookeeper-3.4.11/bin/中

  ```
  bash zkServer.sh start
  ```

  或者用绝对路径启动，例如

  ```
  /opt/zookeeper/zookeeper-3.4.11/bin/zkServer.sh start
  ```

- **集群模式**

  集群模式也是最常用的模式，集群模式有效的保证了容错性，即使一台或多台zookeeper服务器挂掉，整个集群也可以对外提供服务。因为牵扯到leader选取，最后采用奇数台服务器安装zookeeper。安装都是类似的，解压到特定目录即可。集群模式主要的是配置zookeeper之间连通性，然后启动zookeeper即可。

  配置如下：

    ```
    tickTime=2000
    initLimit=10
    syncLimit=5
    dataDir=/opt/zookeeper/zookeeper-3.4.11/data
    clientPort=2181
    maxClientCnxns=60
    server.1=10.127.2.121:2888:3888
    server.2=10.127.2.122:2888:3888
    server.3=10.127.2.123:2888:3888
    ```

  重点说下最后三行配置：
  最后三行提供了集群中所有服务器的信息，其中server.id=host:port:port表示了不同的zookeeper服务器的自身标识，作为集群的一部分，每一台服务器应该知道其他服务器的信息。

  其中id是指此台服务器的唯一标识，这整个集群中不能重复，这个id在服务器的data(dataDir参数所指定的目录)下创建一个文件名为myid的文件，这个文件的内容只有一行，指定的是自身的id值。比如，服务器“1”应该在myid文件中写入“1”。这个id必须在集群环境中服务器标识中是唯一的，且大小在1～255之间。这一样配置中，zoo1代表第一台服务器的IP地址。第一个端口号（port）是从follower连接到leader机器的端口，第二个端口是用来进行leader选举时所用的端口。所以，在集群配置过程中有三个非常重要的端口：clientPort：2181、port:2888、port:3888。

  在每台服务器上配置后，然后启动zookeeper即可。

  **注意**：

  他们在启动的时候需要进行leader选举，此时server1就需要和其他两个zookeeper实例进行通信，但是，另外两个zookeeper实例还没有启动起来，因此将会产生ConnectException的信息。当server2和server3启动后就不会再有这样的警告信息了。


- **伪集群模式**

  伪集群模式与集群模式大致相同，只不过是在同一台服务器上运行多个zookeeper实例，为了保证端口不冲突，分别配置不同的端口，参考匹配文件如下：

  ```
  tickTime=2000
  initLimit=10
  syncLimit=5
  dataDir=/opt/zookeeper/zookeeper-3.4.11/data
  clientPort=2181
  maxClientCnxns=60
  server.1=127.0.0.1:2888:3888
  server.2=127.0.0.1:2889:3889
  server.3=127.0.0.1:2890:3890
  ```

  其余两份配置文件和上面类似，同样需要修改下clientPort即可。





### 2.**开机启动**

网上设置zookeeper开机启动的方法大多数是做成service，通过service命令来管理，这种方式相对复杂，而且不利用自动化安装脚本的编写。另外一种就是利用systemd来进行管理zookeeper，方便而且简单，这个网上资料较少，而且提供的systemd脚本多少有些问题。下面就这两种分别进行介绍，推荐使用systemd来维护管理。

- systemd管理

  下面是自己写的脚本

  ```

  [Unit]
  Description=zookeeper
  After=syslog.target network.target

  [Service]
  Type=forking
  PIDFile=/opt/zookeeper/zookeeper-3.4.11/data/zookeeper_server.pid
  Environment=ZOO_LOG_DIR=/opt/zookeeper/zookeeper-3.4.11/log
  ExecStart=/opt/zookeeper/zookeeper-3.4.11/bin/zkServer.sh start
  ExecStop=/opt/zookeeper/zookeeper-3.4.11/bin/zkServer.sh stop
  ExecReload=/opt/zookeeper/zookeeper-3.4.11/bin/zkServer.sh restart
  KillMode=none
  User=root

  [Install]
  WantedBy=multi-user.target
  ```

  将文件放入/usr/lib/systemd/system目录下即可（不同的os版本目录有所不同）

  字段含义可参考”参考链接“部分的内容，着重说下其中几个字段

  type字段：

  - simple（默认值）：默认，这是最简单的服务类型。意思就是说启动的程序就是主体程序，这个程序要是退出那么一切皆休。这在图形界面里非常好理解，我打开 Amarok，退出它就没有了。但是命令行的大部分程序都不会那么设计，因为命令行的一个最基本原则就是一个好的程序不能独占命令行窗口。所以输入命令，回车，接着马上返回给你提示符，但程序已经执行了。所以只有少数程序比如 python xxx.py 还使用这种方式。在这种类型下面，如果你的主程序是要响应其它程序的，那么你的通信频道应该在启动本服务前就设好（套接字等），因此这种类型的服务，Systemd 运行它后会立刻就运行下面的服务（需要它的服务），这时没有套接字后面的服务会失败，写 After 也没用，因为 simple 类型不存在主进程退出的情况也就不存在有返回状态的情况，所以它一旦启动就认为是成功的，除非没起来。
  - forking：启动程序后会调用 fork() 函数，把必要的通信都设置好之后父进程退出，留下守护精灵的子进程。你要是使用的这种方式，最好也指定下 PIDFILE=，不要让 Systemd 去猜，非要猜也可以，把 GuessMainPID 设为 yes。
  - oneshot：类似于`simple`，但只执行一次，Systemd 会等它执行完，才启动其他服务
  - dbus：类似于`simple`，但会等待 D-Bus 信号后启动
  - notify：类似于`simple`，启动结束后会发出通知信号，然后 Systemd 再启动其他服务
  - idle：类似于`simple`，但是要等到其他任务都执行完，才会启动该服务。一种使用场合是为让该服务的输出，不与其他服务的输出相混合

  判断是 forking 还是 simple 类型非常简单，命令行里运行下你的程序，持续占用命令行要按 Ctrl + C 才可以的，就不会是 forking 类型。

  zookeeper进程是并不会独立占用窗口，因此采用forking类型。

  Environment字段：

  导入环境变量值“ZOO_LOG_DIR"，在上述中已经提到这个目录是用于存放zookeeper日志地方，因此在此指定。

  KillMode字段：

  - control-group（默认值）：当前控制组里面的所有子进程，都会被杀掉
  - process：只杀主进程
  - mixed：主进程将收到 SIGTERM 信号，子进程收到 SIGKILL 信号
  - none：没有进程会被杀掉，只是执行服务的 stop 命令。

  这个字段也很关键，如果在上述配置文件中没有填写此字段，zookeeper的重启、启动都是正常的，但是用systemctl stop zookeeper.service时会报如下错误：

  ```
    systemctl status zookeeper.service 
    zookeeper.service - zookeeper
     Loaded: loaded (/usr/lib/systemd/system/zookeeper.service; enabled; vendor preset: disabled)
     Active: failed (Result: signal) since Fri 2017-12-22 15:18:21 CST; 6s ago
    Process: 1732 ExecStop=/opt/zookeeper/zookeeper-3.4.11/bin/zkServer.sh stop (code=exited, status=0/SUCCESS)
   Main PID: 11135 (code=killed, signal=KILL)

  Dec 21 16:32:41 controller1 zkServer.sh[11126]: Starting zookeeper ... STARTED
  Dec 21 16:32:41 controller1 systemd[1]: Started zookeeper.
  Dec 22 15:18:21 controller1 systemd[1]: Stopping zookeeper...
  Dec 22 15:18:21 controller1 zkServer.sh[1732]: ZooKeeper JMX enabled by default
  Dec 22 15:18:21 controller1 zkServer.sh[1732]: Using config: /opt/zookeeper/zookeeper-3.4.11/bin/../conf/zoo.cfg
  Dec 22 15:18:21 controller1 zkServer.sh[1732]: Stopping zookeeper ... STOPPED
  Dec 22 15:18:21 controller1 systemd[1]: zookeeper.service: main process exited, code=killed, status=9/KILL
  Dec 22 15:18:21 controller1 systemd[1]: Stopped zookeeper.
  Dec 22 15:18:21 controller1 systemd[1]: Unit zookeeper.service entered failed state.
  Dec 22 15:18:21 controller1 systemd[1]: zookeeper.service failed.
  ```
  状态并不是inactive，而是failed，默认情况下systemd会杀死主进程，而且强制杀死后，返回的status却不正常，导致Unit zookeeper.service entered failed state.对于zookeeper而言，使用自己的stop命令就可以结束与进程相关的动作，因此可以指定KillMode=none来解决这个问题。


- zookeeper做成服务（参考文档3，未做验证）

  1、进入到/etc/rc.d/init.d目录下，新建一个zookeeper脚本，并给脚本添加执行权限

  ```
  cd /etc/rc.d/init.d/ 
  touch zookeeper
   chmod +x zookeeper
  ```

  2、使用命令vim zookeeper进行编辑，在脚本中输入如下内容，其中同上面注意事项一样要添加export JAVA_HOME=/usr/java/jdk1.8.0_112这一行，否则无法正常启动。（请根据自己环境上jdk路径进行修改）

  ```
  #!/bin/bash 
  #chkconfig:2345 20 90 
  #description:zookeeper 
  #processname:zookeeper 
  export JAVA_HOME=//usr/java/jdk1.8.0_112
  export ZOO_LOG_DIR=/opt/zookeeper/log  
  case $1 in 
          start) sudo /usr/local/zookeeper-3.4.5/bin/zkServer.sh start;; 
          stop) sudo /usr/local/zookeeper-3.4.5/bin/zkServer.sh stop;; 
          status) sudo /usr/local/zookeeper-3.4.5/bin/zkServer.sh status;; 
          restart) sudo /usr/local/zookeeper-3.4.5/bin/zkServer.sh restart;; 
          *) echo "require start|stop|status|restart" ;; 
  esac
  ```

  3、使用service zookeeper start/stop命令来尝试启动关闭zookeeper，使用service zookeeper status查看zookeeper状态。

  ```
  service zookeeper status 
  JMX enabled by default 
  Using config: /usr/local/zookeeper-3.4.5/bin/../conf/zoo.cfg 
  Mode: standalone 
  ```

  4、添加到开机自启即可

  ```
  chkconfig --add zookeeper
  ```


参考文档：

[zookeeper 安装的三种模式](https://www.cnblogs.com/jxwch/p/6433310.html)

[systemd.service 中文手册](http://www.jinbuguo.com/systemd/systemd.service.html)

[Centos 设置zookeeper开机自启动](https://www.cnblogs.com/zhangmingcheng/p/7455278.html)

[Systemd 入门教程：实战篇](http://www.ruanyifeng.com/blog/2016/03/systemd-tutorial-part-two.html)


