---
title: Performant and optimal Spring WebClient
author: Dhaval Shah
type: post
date: 2021-03-15T14:53:50+00:00
url: /performant-and-optimal-spring-webclient/
categories:
  - Cloud Native Architecture
  - Performance
  - Spring Boot
tags:
  - cloud native
  - performance
  - spring boot
thumbnail: "images/wp-content/uploads/2021/03/webclient-front-image.png"
---

[![](https://www.dhaval-shah.com/images/wp-content/uploads/2021/03/webclient-front-image.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2021/03/webclient-front-image.png)
-----------------------------------------------------------------------------------------------------------------------------------------

# Background

In my previous [post](https://www.dhaval-shah.com/rest-client-with-desired-nfrs-using-springs-resttemplate/) I tried demonstrating how to implement an optimal and performant [REST](https://en.wikipedia.org/wiki/Representational_state_transfer) client using [RestTemplate](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/client/RestTemplate.html)

In this article I will be demonstrating similar stuff but by using [WebClient](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/reactive/function/client/WebClient.html). But before we get started, lets try rationalizing 

> _Why yet another REST client i.e. WebClient_

IMO there are 2 compelling reasons -
1. Maintenance mode of [RestTemplate](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/client/RestTemplate.html)
   > NOTE: As of 5.0 this class is in maintenance mode, with only minor requests for changes and bugs to be accepted going forward. Please, consider using the org.springframework.web.reactive.client.WebClient which has a more modern API and supports sync, async, and streaming scenarios.
2. Enhanced performance with optimum resource utilization. One can refer my older [article](https://www.dhaval-shah.com/refactoring-from-imperative-to-reactive-implementation/) to understand performance gains reactive implementation is able to achieve.

From development standpoint, lets try understanding key aspects of implementing a performant and optimal WebClient

1.  ConnectionProvider with configurable connection pool
2.  HttpClient with optimal configurations
3.  Resiliency
4.  Implementing secured WebClient
5.  WebClient recommendations

[_**Source Code**_ @ Github](https://github.com/dhaval201279/RxWebClientDemo)

# 1. ConnectionProvider with configurable connection pool
----------------------------------

I am pretty much sure that most of us would have encountered issues pertaining to connection pool of REST clients - e.g. Connection pool getting exhausted because of default or incorrect configurations. In order to avoid such issues in WebClient [_ConnectionProvider_](https://projectreactor.io/docs/netty/release/api/index.html?reactor/netty/resources/ConnectionProvider.html) needs to be customized for broader control over :

* maxConnections - Allows to configure maximum no. of connections per connection pool. Default value is derived based on no. of processors
* maxIdleTime - Indicates max. amount of time for which a connection can remain idle in its pool.
* maxLifeTime - Indicates max. life time for which a connection can remain alive. Implicitly it is nothing but max. duration after which channel will be closed

#### Configuring ConnectionProvider
{{< highlight java >}}
ConnectionProvider connProvider = ConnectionProvider
                                    .builder("webclient-conn-pool")
                                    .maxConnections(maxConnections)
                                    .maxIdleTime()
                                    .maxLifeTime()
                                    .pendingAcquireMaxCount()
                                    .pendingAcquireTimeout(Duration.ofMillis(acquireTimeoutMillis))
                                    .build();
{{< /highlight >}}

# 2. HttpClient with optimal configurations
----------------------------------
[Netty's](https://netty.io/) [HttpClient](https://projectreactor.io/docs/netty/release/api/reactor/netty/http/client/HttpClient.html) provides fluent APIs for optimally configuring itself. From WebClient's performance standpoint HttpClient should be configured as shown below -

#### Configuring HttpClient
{{< highlight java >}}
nettyHttpClient = HttpClient
                    .create(connProvider)
                    .secure(sslContextSpec -> sslContextSpec.sslContext(webClientSslHelper.getSslContext()))
                    .tcpConfiguration(tcpClient -> {
                        LoopResources loop = LoopResources.create("webclient-event-loop",
                            selectorThreadCount, workerThreadCount, Boolean.TRUE);

                        return tcpClient
                                .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, connectionTimeoutMillis)
                                .option(ChannelOption.TCP_NODELAY, true)
                                .doOnConnected(connection -> {
                                    connection
                                        .addHandlerLast(new ReadTimeoutHandler(readTimeout))
                                        .addHandlerLast(new WriteTimeoutHandler(writeTimeout));
                                })
                                .runOn(loop);
                    })
                    .keepAlive(keepAlive)
                    .wiretap(Boolean.TRUE);
{{< /highlight >}}

## 2.1 Configuring event loop threads
{{< highlight java >}}
LoopResources loop = LoopResources.create("webclient-event-loop",
                                        selectorThreadCount, workerThreadCount, Boolean.TRUE);
{{< /highlight >}}

* selectorThreadCount - Configures DEFAULT_IO_SELECT_COUNT of [LoopResources](https://projectreactor.io/docs/netty/release/api/reactor/netty/resources/LoopResources.html). Defaults to -1 if not configured
* workerThreadCount - Configures DEFAULT_IO_WORKER_COUNT of [LoopResources](https://projectreactor.io/docs/netty/release/api/reactor/netty/resources/LoopResources.html). Defaults to number of available processors
  
## 2.2 Configuring underlying TCP configurations
* CONNECT_TIMEOUT_MILLIS - Indicates max. duration for which channel will wait to establish connection
* TCP_NODELAY - Indicates whether WebClient should send data packets immediately
* readTimeout - Configures duration for which, if no data was read within this time frame, it would throw [ReadTimeoutException](https://netty.io/4.1/api/io/netty/handler/timeout/ReadTimeoutException.html)
* writeTimeout - Configures duration for which, if no data was written within this time frame, it would throw [WriteTimeoutException](https://netty.io/4.1/api/io/netty/handler/timeout/WriteTimeoutException.html)
* keepAlive - Helps to enable / disable 'Keep Alive' support for outgoing requessts

# 3. Resilient WebClient
----------------------------------
While we all know the reasons for adopting Microservice Architecture, we are also cognizant of the fact that it comes with its own set of complexities and challenges. With distributed architecture, few of the major pain points for any application which consumes REST API are - 

*   Socket Exception - Caused by temporary server overload due to which it rejects incoming requests
*   Timeout Exception - Caused by temporary input / output latency. E.g. Extremely slow DB query resulting in timeout

Since failure in [Distributed Systems](https://en.wikipedia.org/wiki/Distributed_computing) are inevitable we need to make WebClient resilient by using some kind of Retry strategy as shown below 

#### Resilient WebClient
{{< highlight java >}}
cardAliasMono = restWebClient
                  .get()
                  .uri("/{cardNo}", cardNo)
                  .headers(this::populateHttpHeaders)
                  .retrieve()
                  .onStatus(HttpStatus::is4xxClientError, clientResponse -> {
                      log.error("Client error from downstream system");
                      return Mono.error(new HttpClientErrorException(HttpStatus.BAD_REQUEST));
                  })
                  .bodyToMono(String.class)
                  .retryWhen(Retry
                              .onlyIf(this::is5xxServerError)
                              .exponentialBackoff(Duration.ofSeconds(webClientConfig.getRetryFirstBackOff()),
                                      Duration.ofSeconds(webClientConfig.getRetryMaxBackOff()))
                              .retryMax(webClientConfig.getMaxRetryAttempts())
                              .doOnRetry(this::processOnRetry)
                  )
                  .doOnError(this::processInvocationErrors);
{{< /highlight >}}

Here we have configured retry for specific errors i.e. _5xxServerError_ and whenever its condition gets satisfied it will enforce exponential backoff strategy 

# 4. Implementing secured WebClient
--------------------------------

In this world of APIs, depending on the nature of data it deals, one may need to implement secured REST client. This is also demonstrated in code by using [WebClientSslHelper](https://github.com/dhaval201279/RxWebClientDemo/blob/master/src/main/java/com/its/RxWebClientDemo/infrastructure/ssl/WebClientSslHelper.java) which will be responsible for setting up the [SSLContext](https://netty.io/4.0/api/io/netty/handler/ssl/SslContext.html). 

Note - [WebClientSslHelper](https://github.com/dhaval201279/RxWebClientDemo/blob/master/src/main/java/com/its/RxWebClientDemo/infrastructure/ssl/WebClientSslHelper.java) should be conditionally instantiated based on what kind of security strategy (i.e. Untrusted / Trusted) needs to be in place for the corresponding environment and profile (viz test, CI, dev etc.)

# 5. WebClient recommendations
--------------------------------

## 5.1 Determining max idle time
It should always be less than keep alive time out configured on the downstream system

## 5.2 Leaky *exchange* 
While using _exchangeToMono()_ and _exchangeToFlux()_, returned response i.e. _Mono_ and _Flux_ should __ALWAYS__ be consumed. Faililng to do so may result in memory and connection leaks

## 5.3 Connection pool leasing strategy
Default [leasing strategy](https://projectreactor.io/docs/netty/release/api/reactor/netty/ReactorNetty.html) is [FIFO](https://en.wikipedia.org/wiki/FIFO) which means oldest connection is used from the pool. However, with keep alive timeout we may want to use [LIFO](https://en.wikipedia.org/wiki/LIFO), which will in turn ensure that most recent available connection is used from the pool.

## 5.4 Processing response without response body
If use case does not need to process response body, than one can implement it by using _releaseBody()_ and _toBodilessEntity()_. This will ensure that connections are released back to connection pool

# Conclusion
By knowing and understanding various aspects of [WebClient](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/reactive/function/client/WebClient.html) along with its key configuration parameters we can now build a highly performant, resilient and secured REST client using Spring's WebClient.

