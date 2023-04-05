## Kombu源码分析(一)概述

Celery是Python中最流行的异步消息队列框架，支持RabbitMQ、Redis、ZoopKeeper等作为Broker，而对这些消息队列的抽象，都是通过[Kombu](https://github.com/celery/kombu)实现的。Kombu实现了对AMQP transport和non-AMQP transports(Redis、Amazon SQS、ZoopKeeper等)的兼容。

AMQP中的各种概念，Message、Producer、Exchange、Queue、Consumer、Connection、Channel在Kombu中都相应做了实现，另外Kombu还实现了Transport，就是存储和发送消息的实体，用来区分底层消息队列是用amqp、Redis还是其它实现的。

- Message：消息，发送和消费的主体
- Producer: 消息发送者
- Consumer：消息接收者
- Exchange：交换机，消息发送者将消息发至Exchange，Exchange负责将消息分发至队列
- Queue：消息队列，存储着即将被应用消费掉的消息，Exchange负责将消息分发Queue，消费者从Queue接收消息
- Connection：对消息队列连接的抽象
- Channel：与AMQP中概念类似，可以理解成共享一个Connection的多个轻量化连接
- Transport：真实的MQ连接，区分底层消息队列的实现

对于不同的Transport的支持：

![](https://gitarticle.oss-cn-shanghai.aliyuncs.com/mq/images/kombu-1.png)

### 代码示例

先从官网示例代码开始:

```python
from kombu import Connection, Exchange, Queue

media_exchange = Exchange('media', 'direct', durable=True)
video_queue = Queue('video', exchange=media_exchange, routing_key='video')

def process_media(body, message):
    print body
    message.ack()

# connections
with Connection('amqp://guest:guest@localhost//') as conn:

    # produce
    producer = conn.Producer(serializer='json')
    producer.publish({'name': '/tmp/lolcat1.avi', 'size': 1301013},
                      exchange=media_exchange, routing_key='video',
                      declare=[video_queue])

    # the declare above, makes sure the video queue is declared
    # so that the messages can be delivered.
    # It's a best practice in Kombu to have both publishers and
    # consumers declare the queue. You can also declare the
    # queue manually using:
    #     video_queue(conn).declare()

    # consume
    with conn.Consumer(video_queue, callbacks=[process_media]) as consumer:
        # Process messages and handle events on all channels
        while True:
            conn.drain_events()

# Consume from several queues on the same channel:
video_queue = Queue('video', exchange=media_exchange, key='video')
image_queue = Queue('image', exchange=media_exchange, key='image')

with connection.Consumer([video_queue, image_queue],
                         callbacks=[process_media]) as consumer:
    while True:
        connection.drain_events()
```

基本上，各种角色都出场了。各种角色的使用都要从建立Connection开始。

### Connection

获取连接很简单：
```python
>>> from kombu import Connection
>>> connection = Connection('amqp://guest:guest@localhost:5672//')
```

现在的连接其实并未真正建立，只有在需要使用的时候才真正建立连接并将连接缓存：

```python
@property
def connection(self):
    """The underlying connection object.
    Warning:
        This instance is transport specific, so do not
        depend on the interface of this object.
    """
    if not self._closed:
        if not self.connected:
            self.declared_entities.clear()
            self._default_channel = None
            self._connection = self._establish_connection()
            self._closed = False
        return self._connection
```

也可以主动连接：
```python
>>> connection.connect()
```

```python
def connect(self):
    """Establish connection to server immediately."""
    self._closed = False
    return self.connection
```

当然，连接底层是由各自使用的不同的`Transport`建立的：
```python
conn = self.transport.establish_connection() 
```

连接需要显式的关闭：
```python
>>> connection.release()
```

由于`Connection`实现了上下文生成器：
```python
def __enter__(self):
    return self

def __exit__(self, *args):
    self.release()
```

所以可以使用with语句，以免忘记关闭连接：
```
with Connection() as connection:
    # work with connection
```

可以使用`Connection`直接建立`Procuder`和`Consumer`，其实就是调用了各自的创建类：

```python
def Producer(self, channel=None, *args, **kwargs):
    """Create new :class:`kombu.Producer` instance."""
    from .messaging import Producer
    return Producer(channel or self, *args, **kwargs)

def Consumer(self, queues=None, channel=None, *args, **kwargs):
    """Create new :class:`kombu.Consumer` instance."""
    from .messaging import Consumer
    return Consumer(channel or self, queues, *args, **kwargs)
```

### Producer

连接创建后，可以使用连接创建`Producer`：
```
producer = conn.Producer(serializer='json')
```

也可以直接使用Channel创建：
```
with connection.channel() as channel:
    producer = Producer(channel, ...)
```

`Producer`实例初始化的时候会检查第一个`channel`参数：
```python
self.revive(self.channel)
channel = self.channel = maybe_channel(channel)
```

这里会检查`channel`是不是`Connection`实例，是的话会将其替换为`Connection`实例的`default_channel`属性：
```
def maybe_channel(channel):
    """Get channel from object.
    Return the default channel if argument is a connection instance,
    otherwise just return the channel given.
    """
    if is_connection(channel):
        return channel.default_channel
    return channel
```

所以`Producer`还是与`Channel`联系在一起的。

`Producer`发送消息：
```PYTHON
producer.publish({'name': '/tmp/lolcat1.avi', 'size': 1301013},
                  exchange=media_exchange, routing_key='video',
                  declare=[video_queue])
```

`pulish`做的事情，主要是由`Channel`完成的：
```python
def _publish(self, body, priority, content_type, content_encoding,
┆   ┆   ┆   ┆headers, properties, routing_key, mandatory,
┆   ┆   ┆   ┆immediate, exchange, declare):
┆   channel = self.channel
┆   message = channel.prepare_message(
┆   ┆   body, priority, content_type,
┆   ┆   content_encoding, headers, properties,
┆   )   
┆   if declare:
┆   ┆   maybe_declare = self.maybe_declare
┆   ┆   [maybe_declare(entity) for entity in declare]

┆   # handle autogenerated queue names for reply_to
┆   reply_to = properties.get('reply_to')
┆   if isinstance(reply_to, Queue):
┆   ┆   properties['reply_to'] = reply_to.name
┆   return channel.basic_publish(
┆   ┆   message,
┆   ┆   exchange=exchange, routing_key=routing_key,
┆   ┆   mandatory=mandatory, immediate=immediate,
┆   )
```
`Channel`组装消息`prepare_message`，并且发送消息`basic_publish`。

而`Channel`又是`Transport`创建的：
```PYTHON
chan = self.transport.create_channel(self.connection)
```

### Transport

当创建`Connection`时，需要传入`hostname`，类似于：
```
amqp://guest:guest@localhost:5672//
```
然后获取`hostname`的`scheme`，比如`redis`:
```python
transport = transport or urlparse(hostname).scheme
```
以此来区分创建的`Transport`的类型。

创建过程：
```python
self.transport_cls = transport

transport_cls = get_transport_cls(transport_cls)

def get_transport_cls(transport=None):
    """Get transport class by name.

    The transport string is the full path to a transport class, e.g.::

    ┆   "kombu.transport.pyamqp:Transport"

    If the name does not include `"."` (is not fully qualified),
    the alias table will be consulted.
    """
    if transport not in _transport_cache:
    ┆   _transport_cache[transport] = resolve_transport(transport)
    return _transport_cache[transport]

transport = TRANSPORT_ALIASES[transport]

TRANSPORT_ALIASES = {
    ...

    'redis': 'kombu.transport.redis:Transport',
    
    ...
}
```

以`Redis`为例，`Transport`类在`/kombu/transport/redis.py`文件，继承自`/kombu/transport/virtual/base.py`中的`Transport`类。

创建`Channel`:
```
channel = self.Channel(connection)
```

然后`Channel`组装消息`prepare_message`，并且发送消息`basic_publish`。

### Channel

`Channel`实例有几个属性关联着Consumer、Queue等，`virtual.Channel`：

```python
class Channel(AbstractChannel, base.StdChannel):
    def __init__(self, connection, **kwargs):
        self.connection = connection
        self._consumers = set()
        self._cycle = None 
        self._tag_to_queue = {} 
        self._active_queues = [] 
        ... 
```

其中，`_consumers`是相关联的消费者标签集合，`_active_queues`是相关联的Queue列表，`_tag_to_queue`则是消费者标签与Queue的映射：
```python
self._tag_to_queue[consumer_tag] = queue
self._consumers.add(consumer_tag)
self._active_queues.append(queue)
```

`Channel`对于不同的底层消息队列，也有不同的实现，以`Redis`为例：
```python
class Channel(virtual.Channel):
    """Redis Channel."""

```
继承自`virtual.Channel`。

组装消息函数`prepare_message`:

```python
def prepare_message(self, body, priority=None, content_type=None,
┆   ┆   ┆   ┆   ┆   content_encoding=None, headers=None, properties=None):
┆   """Prepare message data."""
┆   properties = properties or {}
┆   properties.setdefault('delivery_info', {})
┆   properties.setdefault('priority', priority or self.default_priority)

┆   return {'body': body,
┆   ┆   ┆   'content-encoding': content_encoding,
┆   ┆   ┆   'content-type': content_type,
┆   ┆   ┆   'headers': headers or {},
┆   ┆   ┆   'properties': properties or {}}
```

基本上是为消息添加各种属性。

发送消息`basic_publish`方法是调用`_put`方法：
```python
def _put(self, queue, message, **kwargs):
┆   """Deliver message."""
┆   pri = self._get_message_priority(message, reverse=False)

┆   with self.conn_or_acquire() as client:
┆   ┆   client.lpush(self._q_for_pri(queue, pri), dumps(message))

```

`client`是一个`redis.StrictRedis`连接：
```python
def _create_client(self, asynchronous=False):
┆   if asynchronous:
┆   ┆   return self.Client(connection_pool=self.async_pool)
┆   return self.Client(connection_pool=self.pool)

self.Client = self._get_client()

def _get_client(self):
┆   if redis.VERSION < (3, 2, 0):
┆   ┆   raise VersionMismatch(
┆   ┆   ┆   'Redis transport requires redis-py versions 3.2.0 or later. '
┆   ┆   ┆   'You have {0.__version__}'.format(redis))
┆   return redis.StrictRedis
```

`Redis`将消息置于某个列表(lpush)中。还会根据是否异步的选项选择不同的`connection_pool`。

### Consumer

现在消息已经被放置与队列中，那么消息又被如何使用呢？

`Consumer`初始化需要声明`Channel`和要消费的队列列表以及处理消息的回调函数列表：

```python
with Consumer(connection, queues, callbacks=[process_media], accept=['json']):
    connection.drain_events(timeout=1)
```

当`Consumer`实例被当做上下文管理器使用时，会调用`consume`方法：
```python
def __enter__(self):
    self.consume()
    return self
```

`consume`方法代码：
```python
def consume(self, no_ack=None):
    """Start consuming messages.

    Can be called multiple times, but note that while it
    will consume from new queues added since the last call,
    it will not cancel consuming from removed queues (
    use :meth:`cancel_by_queue`).

    Arguments:
        no_ack (bool): See :attr:`no_ack`.
    """
    queues = list(values(self._queues))
    if queues:
        no_ack = self.no_ack if no_ack is None else no_ack

        H, T = queues[:-1], queues[-1]
        for queue in H:
            self._basic_consume(queue, no_ack=no_ack, nowait=True)
        self._basic_consume(T, no_ack=no_ack, nowait=False)
```
使用`_basic_consume`方法处理相关的队列列表中的每一项，其中处理最后一个Queue时设置标志`nowait=False`。

`_basic_consume`方法代码：
```python
def _basic_consume(self, queue, consumer_tag=None,
                   no_ack=no_ack, nowait=True):
    tag = self._active_tags.get(queue.name)
    if tag is None:
        tag = self._add_tag(queue, consumer_tag)
        queue.consume(tag, self._receive_callback,
                      no_ack=no_ack, nowait=nowait)
    return tag
```
是将消费者标签以及回调函数传给`Queue`的`consume`方法。


`Queue`的`consume`方法代码：
```python
def consume(self, consumer_tag='', callback=None, 
            no_ack=None, nowait=False):     
    """Start a queue consumer.      

    Consumers last as long as the channel they were created on, or
    until the client cancels them.

    Arguments:             
        consumer_tag (str): Unique identifier for the consumer.
            The consumer tag is local to a connection, so two clients
            can use the same consumer tags. If this field is empty
            the server will generate a unique tag.

        no_ack (bool): If enabled the broker will automatically
            ack messages.

        nowait (bool): Do not wait for a reply.

        callback (Callable): callback called for each delivered message.
    """
    if no_ack is None:
        no_ack = self.no_ack            
    return self.channel.basic_consume(
        queue=self.name,
        no_ack=no_ack,
        consumer_tag=consumer_tag or '',
        callback=callback,
        nowait=nowait,
        arguments=self.consumer_arguments)
```

又回到了`Channel`，`Channel`的`basic_consume`代码：
```python
def basic_consume(self, queue, no_ack, callback, consumer_tag, **kwargs):
    """Consume from `queue`."""     
    self._tag_to_queue[consumer_tag] = queue
    self._active_queues.append(queue)

    def _callback(raw_message):         
        message = self.Message(raw_message, channel=self)
        if not no_ack:
            self.qos.append(message, message.delivery_tag)
        return callback(message)

    self.connection._callbacks[queue] = _callback
    self._consumers.add(consumer_tag)

    self._reset_cycle()   
```

`Channel`将`Consumer`标签，`Consumer`要消费的队列，以及标签与队列的映射关系都记录下来，等待循环调用。另外，还通过`Transport`将队列与回调函数列表的映射关系记录下来，以便于从队列中取出消息后执行回调函数。

真正的调用是下面这行代码实现的：
```
connection.drain_events(timeout=1)
```

现在来到`Transport`的`drain_events`方法：

```python
def drain_events(self, connection, timeout=None):
    time_start = monotonic()
    get = self.cycle.get
    polling_interval = self.polling_interval
    if timeout and polling_interval and polling_interval > timeout:
        polling_interval = timeout
    while 1:
        try: 
            get(self._deliver, timeout=timeout)
        except Empty:
            if timeout is not None and monotonic() - time_start >= timeout:
                raise socket.timeout()
            if polling_interval is not None:
                sleep(polling_interval)
        else:
            break
```
看上去是在无限执行`get(self._deliver, timeout=timeout)`

`get`是`self.cycle`的一个方法，`cycle`是一个`FairCycle`实例：
```python
self.cycle = self.Cycle(self._drain_channel, self.channels, Empty)

@python_2_unicode_compatible
class FairCycle(object):
    """Cycle between resources.

    Consume from a set of resources, where each resource gets
    an equal chance to be consumed from.

    Arguments: 
        fun (Callable): Callback to call.
        resources (Sequence[Any]): List of resources.
        predicate (type): Exception predicate.
    """

    def __init__(self, fun, resources, predicate=Exception):
        self.fun = fun
        self.resources = resources
        self.predicate = predicate
        self.pos = 0

    def _next(self):
        while 1:
            try:
                resource = self.resources[self.pos]
                self.pos += 1
                return resource
            except IndexError:
                self.pos = 0
                if not self.resources:
                    raise self.predicate()

    def get(self, callback, **kwargs):
        """Get from next resource."""
        for tried in count(0):  # for infinity
            resource = self._next()
            try:
                return self.fun(resource, callback, **kwargs)
            except self.predicate:
                # reraise when retries exchausted.
                if tried >= len(self.resources) - 1:
                    raise
```

`FairCycle`接受两个参数，`fun`是要执行的函数`fun`，而`resources`作为一个迭代器，每次提供一个item供`fun`调用。

此处的`fun`是`_drain_channel`，`resources`是`channels`:
```
def _drain_channel(self, channel, callback, timeout=None):
    return channel.drain_events(callback=callback, timeout=timeout)
```

`Transport`相关联的每一个channel都要执行`drain_events`。

`Channel`的`drain_events`代码：

```python
def drain_events(self, timeout=None, callback=None):
    callback = callback or self.connection._deliver
    if self._consumers and self.qos.can_consume():
        if hasattr(self, '_get_many'):
            return self._get_many(self._active_queues, timeout=timeout)
        return self._poll(self.cycle, callback, timeout=timeout)
    raise Empty()
```

`_poll`代码：
```python
def _poll(self, cycle, callback, timeout=None):
    """Poll a list of queues for available messages."""
    return cycle.get(callback)
```

又回到了`FairCycle`，`Channel`的`FairCycle`实例：
```python
def _reset_cycle(self):
    self._cycle = FairCycle(
        self._get_and_deliver, self._active_queues, Empty)
```

`_get_and_deliver`方法从队列中取出消息，然后调用`Transport`传递过来的`_deliver`方法：
```python
def _get_and_deliver(self, queue, callback):
    message = self._get(queue)
    callback(message, queue)
```

`_deliver`代码：
```python
def _deliver(self, message, queue):
    if not queue:
        raise KeyError(
            'Received message without destination queue: {0}'.format(
                message))
    try:
        callback = self._callbacks[queue]
    except KeyError:
        logger.warning(W_NO_CONSUMERS, queue)
        self._reject_inbound_message(message)
    else:
        callback(message)
```
做的事情是根据队列取出注册到此队列的回调函数列表，然后对消息执行列表中的所有回调函数。

### 回顾

可见，Kombu中`Channel`和`Transport`非常重要，`Channel`记录了队列列表、消费者列表以及两者的映射关系，而`Transport`记录了队列与回调函数的映射关系。Kombu对所有需要监听的队列`_active_queues`都查询一遍，直到查询完毕或者遇到一个可以使用的Queue，然后就获取消息，回调此队列对应的callback。
