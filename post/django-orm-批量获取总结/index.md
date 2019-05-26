
<meta name="referrer" content="no-referrer" />

# Django ORM 批量获取总结

## 背景



其实在上一家的公司的时候，大家在代码Review的时候都建议使用**批量获取**的方法去操作数据库，然后在这家公司的时候，我也就理所当然的继承上一家公司的做法。然而最近在做一个需求的时候，我把原来的代码进行了改动，并称之为"性能优化"，但是说实话，我是打了个疑问的，究竟这么做是不是对于性能的优化🤔️？因此我做了以下的实验。



## 实现



### 基础表

我首先创建了相关的表，如下：

```python

from django.db import models


class EventType(models.Model):
    name = models.CharField(primary_key=True, max_length=50, unique=True)

    def __str__(self):
        return self.name


class Connection(models.Model):
    start_time = models.DateTimeField(editable=False, blank=True, null=True)

    def json(self):
        response = {}
        event_types = [
            {"name": x[0]}
            for x in self.honeypotevent_set.order_by("time").values_list("type__name")
        ]
        response['event_types'] = event_types
        return response

    class Meta:
        ordering = ['-start_time']


class HoneypotEvent(models.Model):
    connection = models.ForeignKey(Connection, on_delete=models.CASCADE)
    type = models.ForeignKey(EventType, models.CASCADE)

    time = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ['-time']
        indexes = [
            models.Index(fields=['type']),
            models.Index(fields=['connection']),
        ]
```



一个Connection存在多个HoneypotEvent，然后这个HoneypotEvent又对应着一个EventType。



### 需求/做法



需求：获取一些Connection的所有的EventType。



按照目前的做法有三种做法：

* 批量获取connection的id，再批量获取honey_event获取event_type；
* 遍历每个connection，再批量获取honey_event获取event_type；
* 通过Ddjango提供的`foreign_key_set`的方法获取event_type；





### 具体测试实现



因此我按照以上的方法都实验了一下，具体的实现如下：

db_mananger.py

```python
#!/usr/bin/env python
# -*- coding:utf-8 -*-

"""
@author:    george wang
@datetime:  2019-05-26
@file:      db_mananger.py
@contact:   georgewang1994@163.com
@desc:      ...
"""

from django.db.models import Q

from connection.models import Connection, EventType, HoneypotEvent


def create_event_type_db(name):
    return EventType.objects.create(name=name)


def create_connection_db():
    return Connection.objects.create()


def create_honeypot_event_db(connection, type):
    return HoneypotEvent.objects.create(connection=connection, type=type)


def get_honeypot_event_qs_db(conn_id_list=None, conn_id=None):
    query = Q()
    if conn_id_list is not None:
        query &= Q(connection_id__in=conn_id_list)

    if conn_id is not None:
        query &= Q(connection_id=conn_id)

    return HoneypotEvent.objects.filter(query)


def get_event_type_qs_db(name_list=None):
    query = Q()
    if name_list is not None:
        query &= Q(name__in=name_list)

    return EventType.objects.filter(query)


def delete_event_type_db():
    EventType.objects.all().delete()


def delete_connection_db():
    Connection.objects.all().delete()


def delete_honeypot_event_db():
    HoneypotEvent.objects.all().delete()
```



testsql.py

```python
#!/usr/bin/env python
# -*- coding:utf-8 -*-

"""
@author:    george wang
@datetime:  2019-05-26
@file:      testsql.py
@contact:   georgewang1994@163.com
@desc:      ...
"""
import os

import django

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "testsql.settings")  # project_name 项目名称
django.setup()

import time
import random
from collections import defaultdict

from connection.manager.db_mananger import create_event_type_db, create_connection_db, create_honeypot_event_db, \
    get_honeypot_event_qs_db, get_event_type_qs_db, delete_connection_db, delete_event_type_db, delete_honeypot_event_db


def get_random_str(count):
    return "".join([random.choice('abcdefghijklmnopqrstuvwxyz') for _ in range(count)])


class TestSql(object):
    def __init__(self):
        self.type_list = []
        self.connection_list = []
        self.honeypot_event_list = []
        self.select_connection_list = []

        delete_connection_db()
        delete_event_type_db()
        delete_honeypot_event_db()
        self.init_data()

    def init_data(self):
        # 创建type20种
        self.type_list = []
        for i in range(20):
            type = create_event_type_db(name=get_random_str(8))
            self.type_list.append(type)

        # 创建connection 100 条
        self.connection_list = []
        for i in range(20):
            connection = create_connection_db()
            self.connection_list.append(connection)

        # 创建honeypot event 1w条
        self.honeypot_event_list = []
        for i in range(100):
            for j in range(100):
                type = self.type_list[random.randint(0, 19)]
                connection = self.connection_list[random.randint(0, 19)]
                honeypot_event = create_honeypot_event_db(type=type, connection=connection)
                self.honeypot_event_list.append(honeypot_event)

        self.select_connection_list = [self.connection_list[0], self.connection_list[1]]

    def get_result_from_foreign_set(self):
        result = []
        for conn in self.select_connection_list:
            result.append(conn.json())

        return result

    def _get_result_from_multi_id(self, honeypot_event_qs):
        result = []
        type_id_list = set(honeypot_event_qs.values_list('type_id', flat=True))
        event_type_qs = get_event_type_qs_db(name_list=type_id_list)
        name_2_event_type = {event_type.name: event_type for event_type in event_type_qs}

        # 获取每个连接id对应的事件列表
        conn_id_2_event_type_list = defaultdict(list)
        for honeypot_event in honeypot_event_qs:
            event_type = name_2_event_type.get(honeypot_event.type_id)
            if not event_type:
                continue

            conn_id_2_event_type_list[honeypot_event.connection_id].append(event_type.name)

        for conn in self.select_connection_list:
            event_type_name_list = conn_id_2_event_type_list.get(conn.id, [])
            event_or_port = []

            for event_type_name in event_type_name_list:
                event_type = name_2_event_type.get(event_type_name)
                if not event_type:
                    continue
                event_or_port.append({
                    "name": event_type.name,
                })

            result.append({"event_types": event_or_port})

        return result

    def get_result_from_per_one_id(self):
        result = []
        for conn in self.select_connection_list:
            honeypot_event_qs = get_honeypot_event_qs_db(conn_id=conn.id).order_by('time')
            result.append(self._get_result_from_multi_id(honeypot_event_qs))
        return result

    def get_result_from_id_list(self):
        conn_id_list = []
        for conn in self.select_connection_list:
            conn_id_list.append(conn.id)

        honeypot_event_qs = get_honeypot_event_qs_db(conn_id_list=conn_id_list).order_by('time')
        return self._get_result_from_multi_id(honeypot_event_qs)


if __name__ == '__main__':

    test = TestSql()

    start_time = time.time()
    result1 = test.get_result_from_foreign_set()
    end_time = time.time()
    print("通过set方法获取的时间为%s, 结果个数为%s" % (end_time - start_time, len(result1)))

    start_time = time.time()
    result2 = test.get_result_from_id_list()
    end_time = time.time()
    print("通过批量获取方法获取的时间为%s, 结果个数为%s" % (end_time - start_time, len(result2)))

    start_time = time.time()
    result3 = test.get_result_from_per_one_id()
    end_time = time.time()
    print("通过每次都去获取方法获取的时间为%s, 结果个数为%s" % (end_time - start_time, len(result3)))
```



### 结果分析



得到的结果是（按照获取结果顺序排序，快的在前面）：

1. 通过Ddjango提供的`foreign_key_set`的方法获取event_type；
2. 批量获取connection的id，再批量获取honey_event获取event_type；
3. 遍历每个connection，再批量获取honey_event获取event_type；



时间如下：

```
1. 最快
通过set方法获取的时间为0.07566714286804199, 结果个数为20
通过set方法获取的时间为0.005182981491088867, 结果个数为2

2. 中等
通过批量获取方法获取的时间为0.16964387893676758, 结果个数为20
通过批量获取方法获取的时间为0.02353692054748535, 结果个数为2

3. 最慢
通过每次都去获取方法获取的时间为0.2293407917022705, 结果个数为20
通过每次都去获取方法获取的时间为0.02280402183532715, 结果个数为2
```



### 分析SQL



接着分析它们的SQL，如下：

```sql
1. 最快
(0.002) SELECT `connection_honeypotevent`.`type_id` FROM `connection_honeypotevent` WHERE `connection_honeypotevent`.`connection_id` = 381 ORDER BY `connection_honeypotevent`.`time` ASC; args=(381,)
(0.002) SELECT `connection_honeypotevent`.`type_id` FROM `connection_honeypotevent` WHERE `connection_honeypotevent`.`connection_id` = 382 ORDER BY `connection_honeypotevent`.`time` ASC; args=(382,)

2. 中等
(0.004) SELECT `connection_honeypotevent`.`type_id` FROM `connection_honeypotevent` WHERE `connection_honeypotevent`.`connection_id` IN (361, 362) ORDER BY `connection_honeypotevent`.`time` ASC; args=(361, 362)
(0.000) SELECT `connection_eventtype`.`name` FROM `connection_eventtype` WHERE `connection_eventtype`.`name` IN ('monfwnof', 'ycxvmndz', 'nnymfggs', 'eatopoct', 'qxhnijzx', 'hwlvcgfl', 'btrijiig', 'poebyuzd', 'cloqczmr', 'xgjqbkpo', 'ujxqhclg', 'dwtkelxl', 'cgowszbt', 'pdfdsmhj', 'lsxixzuw', 'mwepdics', 'vqxinmnv', 'pssfzqox', 'rrmrfkys', 'kbdrsciv'); args=('monfwnof', 'ycxvmndz', 'nnymfggs', 'eatopoct', 'qxhnijzx', 'hwlvcgfl', 'btrijiig', 'poebyuzd', 'cloqczmr', 'xgjqbkpo', 'ujxqhclg', 'dwtkelxl', 'cgowszbt', 'pdfdsmhj', 'lsxixzuw', 'mwepdics', 'vqxinmnv', 'pssfzqox', 'rrmrfkys', 'kbdrsciv')
(0.006) SELECT `connection_honeypotevent`.`id`, `connection_honeypotevent`.`connection_id`, `connection_honeypotevent`.`type_id`, `connection_honeypotevent`.`time` FROM `connection_honeypotevent` WHERE `connection_honeypotevent`.`connection_id` IN (361, 362) ORDER BY `connection_honeypotevent`.`time` ASC; args=(361, 362)


3. 最慢
(0.001) SELECT `connection_honeypotevent`.`type_id` FROM `connection_honeypotevent` WHERE `connection_honeypotevent`.`connection_id` = 301 ORDER BY `connection_honeypotevent`.`time` ASC; args=(301,)
(0.000) SELECT `connection_eventtype`.`name` FROM `connection_eventtype` WHERE `connection_eventtype`.`name` IN ('loplrohi', 'wkdvzher', 'lznynqpl', 'silywdsf', 'zxzudxqf', 'unckucpd', 'sqevfbdn', 'oaxrybvh', 'qlmfhojy', 'oanaxtyi', 'bgoznhoq', 'lwqlcnwe', 'gunhoclr', 'bztsnitx', 'zqfqyrgt', 'nfvgvjiy', 'rlcerncb', 'ysxnppfc', 'edbshymu', 'plnzvzto'); args=('loplrohi', 'wkdvzher', 'lznynqpl', 'silywdsf', 'zxzudxqf', 'unckucpd', 'sqevfbdn', 'oaxrybvh', 'qlmfhojy', 'oanaxtyi', 'bgoznhoq', 'lwqlcnwe', 'gunhoclr', 'bztsnitx', 'zqfqyrgt', 'nfvgvjiy', 'rlcerncb', 'ysxnppfc', 'edbshymu', 'plnzvzto')
(0.004) SELECT `connection_honeypotevent`.`id`, `connection_honeypotevent`.`connection_id`, `connection_honeypotevent`.`type_id`, `connection_honeypotevent`.`time` FROM `connection_honeypotevent` WHERE `connection_honeypotevent`.`connection_id` = 301 ORDER BY `connection_honeypotevent`.`time` ASC; args=(301,)

(0.001) SELECT `connection_honeypotevent`.`type_id` FROM `connection_honeypotevent` WHERE `connection_honeypotevent`.`connection_id` = 302 ORDER BY `connection_honeypotevent`.`time` ASC; args=(302,)
(0.000) SELECT `connection_eventtype`.`name` FROM `connection_eventtype` WHERE `connection_eventtype`.`name` IN ('loplrohi', 'wkdvzher', 'lznynqpl', 'silywdsf', 'zxzudxqf', 'unckucpd', 'sqevfbdn', 'oaxrybvh', 'qlmfhojy', 'oanaxtyi', 'bgoznhoq', 'lwqlcnwe', 'gunhoclr', 'bztsnitx', 'zqfqyrgt', 'nfvgvjiy', 'rlcerncb', 'edbshymu', 'ysxnppfc', 'plnzvzto'); args=('loplrohi', 'wkdvzher', 'lznynqpl', 'silywdsf', 'zxzudxqf', 'unckucpd', 'sqevfbdn', 'oaxrybvh', 'qlmfhojy', 'oanaxtyi', 'bgoznhoq', 'lwqlcnwe', 'gunhoclr', 'bztsnitx', 'zqfqyrgt', 'nfvgvjiy', 'rlcerncb', 'edbshymu', 'ysxnppfc', 'plnzvzto')
(0.002) SELECT `connection_honeypotevent`.`id`, `connection_honeypotevent`.`connection_id`, `connection_honeypotevent`.`type_id`, `connection_honeypotevent`.`time` FROM `connection_honeypotevent` WHERE `connection_honeypotevent`.`connection_id` = 302 ORDER BY `connection_honeypotevent`.`time` ASC; args=(302,)
```



通过查看SQL可以很明显的发现：



1. 最快的方法，直接通过查event_type对应的表查出结果，显然就是最快的；
2. 中等的方法，多次批量的获取消耗的时间比较多，并且中间还需要代码层面的遍历运算；
3. 最慢的方法，显然就是在有多个connection，就要处理多少次，显然就是方法2的n倍，当然最慢；



## 结论



1. 一般比较大的表，通常是不会存外键foreign key，具体原因[这里](https://www.cnblogs.com/mcad/archive/2014/12/28/4190288.html)，通常就单纯记录一个对应表的id，免去了维护造成的性能影响，这个时候也就只能使用批量获取查询的方法；
2. 相对而言比较小的表，可以使用外键foreign key，那么直接用set方法就好。
3. 无论啥情况，都不要全部数据一条条的查询数据库。