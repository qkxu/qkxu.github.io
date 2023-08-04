---
layout: post
title: Ceilometer 源码学习 - Collection
category: telemetry
description: Ceilometer 源码学习 - Collection
---
# Ceilometer 源码学习 - Collection组件#

collection组件主要是通过mq以及udp采集sample，并存储到相应的位置（数据库、file、gnocchi等）

### 1.**入口**

- Ceilometer采用[pbr](http://docs.openstack.org/developer/pbr/)的方式管理配置，
- setup.cfg中定义了Polling Agent 入口位置，如下：

```
console_scripts =
    ...
    ceilometer-collector = ceilometer.cmd.collector:main
```

### 2. **ceilometer.cmd.collector:main**

在ceilometer/cmd/collector.py 文件中找到该函数，如下：

```
def main():
    service.prepare_service()
    os_service.launch(CONF, collector.CollectorService(),
                      workers=CONF.collector.workers).wait()
```

- prepare_service中做了一些初始化工作，如初始化日志，加载配置文件等；
- 第二句为核心，配置并启动了collector.CollectorService


### 3. **启动组件**

```
def start(self):
    """Bind the UDP socket and handle incoming data."""
    # ensure dispatcher is configured before starting other services
    dispatcher_managers = dispatcher.load_dispatcher_manager()
    (self.meter_manager, self.event_manager) = dispatcher_managers
    ...
    if cfg.CONF.collector.udp_address:
        self.tg.add_thread(self.start_udp)
```

- 加载dispatchmanager


```
namespace = 'ceilometer.dispatcher.%s' % dispatcher_type
conf_name = '%s_dispatchers' % dispatcher_type
```

读取setup.cfg中的ceilometer.dispatcher.meter以及ceilometer.dispatcher.event命令空间下的注册项
并且根据conf_name匹配相应的处理类，以meter为例，conf_name = meter_dispatchers，读取配置文件中meter_dispatchers的配置项的值，默认为default=['database']，作为消息队列的endpoint用于接收处理sample。

- 通过udp（如果有）来接收sample

      if cfg.CONF.collector.udp_address:
          self.tg.add_thread(self.start_udp)
- 初始化监听队列以及topic用于接收sample，以sample为例

```
if list(self.meter_manager):
    sample_target = oslo_messaging.Target(
        topic=cfg.CONF.publisher_notifier.metering_topic)
    self.sample_listener = (
        messaging.get_batch_notification_listener(
            transport, [sample_target],
            [SampleEndpoint(self.meter_manager)],
            allow_requeue=True,
            batch_size=cfg.CONF.collector.batch_size,
            batch_timeout=cfg.CONF.collector.batch_timeout))
    self.sample_listener.start()
```

topic是“metering”，即notification组件中publish存放到mq队列时指定的topic，在collection中指定topic用于接收，同时指定SampleEndpoint用于处理消息。

SampleEndpoint的父类为CollectorEndpoint，其中定义了sample，即只处理sample级别的消息，处理方式为调用self.method的方法

```
self.dispatcher_manager.map_method(self.method, samples)
```

SampleEndpoint定义为：

```
method = 'record_metering_data'
ep_type = 'sample'
```

最终会调用record_metering_data方法用于记录数据。

### 4. **数据库分发（默认方式）**

- 在database文件中会初始化数据库连接

  ```
  def __init__(self, conf):
      super(DatabaseDispatcher, self).__init__(conf)

      self._meter_conn = self._get_db_conn('metering', True)
      self._event_conn = self._get_db_conn('event', True)
  ```

- 读取并解析配置

  ```
  def _inner():
      if conf.database_connection:
          conf.set_override('connection', conf.database_connection,
                            group='database')
      namespace = 'ceilometer.%s.storage' % purpose
      url = (getattr(conf.database, '%s_connection' % purpose) or
             conf.database.connection)
      return get_connection(url, namespace)
  ```
  根据配置conf.database_connection来解析url，并且根据命名空间ceilometer.metering.storage中的相关类处理
  需要注意的是dispatch的log是放在database中实现的，查看命名空间中的类也可以看到，log方式只是打印了下日志。
- 具体driver实现

  ```
  mgr = driver.DriverManager(namespace, engine_name)
  return mgr.driver(url)
  ```

  在mongodb初始化时会调用self.upgrade()，其中会设置超时时间之类的，这样就完成了数据库连接的初始化。在database中record_metering_data的函数用于记录meter


### 5.**gnocchi分发**

- 初始化

  ```
  def __init__(self, conf):
      super(GnocchiDispatcher, self).__init__(conf)
      self.conf = conf
      self.filter_service_activity = (
          conf.dispatcher_gnocchi.filter_service_activity)
      self._ks_client = keystone_client.get_client()
      self.resources_definition = self._load_resources_definitions(conf)
      ...
      self._gnocchi = gnocchi_client.get_gnocchiclient(conf)
  ```

主要是初始化配置，获取keystone客户端以及gnocchi的客户端，用与连接gnocchi组件，目前url方式已经废弃，通过keystone自动获取gnocchi。

- 分发数据


  record_metering_datas函数首先会对数据进行组合处理，以便gnocchi接收：

```
for resource_id, samples_of_resource in resource_grouped_samples:
    ...
    for metric_name, samples in metric_grouped_samples:
        stats['metrics'] += 1

        samples = list(samples)
        #从gnocchi_resources.yaml中读取gnocchi接受的资源，包括resource_type、
        #metrics、attributes，如果在此文件中未定义的meter将会被忽略。
        #特别注意，根据metric_name会得到匹配的某一个resource，然后将resource中
        #定义的metric全部添加进去（rd.metrics），即使此次sample中并不包含此metric项
        rd = self._get_resource_definition_from_metric(metric_name)
        if rd is None:
            LOG.warning(_LW("metric %s is not handled by Gnocchi") %
                        metric_name)
            continue
        if rd.cfg.get("ignore"):
            continue

        res_info['resource_type'] = rd.cfg['resource_type']
        res_info.setdefault("resource", {}).update({
            "id": resource_id,
            "user_id": samples[0]['user_id'],
            "project_id": samples[0]['project_id'],
            "metrics": rd.metrics,
        })

        #在ceilometer中的meter对应于gnocchi中metric的name属性
        for sample in samples:
            ...
            unit = sample['counter_unit']
            metric = sample['counter_name']
            #对于首次发送数据，batch_measures方法会走异常分支，
            #然后创建resource，本次sample中包含的metric项拥有unit，其余的都unit为Null。
            res_info['resource']['metrics'][metric]['unit'] = unit

        ...
        for gnocchi_id, info in gnocchi_data.items():
            resource = info["resource"]
            resource_type = info["resource_type"]
            resource_extra = info["resource_extra"]
            if not resource_extra:
                continue
            try:
                #当此资源中包含其余的metic的sample时，更新resource，此时上文中unit这些项才有
                #真正的单位，而不是Null
                self._if_not_cached("update", resource_type, resource,
                                    self._update_resource, resource_extra)
```

然后调用  self.batch_measures(measures, gnocchi_data, stats)来发送数据。

```
def batch_measures(self, measures, resource_infos, stats):
    # NOTE(sileht): We don't care about error here, we want
    # resources metadata always been updated
    try:
        #向gnocchi_client发送批量处理measures请求
        self._gnocchi.metric.batch_resources_metrics_measures(measures)
    except gnocchi_exc.BadRequest as e:
        #首次发送数据时，metric在gnocchi中还不存在，因此会进入异常分支
        #resource_infos中包含了所有的metric项
        m = self.RE_UNKNOW_METRICS.match(six.text_type(e))
        if m is None:
            raise

        # NOTE(sileht): Create all missing resources and metrics
        metric_list = self.RE_UNKNOW_METRICS_LIST.findall(m.group(1))
        gnocchi_ids_freshly_handled = set()
        for gnocchi_id, metric_name in metric_list:
            if gnocchi_id in gnocchi_ids_freshly_handled:
                continue
            resource = resource_infos[gnocchi_id]['resource']
            resource_type = resource_infos[gnocchi_id]['resource_type']
            try:
            #首先会创建resoure，创建resource会带有resource_type，在gnocchi初始化时要执行
            #gnocchi-upgrade，或者在ceilometer组件中初始化时执行ceilometer-upgrade用于
            #创建resource_type
            #创建resource时会将resource的metric一并创建，在gnocchi中metric是属
            #于resource资源中的一个属性，meteric对应于meter，resource中可以包含多个metric
            #gnocchi resource show查看
                self._if_not_cached("create", resource_type, resource,
                                    self._create_resource)
            except gnocchi_exc.ResourceAlreadyExists:
                metric = {'resource_id': resource['id'],
                          'name': metric_name}
                metric.update(resource["metrics"][metric_name])
                try:
                    #此分支一般不会进入
                    self._gnocchi.metric.create(metric)
                ...
        # NOTE(sileht): we have created missing resources/metrics,
        # now retry to post measures
        #再次调用创建measures
        self._gnocchi.metric.batch_resources_metrics_measures(measures)
```




