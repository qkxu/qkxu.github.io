---
layout: post
title: django数据库migrate
category: UI
description: django数据库migrate
---
# django数据库migrate#

### 1.**django数据库表生成过程**

1. 生成数据结构文件

   设计好models以后，可以通过以下命令生成将要同步给数据库的数据结构文件：

    ```
    python manage.py makemigrations 
    ```

   该命令会在app下建立 migrations目录，并记录下你所有的关于modes.py的改动（即数据库操作记录），比如0001_initial.py，但是这个改动还没有作用到数据库文件。

   你可以查看下该migrations会对应于什么样子的SQL命令

   ```
   python manage.py sqlmigrate TestModel 0001
   ```

   大致内容为：

   ```
   BEGIN;
   --
   -- Create model Test
   --
   CREATE TABLE `TestModel_test` (`id` integer AUTO_INCREMENT NOT NULL PRIMARY KEY, `name` varchar(20) NOT NULL);
   COMMIT;
   ```

   ​

2. 写入数据库

   执行

   ```
   python manage.py migrate
   ```

   将该改动作用到数据库文件，比如生成table之类，将数据结构写入数据库。

   那么是不是每一次都重复执行所有的migrations内的文件呢？

   当然不是，每次执行migrate的时候，django会在django_migrations数据表内记录已经执行了的migrations文件。去数据库里查询该表就能看到对应APP里执行了的migrations。

   ```
   mysql> select * from  django_migrations;
   +----+--------------+------------------------------------------+----------------------------+
   | id | app          | name                                     | applied                    |
   +----+--------------+------------------------------------------+----------------------------+
   |  1 | TestModel    | 0001_initial                             | 2018-05-02 06:43:45.788446 |
   |  2 | TestModel2   | 0001_initial                             | 2018-05-02 06:43:46.074193 |
   |  3 | contenttypes | 0001_initial                             | 2018-05-02 06:43:46.549206 |
   |  4 | auth         | 0001_initial                             | 2018-05-02 06:43:52.860644 |
   ```
   ​



### 2.**makemigrations注意点**

在django中由于某些原因可能在model中手动配置app_label，此时需要注意下makemigrations生成的数据库表的方式。例如：

在HelloWorld/TestModel2/models.py

```
from __future__ import unicode_literals

from django.db import models

# Create your models here.
class Test1(models.Model):
    name1 = models.CharField(max_length=20)
    class Meta:
        app_label = 'TestModel'
        db_table = 'TestModel_Test1'
```

此model在TestModel2这个app中，但是app_label是隶属于TestModel，因此执行

```
python manage.py makemigrations TestModel2
```
该model的改变并未被发现，因为它的app为TestModel。

每个model默认的app_label是属于该app本身，除非你显示指明所属app。

执行下列命令会将Test1变化更改到TestModel

```
python manage.py makemigrations TestModel
```

makemigrations后面不加具体的app时，执行会遍历所有app下的model，查看其所属app（如果没有此app，将会被忽略），然后再生成相应的数据库文件。

生成数据库文件后，执行下面命令以手动指定数据库的方式创建表：

```
python manage.py migrate --database resource TestModel
```

至于model的migrate关于多数据库的控制可参考上篇文章[django配置多数据库](https://qkxu.github.io/2018/04/28/django%E9%85%8D%E7%BD%AE%E5%A4%9A%E6%95%B0%E6%8D%AE%E5%BA%93.html)，可以根据数据库路由中的allow_migrate函数来指定。

### 3.重新建表

我们了解了django去数据库内生成表结构的过程后，那么如何清理就很简单了。

 第一步，我们需要清理migrations文件夹内除了\_\_init\_\_.py这个文件外的所有文件。（当然部分清理的时候我们也可以考虑直接修改这个文件。） 

第二步，我们需要清理数据库内django_migrations对应app下的migrations记录。 清理完以后我们再重新做 python manage.py makemigrations 和 python manage.py migrate 就可以重新生成表结构文件了。




参考文档：

[django清理migration终极解决办法](http://www.hylinux.cn/Django/87.html)

[migrate 和makemigrations的差别](https://blog.csdn.net/yang1z1/article/details/52235424) 