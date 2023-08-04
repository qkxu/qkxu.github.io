---
layout: post
title: Ceilometer 源码学习 - Polling Agent
category: telemetry
description: Ceilometer 源码学习 - Polling Agent组件
---
# Ceilometer 源码学习 - Polling Agent

## 简介

Ceilometer是Openstack中用于数据采集的基础设施，包括多个组件：Central Agent，Compute Agent，Notification Agent，Collector等。其中Central Agent和Compute Agent分别运行在Controller和Compute机器上，通过定期调用其他服务的api来完成数据采集。由于二者的区别只是所负责的数据来源，这里我们统称为Polling Agent。

## 需求导向

Polling Agent的功能很简单：

**周期性地向其他服务主动拉取需要的数据，并将数据发送到消息队列。**

其结构图如下： ![图1 Central Agent结构图](https://github.com/qkxu/image/blob/master/2-2-collection-poll.png?raw=true)

站在设计者的角度，要完成上述功能，需要处理的有如下几个基本问题：

1. 怎么执行拉取；
2. 向哪些服务拉取数据；
3. 对于某个服务收集哪些数据以及如何收集。

下面分别针对上述问题依次介绍Ceilometer的实现方式:

1. **常驻进程**：自然的我们需要一个常驻进程来完成上述调度任务，基本操作包括：
   - 记录全局状态；
   - 周期性的触发；
   - 负责消息的发送。
2. **插件形式**：Ceilometer中用定义插件的方式定义多个收集器（Pollster），程序从配置文件中获得需要加载的收集器列表，用插件的形式是一个很好的选择，因为：
   - python对插件的良好支持：[stevedore](http://docs.openstack.org/developer/stevedore/)
   - 简化核心逻辑；
   - 方便扩展。
3. **共同基类**：数据来源多种多样，针对不同的数据来源获取数据方式各有不同，但他们需要完成同样的的动作，Ceilometer中设计Pollster的共同基类，定义了如下接口，是每个Pollster都是要实现的：
   - 默认获取数据来源的方式：default_discovery；
   - 拉取数据：get_samples。

## 流程简介

正是由于上面所说的实现方式使得Polling Agent的核心逻辑变得非常简单，不需要关注具体的数据收集过程，而将自己解放成一个调度管理者，下面将简单介绍其实现逻辑。在此之前为了方便说明，先介绍其中涉及到的角色或组件:

- **AgentManager**：Polling Agent的核心类，Central Agent和Compute Agent用不同的参数初始化AgentManager；
- **Pollster**：数据收集器，以插件的形式动态载入；
- **Discover**：以一定的方式发现数据源Resource；
- **Pipeline**：Ceilometer通过pipleline.yml文件的形式定义了所收集数据的一系列转换发送操作，很好的降低了各组件的耦合性和系统的复杂性。该文件中以sources标签定义了不同数据的分组信息，这部分是在Polling Agent中需要关心的；
- **PollingTask**：望文生义，表示一个拉取数据的任务；
- **Resource**：代表一个可提供数据的源，如一个镜像或一个虚拟机实例。

**基本流程**如下：

- AgentManger初始化，主要完成如下动作：
  - 从配置文件中动态加载所有收集器插件Pollster；
  - 从配置文件中动态加载所有资源发现器插件Discover。
- AgentManger启动：
  - 从pipeline文件读取sources信息；
  - 为每一个从文件中加载的Pollster根据Pipeline信息分配一个PollingTask；
  - 为每个PollingTask建立Timer执行。
- PollingTask执行：
  - 通过Pollster的default_discovery函数定义，从已加载的资源发现器Discover中选取合适的一个；
  - 调用Discover的discovery函数获取Resource；
  - 调用Pollster的get_samples函数，从Resource中获得采样数据；
  - 发送给消息队列。

## 代码细节

接下来从代码层面详细介绍上述逻辑实现：

### 1. **入口**

- Ceilometer采用[pbr](http://docs.openstack.org/developer/pbr/)的方式管理配置，
- setup.cfg中定义了Polling Agent 入口位置，如下：

```
console_scripts =
    ceilometer-polling = ceilometer.cmd.eventlet.polling:main
    ...
```

### 2. **ceilometer.cmd.eventlet.polling**

相应的，在ceilometer/cmd/eventlet/polling.py 文件中找到该函数，如下：

```
def main():
     service.prepare_service()
     os_service.launch(CONF, manager.AgentManager(CONF.polling_namespaces,
                                                CONF.pollster_list)).wait()
```

- prepare_service中做了一些初始化工作，如初始化日志，加载配置文件等；

- 第二句为核心，配置并启动了manager.AgentManager，进一步了解到主要工作发生在该类的父类中，即base.AgentManger

- polling有三种方式central、compute、impi，启动时过程一致，只不过传入的polling_namespaces不同，例如：

  ExecStart=/usr/bin/ceilometer-polling --polling-namespaces central --config-file /etc/ceilometer/ceilometer.conf --log-file /var/log/ceilometer/ceilometer-acentral.log

  默认情况下（如果不传polling_namespaces）：

  ```
  CLI_OPTS = [
      MultiChoicesOpt('polling-namespaces',
                      default=['compute', 'central'],
                      choices=['compute', 'central', 'ipmi'],
                      dest='polling_namespaces',
                      help='Polling namespace(s) to be used while '
                           'resource polling'),
      MultiChoicesOpt('pollster-list',
                      default=[],
                      dest='pollster_list',
                      help='List of pollsters (or wildcard templates) to be '
                           'used while polling'),
  ]
  ```

  ​

### 3. **base.AgentManager 初始化**

ceilometer/agent/base.py下找到AgentManager的初始化部分代码，部分如下所示：

```
from stevedore import extension

...
def __init__(self, namespaces, pollster_list, group_prefix=None):
    ...
    # 从配置文件中动态加载收集器Pollster
    extensions = (self._extensions('poll', namespace).extensions
                   for namespace in namespaces)
    ... 
    
    self.extensions = list(itertools.chain(*list(extensions))) + list(
         itertools.chain(*list(extensions_fb)))
    # 从配置文件中动态加载资源发现器Discover
    self.discovery_manager = self._extensions('discover')
    ...

 @staticmethod
 def _get_ext_mgr(namespace):
     def _catch_extension_load_error(mgr, ep, exc):
       ...

     return extension.ExtensionManager(
         namespace=namespace,
         invoke_on_load=True,
         on_load_failure_callback=_catch_extension_load_error,
     )

 def _extensions(self, category, agent_ns=None):
     namespace = ('ceilometer.%s.%s' % (category, agent_ns) if agent_ns
                  else 'ceilometer.%s' % category)
     return self._get_ext_mgr(namespace)
```

- 可以看出_extensions函数中通过[stevedore](http://docs.openstack.org/developer/stevedore/)加载了配置文件中的对应namespace下的插件；
- 初始化过程init中，主要做了两件事情：
  - 加载ceilometer.poll.central下的插件到self.extensions，即上面所说的收集器Pollster；
  - 加载ceilometer.discover下的插件到self.discovery_manager，即上面所说的资源发现器Discover。
- 而在配置文件setup.cfg中可以看到对应的定义，截取部分在这里：

```
...
ceilometer.poll.central =
      ip.floating = ceilometer.network.floatingip:FloatingIPPollster
      image = ceilometer.image.glance:ImagePollster
      image.size = ceilometer.image.glance:ImageSizePollster
      ...
...   
ceilometer.discover =
      local_instances = ceilometer.compute.discovery:InstanceDiscovery
      endpoint = ceilometer.agent.discovery.endpoint:EndpointDiscovery
      tenant = ceilometer.agent.discovery.tenant:TenantDiscovery
      ...
 ...
```

### 4. **base.AgentManager 启动**

了解AgentManager初始化之后，再来看启动部分的代码实现：

```
def start(self):
    # 读取pipeline.yaml配置文件
    self.polling_manager = pipeline.setup_polling()
    ...
    # 
    self.pollster_timers = self.configure_polling_tasks()
    ...
...
```

下面分别介绍这两行代码的功能：

- pipeline.setup_polling中加载解析pipeline.yaml文件，来看一个pipeline.yaml中的示例，更多内容：[pipeline](http://docs.openstack.org/developer/ceilometer/architecture.html#pipeline-manager)；

```
---
  sources:
      - name: meter_source
        interval: 600
        meters:
            - "*"
        sinks:
            - meter_sink
      - name: cpu_source
        ...
      ...
  sinks:
      - name: meter_sink
        transformers:
        publishers:
      ...
  ...
```

ceilometer中用pipeline配置文件的方式定义meter数据从收集到处理到发送的过程，在Polling Agent中我们只需要关心sources部分，在上述pipeline.setup_polling()中读取pipeline文件并解析封装其中的sources内容，供后面使用，并不做数据处理，数据处理的pipeline功能在notification组件中完成。

- configure_polling_tasks代码如下：

```
def configure_polling_tasks(self):
    ...
    pollster_timers = []
    # 创建PollingTask
    data = self.setup_polling_tasks()
    # PollingTask定时执行
    for interval, polling_task in data.items():
        delay_time = (interval + delay_polling_time if delay_start
                      else delay_polling_time)
        pollster_timers.append(self.tg.add_timer(interval,
                               self.interval_task,   #PollsterTask执行内容
                               initial_delay=delay_time,
                               task=polling_task)) 
    ...
    return pollster_timers
```

其中，setup_polling_tasks中新建PollingTask，并根据上一步中封装的sources内容，将每一个收集器Pollster根据其interval设置分配到不同的PollingTask中，interval相同的收集器会分配到同一个PollingTask中。之后每个PollingTask都根据其运行周期设置Timer定时执行。 注意，其中interval_task函数指定timer需要执行的任务。

### 5. **PollingTask 执行**

上边我们了解到PollingTask会定时执行，而interval_task中定义了他的内容：

```
@staticmethod
def interval_task(task):
     task.poll_and_notify()

def poll_and_notify(self):
    ...
    for source_name in self.pollster_matches:
       # 循环处理PollingTask中的每一个收集器Pollster
       for pollster in self.pollster_matches[source_name]:
           key = Resources.key(source_name, pollster)
                candidate_res = list(
                    self.resources[key].get(discovery_cache))
           # Discover发现可用的数据源
           # 其中discovery_cache作用是用于缓存已经查询的资源，如果没有在pipeline
           # 中指定discovery以及resource，则candidate_res为空
           # discovery_cache在首次执行下面的函数后将会赋予缓存的资源，用于循环中相同的
           # discovery_cach直接取得，不需要重复查询
           if not candidate_res and pollster.obj.default_discovery:
                candidate_res = self.manager.discover(
                    [pollster.obj.default_discovery], discovery_cache)
                    
                # Remove duplicated resources and black resources. Using
                # set() requires well defined __hash__ for each resource.
                # Since __eq__ is defined, 'not in' is safe here.
                # 周期性任务是根据pipeline中的时间间隔进行分组，在同一组中如果出现同一个meter，
                # 则第二次meter将会直接略过，因为polling_resources此时为空。
                polling_resources = []
                black_res = self.resources[key].blacklist
                history = poll_history.get(pollster.name, [])
                for x in candidate_res:
                    if x not in history:
                        history.append(x)
                        if x not in black_res:
                            polling_resources.append(x)
                poll_history[pollster.name] = history
                
                # If no resources, skip for this pollster
                if not polling_resources:
                    p_context = 'new ' if history else ''
                    LOG.info(_LI("Skip pollster %(name)s, no %(p_context)s"
                                 "resources found this cycle"),
                             {'name': pollster.name, 'p_context': p_context})
                    continue

             ... 

             try:
                 # 从数据源处拉取采样数据
                 samples = pollster.obj.get_samples(
                     manager=self.manager,
                     cache=cache,
                     resources=polling_resources
                 )
                 sample_batch = []

                 # 发送数据到消息队列
                 for sample in samples:
                     sample_dict = (
                         publisher_utils.meter_message_from_counter(
                             sample, self._telemetry_secret
                         ))
                     if self._batch:
                         sample_batch.append(sample_dict)
                     else:
                         self._send_notification([sample_dict])

                 if sample_batch:
                     self._send_notification(sample_batch)

            except 
                ...
    
```

可以看出，在这段代码中完成了比较核心的几个步骤： 1. 资源发现器Discover发现可用数据源； 2. 收集器Pollster拉取采样数据； 3. 发送数据到消息队列。

### 6. **Pollster示例**

上面介绍了Polling Agent中如何是如何加载Pollster执行数据的收集工作的。下面以获取image基本信息的ImagePollster为例，看一下具体的实现：

```
class _Base(plugin_base.PollsterBase):

     @property
     def default_discovery(self):
         return 'endpoint:%s' % cfg.CONF.service_types.glance
         #此处为endpoint:glance，从discover找到相应的处理类，即：
         endpoint = ceilometer.agent.discovery.endpoint:EndpointDiscovery
         并将glance作为参数传入。

     def get_glance_client(ksclient, endpoint):
         ...
    
     def _get_images(self, ksclient, endpoint):
         client = self.get_glance_client(ksclient, endpoint)
         ...
         return client.images.list(filters={"is_public": None}, **kwargs)

     def _iter_images(self, ksclient, cache, endpoint):
         key = '%s-images' % endpoint
         if key not in cache:
             cache[key] = list(self._get_images(ksclient, endpoint))
         return iter(cache[key])
```

```
class ImagePollster(_Base):
    def get_samples(self, manager, cache, resources):
        for endpoint in resources:
            for image in self._iter_images(manager.keystone, cache, endpoint):
                yield sample.Sample(
                    name='image',
                    type=sample.TYPE_GAUGE,
                    unit='image',
                    ...
                )


```

像上边介绍过的，Pollster需要实现两个接口:

- default_discovery：指定默认的discover
- get_samples：对每个image获取采样数据

### 7. **Discover示例**

```
class EndpointDiscovery(plugin.DiscoveryBase):
    """Discovery that supplies service endpoints.
    """

    @staticmethod
    def discover(manager, param=None):
        endpoints = manager.keystone.service_catalog.get_urls(
            service_type=param,
            endpoint_type=cfg.CONF.service_credentials.os_endpoint_type,
            region_name=cfg.CONF.service_credentials.os_region_name)
        if not endpoints:
            LOG.warning(_LW('No endpoints found for service %s'),
                        "<all services>" if param is None else param)
            return []
        return endpoints

```

可以看到上面的ImagePollster所指定的Discover中慧从keystone获取所有的glance的endpoint列表, 这些endpoint列表最终会作为数据来源传给ImagePollster的get_samples

### 8. **其他**

除了上述提到的内容外，还有一些点需要注意：

- polling agent采用[tooz](http://docs.openstack.org/developer/tooz/)实现了agent的高可用，不同的agent实例之间通过tooz进行通信。在base.AgentManager的初始化和运行过程中都有相关处理，其具体实现可以在ceilometer/coordination.py中看到。

  ```
  self.partition_coordinator.start()
  self.join_partitioning_groups()
  ```

- 除了上述动态加载Pollster和Discover的方式外，pipeline还提供的静态的加载方式，可以在pipeline文件中通过sources的resources和discovery参数指定。在base.AgentManager 启动阶段，保存在对象SampleSource中，根据指定的discovery获取source，并且与指定的resources做合并。如果手动指定discovery或者resources，则后续不会从Pollster中根据discovery来获取resources。

  ```
  def get(self, discovery_cache=None):
      source_discovery = (self.agent_manager.discover(self._discovery,
                                                      discovery_cache)
                          if self._discovery else [])
      static_resources = []
      if self._resources:
          static_resources_group = self.agent_manager.construct_group_id(
              utils.hash_of_set(self._resources))
          p_coord = self.agent_manager.partition_coordinator
          static_resources = p_coord.extract_my_subset(
              static_resources_group, self._resources)
      return static_resources + source_discovery
  ```

## 核心实体

### **AgentManger**

- oslo.service 子类；
- polling agent 的核心实现类；
- 函数：
  - 定义agent的初始化、启动、关闭等逻辑；
  - 定义读取扩展的相关函数；
  - 定义pollingtask相关函数。
- 成员
  - self.extensions：从setup.cfg中读入的pollster插件；
  - self.discover_manager：从setup.cfg中读入的discover插件；
  - self.context：oslo_context 的RequestContext；
  - self.partition_coordinator：用于active-active高可用实现的PartitionCoordinator；
  - self.notifier：oslo_messaging.Notifier 用于将收集的meter信息发送到消息队列；
  - self.polling_manager: PollingManager实例，主要用其通过pipeline配置文件封装的source；
  - self.group_prefix：用来计算partition_coordination的group_id。

### **PollingTask**

- Polling task for polling samples and notifying
- 函数：
  - add：向pollstertask中增加pollster；
  - poll_and_notify：发现资源，拉取采样数据，将其转化成meter message后发送到消息队列；
- 成员：
  - self.manager: 记录当前的agent_manager；
  - self.pollster_matches，value为set类型的dic，用来记录source到pollster set的映射；
  - self.resources用来记录“source_name-pollster” key 到Resource的映射；
  - self._batch 是否要批量发送；
  - self._telemetry_secret : 配置文件中的_telemetry_secret；

## 参考

官方文档：[Ceilometer Architecture](http://docs.openstack.org/developer/ceilometer/architecture.html) 

Github：[Ceilometer Source Code](https://github.com/openstack/ceilometer)

 [Ceilometer 源码学习 - Polling Agent](https://catkang.github.io/2015/11/16/source-ceilometer-notification.html)（作者：catkang）