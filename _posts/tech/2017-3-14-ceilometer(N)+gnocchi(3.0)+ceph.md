---
layout: post
title: Ceilometer(N)+Gnocchi(3.0)+ceph集成
category: telemetry
description: Ceilometer(N)+Gnocchi(3.0)+ceph集成
---
# ceilometer(N)+gnocchi(3.0)+ceph集成

- ## 安装gnocchi
  - pip安装，可以参考官网文档，pip install gnocchi
  - 源码安装 
      - 官网下载源码包解压或者用git clone [https://github.com/openstack/gnocchi.git](https://github.com/openstack/gnocchi.git)
      - cd gnocchi/ 
      - pip install -r requirements.txt
      - python setup.py install

  说明：源码安装最主要的是解决软件包依赖文件，gnocchi依赖软件包比较多，pandas、pytimeparse、Cython 、cradox 等，根据安装报错，下载相应的软件包安装即可。

- ## 准备数据库以及keystone鉴权

  #### 创建数据库

  登陆数据库节点，为gnocchi创建数据库。

  ```
  mysql -u root -p

  CREATE DATABASE gnocchi;
   
  GRANT ALL PRIVILEGES ON  gnocchi.* TO 'gnocchi'@'localhost' IDENTIFIED BY 'Abc12345';
   
  GRANT ALL PRIVILEGES ON  gnocchi.* TO 'gnocchi'@'%' IDENTIFIED BY 'Abc12345';
  ```

  #### 创建gnocchi用户

  ```
  openstack user create --domain default --password-prompt gnocchi
  ```

  #### 给gnocchi用户赋admin角色

  ```
  openstack role add --project service --user gnocchi admin
  ```

  #### 创建服务

  ```
  openstack service create --name gnocchi --description "OpenStack Resource metering as a Service" metric
  ```

  #### 创建endpoint

  注意： 如果需要HA，则需要HAproxy配置 并把这里的Ip在实际环境中替换为haproxy的IP

  ```
  openstack endpoint create --region RegionOne metric public http://192.168.2.203:8041

  openstack endpoint create --region RegionOne metric internal http://192.168.2.203:8041

  openstack endpoint create --region RegionOne metric admin http://192.168.2.203:8041
  ```

- ## 准备ceph存储

  - 给gnocchi创建一个专用的ceph pool，用来存放计量数据。（ceph mon node）
    ceph osd pool create gnocchi 128 128
    gnocchi pool的pg_num需要根据实际的ceph环境确定。

  - 给gnocchi创建一个ceph用户。

    ceph auth get-or-create client.gnocchi mon ‘allow r’ osd ‘allow class-read object_prefix rbd_children, allow rwx pool=gnocchi’

  - 保存keyring文件

    ceph auth get-or-create client.gnocchi | tee ceph.client.gnocchi.keyring

  - 拷贝gnocchi的keyring文件ceph.client.gnocchi.keyring到所有controller节点的/etc/ceph目录下。

  - 获取gnocchi用户的ceph key，用于gnocchi.conf中的ceph_secret配置项。(secret与key配置一个即可，下文有提到)

    ceph auth get-key client.gnocchi
    AQAvAr1XN4BtDRAAR8O6ubQMZhCov26b8/f7hg==

  ​

- ## 配置

- 配置ceilometer

  主要配置三个地方，url是指gnocchi提供的api服务地址

  [DEFAULT]
  
  meter_dispatchers = gnocchi

  [api]

  gnocchi_is_enable = True

  [dispatcher_gnocchi]
  
  filter_project = service
  
  filter_service_activity = False
  
  archive_policy = low
  
  url = http://192.168.2.203:8041

  ​

- 配置gnocchi

  修改gnocchi.conf（执行gnocchi-config-generator > /etc/gnocchi/gnocchi.conf生成conf）

  修改相应组中的值，如下：

  create_legacy_resource_types用于创建初始化时创建resource-type

  database与indexer是用于存放indexer的数据库

  storage用于存放measure的文件系统

  ```
  [DEFAULT]
  create_legacy_resource_types = True

  ...

  [database]
  connection = mysql+pymysql://{{gnocchi_user}}:{{gnocchi_password}}@{mariadb_host}}/gnocchi

  ...

  [indexer]
  url = mysql+pymysql://{{gnocchi_user}}:{{gnocchi_password}}@{{mariadb_host}}/gnocchi

  ....

  [keystone_authtoken]
  auth_uri = http://{{openstack.keystone_host}}:5000
  auth_url = http://{{openstack.keystone_host}}:35357
  memcached_servers = {{openstack.keystone_host}}:11211
  auth_type = password
  project_domain_name = default
  user_domain_name = default
  project_name = service
  username = gnocchi
  password = gnocchi

  [oslo_messaging_rabbit]
  rabbit_host = {{openstack.rabbit_host}}
  rabbit_userid = {{openstack.rabbit.user}}
  rabbit_password = {{openstack.rabbit.password}}
  amqp_durable_queues = True
  rabbit_retry_interval = 1
  rabbit_retry_backoff = 3
  rabbit_interval_max = 60
  rabbit_max_retries = 0
  rabbit_ha_queues = True
  rabbit_transient_queues_ttl = 600
  rpc_queue_expiration = 60

  ...

  [storage]
  driver = ceph
  ceph_pool = gnocchi
  ceph_username = gnocchi
  ceph_conffile = /etc/ceph/ceph.conf
  ceph_keyring = /etc/ceph/ceph.client.gnocchi.keyring
  ceph_secret = AQAvAr1XN4BtDRAAR8O6ubQMZhCov26b8/f7hg==
  #ceph_keyring与ceph_secret两者选择一个即可
  ```

- 配置文件到特定目录

  api-paste.ini文件默认的鉴权方式为

  ```
  [pipeline:main]
  pipeline = gnocchi+noauth
  ```

  如果采用kestone鉴权，pipe修改为gnocchi+auth（3.1版本有配置项，3.0只支持这样修改）

  对于配置文件放置到/etc/gnocchi目录下 （api-paste.ini，gnocchi.conf，policy.json ）

  ​

- ## 启动

- 初始化indexes数据库表以及storage

  ```
  gnocchi-upgrade
  ```

- 启动gnocchi

  - 手动执行api

    gnocchi-api --port 8041 -- --log-file /var/log/gnocchi/gnocchi-api.log  --config-file /etc/gnocchi/gnocchi.conf

    其中port是指端口号，这种方式只用于测试，而非正式推荐使用方法，建议用httpd或者usgi来启动。

    而且这种方式只是对于api启动了一个线程，对于ceilometer发送数据过大时，会直接超过单线程api负荷导致api不可用。

  - 用httpd执行api

    根据gnocchi源码中/destack/apache-ported-gnocchi.template进行参数的修改，改为环境中的信息即可，其中processes可以执行启动api的进程数。例如：

    ```
    Listen 192.168.31.1:8041

    <VirtualHost *:8041>
        WSGIDaemonProcess gnocchi processes=10 threads=10 user=gnocchi display-name=%{GROUP}
        WSGIProcessGroup gnocchi
        WSGIScriptAlias / /usr/lib/python2.7/site-packages/gnocchi/rest/app.wsgi
        WSGIApplicationGroup %{GLOBAL}
        <Directory /usr/lib/python2.7/site-packages/gnocchi/rest/>
            Require all granted
        </Directory>
        ErrorLog /var/log/gnocchi/gnocchi_error.log
        CustomLog /var/log/gnocchi/gnocchi_access.log combined
    </VirtualHost>

    WSGISocketPrefix /var/run/httpd
    ```
    将配置文件放入httpd配置目录（一般位于/etc/httpd/conf.d/），重启httpd服务即可

    ```
    httpd -k restart
    ```

- 启动metricd进程

  - 手动执行

    ```
    gnocchi-metricd --log-file /var/log/gnocchi/gnocchi-metricd.log
    ```

  - systemd执行

    将以下内容写入/usr/lib/systemd/system/gnocchi-metricd.service，
    ```
    [Unit]
    Description=OpenStack gnocchi API service
    After=syslog.target network.target

    [Service]
    Type=simple
    User=root
    ExecStart=/usr/bin/gnocchi-metricd --config-file /etc/gnocchi/gnocchi.conf --log-file /var/log/gnocchi/gnocchi-metricd.log

    [Install]
    WantedBy=multi-user.target
    ```

    使用systemctl命令来启动

    ```
    systemctrl start gnocchi-metricd.service
    ```


