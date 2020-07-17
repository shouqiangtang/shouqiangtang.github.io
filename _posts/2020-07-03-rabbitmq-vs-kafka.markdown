---
layout: post
title: RabbitMQ VS Kafka
date: 2020-07-03 19:01:00 +0800
categories: mq
tags: RabbitMQ和Kafka区别
---

# 介绍

作为一个微服务系统软件架构师，我经常遇到一个不断重复的问题问题：“我应该使用RabbitMQ还是Kafka“，由于某些原因，许多开发者认为这些技术是可互换的。尽管某些情况下确实如此，但这些平台之间存在各种潜在的差异。

结论是，不同的场景需要不同的解决方案，选择错误的方案可能会严重影响您的设计、开发和维护软件解决方案的能力。

本文的目的是首先介绍基本的异步消息传递模式。然后，继续介绍RabbitMQ和Kafka及其内部结构。[第2部分](xxxxxxx)将重点介绍这些平台之间的关键区别，它们的各种优缺点以及如何在两者之间进行选择。

# 异步消息传递模式

异步消息传递是一个消息传递方案，其中生产者生产消息和消费者处理消息是解耦的。在处理消息传递系统时，我们通常会确定两种主要的消息传递模式：消息排队(Message Queue)和发布/订阅。

## 消息排队(Message queueing)

在消息排队(message-queuing)通信模式中，队列(Queue)暂时将生产者和消费者解耦。多个生产者可以将消息发送到同一个队列(Queue)。但是，当消费者处理消息时，该消息将被锁定或从队列(Queue)中删除并且不再可用。某个消息只能被一个消费者使用。

![message queuing](/assets/message_queuing.png)

附带说明一下，如果消费者无法处理某条消息，则消息传递平台通常将消息返回到队列(Queue)中，以供其它消费者使用。除了临时解耦，队列(Queue)还使我们能够独立地扩容生产者和消费者的规模，并提供一定程度的容错能力以应对错误处理。

## 发布/订阅(Publish/Subscribe)

在发布/订阅(或pub/sub)通信模式中，单条消息可以被多个消费者同时接收和处理。

![Publish/Subcribe](/assets/publish_subscribe.png)

例如，该模式允许发布者通知所有订阅者系统中发生了某些事情。许多队列(queuing)平台通常将pub/sub与术语topics相关联。在RabbitMQ中，topics是pub/sub实现的一种特定类型（确切的说是exchange类型），但是在本文中，我将topics称为pub/sub整体的表示。

一般来说，订阅有两种类型：

1. 临时订阅，该订阅仅在消费者启动并运行时才活跃。一旦消费者关闭，该订阅和尚未处理的消费便会丢失。
2. 持久订阅，该订阅只要不被明确删除并会一直保留。当消费者关闭时，消息传递平台将维护该订阅，并且在以后恢复消息处理。

# RabbitMQ

RabbitMQ实现了一个消息代理(message broler)，该消息代理通常称为服务总线(service bus)。它本身支持上述两种消息传递模式。消息代理(message broker)的其它流行实现包括ActiveMQ, ZeroMQ, Azure Service Bus, 和 Amazon Simple Queue Service (SQS)。所有的这些实现都有很多共同点，本文中描述的许多概念也大多数适用于它们。

## Queues

RabbitMQ支持开箱即用的经典消息队列。开发人员定义命名队列，然后生产者可以将消息发送到该命名队列。消费者进而使用相同的队列来取消息并对其进行处理。

## Message exchanges

RabbitMQ通过使用message exchanges来实现pub/sub。发布者将消息发布到message exchanges，而不知道这些消息的订阅者是谁。

每个希望订阅exchange的消费者都会创建一个队列。然后，message exchange将生产的消息发送到所有队列以供消费者消费。它还可以基于各种路由规则为某些消费者过滤消息。

![RabbitMQ message exchange](/assets/rabbitmq_message_exchange.png)

请注意RabbitMQ支持临时订阅和持久订阅。消费者可以通过RabbitMQ API来决定使用哪种订阅类型。

基于RabbitMQ的体系结构，我们还可以创建一种混合方式 - 将一些订阅者形成消费者组，该消费者组在特定队列上竞争消费消息。通过这种方式，我们实现了发布/订阅模式，同时还允许某些订阅者扩大规模以处理接收到的消息。

![Pub/sub and queuing combined](/assets/pubsub_and_queuing_combined.png)

# Apache Kafka

Apache Kafka不是消息代理(message broker)的实现，它是一个分布式流数据平台。

不像RabbitMQ是基于queues和exchanges，Kafka的存储层是使用分区事务日志来实现的。Kafka还提供了用于处理实时数据流的Streams API，和容易与其它数据源集成的Connector API。但是这些不再本人的讨论范围之内。

云供应商为Kafka的存储层提供了替代解决方案。这些解决方案包括[Azure Event Hubs](https://azure.microsoft.com/en-us/services/event-hubs/)和[AWS Kinesis Data Streams](https://aws.amazon.com/kinesis/data-streams/)。

## Topics

Kafka没有实现queue的概念。相反，Kafka将记录的集合存储在成为Topic的类别中。对于每个Topic，Kafka维护一个消息分区日志。每个分区是一个有序的、不可变的记录序列。当消息到达时，Kafka会将消息追加到这些分区里面。默认情况下，Kafka使用轮询分区程序在各个分区之间均匀分布消息。生产者可以修改此行为以创建消息的逻辑流。例如在多租户应用程序中，我们可能根据每个消息的租户ID来创建逻辑消息流。在物联网场景中，我们可能希望不断将每个生产者的身份映射到特定分区。确保来自同一逻辑流的所有消息映射到同一分区进而将其传递给消费者。

![Kafka producers](/assets/kafka_producers.png)

消费者通过维护这些分区的偏移量offset(或索引index)并顺序消费消息。一个消费者可以消费多个Topic，并且消费者数量可以扩展到可用分区数量。因此，在创建Topic时，应仔细考虑该Topic上消息传递的预期吞吐量。一起消费topic的一组消费者被成为消费者组(consumer group)。Kafka的API通常会处理消费者组中消费者之间的分区处理与消费者当前分区偏移量的存储之间的平衡。

![Kafka consumers](/assets/kafka_consumers.png)

## 使用Kafka实现消息传递模式

Kafka的实现非常适合pub/sub模式。生产者可以将消息发送到特定的Topic，而多个消费者组可以消费相同的消息。每个消费者组可以单独扩容以应对负载。消费者可以选择持久订阅，即重新启动时维护偏移量；或者临时订阅，即丢掉偏移量并在每次重启时从每个分区的最新记录处开始消费。

然而，Kafka不适用于message-queuing模式。当然，我们可以只包含一个消费者组来模拟一个经典消息队列。然而，这有多个缺点。本文的[第2部分](xxxx)将详细讨论。

请务必注意，Kafka会将数据留在分区中，直到预定的时间，无论消费者是否消费了这些消息。这就意味着消费者可以自由地重读以往的数据。此外，开发人员还可以使用Kafka的存储层来实现各种机制，比如事件源和审核日志。

# 结束语

尽管RabbitMQ和Kafka有时可以互换，但是它们的实现却有很大的不同。 因此，我们无法将它们视为同一类工具的成员； 一个是消息代理，另一个是分布式流平台。作为解决方案架构师，我们应该承认这些差异，并积极考虑在给定场景中应使用哪种类型的解决方案。 [第2部分](xxxxx)解决了这些差异，并提供了何时使用它们的指南。

# 延伸阅读

如果您想了解有关RabbitMQ和Kafka的内部实现的更多信息，我建议您看下以下资源：

* [AMQP 0.9.1 Model Explained — RabbitMQ](https://www.rabbitmq.com/tutorials/amqp-concepts.html)
* [Introduction to Apache Kafka](https://kafka.apache.org/intro)
