---
title: Performance Comparison -  RSocket Vs Webflux
author: Dhaval Shah
type: post
date: 2021-09-12T14:53:50+00:00
url: /performance-comparison-rsocket-webflux/
categories:
  - Cloud Native Architecture
  - Performance
  - Spring Boot
tags:
  - cloud native
  - performance
  - spring boot
thumbnail: "images/wp-content/uploads/2021/09/rsock-rx-thumbnail-initial.png"
---

[![](https://www.dhaval-shah.com/images/wp-content/uploads/2021/09/rsock-rx-thumbnail-initial.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2021/09/rsock-rx-thumbnail-initial.png)
-----------------------------------------------------------------------------------------------------------------------------------------

# Background

In one of my previous [post](https://www.dhaval-shah.com/refactoring-from-imperative-to-reactive-implementation/) we saw tangible advantages (w.r.t throughput, latency and resource utilization) of refactoring existing Microservice application from imperative to [reactive](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html) constructs.

So an obvious question that comes to an inquisitive mind - 
> Can we apply [Reactive](https://www.reactivemanifesto.org/) principles to the underlying communication layer

Answer to above question is - Yes and  [**RSocket**](https://rsocket.io/) is the way to go

# What is RSocket
RSocket is a binary protocol that can perform bi-directional communication via [TCP](https://en.wikipedia.org/wiki/Transmission_Control_Protocol). RSocket tries to overcome few of the limitations of HTTP1 and HTTP2 by providing following features -
  - Back-Pressure
  - Multiplexing
  - Resumption
  - Routing

It has been developed to match the semantics of [**Reactive Streams**](http://www.reactive-streams.org/). Hence it seamlessly integrates with [**Project Reactor**](https://projectreactor.io/) and [**ReactiveX**](http://reactivex.io/)

It mainly supports 4 communication models :
  1. Fire and Forget (No Response)
  2. Request Response (Single Response)
  3. Request Stream (Multiple Responses)
  4. Channel (Bi-directional Response)

Lets try to understand _Request-Response_ model with a working example along with performance comparison w.r.t Spring Webflux APIs

# Bird's eye view of demo example

[![Demo App - Bird's eye view](https://www.dhaval-shah.com/images/wp-content/uploads/2021/09/demo-app-view.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2021/09/demo-app-view.png)

Working example (along with its source code) which we will be using for demonstration, primarily consists of 2 simple Spring Boot applications -

1.  [Card Client](https://github.com/dhaval201279/RxVsRsocket) - Public facing edge application with below endpoints. It uses _WebClient_ to invoke downstream system's APIs and and _RSocket_ client to communicate with downstream system's destination based on declared pattern.

{{< highlight java >}}
    @GetMapping("/rsocket/card/{cardId}")
    public Mono<CardEntity> cardByIdWithRSocket(@PathVariable String cardId) {
        log.info("Entering CardClientController : cardByIdWithRSocket with path variable as {} ", cardId);
        Mono<CardEntity> cardEntityMono = rSocket
                                            .route("rsocket-find-specific")
                                            .data(cardId)
                                            .retrieveMono(CardEntity.class);

        log.info("Card details received");
        return cardEntityMono;
    }
{{< /highlight >}}

#### APIs exposed by application

| HTTP Method   | URI     | Description   |
| :--------:  | :--------: | :------ |
| GET | rx/card/{id} | Fetches a card based on cardId by using _WebClient_ |
| GET | rsocket/card/{id} | Fetches a card based on cardId by using _RSocket_ |

2.  [Card Service](https://github.com/dhaval201279/RxVsRSocketServer) - Application which not only exposes _Webflux_ endpoint but also has a connection point for accepting _RSocket_ client connection via [**@MessageMapping**](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/messaging/handler/annotation/MessageMapping.html). It will be using  [Redis](https://redis.io/) as a persistent store. It also has a bulk loader which will load 5000 cards into the database whenever it bootstraps.

{{< highlight java >}}
    @MessageMapping("rsocket-find-specific")
    public Mono<CardEntity> rSocketFindSpecific(String cardId) {
        log.info("Entering CardViewController : rSocketFindSpecific with cardId : {} ", cardId);
        return cardService
                .getCardById(cardId)
                //.delayElement(Duration.ofSeconds(1))
                .doFinally(sig -> log.info("Response sent"));
    }
{{< /highlight >}}

# Comparison based on performance metrics
Lets try to comparatively analyze _RSocket_ Vs _Webflux_ from performance standpoint. We will be using [Gatling](https://gatling.io/)
to simulate a load test.

## Throughput & Latency
[![Load Testing Result](https://www.dhaval-shah.com/images/wp-content/uploads/2021/09/gatling-result-comp.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2021/09/gatling-result-comp.png)

Above reports clearly shows that -
1. 95th Percentile response time of RSocket based communication is 1/3 of Webflux based API consumption
2. Mean response time of RSocket based communication is 1/2 of Webflux based API consumption
3. Failures in RSocket based communication is 1/2 of WebFlux based API consumption
4. TPS for successful requests in RSocket based communication is twice more than  that of Webflux based API consumption

## [JVM Memory](https://www.dhaval-shah.com/understanding-jvm-memory-management/) usage and [Garbage Collection]() behavior
GC logging was enabled while performing load testing. GC logs were parsed via [GC Easy](https://gceasy.io/) to infer detailed JVM heap and GC behavior. Below are the observations w.r.t key performance parameters :

### GC Pause Time
Time taken by single Stop the World (STW) GC

[![GC Pause Time](https://www.dhaval-shah.com/images/wp-content/uploads/2021/09/gc-pause-time.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2021/09/gc-pause-time.png)

### Object Promotion Metrics
Metrics indicating size and rate at which objects are promoted from Young Generation to Old Generation. Higher the promotion rate, higher the frequence of full GCs i.e. STW GC

[![Object Promotion Metrics](https://www.dhaval-shah.com/images/wp-content/uploads/2021/09/obj-promotion-metrics.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2021/09/obj-promotion-metrics.png)

### GC Throughput
Percentage of time spent in performing business transaction against time spent in GC activity

[![GC Throughput](https://www.dhaval-shah.com/images/wp-content/uploads/2021/09/GC-throughput.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2021/09/GC-throughput.png)

> Reference - One may want to refer my old article to understand [nuances of Garbage Collection](https://www.dhaval-shah.com/understanding-and-optimizing-garbage-collection/)

# Conclusion
As we saw in this article that along with obvious [motivations behind using RSocket](https://rsocket.io/about/motivations), there are compelling performance reasons which may incline us to leverage this protocol in justifiable way for inter service communication.