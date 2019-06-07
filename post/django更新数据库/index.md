
# Django更新数据库



## 背景



最近发现在代码中使用了非常多的`select_for_update`，因此想测试一下这么写会不会有性能影响，因此做了一些的实验的比较。



## 思考



写一个更新数据库的操作实现一样的功能，提供以下几种的写法：

1. 先`get`再`update`；
2. 直接使用`select_for_update`（这么写是错误的❌）；
3. 使用事务，再使用`select_for_update`；
4. 使用事务，先`get`再`update`；
5. `filter`后直接`update`；
6. 数据库语句直接`update`；



## 实现



### 数据库操作层面

```python
#!/usr/bin/env python
# -*- coding:utf-8 -*-

"""
@author:    george wang
@datetime:  2019-06-03
@file:      db_manager.py
@contact:   georgewang1994@163.com
@desc:      数据库操作
"""

import time

from django.db import connection, transaction
from django.db.models import F

from sql.models import Book


def get_book_safe_db(book_id):
    """

    :param book_id:
    :return:
    """
    try:
        return Book.objects.get(id=book_id)
    except Book.DoesNotExist:
        return None


def update_book_count_db1(book_id):
    """
    :param book_id:
    :return:
    """
    start_time = time.time()
    book = get_book_safe_db(book_id)
    if not book:
        return False

    book.read_num += 1
    book.save()
    end_time = time.time()
    print("view1 花费的总时间 :%s" % (end_time - start_time))
    return True


def update_book_count_db2(book_id):
    """
    :param book_id:
    :return:
    """
    try:
        book = Book.objects.select_for_update().get(id=book_id)
    except Book.DoesNotExist:
        return None

    book.read_num += 1
    book.save()
    return True


def update_book_count_db3(book_id):
    """
    :param book_id:
    :return:
    """
    start_time = time.time()
    with transaction.atomic():
        try:
            book = Book.objects.select_for_update().get(id=book_id)
        except Book.DoesNotExist:
            return None

        book.read_num += 1
        book.save()
        end_time = time.time()
        print("view3 花费的总时间 :%s" % (end_time - start_time))
        return True


def update_book_count_db4(book_id):
    """
    :param book_id:
    :return:
    """
    start_time = time.time()
    with transaction.atomic():
        try:
            book = Book.objects.get(id=book_id)
        except Book.DoesNotExist:
            return None

        book.read_num = F('read_num') + 1
        book.save()
        end_time = time.time()
        print("view4 花费的总时间 :%s" % (end_time - start_time))
        return True


def update_book_count_db5(book_id):
    """
    :param book_id:
    :return:
    """
    start_time = time.time()
    Book.objects.filter(id=book_id).update(read_num=F('read_num') + 1)
    end_time = time.time()
    print("view5 花费的总时间 :%s" % (end_time - start_time))
    return True


def update_book_count_db6(book_id):
    """
    :param book_id:
    :return:
    """
    start_time = time.time()
    cursor = connection.cursor()
    result = cursor.execute(f"UPDATE sql_book SET read_num = read_num + 1 WHERE id = { book_id }")
    end_time = time.time()
    print("view6 花费的总时间 :%s" % (end_time - start_time))
    return True


def create_book_db(name):
    return Book.objects.create(name=name)


def delete_book_db(book_id):
    Book.objects.filter(id=book_id).delete()


def delete_all_book_db():
    Book.objects.all().delete()
```



### 测试

```python
#!/usr/bin/env python
# -*- coding:utf-8 -*-

"""
@author:    george wang
@datetime:  2019-06-03
@file:      testsql.py
@contact:   georgewang1994@163.com
@desc:      测试
"""

import os

import django

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "test_sql.settings")  # project_name 项目名称
django.setup()

import requests

from sql.manager.db_manager import *
from sql.thread import generate_thread


def request_update_view(view_idx, book_id):
    count = 0
    while count < 1:
      	# view2写的有问题。。
        if view_idx == 2:
            break

        resp = requests.post(f"http://127.0.0.1:8000/book/read/update{view_idx}/", data={"book_id": book_id})
        result = resp.json()

        if not result.get('success'):
            break

        count += 1


if __name__ == "__main__":

    delete_all_book_db()
    book_list = []

    for i in range(6):
        book = create_book_db("book_%s" % i)

        if len(book_list) >= 6:
            continue

        book_list.append(book)

    thread_list = []
    for idx, book in enumerate(book_list, start=1):
        if idx == 2:
            continue

        thread_sub_list = generate_thread(thread_num=1, name="view_%s" % idx, target=request_update_view, args=(idx, book.id))
        thread_list.extend(thread_sub_list)

    for thread in thread_list:
        thread.start()

    for thread in thread_list:
        thread.join()

    for idx, book in enumerate(book_list, start=1):
        book = get_book_safe_db(book.id)
        print('book: %s, view: %s, read_num: %s' % (book.id, idx, book.read_num))
```



## 结果



在执行的时间消耗方面和最终的结果方面的比较如下：

```
view1 花费的总时间 :0.01191091537475586
view2 花费的总时间 :0
view3 花费的总时间 :0.013716936111450195
view4 花费的总时间 :0.012083053588867188
view5 花费的总时间 :0.009126901626586914
view6 花费的总时间 :0.011317968368530273


book: 198006, view: 1, read_num: 954
book: 198007, view: 2, read_num: 0
book: 198008, view: 3, read_num: 1000
book: 198009, view: 4, read_num: 1000
book: 198010, view: 5, read_num: 1000
book: 198011, view: 6, read_num: 1000
```



可以发现加了锁的执行时间都普遍比较高，反而直接操作数据库的会比较快一些，但是像是在`第一种写法`中，会出现`并发导致数据库不一致`的情况，至于为什么在第一种写法会出现问题，原因很简单，函数`update_book_count_db1`中将更新操作分成两步，先获取，后更新，这就有可能会出现多个线程都执行完了第一步，接下来分别进行更新数据，而这个数据是旧的数据，自然就会出现错误。



至于前公司都这么使用，我想可能是因为加了`缓存`的缘故，因此等我把`函数缓存`、`model缓存`完成的时候再测试一下。



至于为什么`select_for_update`可以保证一致性，这是因为其是个`悲观锁`，当进行更新数据的时候，如果没有明确的指定`主键`，那么`Innodb`就会进行表锁，反之，如果有明确的`主键`，则只进行`行锁`。需要注意⚠️的是，该语法只能配合事务一起使用，并且仅使用在`Innodb`上，因为`MyAsim`不支持行级锁，而`Innodb`默认就是行级锁。

