


# 线程安全总结

## 背景



记得之前在面试的过程中的时候，问了一道题目，关于线程安全的：多个线程同时对一个公共的数字进行加1，最后的结果可以保持一致吗？然而当时我并不知道😢。而且最近在自己的开源项目中也用到了线程的队列，因此想说正好给总结一下好了。



## 实验



```python

#!/usr/bin/env python
# -*- coding:utf-8 -*-

"""
@author:    george wang
@datetime:  2019-06-01
@file:      test_threading.py
@contact:   georgewang1994@163.com
@desc:      测试线程
"""

import threading

counter = 0
counter_list = []
counter_lock = threading.Lock()  # 只是定义一个锁,并不是给资源加锁,你可以定义多个锁,像下两行代码,当你需要占用这个资源时，任何一个锁都可以锁这个资源


# 可以使用上边三个锁的任何一个来锁定资源

class MyThread(threading.Thread):  # 使用类定义thread，继承threading.Thread
    def __init__(self, name):
        threading.Thread.__init__(self)
        self.name = "Thread-" + str(name)

    def run(self):  # run函数必须实现
        global counter, counter_list  # 多线程是共享资源的，使用全局变量
        for i in range(100000):
            counter += 1
            counter -= 1
            counter_list.append(i)
            counter_list.pop()

        print("I am %s, set counter:%s, set counter_list:%s" % (self.name, counter, counter_list))


if __name__ == "__main__":
    my_thread_list = []
    for i in range(1, 101):
        my_thread = MyThread(i)
        my_thread.start()
        my_thread_list.append(my_thread)

    for thread in my_thread_list:
        thread.join()

    print("last counter: %s, last counter_list: %s" % (counter, counter_list))
```



得到的结果如下：

```

last counter: 23, last counter_list: []

# last counter 得到的结果每次都不一样
# last counter_list 得到的结果始终是一致
```



## 思考



上面的结果中`counter_list`的结果和我们设想的保持一致，但是`counter`的结果却不是0？



我们先来分析一下他们的原子操作：

```python
from dis import dis

def add_counter():
  global counter
  counter += 1
  
def add_counter_list():
  global counter_list
  counter_list.append(1)

print(dis(add_counter))
print(dis(add_counter_list))
```



得到的结果如下：



![image](https://wx3.sinaimg.cn/large/007FyU7Tly1g3lvu9j2jlj318m0mo49v.jpg)



可以发现`counter += 1`包含了两个步骤，分别是`+1`和`赋值`的操作，并不是一个原子操作，也就说，当多个线程在执行的过程中，在这两个步骤中的任意时刻线程被切换成其他的线程，那么数据就会打乱，举例如下：



![image](https://wx1.sinaimg.cn/large/007FyU7Tly1g3lvuq5s3ij318i0awn11.jpg)



但是可以发现`list.append`操作是原子操作，只有一个操作命令，也就是说这个操作在执行过程中不会切换线程，要么执行成功，要么不执行，因此最终的结果和我们的设想是一致的。



如果要将上述的线程安全的代码改为线程安全的，需要加上锁，对这个公共资源进行保护操作，当一个线程获取到锁的时候才能够操作该资源，并且操作完成后释放锁提供给下一个线程继续使用。



```python
counter_lock = threading.Lock()

# 使用`with`可以自动实现锁的获取和释放
# 相当于counter_lock.acquire()和counter_lock.release()
with counter_lock:
    for i in range(100000):
        counter += 1
        counter -= 1
```



如果需要在多线程环境中使用队列，`theading`模块则提供了`Queuue`保证资源的安全。



>  至于上面的代码中为什么要遍历这么多次，比如遍历100次或者不遍历，那么最终的结果和我们的猜想是一致的。我的猜想是需要对这个资源不断的操作，直至有很大的可能出现在操作这个资源的过程中，线程被切换，如果仅仅执行少的次数或者不执行，那么这个操作会非常快的结束，根本不会出现操作的过程中被切换线程的情况。