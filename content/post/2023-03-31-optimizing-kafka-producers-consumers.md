---
title: Optimizing Kafka Producers and Consumers
author: Dhaval Shah
type: post
date: 2023-04-01T06:00:50+00:00
url: /optimizing-kafka-producers-consumers/
categories:
  - Performance
  - Kafka
tags:
  - performance
  - tuning
  - kafka
thumbnail: "images/wp-content/uploads/2023/04/apache-kafka-speedometer.png"
---


[![](https://www.dhaval-shah.com/images/wp-content/uploads/2023/04/apache-kafka-speedometer.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2023/04/apache-kafka-speedometer.png)
-----------------------------------------------------------------------------------------------------------------------------------------
# Background
In this current era, [Distributed Architecture](https://en.wikipedia.org/wiki/Distributed_computing) has become de-facto architectural paradigm, which necessitates implementation of loosely coupled [Microservices](https://en.wikipedia.org/wiki/Microservices) which would talk with each other via 
1. [REST](https://en.wikipedia.org/wiki/Representational_state_transfer) API
2. [Message Oriented Middleware](https://en.wikipedia.org/wiki/Message-oriented_middleware)

As far as Message Oriented Middleware is concerned, [Apache Kafka](https://kafka.apache.org/) has become quite ubiquitous in today's world of Distributed Systems. Apache Kafka is a powerful, distributed, replicated messaging service platform that is mainly responsible for storing and sharing data in a scalable, robust and fault tolerant manner. From application standpoint, application developers mainly leverage **Kafka Producer** and **Kafka Consumer** to mainly publish and consume messages. So both Producer and Consumer assumes lot of importance for optimizing Kafka based interactions.

Primary focus of this article will be to understand ways to improve performance of Kafka Producers and Consumers. Performance Engineering as a whole has two orthogonal dimensions -
1. **Throughput**
2. **Latency**
# Kafka end to end latency
_Kafka end to end latency_ is the amount of time spent between an application publishing a record via _KafkaProducer.send()_ and application consuming the published record via _KafkaConsumer.poll()_. Below image clearly indicates various phases Kafka record undergoes

[![Kafka - End to End Latency](https://www.dhaval-shah.com/images/wp-content/uploads/2023/04/kafka-end-to-end-latency.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2023/04/kafka-end-to-end-latency.png)

1. **Produce Time** - Time elapsed between the point at which application publishes a record via _KafkaProducer.send()_ and the point at which published record is sent to leader broker of topic partition.
2. **Publish Time** - Time elapsed between the point at which Kafka's internal Producer publishes batch of messages to broker and the point at which published messages gets appended to replica log of leader
3. **Commit Time** - Time taken by Kafka to replicate messages to all in-sync replicas
4. **Catch-up Time** - Once message is committed, if Consumer's offset is __N__ messages behind the committed message than _Catch-up Time_ is the time Consumer would take to consume those __N__ messages
5. **Fetch Time** - Time taken by Kafka Consumer to fetch messages from leader broker

# Optimization Approach
Typically speaking, data flow through Kafka will involve following actors -
1. Producer
2. Topic
3. Consumer

So we will mainly focus on Producer and Consumer from application optimization standpoint

## Optimizing Kafka Producer
Apart from above phases of Kafka messages, understanding delivery time breakdown of Kafka Producer is equally important from optimization perspective

[![Kafka - Producer time split up ](https://www.dhaval-shah.com/images/wp-content/uploads/2023/04/kafka-producer-write-time-splitup.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2023/04/kafka-producer-write-time-splitup.png)

### Key Configurations
1. _**batch.size**_ - Controls amount of memory (in bytes) that will be used by Producer for each batch. Increasing batch size may improve throughput at the cost of memory footprint of your application.
**Note** - Higher value doesn't mean delay in sending messages
2. _**linger.ms**_ - Defines amount of time (in milliseconds) Producer waits until _batch.size_ messages are available for sending to Broker. Increasing its value will reduce network I/O and thereby ensure higher throughput. However, higher value may increase latency of Producer.
3. _**max.inflight.requests.per.connection**_ - Controls number of message batches Producer will send without receiving responses. Higher value will improve throughput at the cost of memory usage.


## Optimizing Kafka Consumer

1. _**fetch.min.bytes**_ - Defines minimum number of bytes Consumer intends to receive from Broker. Lower value will reduce latency at the cost of throughput.
2. _**fetch.wait.max.ms**_ - Defines maximum time Broker will wait before responding to _Fetch_ request from Consumer. Higher value will reduce network I/O and thereby ensure improved throughput at the cost of latency.
3. _**max.poll.records**_ - Controls maximum number of records a single call will return. Reducing its value will reduce latency by sacrificing throughput.

## Kafka - Producer Consumer Optimization Axes
Pictorially we can collate above understanding and prepare Kafka Producer Consumer Axes to easily remember key configurations along with its impact on performance of application.

[![Kafka - Producer Consumer Optimization Axes ](https://www.dhaval-shah.com/images/wp-content/uploads/2023/04/kafka-producer-consumer-performance-axes.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2023/04/kafka-producer-consumer-performance-axes.png)

# Conclusion
In this article we saw what exactly end to end latency within Kafka is along with various phased through which a message typically undergoes. Now we clearly understand what phases impacts performance of Kafka Producers and Consumers. It also covered key configurations that can help in reducing latency and increasing throughput of Kafka Producers and Consumers. By understanding impact of these configurations, we can say that there is a tradeoff between high throughput and low latency from performance optimization standpoint. And one can find right balance through experimentation by understanding nature of application (i.e. Throughput / Latency sensitive) and volume of load.

P.S - By default Apache Kafka is configured to favor latency over throughput
