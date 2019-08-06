## Kombu源码分析

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

+---------------+----------+------------+------------+---------------+--------------+-----------------------+
| **Client**    | **Type** | **Direct** | **Topic**  | **Fanout**    | **Priority** | **TTL**               |
+---------------+----------+------------+------------+---------------+--------------+-----------------------+
| *amqp*        | Native   | Yes        | Yes        | Yes           | Yes          | Yes                   |
+---------------+----------+------------+------------+---------------+--------------+-----------------------+
| *qpid*        | Native   | Yes        | Yes        | Yes           | No           | No                    |
+---------------+----------+------------+------------+---------------+--------------+-----------------------+
| *redis*       | Virtual  | Yes        | Yes        | Yes (PUB/SUB) | Yes          | No                    |
+---------------+----------+------------+------------+---------------+--------------+-----------------------+
| *mongodb*     | Virtual  | Yes        | Yes        | Yes           | Yes          | Yes                   |
+---------------+----------+------------+------------+---------------+--------------+-----------------------+
| *SQS*         | Virtual  | Yes        | Yes        | Yes           | No           | No                    |
+---------------+----------+------------+------------+---------------+--------------+-----------------------+
| *zookeeper*   | Virtual  | Yes        | Yes        | No            | Yes          | No                    |
+---------------+----------+------------+------------+---------------+--------------+-----------------------+
| *in-memory*   | Virtual  | Yes        | Yes        | No            | No           | No                    |
+---------------+----------+------------+------------+---------------+--------------+-----------------------+
| *SLMQ*        | Virtual  | Yes        | Yes        | No            | No           | No                    |
+---------------+----------+------------+------------+---------------+--------------+-----------------------+
| *Pyro*        | Virtual  | Yes        | Yes        | No            | No           | No                    |
+---------------+----------+------------+------------+---------------+--------------+-----------------------+

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
```PYTHON
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

`Channel`也有各自的实现，以`Redis`为例：
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

而`client`获取：
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

显然是建立一个`redis.StrictRedis`连接，将消息置于某个列表(lpush)中。还会根据是否异步的选项选择不同的`connection_pool`。

### Consumer

