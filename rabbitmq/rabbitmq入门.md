## 概述

RabbitMQ 是一个由 Erlang 语言开发的 AMQP 的开源实现。

AMQP ：Advanced Message Queue，高级消息队列协议。它是应用层协议的一个开放标准，为面向消息的中间件设计，基于此协议的客户端与消息中间件可传递消息，并不受产品、开发语言等条件的限制。



## 相关概念



### 消息模型

所有的消息队列可以大致认为执行的是以下的消息模型，生产者生产消息并发布到队列中，消费者订阅队列并从队列中消费消息。

![](img/5015984-066ff248d5ff8eed.webp)

### AMQP协议模型

![](img/5015984-367dd717d89ae5db.webp)



+ Broker

  又称 `Server`，接受客户端的连接，实现 `AMQP` 实体服务

+ Connection

  连接，指应用程序与 `Broker` 的网络连接

+ Channel

  网络信道，是进行消息读写的通道，每个 `Channel` 代表一个会话任务

+ Message

  消息，服务器和应用程序之间传送的数据，由 `properties` 和 `body` 组成。`properties`可以对消息进行修饰，如消息优先级、延迟等特性，`Body` 则是消息体内容。

+ Virtual host

  虚拟地址，用于进行逻辑隔离，最上层的消息路由。（如 redis 16个db也是逻辑隔离）一个 `Virtual host` 里面可以有若干个 `Exchange` 和 `Queue`，同一个 `Virtual Host` 不能有相同名称的 `Exchange` 和 `Queue`。

+ Exchange

  交换机，接受消息，根据路由建转发消息到绑定的队列。

+ Binding

  `Exchange` 和 `Queue` 之间的虚拟连接，`binding` 中可以包含 `routing key`。

+ routing key

  一个路由规则，虚拟机根据它来确定如何路由一个特定的消息。

+ Queue

  消息队列，保存和转发消息。

#### RabbitMQ架构

![](img/rabbitmq_example.png)



 