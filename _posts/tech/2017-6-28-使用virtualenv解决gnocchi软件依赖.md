---
layout: post
title: 使用virtualenv解决gnocchi软件依赖
category: telemetry
description: 使用virtualenv解决gnocchi软件依赖
---
# 使用virtualenv解决gnocchi软件依赖#

安装最新版本的gnocchi，gnocchi需要升级许多公共软件，例如oslo.config、oslo.middleware等，这样就会影响到其他组件的正常运行，利用virtualenv搭建独立的执行环境来解决此问题。

virtualenv可以搭建虚拟且独立的python环境，可以使每个项目环境与其他项目独立开来，保持环境的干净，解决包冲突问题。

### 1.**virtualenv 安装**

```
pip install virtualenv
```

或者

```
yum install python-virtualenv
```

### 2.**创建python虚拟环境**

使用virtualenv命令创建python虚拟环境：virtualenv [虚拟环境名称]。

  ```
  virtualenv gncochi-env
  ```
 执行后，在本地会生成一个与虚拟环境同名的文件夹。

如果你的系统里安装有不同版本的python，可以使用--python参数指定虚拟环境的python版本：

```
virtualenv --python=/usr/local/python-2.7.8/bin/python2.7 gncochi-env
```

默认情况下虚拟环境不会依赖系统环境的global site-packages。比如系统环境里安装了MySQLdb模块，在虚拟环境里import MySQLdb会提示ImportError。如果想依赖系统环境的第三方软件包，可以使用参数--system-site-packages。

```
virtualenv --system-site-packages gncochi-env
```

### 3.**启动虚拟环境**

进入虚拟环境目录，启动虚拟环境

```
cd /opt/openstack/gnocchi
source gnocchi-env/bin/activate
```
启动虚环境其实就是利用source改变当前的环境变量

虚环境中默认有安装pip，因此可以直接用pip进行安装，在虚环境中安装的软件不会与外界环境有关联，可以安装与外界环境不同的软件版本，以供gnocchi使用。

```
pip install oslo.config==4.3.0;
pip install oslo.middleware==3.26.0;
pip install pbr==2.0.0;
pip install scipy==0.19.0;
...
```

需要注意的是在虚环境中使用yum安装的软件其实是调用外界的yum，因此安装卸载软件都是安装卸载外界环境上的软件。

### 4.**退出虚拟环境**

在虚环境中执行

```
deactivate
```

### 5.**使用虚拟环境中的软件**

在虚环境中安装好软件之后，软件依赖不会影响到外界环境，当在外部使用虚环境的可执行脚本时，只需要更改可执行脚本所在目录即可：

例如执行：

```
/opt/openstack/gnocchi/gnocchi-env/bin/gnocchi-upgrade
```
### 6.解决WSGI中的启动脚本

gnocchi的api启动方式目前采用的是wsgi方式启动，而不是直接调用gnocchi-api启动脚本：

```
<VirtualHost *:8041>
    WSGIDaemonProcess gnocchi processes=10 threads=10 user=gnocchi display-name=%{GROUP}
    WSGIProcessGroup gnocchi
    WSGIScriptAlias / /opt/openstack/gnocchi/gnocchi-env/lib/python2.7/site-packages/gnocchi/rest/app.wsgi
    WSGIApplicationGroup %{GLOBAL}
    ...
</VirtualHost>
```

app.wsgi中部分代码为：

```
import debtcollector

from gnocchi.rest import app

application = app.build_wsgi_app()
```
导致，在代码调用的时候找不到相关的model。这是因为执行该app.wsgi时使用的Python-path是系统默认的，并不是虚环境中的，而在外部系统并没有安装gnocchi，所以导致找到不到gnocchi.rest 该model。

解决方法：wsgi支持配置不同的Python-path（感觉wsgi还是很强大的）

```
<VirtualHost *:8041>
    WSGIDaemonProcess gnocchi processes=10 threads=10 user=gnocchi display-name=%{GROUP} python-path=/opt/openstack/gnocchi/gnocchi-env/lib/python2.7/site-packages
    WSGIProcessGroup gnocchi
    WSGIScriptAlias / /opt/openstack/gnocchi/gnocchi-env/lib/python2.7/site-packages/gnocchi/rest/app.wsgi
    WSGIApplicationGroup %{GLOBAL}
    ...
</VirtualHost>
```

这样gnocchi的两个进程都使用虚环境中的gnocchi脚本进行运行，完美解决软件版本升级依赖问题。