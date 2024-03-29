# RabbitMQ tutorial - Topics

### (using the Pika Python client)

### Prerequisites

上一篇教程里，我们改进了我们的日志系统。我们使用直连交换机替代了扇型交换机，从只能盲目的广播消息改进为有可能选择性的接收日志。

尽管直连交换机能够改善我们的系统，但是它也有它的限制 —— 没办法基于多个标准执行路由操作。

在我们的日志系统中，我们不只希望订阅基于严重程度的日志，同时还希望订阅基于发送来源的日志。Unix工具syslog就是同时基于严重程度-severity (info/warn/crit...) 和 设备-facility (auth/cron/kern...)来路由日志的。

如果这样的话，将会给予我们非常大的灵活性，我们既可以监听来源于“cron”的严重程度为“critical errors”的日志，也可以监听来源于“kern”的所有日志。

为了实现这个目的，接下来我们学习如何使用另一种更复杂的交换机 —— 主题交换机。

### Topic exchange

发送到主题交换机（topic exchange）的消息不可以携带随意什么样子的路由键（routing_key），它的路由键必须是一个由.分隔开的词语列表。这些单词随便是什么都可以，但是最好是跟携带它们的消息有关系的词汇。以下是几个推荐的例子：`stock.usd.nyse`, `nyse.vmw`, `quick.orange.rabbit`。词语的个数可以随意，但是不要超过255字节。

绑定键也必须拥有同样的格式。主题交换机背后的逻辑跟直连交换机很相似 —— 一个携带着特定路由键的消息会被主题交换机投递给绑定键与之想匹配的队列。但是它的绑定键和路由键有两个特殊应用方式：

  * `*` (star) can substitute for exactly one word.
  * `#` (hash) can substitute for zero or more words.

It's easiest to explain this in an example:

  <img src="https://gitarticle.oss-cn-shanghai.aliyuncs.com/mq/images/python-five.png" height="170" />

这个例子里，我们发送的所有消息都是用来描述小动物的。发送的消息所携带的路由键是由三个单词所组成的，这三个单词被两个.分割开。路由键里的第一个单词描述的是动物的手脚的利索程度，第二个单词是动物的颜色，第三个是动物的种类。所以它看起来是这样的： `<celerity>.<colour>.<species>`。

我们创建了三个绑定：Q1的绑定键为 `*.orange.*`，Q2的绑定键为 `*.*.rabbit` 和 `lazy.#` 。

这三个绑定键被可以总结为：

  * Q1 is interested in all the orange animals.
  * Q2 wants to hear everything about rabbits, and everything about lazy animals.

一个携带有 `quick.orange.rabbit` 的消息将会被分别投递给这两个队列。携带着 `lazy.orange.elephant` 的消息同样也会给两个队列都投递过去。另一方面携带有 `quick.orange.fox` 的消息会投递给第一个队列，携带有 `lazy.brown.fox` 的消息会投递给第二个队列。携带有 `lazy.pink.rabbit` 的消息只会被投递给第二个队列一次，即使它同时匹配第二个队列的两个绑定。携带着 `quick.brown.fox` 的消息不会投递给任何一个队列。

如果我们违反约定，发送了一个携带有一个单词或者四个单词（`orange` or `quick.orange.male.rabbit`）的消息时，发送的消息不会投递给任何一个队列，而且会丢失掉。

但是另一方面，即使 `lazy.orange.male.rabbit` 有四个单词，他还是会匹配最后一个绑定，并且被投递到第二个队列中。

> #### Topic exchange
>
> 主题交换机是很强大的，它可以表现出跟其他交换机类似的行为
> 
> 当一个队列的绑定键为`#`的时候，这个队列将会无视消息的路由键，接收所有的消息。
> 
> 当 `*` 和 `#` 这两个特殊字符都未在绑定键中出现的时候，此时主题交换机就拥有的直连交换机的行为。

### Putting it all together
-----------------------

接下来我们会将主题交换机应用到我们的日志系统中。在开始工作前，我们假设日志的路由键由两个单词组成，路由键看起来是这样的："`<facility>.<severity>`".

The code is almost the same as in the
[previous tutorial](tutorial-four-python.html).

`emit_log_topic.py` ([source](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/python/emit_log_topic.py))

```python
#!/usr/bin/env python
import pika
import sys

connection = pika.BlockingConnection(
    pika.ConnectionParameters(host='localhost'))
channel = connection.channel()

channel.exchange_declare(exchange='topic_logs', exchange_type='topic')

routing_key = sys.argv[1] if len(sys.argv) > 2 else 'anonymous.info'
message = ' '.join(sys.argv[2:]) or 'Hello World!'
channel.basic_publish(
    exchange='topic_logs', routing_key=routing_key, body=message)
print(" [x] Sent %r:%r" % (routing_key, message))
connection.close()
```

`receive_logs_topic.py` ([source](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/python/receive_logs_topic.py))

```python
#!/usr/bin/env python
import pika
import sys

connection = pika.BlockingConnection(
    pika.ConnectionParameters(host='localhost'))
channel = connection.channel()

channel.exchange_declare(exchange='topic_logs', exchange_type='topic')

result = channel.queue_declare('', exclusive=True)
queue_name = result.method.queue

binding_keys = sys.argv[1:]
if not binding_keys:
    sys.stderr.write("Usage: %s [binding_key]...\n" % sys.argv[0])
    sys.exit(1)

for binding_key in binding_keys:
    channel.queue_bind(
        exchange='topic_logs', queue=queue_name, routing_key=binding_key)

print(' [*] Waiting for logs. To exit press CTRL+C')


def callback(ch, method, properties, body):
    print(" [x] %r:%r" % (method.routing_key, body))


channel.basic_consume(
    queue=queue_name, on_message_callback=callback, auto_ack=True)

channel.start_consuming()
```

执行下边命令 接收所有日志：

<pre class="lang-bash">
python receive_logs_topic.py "#"
</pre>

To receive all logs from the facility "`kern`":

<pre class="lang-bash">
python receive_logs_topic.py "kern.*"
</pre>

Or if you want to hear only about "`critical`" logs:

<pre class="lang-bash">
python receive_logs_topic.py "*.critical"
</pre>

You can create multiple bindings:

<pre class="lang-bash">
python receive_logs_topic.py "kern.*" "*.critical"
</pre>

And to emit a log with a routing key "`kern.critical`" type:

<pre class="lang-bash">
python emit_log_topic.py "kern.critical" "A critical kernel error"
</pre>

如果你现在还没被搞晕，想想下边问题:

- 绑定键为 `*` 的队列会取到一个路由键为空的消息吗？
- 绑定键为 `#.*` 的队列会获取到一个名为`..`的路由键的消息吗？它会取到一个路由键为单个单词的消息吗？
- `a.*.#` 和 `a.#`的区别在哪儿？

Have fun playing with these programs. Note that the code doesn't make
any assumption about the routing or binding keys, you may want to play
with more than two routing key parameters.

Move on to [tutorial 6](tutorial-six-python.html) to learn about *RPC*.
