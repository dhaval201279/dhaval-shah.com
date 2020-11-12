---
title: REST client with desired NFRs using Spring RestTemplate
author: Dhaval Shah
type: post
date: 2020-05-19T14:53:50+00:00
url: /rest-client-with-desired-nfrs-using-springs-resttemplate/
categories:
  - Cloud Native Architecture
  - Performance
  - Spring Boot
tags:
  - cloud native
  - performance
  - spring boot
thumbnail: "wp-content/uploads/2020/05/white-mode-150x150.png"
---

[![](http://dhaval-shah.com/wp-content/uploads/2020/05/white-mode.png)](http://dhaval-shah.com/wp-content/uploads/2020/05/white-mode.png)
-----------------------------------------------------------------------------------------------------------------------------------------

# Background

In this contemporary world of enterprise application development, [Microservice Architecture](https://en.wikipedia.org/wiki/Microservices) has become defacto paradigm. With this new paradigm, an application is going to have myriad set of independent and autonomous (micro)services which will be calling each other. One of the fundamental characteristics of Microservice Architecture is

> _Services must be easily consumable_

Hence most of the services implemented will be exposing [REST](https://en.wikipedia.org/wiki/Representational_state_transfer) APIs. In order to consume these REST APIs, each Microservice application will have to implement a REST client. As most of the applications are built using [Spring Boot](https://spring.io/projects/spring-boot), it is quite obvious that REST clients must be realized by using Spring's [RestTemplate](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/client/RestTemplate.html). Since [Performance](https://en.wikipedia.org/wiki/Performance_engineering) of application is of paramount importance, input / output operation that happens whilst invoking REST api via RestTemplate assumes lot of significance. Hence I felt a need to not only pen down various aspects to be kept in mind while implementing REST client using RestTemplate & Apache HttpClient but also demonstrate it via a full blown implementation  
  
[_**Source Code**_ @ Github](https://github.com/dhaval201279/RESTClientDemo)  
  
If you have come this far :) and are convinced to go through remaining of the article, this is what I will be covering :

1.  Optimal connection pool management
2.  Resilient REST client invocation
3.  Observable connection pool
4.  Implementing secured REST client

# 1. Optimal Connection Pool Management
----------------------------------

I am pretty much sure that most of us would have encountered issues pertaining to connection pool of REST clients - e.g. Connection pool getting exhausted because of incorrect configuration, or connections once acquired not being released and cleaned up etc. One can optimize the overall implementation if one really knows key configurable parameters and ways to manage them.

## 1.1 Customizing [_RequestConfig_](https://hc.apache.org/httpcomponents-client-4.5.x/httpclient/apidocs/org/apache/http/client/config/RequestConfig.html)

[_RequestConfig_](https://hc.apache.org/httpcomponents-client-4.5.x/httpclient/apidocs/org/apache/http/client/config/RequestConfig.html) needs to be customized for finer control on key HTTP connection parameters -

*   CONNECT\_TIMEOUT - Indicates timeout in milliseconds until a connection is established
*   CONNECTION\_REQUEST\_TIMEOUT - Indicates timeout when requesting a connection from the connection manager
*   SOCKET\_TIMEOUT - Indicates timeout for waiting for data OR maximum period of inactivity between 2 consecutive data packets

#### Configuring HttpClient using RequestConfig
{{< highlight java >}}
private RequestConfig prepareHttpClientRequestConfig() {
  RequestConfig requestConfig =  RequestConfig
                                  .custom()
                                  .setConnectTimeout(CONNECT_TIMEOUT)
                                  .setConnectionRequestTimeout(CONNECTION_REQUEST_TIMEOUT)
                                  .setSocketTimeout(SOCKET_TIMEOUT)
                                  .build();
  return requestConfig;
}
{{< /highlight >}}

## 1.2 Connection pool management via [_PoolingHttpClientConnectionManager_](https://hc.apache.org/httpcomponents-client-4.5.x/httpclient/apidocs/org/apache/http/impl/conn/PoolingHttpClientConnectionManager.html)

[_ConnectionManager_](https://hc.apache.org/httpcomponents-client-4.5.x/httpclient/apidocs/org/apache/http/impl/conn/PoolingHttpClientConnectionManager.html) needs to be instantiated so that application can manage certain key configurations -

*   MAX\_CONNECTIONS - Sets maximum number of total connections
*   MAX\_PER\_ROUTE_CONNECTION - Sets maximum number of connections per route
*   VALIDATE\_AFTER\_INACTIVITY - Sets duration after which persistent connections needs to be re-validated before leasing

#### HttpClient configuration for ConnectionPool using PoolingHttpClientConnectionManager
{{< highlight java >}}
public PoolingHttpClientConnectionManager poolingConnectionManager() {
	PoolingHttpClientConnectionManager poolingConnectionManager =
		new PoolingHttpClientConnectionManager(getConnectionSocketFactoryRegistry());
	poolingConnectionManager.setMaxTotal(MAX_CONNECTIONS);
	poolingConnectionManager.setDefaultMaxPerRoute(MAX_PER_ROUTE_CONNECTION);
	poolingConnectionManager.setValidateAfterInactivity(VALIDATE_AFTER_INACTIVITY_IN_MILLIS);
	return poolingConnectionManager;
}
{{< /highlight >}}

## 1.3 Configuring [_ConnectionKeepAlive_](https://hc.apache.org/httpcomponents-client-4.5.x/httpclient/apidocs/org/apache/http/conn/ConnectionKeepAliveStrategy.html) strategy

This configuration is also equally important, as its absence from [Http Headers](https://en.wikipedia.org/wiki/List_of_HTTP_header_fields) will lead [httpclient](https://hc.apache.org/httpcomponents-client-ga/) to assume that connection can be kept alive till eternity. For finer control on http connection parameters it would be prudent to implement a custom keep alive strategy

#### Http client config with custom defined keep alive strategy
{{< highlight java >}}
private ConnectionKeepAliveStrategy connectionKeepAliveStrategy() {
  return new DefaultConnectionKeepAliveStrategy() {
    @Override
    public long getKeepAliveDuration(final HttpResponse response, final HttpContext context) {
      long keepAliveDuration = super.getKeepAliveDuration(response, context);
      if (keepAliveDuration < 0) {
        keepAliveDuration = DEFAULT_KEEP_ALIVE_TIME_MILLIS;
      } else if (keepAliveDuration > MAX_KEEP_ALIVE_TIME_MILLIS) {
        keepAliveDuration = MAX_KEEP_ALIVE_TIME_MILLIS;
      }
      return keepAliveDuration;
    }
  };
}
{{< /highlight >}}

## 1.4 Housekeeping connection pool

So far we saw that connection pool as an application resource is being managed with various configuration parameters. As these resources are key to optimal functioning of application, it needs to perform periodic house keeping which does the following -

*   Evicts connection that are expired due to prolonged period of inactivity
*   Evicts connection that have been idle based on application configured IDLE\_CONNECTION\_WAIT\_TIME\_SECS
*   From performance standpoint, it is **recommended** to execute such house keeping jobs at configured interval in a separate thread with its own thread pool configuration using [_ScheduledThreadPoolExecutor_](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ScheduledThreadPoolExecutor.html)

#### Http client config with house keeping of idle and expired connections
{{< highlight java >}}
public Runnable idleAndExpiredConnectionProcessor(final PoolingHttpClientConnectionManager connectionManager) {
  return new Runnable() {
    @Override
    @Scheduled(fixedDelay = 20000)
    public void run() {
      try {
        if (connectionManager != null) {
            connectionManager.closeExpiredConnections();
            connectionManager.closeIdleConnections(IDLE_CONNECTION_WAIT_TIME_SECS, TimeUnit.SECONDS);
        }
      } catch (Exception e) {
        // Log errors
      }
    }
  };
}
{{< /highlight >}}

Note - Considering the fact that [Cloud Native](https://en.wikipedia.org/wiki/Cloud_native_computing) applications are implemented considering [12 factors](https://12factor.net/), all the above configurations should be managed via [externalized configurations](https://12factor.net/config).

# 2. Resilient invocation
--------------------

While we all know the reasons for adopting Microservice Architecture, we are also cognizant of the fact that it comes with its own set of complexities and challenges. With distributed architecture, few of the major pain points for any application which consumes REST API are - 

*   Socket Exception - Caused by temporary server overload due to which it rejects incoming requests
*   Timeout Exception - Caused by temporary input / output latency. E.g. Extremely slow DB query resulting in timeout

Phew ! Things are going damn crazy with Microservice Architecture :) Thankfully we have [Spring](https://spring.io/) to our rescue with [Spring Retry](https://docs.spring.io/spring-batch/docs/current/reference/html/retry.html) library. We can use its capabilities to make our REST API calls more resilient. I have used [_@Retryable_](https://docs.spring.io/spring-retry/docs/api/current/org/springframework/retry/annotation/Retryable.html) and [_@Recover_](https://docs.spring.io/spring-retry/docs/api/current/org/springframework/retry/annotation/Recover.html) to configure retry strategy with exponential back-off and fallback strategy.

For our application we have configured retry (using [_@Retryable_](https://docs.spring.io/spring-retry/docs/api/current/org/springframework/retry/annotation/Retryable.html)) for [RemoteServiceUnavailableException](https://github.com/dhaval201279/RESTClientDemo/blob/master/src/main/java/com/its/RESTClientDemo/service/exception/RemoteServiceUnavailableException.java) with _maxAttempts_ as 8 and with an exponential back-off policy i.e. 1000 \* (2 to the power of n) seconds, where n = no. of retry attempt. We have also configured a fallback strategy using [_@Recover_](https://docs.spring.io/spring-retry/docs/api/current/index.html?org/springframework/retry/annotation/Recover.html)

Spring Retry in itself is a candidate for a separate blog, hence we are not delving deeper beyond this.

#### Retryable RestClient using Spring Retry
{{< highlight java >}}
@Retryable(value = {RemoteServiceUnavailableException.class}, maxAttempts = 8,
    backoff = @Backoff(delay = 1000, multiplier = 2), label = "generate-alias-retry-label")
String generateAlias(String cardNo);

@Recover
String fallbackForGenerateAlias(Throwable th, String cardNo);
{{< /highlight >}}

# 3. Observable connection pool
--------------------------

As we all know that in distributed architecture, [observability](https://en.wikipedia.org/wiki/Observability) of application / infrastructure resources is extremely important. The same holds good for Http connection pool. [ConnectionManager](https://hc.apache.org/httpcomponents-client-4.5.x/httpclient/apidocs/org/apache/http/impl/conn/PoolingHttpClientConnectionManager.html) provides methods to fetch its total statistics and statistics for each route at a given point of time. In order to avoid any performance issues, it is **recommended** to execute such metric logger at configured interval in a separate thread with its own thread pool configuration using [_ScheduledThreadPoolExecutor_](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ScheduledThreadPoolExecutor.html)

#### HttpClient metrics extractor
{{< highlight java >}}
public Runnable connectionPoolMetricsLogger(final PoolingHttpClientConnectionManager connectionManager) {
  return new Runnable() {
    @Override
    @Scheduled(fixedDelay = 30000)
    public void run() {
      final StringBuilder buffer = new StringBuilder();
      try {
        if (connectionManager != null) {
          final PoolStats totalPoolStats = connectionManager.getTotalStats();
          log.info(" ** HTTP Client Connection Pool Stats : Available = {}, Leased = {}, Pending = {}, Max = {} **",
              totalPoolStats.getAvailable(), totalPoolStats.getLeased(), totalPoolStats.getPending(), totalPoolStats.getMax());
          connectionManager
            .getRoutes()
            .stream()
            .forEach(route -> {
              final PoolStats routeStats = connectionManager.getStats(route);
              buffer
                .append(" ++ HTTP Client Connection Pool Route Pool Stats ++ ")
                .append(" Route : " + route.toString())
                .append(" Available : " + routeStats.getAvailable())
                .append(" Leased : " + routeStats.getLeased())
                .append(" Pending : " + routeStats.getPending())
                .append(" Max : " + routeStats.getMax());
            });
          log.info(buffer.toString());
        }
      } catch (Exception e) {
        log.error("Exception occurred whilst logging http connection pool stats. msg = {}, e = {}", e.getMessage(), e);
      }
    }
  };
}
{{< /highlight >}}

Captured metrics can be used for better monitoring and alerting which can not only assist in proactive addressing of Http Client's connection pool issues, but it may also help in troubleshooting production incidents pertaining to connection pool. This implicitly will help in reducing [MTTR](https://en.wikipedia.org/wiki/Mean_time_to_recovery)

# 4. Implementing secured REST client
--------------------------------

For lot of APIs, depending on the nature of data it deals with, one may need to implement secured REST client. This is also demonstrated in code by using [HttpClientSslHelper](https://github.com/dhaval201279/RESTClientDemo/blob/master/src/main/java/com/its/RESTClientDemo/infrastructure/ssl/UntrustedHttpClientSslHelper.java) which will be responsible for setting up the [SSLContext](https://docs.oracle.com/javase/8/docs/api/javax/net/ssl/SSLContext.html). This SSLContext can than be used to instantiate [Registry](https://hc.apache.org/httpcomponents-core-ga/httpcore/apidocs/org/apache/http/config/Registry.html) of [ConnectionSocketFactory](https://hc.apache.org/httpcomponents-client-ga/httpclient/apidocs/index.html?org/apache/http/conn/socket/ConnectionSocketFactory.html).

Note - [HttpClientSslHelper](https://github.com/dhaval201279/RESTClientDemo/blob/master/src/main/java/com/its/RESTClientDemo/infrastructure/ssl/UntrustedHttpClientSslHelper.java) should be conditionally instantiated based on what kind of security strategy (i.e. Untrusted / Trusted) needs to be in place for the corresponding environment and profile (viz test, CI, dev etc.)

# Conclusion
By knowing and understanding various aspects of [HttpClient](https://hc.apache.org/httpcomponents-client-ga/) along with its key configuration parameters we can now build a highly performant, resilient, observable and secured REST client using RestTemplate.