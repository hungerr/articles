# RabbitMQ tutorial - Routing 路由

### (using the Pika Python client)

### Prerequisites

As with other Python tutorials, we will use the [Pika](https://pypi.python.org/pypi/pika) RabbitMQ client
[version 1.0.0](https://pika.readthedocs.io/en/stable/).

### What This Tutorial Focuses On

在前面的教程中，我们实现了一个简单的日志系统。可以把日志消息广播给多个接收者。

本篇教程中我们打算新增一个功能 —— 使得它能够只订阅消息的一个字集。例如，我们只需要把严重的错误日志信息写入日志文件（存储到磁盘），但同时仍然把所有的日志信息输出到控制台中

### Bindings绑定

前面的例子，我们已经创建过绑定（bindings），代码如下：

```python
channel.queue_bind(exchange=exchange_name,
                   queue=queue_name)
```

绑定（binding）是指交换机（exchange）和队列（queue）的关系。可以简单理解为：这个队列（queue）对这个交换机（exchange）的消息感兴趣。

绑定的时候可以带上一个额外的routing_key参数。为了避免与basic_publish的参数混淆，我们把它叫做绑定键（binding key）。以下是如何创建一个带绑定键的绑定。

```python
channel.queue_bind(exchange=exchange_name,
                   queue=queue_name,
                   routing_key='black')
```

绑定键的意义取决于交换机（exchange）的类型。我们之前使用过的扇型交换机（fanout exchanges）会忽略这个值。

### Direct exchange直连交换机

我们的日志系统广播所有的消息给所有的消费者（consumers）。我们打算扩展它，使其基于日志的严重程度进行消息过滤。例如我们也许只是希望将比较严重的错误（error）日志写入磁盘，以免在警告（warning）或者信息（info）日志上浪费磁盘空间。

我们使用的扇型交换机（fanout exchange）没有足够的灵活性 —— 它能做的仅仅是广播。

我们将会使用直连交换机（direct exchange）来代替。路由的算法很简单 —— 交换机将会对绑定键（binding key）和路由键（routing key）进行精确匹配，从而确定消息该分发到哪个队列。

下图能够很好的描述这个场景：

  <img src="https://gitarticle.oss-cn-shanghai.aliyuncs.com/mq/images/direct-exchange.png" height="170" />

在这个场景中，我们可以看到直连交换机X和两个队列进行了绑定。第一个队列使用orange作为绑定键，第二个队列有两个绑定，一个使用black作为绑定键，另外一个使用green。

这样以来，当路由键为orange的消息发布到交换机，就会被路由到队列Q1。路由键为black或者green的消息就会路由到Q2。其他的所有消息都将会被丢弃。

### Multiple bindings多绑定

  <img src="https://gitarticle.oss-cn-shanghai.aliyuncs.com/mq/images/direct-exchange-multiple.png" height="170" />

多个队列使用相同的绑定键是合法的。这个例子中，我们可以添加一个`X`和`Q1`之间的绑定，使用`black`绑定键。这样一来，直连交换机就和扇型交换机的行为一样，会将消息广播到所有匹配的队列。带有`black`路由键的消息会同时发送到`Q1`和`Q2`。


### Emitting logs发送日志

我们将会发送消息到一个直连交换机，把日志级别作为路由键。这样接收日志的脚本就可以根据严重级别来选择它想要处理的日志。我们先看看发送日志。

我们需要创建一个交换机（exchange）：

```python
channel.exchange_declare(exchange='direct_logs',
                         exchange_type='direct')
```

然后我们发送一则消息：

```python
channel.basic_publish(exchange='direct_logs',
                      routing_key=severity,
                      body=message)
```

我们先假设`severity`的值是`info`、`warning`、`error`中的一个。

### Subscribing订阅

处理接收消息的方式和之前差不多，只有一个例外，我们将会为我们感兴趣的每个严重级别分别创建一个新的绑定。

```python
result = channel.queue_declare(queue='', exclusive=True)
queue_name = result.method.queue

for severity in severities:
    channel.queue_bind(exchange='direct_logs',
                       queue=queue_name,
                       routing_key=severity)
```

### 代码整合

  <img src="https://gitarticle.oss-cn-shanghai.aliyuncs.com/mq/images/python-four.png" height="170" />
 
`emit_log_direct.py` ([source](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/python/emit_log_direct.py))

```python
#!/usr/bin/env python
import pika
import sys

connection = pika.BlockingConnection(
    pika.ConnectionParameters(host='localhost'))
channel = connection.channel()

channel.exchange_declare(exchange='direct_logs', exchange_type='direct')

severity = sys.argv[1] if len(sys.argv) > 1 else 'info'
message = ' '.join(sys.argv[2:]) or 'Hello World!'
channel.basic_publish(
    exchange='direct_logs', routing_key=severity, body=message)
print(" [x] Sent %r:%r" % (severity, message))
connection.close()
```

`receive_logs_direct.py` ([source](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/python/receive_logs_direct.py))

```python
#!/usr/bin/env python
import pika
import sys

connection = pika.BlockingConnection(
    pika.ConnectionParameters(host='localhost'))
channel = connection.channel()

channel.exchange_declare(exchange='direct_logs', exchange_type='direct')

result = channel.queue_declare(queue='', exclusive=True)
queue_name = result.method.queue

severities = sys.argv[1:]
if not severities:
    sys.stderr.write("Usage: %s [info] [warning] [error]\n" % sys.argv[0])
    sys.exit(1)

for severity in severities:
    channel.queue_bind(
        exchange='direct_logs', queue=queue_name, routing_key=severity)

print(' [*] Waiting for logs. To exit press CTRL+C')


def callback(ch, method, properties, body):
    print(" [x] %r:%r" % (method.routing_key, body))


channel.basic_consume(
    queue=queue_name, on_message_callback=callback, auto_ack=True)

channel.start_consuming()
```

如果你希望只是保存`warning`和`error`级别的日志到磁盘，只需要打开控制台并输入：

<pre class="lang-bash">
python receive_logs_direct.py warning error > logs_from_rabbit.log
</pre>

如果你希望所有的日志信息都输出到屏幕中，打开一个新的终端，然后输入：

<pre class="lang-bash">
python receive_logs_direct.py info warning error
# => [*] Waiting for logs. To exit press CTRL+C
</pre>

如果要触发一个error级别的日志，只需要输入：

<pre class="lang-bash">
python emit_log_direct.py error "Run. Run. Or it will explode."
# => [x] Sent 'error':'Run. Run. Or it will explode.'
</pre>

Move on to [tutorial 5](tutorial-five-python.html) to find out how to listen
for messages based on a pattern.
