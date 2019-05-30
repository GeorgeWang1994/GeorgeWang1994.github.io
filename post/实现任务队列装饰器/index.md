
# 实现任务队列装饰器

## 总结



![image](https://wx2.sinaimg.cn/large/007FyU7Tly1g3dviq38puj30sh0dwwn3.jpg)



## 实现



```python
#!/usr/bin/env python
# -*- coding:utf-8 -*-

"""
@author:    george wang
@datetime:  2019-05-25
@file:      task.py
@contact:   georgewang1994@163.com
@desc:      实现task的装饰器
"""

import random


def get_unique_id(count=8):
    return "".join([random.choice('abcdefghijklmnopqrstuvwxyz1234567890') for _ in range(count)])


class Task(object):
    def __init__(self, task_id, *args, **kwargs):
        self.task_id = task_id
        self.args = args
        self.kwargs = kwargs

    @property
    def parameter(self):
        return self.args, self.kwargs

    def execute(self):
        return NotImplemented


class Result(object):
    def __init__(self, task_id, return_value, *args, **kwargs):
        self.task_id = task_id
        self.return_value = return_value

    def get(self):
        return self.return_value


class TaskWrapper(object):
    base_task_class = Task

    def __init__(self, func, *args, **kwargs):
        self.func = func
        self.task_id = get_unique_id()
        self.task_class = self.create_task_class(**kwargs)

    def create_task_class(self, **kwargs):
        execute_func = self.func

        def execute(self):
            args, kwargs = self.parameter
            value = execute_func(*args, **kwargs)
            return value

        settings = {
            'task_id': self.task_id,
            'execute': execute,
            '__doc__': execute_func.__doc__,
            '__module': execute_func.__module__,
        }
        settings.update(kwargs)
        return type(self.func.__name__, (self.base_task_class, ), settings)

    def enqueue(self, task):
        value = task.execute()
        print("doc: %s\nmodule: %s\nvalue: %s" % (task.__doc__, task.__module__, value))
        return Result(task.task_id, value)

    def __call__(self, *args, **kwargs):
        return self.enqueue(self.task_class(self.task_id, *args, *kwargs))


class Ammonia(object):
    @classmethod
    def task(cls, *args, **kwargs):
        def decorator(func):
            return TaskWrapper(func, *args, **kwargs)
        return decorator


@Ammonia.task()
def get_sum(num1, num2):
    """
    计算两者的和
    :param num1:
    :param num2:
    :return:
    """
    return num1 + num2


if __name__ == '__main__':
    result = get_sum(1, 2)
    print(result.get())
```

