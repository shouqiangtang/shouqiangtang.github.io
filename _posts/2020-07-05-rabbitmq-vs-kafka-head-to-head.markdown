---
layout: post
title: RabbitMQ vs. Kafka Head-To-Head
date: 2020-07-05 14:51:00 +0800
categories: mq
tags: RabbitMQ和Kafka区别
---

# 介绍

作为一个微服务系统软件架构师，我经常遇到一个不断重复的问题问题：“我应该使用RabbitMQ还是Kafka“，由于某些原因，许多开发者认为这些技术是可互换的。尽管某些情况下确实如此，但这些平台之间存在各种潜在的差异。

结论是，不同的场景需要不同的解决方案，选择错误的方案可能会严重影响您的设计、开发和维护软件解决方案的能力。

本系列的[第1部分](/mq/2020/07/03/rabbitmq-vs-kafka.html)介绍了[RabbitMQ](https://www.rabbitmq.com/)和[Apache Kafka](https://kafka.apache.org/)的内部实现概念。 本部分将继续回顾这两个平台之间的显着差异，我们作为软件架构师和开发人员应注意的差异。然后，它继续说明我们通常尝试使用这些工具实现的体系结构模式，并评估何时使用每种工具。

## 提示1

如果您不熟悉RabbitMQ和Kafka的内部结构，我强烈建议您首先阅读本文的[第1部分](/mq/2020/07/03/rabbitmq-vs-kafka.html)。 如果不确定，请随意浏览标题和图表，至少可以瞥见这些差异。

## 提示2

在上一篇文章之后，一些读者问我关于Apache Pulsar的问题。 Pulsar是另一个消息传递平台，旨在结合RabbitMQ和Kafka所提供的一些最佳功能。作为一个现代平台，它看起来很有希望； 但是，与其他平台一样，它也有其优点和缺点。 我将在以后的文章中尝试比较Apache Pulsar，因为该文章着重于RabbitMQ和Kafka。

# RabbitMQ和Kafka之间的显著差异

RabbitMQ是一个消息代理(message broker)，而Apache Kafka是一个分布式的流数据平台。这种差异似乎是语义上的，但它带来了严重的影响，影响了我们合适地实现各种实例的能力。

例如，Kafka最适合用于处理数据流，而RabbitMQ对流中消息的顺序几乎没有保证。另一方面，RabbitMQ内置了对重试逻辑和死信交换的支持，而Kafka则将此类实现留在用户手中。本节重点介绍它俩之间的这些及其它显著差异。

## 消息顺序(Message ordering)

RabbitMQ对于发送到queue和exchange的消息的顺序几乎没有保证。尽管似乎消费者可以按照生产者发送消息的顺序来处理消息，但这是非常误导的。RabbitMQ文档陈述了有关消息顺序的保证：

> “Messages published in one channel, passing through one exchange and one queue and one outgoing channel will be received in the same order that they were sent.” — RabbitMQ Broker Semantics

换句话说，只要我们有一个消息消费者，它就会按顺序接收消息。 但是，一旦我们有多个消费者从同一个队列中读取消息，就无法保证消息的处理顺序。缺乏消息顺序保证的原因是，消费者可以在读取消息后将消息返回（或重新传递）到队列中（例如，在处理失败的情况下）。

一旦返回一条消息，另一个使用者就可以将其拾取进行处理，即使它已经使用了以后的消息。 因此，消费者组无序处理消息，如下图所示：

![An example of lost message ordering when using RabbitMQ](/assets/an_example_of_lost_message_ordering_when_using_RabbitMQ.png)

当然，我们可以通过将消费者并发限制为一个来确保RabbitMQ中的消息顺序。 更确切地说，单个消费者内的线程数应限制为一个，因为任何并行消息处理都可能导致的乱序问题。但是限制为一个单线程使用者会严重影响我们随着系统增长而扩展消息处理的能力。因此，我们不应该轻易的进行这种折中。

另一方面，Kafka在消息处理方面提供了可靠的消息顺序保证。Kafka保证发送到相同topic分区的所有消息被顺序处理。

回顾下[第1部分](/mq/2020/07/03/rabbitmq-vs-kafka.html)，Kafka使用轮询分区程序将消息均匀的放在不同的分区中。但是，生产者可以在每个消息上设置分区key，以创建逻辑数据流（例如来自同一设备的消息或属于同一租户的消息）。来自同一个数据流的所有消息都放在同一分区中，从而使消费者组按顺序消息消息。

然而，我们应该注意到，在消费者组中，每个分区是由单个消费者的单个线程来处理，我们无法扩展单个分区的处理能力。但是在Kafka中，我们可以扩展topic分区的数量，从而使每个分区收到的消息变少，并为补充的分区添加额外的消费者。

**优胜者**

Kafka无疑是赢家，因为它允许按顺序处理消息。RabbitMQ在顺序处理消息方面的保证很弱。

## 消息路由(Message routing)

RabbitMQ可以将消息路由到message exchange的订阅者，基于订阅者自定义的路由规则。[Topic Exchange](https://www.rabbitmq.com/tutorials/amqp-concepts.html#exchange-topic)可以基于专用的routing_key头来路由消息。此外，[Header Exchange](https://www.rabbitmq.com/tutorials/amqp-concepts.html#exchange-headers)可以基于任意的消息头信息来路由消息。两种exchange有效的使消费者可以指定他们想接收的消息类型，从而为架构师提供了极大的灵活性。

另一方面，Kafka不允许消费者在拉取topic之前过滤其中的消息。订阅消费者需毫无例外的接收分区中的所有消息。

作为开发人员，你可以使用[Kafka流作业](https://kafka.apache.org/documentation/streams/)，该作业从topic中读取消息，对其进行过滤，然后将其推动到消费者可以订阅的另一个topic。然而，这需要更多的努力，维护，和更多的移动组件。

**优胜者**

RabbitMQ在路由和过滤消息方面提供了出色的支持。

## 消息计时(Message timing)

RabbitMQ提供了有关队列消息计时的各种功能：

### 消息生存时间(Message Time-To-Live (TTL))

发送到RabbitMQ的每条消息都可关联TTL属性。TTL设置可直接由发布者设置，也可通过策略在队列上设置。

指定TTL可限制消息的有效期限。如果消费者没在有效期内消费，则该消息将会自动从队列中删除（并转移到死信交换dead-letter exchange，稍后处理）。TTL对于时间敏感的命令特别有用。

### 延迟/计划消息

RabbitMQ通过一个插件来支持延迟/计划消息。在message exchange上启用此插件后，生产者可以发送消息到RabbitMQ，生产者可以延迟RabbitMQ将消息路由到消费者队列的时间。此功可使开发人员设置将来执行的命令。例如，当生产者达到限流规则，我们可能希望将特定命令延迟到以后执行。

Kafka不支持此类功能，当消息到达时会被写入到分区，供消费者立即使用。此外，Kafka不提供消息的TTL机制，尽管我们可以在应用程序级别实现。

我们还必须记住，Kafka分区是仅追加事务日志。因此，它无法操作消息时间（或分区内消息的位置）。

**优胜者**

RabbitMQ赢得了此殊荣，因为Kafka的实现机制限制了此功能。


## 消息保留(Message retention)

一旦消费者成功消费了消息，RabbitMQ便立即将其从存储中逐出。这种行为不能被修改，这几乎是所有消息代理设计的一部分。

相比之下，Kafka会按设计持久保留所有消息，直到每个topic设置的超时期限。关于消息保留，Kafka并不关心消费者的消费状态，因为它充当消息日志。消费者可以消费任意数量的消息，并且可以通过操纵分区偏移量"及时"往返。。Kakfa会定期审查topic中消息的存在时长，并逐出足够久的消息。

Kafka的性能与存储容量无关。因此，从理论上讲，可以无限制的存储消息而不会影响性能（只要你的节点足够大以存储这些分区）。

> Kafka的性能与存储容量无关的理解，由于kafka数据会存入到分区，分区是一个文件夹，数据实际上是存入segment，而Kafka的每个segment大小为：log.segment.bytes (默认: 1GB)；Kafka查找消息时，首先定位segment，然后在segment里取数据，因此存储容量大小对kafka的性能影响很小。

**优胜者**

Kafka旨在保留消息，而RabbitMQ则不保留。这里不存在竞争，因此Kafka是获胜者。
