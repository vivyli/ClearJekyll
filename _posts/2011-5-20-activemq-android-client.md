---
layout: post
title: "ActiveMQ Android Client"
description: ""
category: 
tags: [android]
---
{% include JB/setup %}

[ActiveMQ](http://activemq.apache.org/)是一个JMS的消息队列的一种实现，在高并发的服务端用的较多，据称能够支持每秒20000的并发数据访问。

他实现了JMS标准的基本操作，如点对点的消息收发，发布/订阅消息等等，并且支持了许多协议，如Openwire、REST、Stomp、XMPP等等，因此支持了多语言和多平台，它自带的Java库实现了openwire和stomp的访问，但是如要手机端（如Android）对JMS的支持很差（或者还不支持？），因此对于手机端来说操作ActiveMQ不得不使用XMPP或者REST。

<!-- more -->

XMPP我尝试过，使用[smack](http://www.igniterealtime.org/projects/smack/index.jsp)，smack是一个XMPP的Java库，原本是用来做IM开发的（作为同一系列的[spark](http://www.igniterealtime.org/projects/spark/index.jsp)是XMPP的IM客户端，基于smack，[Openfire](http://www.igniterealtime.org/projects/openfire/index.jsp)是XMPP的服务器，因此spark+openfire能够搭建一个IM系统，同样ActiveMQ也可以作为XMPP的服务器，ActiveMQ+spark也可以搭建一个IM系统。）使用smack能够连接ActiveMQ，并建立一个Topic，向Topic队列发送消息，但是一直没有找到如何建立Queue，生产消息和消费消息。有人还特意为Android写了个[asmack](http://code.google.com/p/asmack/)。

我最后还是采用了最简单的REST，因为这基本上是HTTP的GET/POST来做，不需要任何第三方库。

我需要的操作有3个：

- produce：向一个消息队列中发送消息。

- consume：消费一个队列中的一条消息。

- unsubscribe：撤销一个队列的一个conumer。

实现比较简单，produce操作即向http://localhost:8161/demo/message POST一条HTTP请求，POST的数据如：

- destination：队列名称

- type：队列类型，queue或者topic

- TimeToLive：消息在队列中的有效期

- body：消息的内容

其他还有一些能够设置，不过用处不是很大。

consume操作则是向http://localhost:8161/demo/message GET一条HTTP请求，一般格式如：

http://localhost:8161/demo/message/destination?type=queue&clientId=cid&readTimeout=timeout

- destination：想要消费数据的队列名称

- type：queue或者topic

- clientId：客户端的ID，不同的ID对应不同的consumer

- readTimeout：读队列等待的时长，超过时长请求等待结束

unsubscribe操作指的是取消队列的consumer，两个consumer同时在consume一个队列时REST请求似乎没有响应，因此在consume一条消息之后需要移除这个consumer，而这个操作是到了ActiveMQ 5.4.0之后才有这个操作。做法是向http://localhost:8161/demo/message POST一条请求，其中数据为：

- action：为unsubscribe

- clientId：要移除的consumer的id

这样手机端就可以与ActiveMQ通信。