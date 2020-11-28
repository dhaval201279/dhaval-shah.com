---
title: Refactoring from imperative to reactive implementation
author: Dhaval Shah
type: post
date: 2020-01-27T21:06:28+00:00
url: /refactoring-from-imperative-to-reactive-implementation/
categories:
  - Performance
  - Reactive
  - Spring Boot
tags:
  - microservice
  - performance
  - reactive
  - spring boot
thumbnail: "images/wp-content/uploads/2020/01/refactoring.jpg"
---

[![](https://www.dhaval-shah.com/images/wp-content/uploads/2020/01/refactoring.jpg)](https://www.dhaval-shah.com/images/wp-content/uploads/2020/01/refactoring.jpg)
-----------------------------------------------------------------------------------------------------------------------------------------
# Background

As software industry is embracing the new [Microservice Architecture](https://en.wikipedia.org/wiki/Microservices) paradigm, 
myriad applications have been built with [Spring Boot](https://spring.io/projects/spring-boot) framework. By the time organizations 
have got its early versions of microservice applications in production, industry has found out newer and better avenues 
for further optimizing microservices, so that systems can be more robust, resilient and responsive a.k.a Reactive Systems 
(as per [Reactive Manifesto](https://www.reactivemanifesto.org/) ). Thanks to [Spring Reactor](https://projectreactor.io/) 
and [Spring Webflux](https://docs.spring.io/spring/docs/current/spring-framework-reference/web-reactive.html) which can 
help us in building reactive systems using Spring framework.

What I intend to demonstrate over here is, how a Spring Boot application can be [refactored](https://en.wikipedia.org/wiki/Code_refactoring) from imperative to reactive implementation along with compelling justifiable reasons.

# Reference Application

[![](https://www.dhaval-shah.com/images/wp-content/uploads/2020/01/I2Rx-System-Diagram.jpg)](http://dhaval-shah.com/wp-content/uploads/2020/01/I2Rx-System-Diagram.jpg)

Lets consider a hypothetical scenario where in we have a Spring Boot based developed application i.e. Card Service App which exposes [REST](https://en.wikipedia.org/wiki/Representational_state_transfer) endpoints using [Spring WebMVC](https://docs.spring.io/spring/docs/3.2.x/spring-framework-reference/html/mvc.html). Similar to a conventional Microservice Architecture, this application in turn not only invokes an API (using Spring's [RestTemplate](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/client/RestTemplate.html)) exposed by downstream system (written in [Go](https://golang.org/)) but also performs some [Database](https://en.wikipedia.org/wiki/Database) operations (using [Spring Data Repository](https://docs.spring.io/spring-data/data-commons/docs/1.6.1.RELEASE/reference/html/repositories.html)). The sole purpose of adding this infrastructure complexity is to mimic a real world scenario and thereby really justify the refactoring efforts that we will be putting in.

#### APIs exposed by application

| HTTP Method   | URI     | Description   |
| --------  | -------- | ------ |
| GET | /card/{id} | Fetches a card based on cardId |
| GET | /cards | Fetches all the cards from the system |
| POST | /card | Adds a card within system |
| DELETE | /cards | Deletes all the cards from the system |

# Source Code

[Imperative Implementation](https://github.com/dhaval201279/imperative2rx/tree/8361a69b3499113dfe52297fe9badaf9818d4bc5) (Before refactoring) 
[Reactive Implementation](https://github.com/dhaval201279/imperative2rx/tree/a8b191b43db507828fea8975fad62d9804297bee) (After refactoring)

# Steps for refactoring

## 1. Build Assembly

Conventionally speaking build assembly of a typical Spring Boot application (which exposes imperative REST APIs) will have [Spring Boot Starter Web](https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-web). So we will be replacing this by [Spring Boot Starter Webflux](https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-webflux), which is parallel version of Spring MVC that supports fully non-blocking reactive streams. We will also be moving from [Tomcat](http://tomcat.apache.org/) to [Reactor Netty](https://github.com/reactor/reactor-netty) (based on [Netty](https://netty.io/) framework) which provides non-blocking HTTP clients and servers. Ideally Spring Boot should automatically configure Reactor Netty as default server if we are using Webflux. Since it was not getting configured automatically due to some dependency issue, I had to explicitly exclude Tomcat and add Reactor Netty as dependency.

[![Build Assembly Refactorings](https://www.dhaval-shah.com/images/wp-content/uploads/2020/01/gradle-build-changes.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2020/01/gradle-build-changes.png)

## 2. REST Controllers

Now that we have added Webflux as one of the dependencies, we can use its APIs to refactor controllers. Easiest way to get started is by refactoring GET APIs first and then follow it up with other APIs

### 2.1 GET API

Basically GET APIs either return a response object or a collection of response objects. We will be using [Mono](https://projectreactor.io/docs/core/release/reference/#mono) and [Flux](https://projectreactor.io/docs/core/release/reference/#flux) to refactor our APIs from imperative to reactive implementation by making use of _just_ and _fromIterable _operator

[![](https://www.dhaval-shah.com/images/wp-content/uploads/2020/01/REST-get-apis-changes.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2020/01/REST-get-apis-changes.png)

### 2.2 POST API

Now here comes the complexity as we will need to refactor _CardService_ before refactoring Controller. And we being [Software Craftsman](https://en.wikipedia.org/wiki/Software_craftsmanship) love solving complex pieces. Hence this is going to be fun and exciting with multi step process as mentioned below -

#### 2.2.1 Refactoring CardService from REST template to WebClient

We will replace RestTemplate with [WebClient](https://docs.spring.io/spring/docs/current/spring-framework-reference/web-reactive.html#webflux-client) which is a non blocking and reactive client to perform HTTP requests

[![](https://www.dhaval-shah.com/images/wp-content/uploads/2020/01/cardservice-resttemplate-to-webclient.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2020/01/cardservice-resttemplate-to-webclient.png)

We will also refactor the method that invokes downstream system's API by replacing instantiated RestTemplate with WebClient object

[![](https://www.dhaval-shah.com/images/wp-content/uploads/2020/01/cardservice-generateAlias.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2020/01/cardservice-generateAlias.png)

#### 2.2.2 Refactoring enroll method of CardService

Another tricky piece will be to refactor `enroll` method. So we will be flattening the returned Mono from downstream system, which will allow us to extract card alias, than set in Card object which will be eventually saved in database

[![](https://www.dhaval-shah.com/images/wp-content/uploads/2020/01/cardservice-enrollCard-method.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2020/01/cardservice-enrollCard-method.png)

And finally we will refactor Controller

#### 2.2.3 Refactoring Controller

[![](https://www.dhaval-shah.com/images/wp-content/uploads/2020/01/Controller-addCard.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2020/01/Controller-addCard.png)

# Comparison on performance metrics

After performing all the above refactoring steps, lets understand whether these efforts are really going to add any business value or is this refactoring activity going to sound like a buzz word driven development to the business / product team :simple_smile:

I certainly do not want to digress from core agenda of this post, hence I will try to keep this section as much concise as I can.

### Throughput

[![](https://www.dhaval-shah.com/images/wp-content/uploads/2020/01/Perf-througput.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2020/01/Perf-througput.png)

*   For 1500 concurrent users throughput of Webflux based implementation **increases by 1.52 times**
*   Spring WebMvc based implementation was so slow that not a single Get Card request got executed for 1000 and 1500 concurrent users

### Latency

[![](https://www.dhaval-shah.com/images/wp-content/uploads/2020/01/Perf-latency.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2020/01/Perf-latency.png)

*   For Webflux based implementation, 95 percentile response time is **50% less** when compared with Spring WebMvc based implementation

### CPU Utilization

[![](https://www.dhaval-shah.com/images/wp-content/uploads/2020/01/Perf-CPU.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2020/01/Perf-CPU.png)

*   Based on average readings Spring Webflux based implementation is **8-10% less** in CPU utilization

### No. of threads

[![](https://www.dhaval-shah.com/images/wp-content/uploads/2020/01/Perf-Thread-dump-analysis.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2020/01/Perf-Thread-dump-analysis.png)

*   For Spring Webflux based implementation, total no. of threads is **reduced by 7 times** (31 Vs 234)

# Conclusion

We clearly saw how to systematically refactor imperative code to reactive implementation. And with above performance metrics it is quiet apparent that even though this refactoring may not enhance any business value of application, but it has significant impact from [NFR](https://en.wikipedia.org/wiki/Non-functional_requirement) standpoint, as it mainly aids in making the system Reactive. Needless to say that comparative analysis would have convinced you regarding the benefits it bring from performance and resource utilization standpoint.

Happy REFACTORING!