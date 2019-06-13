
# Rabbitmq 持久化

## 简介



![398358-20160321171125979-446067508](https://ws4.sinaimg.cn/large/007FyU7Tly1g3z9cjar2ij30ql07l0y2.jpg)



最近需要在自己的项目中需要将消息进行持久化，除了我们所认知的数据库以外，`Rabbitmq`也能够做得到，因此特意在这里总结一下。



## 设置



由于我使用的是`Kombu`，因此下面的设置都是按照这个来的。在默认的`Rabbitmq`实现中，如果相关的配置没有配置成持久化的，那么当连接断开的时候，保存在对列和交换器中的消息则会消失，



```python
1. Connection

不需要设置，和原来的使用一致

2. Exchange

durable=True  # 默认值
auto_delete=False  # 默认值，当没有消息时，自动删除交换器
delivery_mode=PERSISTENT_DELIVERY_MODE(2)  # 默认值

3. Queue

durable=True  # 默认值
exclusive=False  # 默认值，只能被当前连接来消费
auto_delete=False  # 默认值，当没有消息时，自动删除队列

4. Consumer

no_ack=True  # 默认不开启，自动将消息进行确认，有可能在消费者在接收到消息以后还没来得及ack就crash，因此改为手动进行确认

5. Producer

delivery_mode=PERSISTENT_DELIVERY_MODE(2)  # 在public发布消息的时候，将消息设置成持久化
```



需要注意的是，如果已经声明了一个持久化的对列或者交换器，那么就无法再将其更改为非持久化的，只能删除再次重新创建。



## 原理



`Rabbitmq`会维护一个`buffer`，每次都会将消息写入到 buffer 中，如果其已经满了，则将其写入到文件中（但是还未保存到磁盘中），当在每一个`25ms`的间隔 Rabbitmq 都会将 buffer 中的内容以及未保存到磁盘中的文件都存储到磁盘中。或者，当没有后续的消息写入时，也会直接将消息保存到磁盘中。



当所有文件中被删除的消息的比例超过了`阀值`（默认是50%），则会触发将消息的文件合并，提高磁盘的利用率。



## 最后



写入硬盘相比于纯粹写入到内存来说，性能肯定比不上，从而降低了 Rabbitmq 的吞吐量。而且上面并没有考虑到 Rabbitmq 的可靠性。



下次再说吧。

