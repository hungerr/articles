# AMQP 0-9-1 Model Explained

本文提供了AMQP 0-9-1协议的一个概述，它是RabbitMQ所支持的协议之一。

## High-level Overview of AMQP 0-9-1 and the AMQP Model

### What is AMQP 0-9-1?

AMQP 0-9-1 (Advanced Message Queuing Protocol) is a messaging
protocol that enables conforming client applications to
communicate with conforming messaging middleware brokers.

AMQP 0-9-1（Advanced Message Queuing Protocol）是一个消息协议。它支持符合要求的客户端应用（application）和消息中间件代理（messaging middleware broker）之间进行通信。

### Brokers and Their Role

Messaging brokers receive messages from publishers
(applications that publish them, also known as producers) and route them to consumers(applications that process them).

消息代理（message brokers）从发布者（publishers）亦称生产者（producers）那儿接收消息，并根据既定的路由规则把接收到的消息发送给处理消息的消费者（consumers）。

Since it is a network protocol, the publishers,
consumers and the broker can all reside on
different machines.

由于AMQP是一个网络协议，所以这个过程中的发布者，消费者，消息代理 可以存在于不同的设备上。

### AMQP 0-9-1 Model in Brief

The AMQP 0-9-1 Model has the following view of the world:
messages are published to _exchanges_,
which are often compared to post offices or mailboxes. Exchanges then
distribute message copies to _queues_ using rules called
_bindings_. Then the broker either deliver messages to
consumers subscribed to queues, or consumers
fetch/pull messages from queues on demand.

AAMQP 0-9-1的工作过程如下图：消息（message）被发布者（publisher）发送给交换机（exchange），交换机常常被比喻成邮局或者邮箱。然后交换机将收到的消息根据路由规则分发给绑定的队列（queue）。最后AMQP代理会将消息投递给订阅了此队列的消费者，或者消费者按照需求自行获取。

<img src="./images/hello-world-example-routing.png" alt="Publish path from publisher to consumer via exchange and queue" />

When publishing a message, publishers may specify various
_message attributes_ (message meta-data). Some of this
meta-data may be used by the broker, however, the rest of it
is completely opaque to the broker and is only used by
applications that receive the message.

当发布一条消息时，publishers可以指定多个消息属性（message meta-data）。部分的meta-data可能会被broker使用，但是剩下消息对broker是完全不透明的，仅仅会被接收消息的应用程序使用。

Networks are unreliable and applications may fail to process
messages therefore the AMQP 0-9-1 model has a notion of
_message acknowledgements_: when a message is delivered
to a consumer the consumer _notifies the broker_,
either automatically or as soon as the application
developer chooses to do so. When message acknowledgements
are in use, a broker will only completely remove a message
from a queue when it receives a notification for that message
(or group of messages).

网络不稳定，应用程序可能会无法处理消息，因此AMQP model有一个消息确认的概念：当一条消息被传输到consumer，consumer自动通知broker或者应用程序开发人员选择这么做。当消息确认被使用时，broker只在接收到该消息（群消息）的通知时才会从queue中完全移除该消息。

In certain situations, for example, when a message cannot
be routed, messages may be _returned_ to publishers,
dropped, or, if the broker implements an extension,
placed into a so-called "dead letter queue". Publishers choose
how to handle situations like this by publishing messages
using certain parameters.

在某些场合中，比如当一条消息不能被路由时，消息可能会被返回给publishers，或者被丢弃，或者如果broker实现了一个扩展将消息放在“dead letter queue”中。publishers通过使用某些参数发布消息选择如何处理这些情况

Queues, exchanges and bindings are collectively referred to as _AMQP entities_.

queues，exchanges与bindings统称为AMQP实体

### AMQP 0-9-1 is a Programmable Protocol

AMQP 0-9-1 is a programmable protocol in the sense that AMQP 0-9-1
entities and routing schemes are primarily defined by applications
themselves, not a broker administrator. Accordingly,
provision is made for protocol operations that declare
queues and exchanges, define bindings between them,
subscribe to [queues](/queues.html) and so on.

This gives application developers a lot of freedom but
also requires them to be aware of potential definition
conflicts. In practice, definition conflicts are rare and
often indicate a misconfiguration.

Applications declare the AMQP 0-9-1 entities that they need,
define necessary routing schemes and may choose to delete
AMQP 0-9-1 entities when they are no longer used.

AMQP 0-9-1是一个可编程协议，某种意义上说AMQP的实体和路由规则是由应用本身定义的，而不是由消息代理定义。包括像声明队列和交换机，定义他们之间的绑定，订阅队列等等关于协议本身的操作。

这虽然能让开发人员自由发挥，但也需要他们注意潜在的定义冲突。当然这在实践中很少会发生，如果发生，会以配置错误（misconfiguration）的形式表现出来。

应用程序（Applications）声明AMQP实体，定义需要的路由方案，或者删除不再需要的AMQP实体。

## Exchanges and Exchange Types

_Exchanges_ are AMQP 0-9-1 entities where messages are
sent. Exchanges take a message and route it into zero or
more queues. The routing algorithm used depends on the
_exchange type_ and rules called _bindings_.
AMQP 0-9-1 brokers provide four exchange types:

交换机是用来发送消息的AMQP实体。交换机拿到一个消息之后将它路由给一个或零个队列。它使用哪种路由算法是由交换机类型和被称作绑定（bindings）的规则所决定的。AMQP 0-9-1的代理提供了四种交换机:

<table>
  <thead>
    <th>Name</th>
    <th>Default pre-declared names</th>
  </thead>
  <tr>
    <td>Direct exchange</td>
    <td>(Empty string) and amq.direct</td>
  </tr>
  <tr>
    <td>Fanout exchange</td>
    <td>amq.fanout</td>
  </tr>
  <tr>
    <td>Topic exchange</td>
    <td>amq.topic</td>
  </tr>
  <tr>
    <td>Headers exchange</td>
    <td>amq.match (and amq.headers in RabbitMQ)</td>
  </tr>
</table>

Besides the exchange type, exchanges are declared with a number of
attributes, the most important of which are:

 * Name
 * Durability (exchanges survive broker restart)
 * Auto-delete (exchange is deleted when last queue is unbound from it)
 * Arguments (optional, used by plugins and broker-specific features)

Exchanges can be durable or transient. Durable
exchanges survive broker restart whereas transient
exchanges do not (they have to be redeclared
when broker comes back online). Not all scenarios
and use cases require exchanges to be durable.


除exchange类型之外，exchange声明了一系列的属性，最重要的如下：

 * Name
 * Durability（broker重启时exchanges是否survive）
 * Auto-delete（当所有与之绑定的消息队列都完成了对此交换机的使用后，删掉它）
 * Arguments（这些都是broker相关的）

交换机可以有两个状态：持久（durable）、暂存（transient）。持久化的交换机会在消息代理（broker）重启后依旧存在，而暂存的交换机则不会（它们需要在代理再次上线后重新被声明）。然而并不是所有的应用场景都需要持久化的交换机。

### Default Exchange

The default exchange is a direct exchange with no name
(empty string) pre-declared by the broker. It has one special
property that makes it very useful for simple applications:
every queue that is created is automatically bound to it with
a routing key which is the same as the queue name.

For example, when you declare a queue with the name of
"search-indexing-online", the AMQP 0-9-1 broker will bind it
to the default exchange using "search-indexing-online" as
the routing key (in this context sometimes referred to as the binding key).
Therefore, a message published to the default
exchange with the routing key "search-indexing-online"
will be routed to the queue "search-indexing-online". In
other words, the default exchange makes it seem like it
is possible to deliver messages directly to queues, even
though that is not technically what is happening.

默认交换机（default exchange）实际上是一个由消息代理预先声明好的没有名字（名字为空字符串）的直连交换机（direct exchange）。它有一个特殊的属性使得它对于简单应用特别有用处：那就是每个新建队列（queue）都会自动绑定到默认交换机上，绑定的路由键（routing key）名称与队列名称相同。

举个栗子：当你声明了一个名为"search-indexing-online"的队列，AMQP代理会自动将其绑定到默认交换机上，绑定（binding）的路由键名称也是为"search-indexing-online"。因此，当携带着名为"search-indexing-online"的路由键的消息被发送到默认交换机的时候，此消息会被默认交换机路由至名为"search-indexing-online"的队列中。换句话说，默认交换机看起来貌似能够直接将消息投递给队列，尽管技术上并没有做相关的操作。

### Direct Exchange

A direct exchange delivers messages to queues based on the
message routing key. A direct exchange is ideal for the unicast
routing of messages (although they can be used for multicast
routing as well). Here is how it works:

 * A queue binds to the exchange with a routing key K
 * When a new message with routing key R arrives at the
   direct exchange, the exchange routes it to the queue if K = R

Direct exchanges are often used to distribute tasks between
multiple workers (instances of the same application) in
a round robin manner. When doing so, it is important to
understand that, in AMQP 0-9-1, messages are load balanced
between consumers and not between queues.

A direct exchange can be represented graphically as follows:

直连型交换机（direct exchange）是根据消息携带的路由键（routing key）将消息投递给对应队列的。直连交换机用来处理消息的单播路由（unicast routing）（尽管它也可以处理多播路由）。下边介绍它是如何工作的：

 * 一个routing key为K的queue与exchange进行绑定
 * 当一条新的routing key为R的消息到达direct exchange时，exchange 将它路由至该queue如果K=R

直连交换机经常用来循环分发任务给多个工作者（workers）。当这样做的时候，我们需要明白一点，在AMQP 0-9-1中，消息的负载均衡是发生在消费者（consumer）之间的，而不是队列（queue）之间。

direct exchange可以用图形方式表示如下：

<img src="./images/exchange-direct.png" alt="exchange delivering messages to  queues based on routing key" />

### Fanout Exchange

A fanout exchange routes messages to all of the queues
that are bound to it and the routing key is ignored. If N
queues are bound to a fanout exchange, when a new message
is published to that exchange a copy of the message is
delivered to all N queues. Fanout exchanges are ideal for
the broadcast routing of messages.

Because a fanout exchange delivers a copy of a message to
every queue bound to it, its use cases are quite similar:

 * Massively multi-player online (MMO) games can use it
   for leaderboard updates or other global events
 * Sport news sites can use fanout exchanges for
   distributing score updates to mobile clients in near
   real-time
 * Distributed systems can broadcast various state and
   configuration updates
 * Group chats can distribute messages between participants
   using a fanout exchange (although AMQP does not have a
   built-in concept of presence, so XMPP may be a better choice)

A fanout exchange can be represented graphically as follows:

扇型交换机（funout exchange）将消息路由给绑定到它身上的所有队列，而不理会绑定的路由键。如果N个队列绑定到某个扇型交换机上，当有消息发送给此扇型交换机时，交换机会将消息的拷贝分别发送给这所有的N个队列。扇型用来交换机处理消息的广播路由（broadcast routing）。

因为一个fanout exchange传送消息的副本到每一个与其绑定的queue，它的使用情况很相似：

- 大量的多用户在线（multi-player online MMO）游戏使用它更新排行榜或者其他的全体事件
- 体育新闻网站使用fanout exchange向手机客户端实时发送比分更新
- 分布式系统可以广播各种状态与配置更新
- 在群聊的时候，它被用来分发消息给参与群聊的用户。（AMQP没有内置presence的概念，因此XMPP可能会是个更好的选择）

一个fanout exchange图形化的表述如下：

<img src="./images/exchange-fanout.png" alt="exchange delivering messages to three queues" />

### Topic Exchange

Topic exchanges route messages to one or many queues
based on matching between a message routing key and the
pattern that was used to bind a queue to an exchange. The
topic exchange type is often used to implement various
publish/subscribe pattern variations. Topic exchanges are
commonly used for the multicast routing of messages.

Topic exchanges have a very broad set of use cases. Whenever
a problem involves multiple consumers/applications that
selectively choose which type of messages they want to
receive, the use of topic exchanges should be considered.

Example uses:

* Distributing data relevant to specific geographic
  location, for example, points of sale
* Background task processing done by multiple workers,
  each capable of handling specific set of tasks
* Stocks price updates (and updates on other kinds of financial data)
* News updates that involve categorization or tagging
  (for example, only for a particular sport or team)
* Orchestration of services of different kinds in the cloud
* Distributed architecture/OS-specific software builds or
  packaging where each builder can handle only one architecture or OS

主题交换机（topic exchanges）通过对消息的路由键和队列到交换机的绑定模式之间的匹配，将消息路由给一个或多个队列。主题交换机经常用来实现各种分发/订阅模式及其变种。主题交换机通常用来实现消息的多播路由（multicast routing）。

主题交换机拥有非常广泛的用户案例。无论何时，当一个问题涉及到那些想要有针对性的选择需要接收消息的 多消费者/多应用（multiple consumers/applications） 的时候，主题交换机都可以被列入考虑范围。

使用示例：

- 销售与特定地理位置相关的数据，比如销售点
- 由多个工作者（workers）完成的后台任务，每个工作者负责处理某些特定的任务
- 股票价格更新（以及其他类型的金融数据更新）
- 包含分类与标签的新闻更新（例如只针对某一个特定的运动或团队）
- 云端的不同种类服务的协调
- 分布式架构/基于系统的软件封装，其中每个构建者仅能处理一个特定的架构或者系统。

### Headers Exchange

A headers exchange is designed for routing on multiple
attributes that are more easily expressed as message
headers than a routing key. Headers exchanges ignore the
routing key attribute. Instead, the attributes used for
routing are taken from the headers attribute. A message is
considered matching if the value of the header equals the
value specified upon binding.

It is possible to bind a queue to a headers exchange using
more than one header for matching. In this case, the broker
needs one more piece of information from the application
developer, namely, should it consider messages with any
of the headers matching, or all of them? This is what the
"x-match" binding argument is for. When the "x-match"
argument is set to "any", just one matching header value
is sufficient. Alternatively, setting "x-match" to "all"
mandates that all the values must match.

Headers exchanges can be looked upon as "direct exchanges
on steroids". Because they route based on header values,
they can be used as direct exchanges where the routing key
does not have to be a string; it could be an integer or a
hash (dictionary) for example.

Note that headers beginning with the string `x-`
will not be used to evaluate matches.

有时消息的路由操作会涉及到多个属性，此时使用消息头就比用路由键更容易表达，头交换机（headers exchange）就是为此而生的。头交换机使用多个消息属性来代替路由键建立路由规则。通过判断消息头的值能否与指定的绑定相匹配来确立路由规则。

我们可以绑定一个队列到头交换机上，并给他们之间的绑定使用多个用于匹配的头（header）。这个案例中，消息代理得从应用开发者那儿取到更多一段信息，换句话说，它需要考虑某条消息（message）是需要部分匹配还是全部匹配。上边说的“更多一段消息”就是"x-match"参数。当"x-match"设置为“any”时，消息头的任意一个值被匹配就可以满足条件，而当"x-match"设置为“all”的时候，就需要消息头的所有值都匹配成功。

头交换机可以视为直连交换机的另一种表现形式。头交换机能够像直连交换机一样工作，不同之处在于头交换机的路由规则是建立在头属性值之上，而不是路由键。路由键必须是一个字符串，而头属性值则没有这个约束，它们甚至可以是整数或者哈希值（字典）等。

## Queues

[Queues](/queues.html) in the AMQP 0-9-1 model are very similar to queues in
other message- and task-queueing systems: they store
messages that are consumed by applications. Queues share
some properties with exchanges, but also have some additional
properties:

 * Name
 * Durable (the queue will survive a broker restart)
 * Exclusive (used by only one connection and the queue will be deleted when that connection closes)
 * Auto-delete (queue that has had at least one consumer is deleted when last consumer unsubscribes)
 * Arguments (optional; used by plugins and broker-specific features such as message TTL, queue length limit, etc)

Before a queue can be used it has to be declared. Declaring
a queue will cause it to be created if it does not already
exist. The declaration will have no effect if the queue does
already exist and its attributes are the same as those in the
declaration. When the existing queue attributes are not the
same as those in the declaration a channel-level exception
with code 406 (`PRECONDITION_FAILED`) will be  raised.

AMQP中的队列（queue）跟其他消息队列或任务队列中的队列是很相似的：它们存储着即将被应用消费掉的消息。队列跟交换机共享某些属性，但是队列也有一些另外的属性。

- Name
- Durable（当broker重启时，queue是否存在）
- Exclusive（只被一个connection使用并且在connection关闭时queue被删除）
- Auto-delete（当最后一个consumer取消订阅时queue被删除）
- Arguments（一些broker使用它去实现message TTL之类的附加功能）

队列在声明（declare）后才能被使用。如果一个队列尚不存在，声明一个队列会创建它。如果声明的队列已经存在，并且属性完全相同，那么此次声明不会对原有队列产生任何影响。如果声明中的属性与已存在队列的属性有差异，那么一个错误代码为406的通道级异常就会被抛出。

### Queue Names

Applications may pick queue names or ask the broker to generate
a name for them.  Queue names may be up to 255 bytes of UTF-8
characters. An AMQP 0-9-1 broker can generate a unique queue
name on behalf of an app. To use this feature, pass an empty
string as the queue name argument. The generated name will be returned
to the client with queue declaration response.

Queue names starting with "amq." are reserved for internal
use by the broker. Attempts to declare a queue with a name that
violates this rule will result in a channel-level exception
with reply code 403 (`ACCESS_REFUSED`).

队列的名字可以由应用（application）来取，也可以让消息代理（broker）直接生成一个。队列的名字可以是最多255字节的一个utf-8字符串。若希望AMQP消息代理生成队列名，需要给队列的name参数赋值一个空字符串：在同一个通道（channel）的后续的方法（method）中，我们可以使用空字符串来表示之前生成的队列名称。之所以之后的方法可以获取正确的队列名是因为通道可以默默地记住消息代理最后一次生成的队列名称。

以"amq."开始的队列名称被预留做消息代理内部使用。如果试图在队列声明时打破这一规则的话，一个通道级的403 (ACCESS_REFUSED)错误会被抛出。

### Queue Durability

Durable queues are persisted to disk and thus survive broker
restarts. Queues that are not durable are called transient.
Not all scenarios and use cases mandate queues to be durable.

Durability of a queue does not make _messages_ that
are routed to that queue durable. If broker is taken down
and then brought back up, durable queue will be re-declared
during broker startup, however, only _persistent_
messages will be recovered.

持久化队列（Durable queues）会被存储在磁盘上，当消息代理（broker）重启的时候，它依旧存在。没有被持久化的队列称作暂存队列（Transient queues）。并不是所有的场景和案例都需要将队列持久化。

持久化的队列并不会使得路由到它的消息也具有持久性。倘若消息代理挂掉了，重新启动，那么在重启的过程中持久化队列会被重新声明，无论怎样，只有经过持久化的消息才能被重新恢复。

## Bindings

Bindings are rules that exchanges use (among other things)
to route messages to queues. To instruct an exchange E to
route messages to a queue Q,
Q has to be _bound_ to E. Bindings may have an optional
_routing key_ attribute used by some exchange types. The
purpose of the routing key is to select certain messages published
to an exchange to be routed to the bound queue. In other words,
the routing key acts like a filter.

To draw an analogy:

 * Queue is like your destination in New York city
 * Exchange is like JFK airport
 * Bindings are routes from JFK to your destination. There
can be zero or many ways to reach it

Having this layer of indirection enables routing scenarios
that are impossible or very hard to implement using publishing
directly to queues and also eliminates certain amount of
duplicated work application developers have to do.

If a message cannot be routed to any queue (for example,
because there are no bindings for the exchange it was published
to) it is either [dropped or returned to the publisher](/publishers.html#unroutable),
depending on message attributes the publisher has set.

绑定（Binding）是交换机（exchange）将消息（message）路由给队列（queue）所需遵循的规则。如果要指示交换机“E”将消息路由给队列“Q”，那么“Q”就需要与“E”进行绑定。绑定操作需要定义一个可选的路由键（routing key）属性给某些类型的交换机。路由键的意义在于从发送给交换机的众多消息中选择出某些消息，将其路由给绑定的队列。

画一个类比：

- queue就像是你在纽约的目的地
- Exchanges就像是JFK机场
- Bindings是JFK到你目的地的路线。有0或者多条路线到达目的地

拥有了交换机这个中间层，很多由发布者直接到队列难以实现的路由方案能够得以实现，并且避免了应用开发者的许多重复劳动。

如果AMQP的消息无法路由到队列（例如，发送到的交换机没有绑定队列），消息会被就地销毁或者返还给发布者。如何处理取决于发布者设置的消息属性。

## Consumers

Storing messages in queues is useless unless applications
can _[consume](/consumers.html)_ them. In the AMQP 0-9-1 Model, there
are two ways for applications to do this:

 * Have messages delivered to them ("push API")
 * Fetch messages as needed ("pull API")

With the "push API", applications have to indicate interest in
consuming messages from a particular queue. When they do so,
we say that they _register a consumer_
or, simply put, _subscribe to a queue_. It is possible
to have more than one consumer per queue or to register an
_exclusive consumer_ (excludes all other consumers from
the queue while it is consuming).

Each consumer (subscription) has an identifier called a
_consumer tag_. It can be used to unsubscribe from
messages. Consumer tags are just strings.

消息如果只是存储在队列里是没有任何用处的。被应用消费掉，消息的价值才能够体现。在AMQP 0-9-1 模型中，有两种途径可以达到此目的：

- 将消息传送给它们（push API）
- 按照需要拉取消息（pull API）

使用push API，应用（application）需要明确表示出它在某个特定队列里所感兴趣的，想要消费的消息。如是，我们可以说应用注册了一个消费者，或者说订阅了一个队列。一个队列可以注册多个消费者，也可以注册一个独享的消费者（当独享消费者存在时，其他消费者即被排除在外）。

每个消费者（订阅者）都有一个叫做消费者标签的标识符。它可以被用来退订消息。消费者标签实际上是一个字符串。

### Message Acknowledgements

[Consumer applications](/consumers.html) – that is, applications that receive and process
messages – may occasionally fail to process individual
messages or will sometimes just crash. There is also the possibility
of network issues causing problems. This raises a question:
when should the broker remove messages from queues? The
AMQP 0-9-1 specification gives consumers control over this. There are
two [acknowledgement modes](/confirms.html):

 * After broker sends a message to an application (using either `basic.deliver` or `basic.get-ok` method).
 * After the application sends back an acknowledgement (using the `basic.ack` method).

The former choice is called the automatic acknowledgement
model, while the latter is called the explicit acknowledgement
model. With the explicit model the application chooses when
it is time to send an acknowledgement. It can be right after
receiving a message, or after persisting it to a data store
before processing, or after fully processing the message
(for example, successfully fetching a Web page, processing
and storing it into some persistent data store).

If a consumer dies without sending an acknowledgement,
the broker will redeliver it to another consumer or,
if none are available at the time, the broker will wait until
at least one consumer is registered for the same queue before
attempting redelivery.

消费者应用（Consumer applications） - 用来接受和处理消息的应用 - 在处理消息的时候偶尔会失败或者有时会直接崩溃掉。而且网络原因也有可能引起各种问题。这就给我们出了个难题，AMQP代理在什么时候删除消息才是正确的？AMQP 0-9-1 规范给我们两种建议：

- 当消息代理（broker）将消息发送给应用后立即删除。（使用AMQP方法：basic.deliver或basic.get-ok）
- 待应用（application）发送一个确认回执（acknowledgement）后再删除消息。（使用AMQP方法：basic.ack）

前者被称作自动确认模式（automatic acknowledgement model），后者被称作显式确认模式（explicit acknowledgement model）。在显式模式下，由消费者应用来选择什么时候发送确认回执（acknowledgement）。应用可以在收到消息后立即发送，或将未处理的消息存储后发送，或等到消息被处理完毕后再发送确认回执（例如，成功获取一个网页内容并将其存储之后）。

如果一个消费者在尚未发送确认回执的情况下挂掉了，那AMQP代理会将消息重新投递给另一个消费者。如果当时没有可用的消费者了，消息代理会死等下一个注册到此队列的消费者，然后再次尝试投递。

### Rejecting Messages

When a consumer application receives a message, processing
of that message may or may not succeed. An application can
indicate to the broker that message processing has failed
(or cannot be accomplished at the time) by rejecting a
message. When rejecting a message, an application can ask
the broker to discard or requeue it. When there is only one
consumer on a queue, make sure you do not create infinite
message delivery loops by rejecting and requeueing a message
from the same consumer over and over again.

当一个消费者接收到某条消息后，处理过程有可能成功，有可能失败。应用可以向消息代理表明，本条消息由于“拒绝消息（Rejecting Messages）”的原因处理失败了（或者未能在此时完成）。当拒绝某条消息时，应用可以告诉消息代理如何处理这条消息——销毁它或者重新放入队列。当此队列只有一个消费者时，请确认不要由于拒绝消息并且选择了重新放入队列的行为而引起消息在同一个消费者身上无限循环的情况发生。

### Negative Acknowledgements

Messages are rejected with the `basic.reject` method.
There is one limitation that `basic.reject` has:
there is no way to reject multiple messages as you can do
with acknowledgements. However, if you are using RabbitMQ,
then there is a solution. RabbitMQ provides an AMQP 0-9-1
extension known as _negative acknowledgements_ or _nacks_. For
more information, please refer to the [Confirmations](/confirms.html)
and <a href="">basic.nack extension</a> guides.

在AMQP中，basic.reject方法用来执行拒绝消息的操作。但basic.reject有个限制：你不能使用它决绝多个带有确认回执（acknowledgements）的消息。但是如果你使用的是RabbitMQ，那么你可以使用被称作negative acknowledgements（也叫nacks）的AMQP 0-9-1扩展来解决这个问题。更多的信息请参考帮助页面。 

### Prefetching Messages

For cases when multiple consumers share a queue, it is useful
to be able to specify how many messages each consumer can be
sent at once before sending the next acknowledgement. This can
be used as a simple load balancing technique or to improve
throughput if messages tend to be published in batches. For
example, if a producing application sends messages every
minute because of the nature of the work it is doing.

Note that RabbitMQ only supports channel-level prefetch-count,
not connection or size based prefetching.

在多个消费者共享一个队列的案例中，明确指定在收到下一个确认回执前每个消费者一次可以接受多少条消息是非常有用的。这可以在试图批量发布消息的时候起到简单的负载均衡和提高消息吞吐量的作用。For example, if a producing application sends messages every minute because of the nature of the work it is doing.（？？？例如，如果生产应用每分钟才发送一条消息，这说明处理工作尚在运行。）

注意，RabbitMQ只支持通道级的预取计数，而不是连接级的或者基于大小的预取。

## Message Attributes and Payload

Messages in the AMQP 0-9-1 model have _attributes_. Some
attributes are so common that the AMQP 0-9-1 specification
defines them and application developers do not have to think
about the exact attribute name. Some examples are

 * Content type
 * Content encoding
 * Routing key
 * Delivery mode (persistent or not)
 * Message priority
 * Message publishing timestamp
 * Expiration period
 * Publisher application id

Some attributes are used by AMQP brokers, but most are open
to interpretation by applications that receive them. Some
attributes are optional and known
as _headers_. They are similar to X-Headers in
HTTP. Message attributes are set when a message is published.

Messages also have a _payload_ (the data that
they carry), which AMQP brokers treat as an opaque byte array.
The broker will not inspect or modify the payload. It is possible
for messages to contain only attributes and no payload. It
is common to use serialisation formats like JSON, Thrift,
Protocol Buffers and MessagePack to serialize structured data
in order to publish it as the message payload. Protocol peers
typically use the "content-type" and "content-encoding" fields to
communicate this information,  but this is by convention only.

Messages may be published as persistent, which makes the broker persist them to disk.
If the server is restarted the system ensures that received persistent messages are not
lost. Simply publishing a message to a durable exchange or the
fact that the queue(s) it is routed to are durable doesn't make
a message persistent: it all depends on persistence mode of the
message itself. Publishing messages as persistent affects
performance (just like with data stores, durability comes at a
certain cost in performance).

Learn more in the [Publishers guide](/publishers.html).

AMQP模型中的消息（Message）对象是带有属性（Attributes）的。有些属性及其常见，以至于AMQP 0-9-1 明确的定义了它们，并且应用开发者们无需费心思思考这些属性名字所代表的具体含义。例如：

- Content type（内容类型）
- Content encoding（内容编码）
- Routing key（路由键）
- Delivery mode (persistent or not)
- 投递模式（持久化 或 非持久化）
- Message priority（消息优先权）
- Message publishing timestamp（消息发布的时间戳）
- Expiration period（消息有效期）
- Publisher application id（发布应用的ID）

有些属性是被AMQP代理所使用的，但是大多数是开放给接收它们的应用解释器用的。有些属性是可选的也被称作消息头（headers）。他们跟HTTP协议的X-Headers很相似。消息属性需要在消息被发布的时候定义。

AMQP的消息除属性外，也含有一个有效载荷 - Payload（消息实际携带的数据），它被AMQP代理当作不透明的字节数组来对待。消息代理不会检查或者修改有效载荷。消息可以只包含属性而不携带有效载荷。它通常会使用类似JSON这种序列化的格式数据，为了节省，协议缓冲器和MessagePack将结构化数据序列化，以便以消息的有效载荷的形式发布。AMQP及其同行者们通常使用"content-type" 和 "content-encoding" 这两个字段来与消息沟通进行有效载荷的辨识工作，但这仅仅是基于约定而已。

消息能够以持久化的方式发布，AMQP代理会将此消息存储在磁盘上。如果服务器重启，系统会确认收到的持久化消息未丢失。简单地将消息发送给一个持久化的交换机或者路由给一个持久化的队列，并不会使得此消息具有持久化性质：它完全取决与消息本身的持久模式（persistence mode）。将消息以持久化方式发布时，会对性能造成一定的影响（就像数据库操作一样，健壮性的存在必定造成一些性能牺牲）。

## Message Acknowledgements

Since networks are unreliable and applications fail,
it is often necessary to have some kind of processing
acknowledgement. Sometimes it is only necessary to
acknowledge the fact that a message has been received. Sometimes
acknowledgements mean that a message was validated and processed
by a consumer, for example, verified as having mandatory data
and persisted to a data store or indexed.

This situation is very common, so AMQP 0-9-1 has a built-in
feature called _message acknowledgements_ (sometimes
referred to as _acks_) that consumers use to confirm
message delivery and/or processing. If an application crashes
(the AMQP broker notices this when the connection is closed), if
an acknowledgement for a message was expected but not received
by the AMQP broker, the message is re-queued (and possibly
immediately delivered to another consumer, if any exists).

Having acknowledgements built into the protocol helps
developers to build more robust software.

由于网络的不确定性和应用失败的可能性，处理确认回执（acknowledgement）就变的十分重要。有时我们确认消费者收到消息就可以了，有时确认回执意味着消息已被验证并且处理完毕，例如对某些数据已经验证完毕并且进行了数据存储或者索引操作。

这种情形很常见，所以 AMQP 0-9-1 内置了一个功能叫做 消息确认（message acknowledgements），消费者用它来确认消息已经被接收或者处理。如果一个应用崩溃掉（此时连接会断掉，所以AMQP代理亦会得知），而且消息的确认回执功能已经被开启，但是消息代理尚未获得确认回执，那么消息会被从新放入队列（并且在还有还有其他消费者存在于此队列的前提下，立即投递给另外一个消费者）。

协议内置的消息确认功能将帮助开发者建立强大的软件。

## AMQP 0-9-1 Methods

AMQP 0-9-1 is structured as a number of
_methods_. Methods are operations (like HTTP methods)
and have nothing in common with methods in object-oriented
programming languages. Protocol methods in AMQP 0-9-1 are grouped into
_classes_. Classes are just logical groupings of AMQP
methods. The [AMQP 0-9-1 reference](/amqp-0-9-1-reference.html) has full details of all the AMQP methods.

Let us take a look at the _exchange_ class, a group
of methods related to operations on exchanges. It includes
the following operations:

 * `exchange.declare`
 * `exchange.declare-ok`
 * `exchange.delete`
 * `exchange.delete-ok`

(note that the RabbitMQ site reference also includes
RabbitMQ-specific extensions to the exchange class that we
will not discuss in this guide).

The operations above form logical pairs:
`exchange.declare` and
`exchange.declare-ok`,
`exchange.delete` and
`exchange.delete-ok`. These operations are
"requests" (sent by clients) and "responses" (sent by
brokers in response to the aforementioned "requests").

As an example, the client asks the broker to declare a new
exchange using the `exchange.declare` method:

AMQP 0-9-1由许多方法（methods）构成。方法即是操作，这跟面向对象编程中的方法没半毛钱关系。AMQP的方法被分组在类（class）中。这里的类仅仅是对AMQP方法的逻辑分组而已。在 AMQP 0-9-1参考 中有对AMQP方法的详细介绍。

让我们来看看交换机类，有一组方法被关联到了交换机的操作上。这些方法如下所示：

- exchange.declare
- exchange.declare-ok
- exchange.delete
- exchange.delete-ok

（请注意，RabbitMQ网站参考中包含了特用于RabbitMQ的交换机类的扩展，这里我们不对其进行讨论）

以上的操作来自逻辑上的配对：exchange.declare 和 exchange.declare-ok，exchange.delete 和 exchange.delete-ok. 这些操作分为“请求 - requests”（由客户端发送）和“响应 - responses”（由代理发送，用来回应之前提到的“请求”操作）。

如下的例子：客户端要求消息代理使用exchange.declare方法声明一个新的交换机：

<img src="./images/exchange-declare.png" alt="exchange.declare" />

As shown on the diagram above,
`exchange.declare` carries several
_parameters_. They enable the client to specify
exchange name, type, durability flag and so on.

If the operation succeeds, the broker responds with the `exchange.declare-ok` method:

如上图所示，exchange.declare方法携带了好几个参数。这些参数可以允许客户端指定交换机名称、类型、是否持久化等等。

操作成功后，消息代理使用exchange.declare-ok方法进行回应：

<img src="./images/exchange-declare-ok.png" alt="exchange.declare-ok" />

`exchange.declare-ok` does not carry any
parameters except for the channel number (channels will be
described later in this guide).

The sequence of events is very similar for another
method pair on the AMQP 0-9-1 _queue_ method class: `queue.declare` and
`queue.declare-ok`:

exchange.declare-ok方法除了通道号之外没有携带任何其他参数（通道-channel 会在本指南稍后章节进行介绍）。

AMQP队列类的配对方法 - queue.declare方法 和 queue.declare-ok有着与其他配对方法非常相似的一系列事件：

<img src="./images/queue-declare.png" alt="queue.declare" />

<img src="./images/queue-declare-ok.png" alt="queue.declare-ok" />

Not all AMQP 0-9-1 methods have counterparts. Some
(`basic.publish` being the most widely used one)
do not have corresponding "response" methods
and some others (`basic.get`, for example)
have more than one possible "response".

不是所有的AMQP方法都有与其配对的“另一半”。许多（basic.publish是最被广泛使用的）都没有相对应的“响应”方法，另外一些（如basic.get）有着一种以上与之对应的“响应”方法。

## Connections

AMQP 0-9-1 connections are typically long-lived. AMQP 0-9-1 is an
application level protocol that uses TCP for reliable
delivery. Connections use authentication and can be
protected using TLS. When an application no longer needs
to be connected to the server, it should gracefully close
its AMQP 0-9-1 connection instead of abruptly closing the underlying TCP connection.

AMQP连接通常是长连接。AMQP是一个使用TCP提供可靠投递的应用层协议。AMQP使用认证机制并且提供TLS（SSL）保护。当一个应用不再需要连接到AMQP代理的时候，需要优雅的释放掉AMQP连接，而不是直接将TCP连接关闭。

## Channels

Some applications need multiple connections to the
broker. However, it is undesirable to keep many TCP
connections open at the same time because doing so consumes
system resources and makes it more difficult to configure
firewalls. AMQP 0-9-1 connections are multiplexed with
_[channels](/channels.html)_ that can be thought of as "lightweight
connections that share a single TCP connection".

Every protocol operation performed by a client happens on a channel.
Communication on a particular channel is completely separate
from communication on another channel, therefore every protocol
method also carries a channel ID (a.k.a. channel number), an integer
that both the broker and clients use to figure out which channel the method is for.

A channel only exists in the context of a connection and never on its own.
When a connection is closed, so are all channels on it.

For applications that use multiple threads/processes for
processing, it is very common to open a new channel per thread/process
and not share channels between them.

有些应用需要与AMQP代理建立多个连接。无论怎样，同时开启多个TCP连接都是不合适的，因为这样做会消耗掉过多的系统资源并且使得防火墙的配置更加困难。AMQP 0-9-1提供了通道（channels）来处理多连接，可以把通道理解成共享一个TCP连接的多个轻量化连接。

在涉及多线程/进程的应用中，为每个线程/进程开启一个通道（channel）是很常见的，并且这些通道不能被线程/进程共享。

一个特定通道上的通讯与其他通道上的通讯是完全隔离的，因此每个AMQP方法都需要携带一个通道号，这样客户端就可以指定此方法是为哪个通道准备的。

## Virtual Hosts

To make it possible for a single broker to host multiple
isolated "environments" (groups of users, exchanges,
queues and so on), AMQP 0-9-1 includes the concept of _[virtual hosts](/vhosts.html)_ (vhosts).
They are similar to virtual hosts used by many popular Web servers and provide completely isolated
environments in which AMQP entities live. Protocol clients
specify what vhosts they want to use during connection negotiation.

为了在一个单独的代理上实现多个隔离的环境（用户、用户组、交换机、队列 等），AMQP提供了一个虚拟主机（virtual hosts - vhosts）的概念。这跟Web servers虚拟主机概念非常相似，这为AMQP实体提供了完全隔离的环境。当连接被建立的时候，AMQP客户端来指定使用哪个虚拟主机。

## AMQP is Extensible

AMQP 0-9-1 has several extension points:

 * [Custom exchange types](/devtools.html#miscellaneous) let developers
   implement routing schemes that exchange types provided out-of-the-box do
   not cover well, for example, geodata-based routing.
 * Declaration of exchanges and queues can include additional attributes that the broker
   can use. For example, [per-queue message TTL](/ttl.html) in RabbitMQ is implemented this way.
 * Broker-specific extensions to the protocol. See, for example,
   [extensions that RabbitMQ implements](/extensions.html).
 * [New AMQP 0-9-1 method classes](/amqp-0-9-1-quickref.html#class.confirm) can be introduced.
 * Brokers can be extended with [additional plugins](/plugins.html),
   for example, the [RabbitMQ management](/management.html)
   frontend and HTTP API are implemented as a plugin.

These features make the AMQP 0-9-1 Model even more flexible
and applicable to a very broad range of problems.

AMQP 0-9-1 拥有多个扩展点：

- 定制化交换机类型 可以让开发者们实现一些开箱即用的交换机类型尚未很好覆盖的路由方案。例如 geodata-based routing。
- 交换机和队列的声明中可以包含一些消息代理能够用到的额外属性。例如RabbitMQ中的per-queue message TTL即是使用该方式实现。
- 特定消息代理的协议扩展。例如RabbitMQ所实现的扩展。
- 新的 AMQP 0-9-1 方法类可被引入。
- 消息代理可以被其他的插件扩展，例如RabbitMQ的管理前端 和 已经被插件化的HTTP API。

这些特性使得AMQP 0-9-1模型更加灵活，并且能够适用于解决更加宽泛的问题。

## AMQP 0-9-1 Clients Ecosystem

There are [many AMQP 0-9-1 clients](/devtools.html) for many
popular programming languages and platforms. Some of them follow AMQP terminology
closely and only provide implementation of AMQP methods. Some
others have additional features, convenience methods
and abstractions. Some of the clients are asynchronous
(non-blocking), some are synchronous (blocking), some support
both models. Some clients support vendor-specific extensions
(for example, RabbitMQ-specific extensions).

Because one of the main AMQP goals is interoperability, it is
a good idea for developers to understand protocol operations
and not limit themselves to terminology of a particular client
library. This way communicating with developers using different
libraries will be significantly easier.

AMQP 0-9-1 拥有众多的适用于各种流行语言和框架的客户端。其中一部分严格遵循AMQP规范，提供AMQP方法的实现。另一部分提供了额外的技术，方便使用的方法和抽象。有些客户端是异步的（非阻塞的），有些是同步的（阻塞的），有些将这两者同时实现。有些客户端支持“供应商的特定扩展”（例如RabbitMQ的特定扩展）。

因为AMQP的主要目标之一就是实现交互性，所以对于开发者来讲，了解协议的操作方法而不是只停留在弄懂特定客户端的库就显得十分重要。这样一来，开发者使用不同类型的库与协议进行沟通时就会容易的多。

### 参考

- [AMQP 0-9-1 Model Explained](https://www.rabbitmq.com/tutorials/amqp-concepts.html)