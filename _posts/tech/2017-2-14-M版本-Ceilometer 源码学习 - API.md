---
layout: post
title: Ceilometer 源码学习 - API
category: telemetry
description: Ceilometer 源码学习 - API
---
# Ceilometer 源码学习 - API#

ceilometer API是基于HTTP协议，使用JSON格式的RESTful API。

### 1.入口

- Ceilometer采用[pbr](http://docs.openstack.org/developer/pbr/)的方式管理配置，
- setup.cfg中定义了Polling Agent 入口位置，如下：

```
console_scripts =
    ceilometer-api = ceilometer.cmd.api:main
    ...
```

### 2. ceilometer.cmd.api:main

在ceilometer/cmd/collector.py 文件中找到该函数，如下：

```
def main():
    service.prepare_service()
    app.build_server()
```

- prepare_service中做了一些初始化工作，如初始化日志，加载配置文件等；
- 第二句为核心，加载了app，并启动wsgi服务。


### 3. 启动组件

```
def build_server():
    app = load_app()
    ...
    serving.run_simple(cfg.CONF.api.host, cfg.CONF.api.port,
                       app, processes=CONF.api.workers)
```

- 读取配置文件

  ```
  def load_app():
      # Build the WSGI app
      cfg_file = None
      cfg_path = cfg.CONF.api_paste_config
      ...
      return deploy.loadapp("config:" + cfg_file)
  ```

  ceilometer API 应用使用PasteDeploy部署，load_app函数中传入的paste.ini配置文件，paste.ini文件的格式类似于INI格式，每个section的格式为[type:name]。api_paste_config默认配置文件为：“api_paste.ini”，因此会解析该配置文件。

  ceilometer/etc/ceilometer/api_paste.ini

  ```
  [pipeline:main]
  pipeline = cors request_id authtoken api-server

  [app:api-server]
  paste.app_factory = ceilometer.api.app:app_factory

  [filter:authtoken]
  paste.filter_factory = keystonemiddleware.auth_token:filter_factory

  [filter:request_id]
  paste.filter_factory = oslo_middleware:RequestId.factory

  [filter:cors]
  paste.filter_factory = oslo_middleware.cors:filter_factory
  oslo_config_project = ceilometer
  ```
  app即是具体的app，filter表示过滤器，pipepline把filter串起来，最后一项必须是app，因此会加载ceilometer.api.app:app_factory

- 加载app

  ```
  def setup_app(pecan_config=None):
      # FIXME: Replace DBHook with a hooks.TransactionHook
      app_hooks = [hooks.ConfigHook(),
                   hooks.DBHook(),
                   hooks.NotifierHook(),
                   hooks.TranslationHook()]

      pecan_config = pecan_config or {
          "app": {
              #设置路由入口
              'root': 'ceilometer.api.controllers.root.RootController',
              'modules': ['ceilometer.api'],
          }
      }
      ...
      app = pecan.make_app(
          pecan_config['app']['root'],
          debug=pecan_debug,
          hooks=app_hooks,
          wrap_app=middleware.ParsableErrorMiddleware,
          guess_content_type_from_ext=False
      )

      return app
  ```

  Python Application创建完成，并且指定好了解析HTTP Request的RootController

- 处理请求

  ```
  class RootController(object):

      def __init__(self):
          self.v2 = v2.V2Controller()

      @pecan.expose('json')
      def index(self):
          base_url = pecan.request.application_url
          available = [{'tag': 'v2', 'date': '2013-02-13T00:00:00Z', }]
          collected = [version_descriptor(base_url, v['tag'], v['date'])
                       for v in available]
          versions = {'versions': {'values': collected}}
          return versions
  ```

  v2.V2Controller()用于处理/v2开头的请求，pecan采用对象分发的路由方式，RootController处理/路径，对象依次处理相应的路径，例如RootController下面有v2此对象，则处理/v2这样的请求路径，如果RootController下面有v3，则处理/v3这样的请求路径，对象之间可以嵌套，用于处理更深层次的路径。index函数类似于首页，用于处理GET /返回的默认信息。

  在V2Controller中定义了url相应的处理类

  ```
  @pecan.expose()
  def _lookup(self, kind, *remainder):
      ...
      elif kind == 'meters':
          return meters.MetersController(), remainder
      elif kind == 'resources':
          return resources.ResourcesController(), remainder
      elif kind == 'samples':
          return samples.SamplesController(), remainder
      elif kind == 'query':
          return QueryController(
              gnocchi_is_enabled=self.gnocchi_is_enabled,
              aodh_url=self.aodh_url,
          ), remainder
      ...
  ```

  Pecan的`_lookup()`方法是controller中的一个特殊方法，Pecan会在特定的时候调用这个方法来实现更灵活的URL路由。Pecan还支持用户实现`_default()`和`_route()`方法。这些方法的具体说明，请阅读Pecan的文档：[routing](https://pecan.readthedocs.org/en/latest/routing.html)。

  我们这里只用到`_lookup()`方法，这个方法会在controller中没有其他方法可以执行且没有`_default()`方法的时候执行。

  `_lookup()`方法需要返回一个元组，元组的第一个元素是下一个controller的实例，第二个元素是URL path中剩余的部分。

  以查询某个resources为例，执行命令

  ceilometer resource-show 028def79-06e3-417a-aa7c-e1da648e8edb

  在api日志中可以看到请求为：

  GET /v2/resources/028def79-06e3-417a-aa7c-e1da648e8edb

  按照上述分析，则会路由RootController--->v2.V2Controller

  在V2Controller中没有resources对象也没有_default()函数，因此会执行\_lookup()，对应于kind为resources，如果没有开启gnocchi，则调用resources.ResourcesController()来处理

  get_one函数是pecan定义获取单个对象的接口，因此会调用get_one进行处理。


### 4.注意点：

-   需要注意的修饰类

  ```
  pecan.decorators.expose(template=None,
                          content_type='text/html',
                          generic=False)
  ```

  其中**template**参数用来指定返回值的模板，如果是'json'就会返回JSON内容，这里可以指定一个HTML文件，或者指定一个mako模板。

  *content_type*指定响应的content-type，默认值是'text/html'。

  generic参数表明该方法是一个“泛型”方法，可以指定多个不同的函数对应同一个路径的不同的HTTP方法。

  - WSME，WSME的全称是**Web Service Made Easy**，是专门用于实现REST服务的typing库，让你不需要直接操作请求和响应，和Pecan结合得非常好，所以OpenStack的很多项目都使用了Pecan + WSME的组合来实现API。WSME的理念是：在大部分情况下，Web服务的输入和输出对数据类型的要求都是严格的，用于请求参数和响应内容的类型检查。

  - WSME中主要使用两个控制器：

    - @signature: 这个装饰器用来描述一个函数的输入和输出。
    - @wsexpose: 这个装饰器包含@signature的功能，同时会把函数的路由信息暴露给Web框架，效果就像Pecan的expose装饰器。

  - 常见到的装饰类为

    @wsexpose(int, int)

    表示如果提供的参数为空或者不是整型，访问就会失败：


