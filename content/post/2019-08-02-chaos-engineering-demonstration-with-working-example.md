---
title: Chaos Engineering – Demonstration with working example (Part-2)
author: Dhaval Shah
type: post
date: 2019-08-02T17:21:13+00:00
url: /chaos-engineering-demonstration-with-working-example/
categories:
  - Chaos Engineering
  - Cloud Native Architecture
  - Spring Boot
  - Testing
tags:
  - Chaos Engineering
  - cloud native
  - Grafana
  - microservice
  - Prometheus
  - spring boot

---


# Background

From [first part](http://dhaval-shah.com/chaos-engineering-a-quick-primer/) of blog we gathered understanding about basics of Chaos Engineering. Now we will further deep dive to understand how to perform Chaos Engineering with a working example - which to me is going to be quite interesting. First lets start with understanding basics of working example which will be used to demonstrate following-

*   How to perform chaos engineering within an application
*   How to monitor the behavior of the system
*   What to monitor whilst executing experiments on our system

# Bird's eye view of demo example

[![Demo App - Architecture Overview](http://dhaval-shah.com/wp-content/uploads/2019/06/blog-demo-archictecture.png)](http://dhaval-shah.com/wp-content/uploads/2019/06/blog-demo-archictecture.png)

The working example (along with its source code) which we will be using for demonstration, primarily consists of 2 simple Spring Boot applications -

1.  [Card Client](https://github.com/dhaval201279/cm4sb-card-client) - Public facing edge application
2.  [Card Service](https://github.com/dhaval201279/cm4sb-card-service) - Application which has core domain of card. Since it owns business workflow, it will be using  [Redis](https://redis.io/) as persistent store. It also has a bulk loader which will load 5000 cards into the database whenever it bootstraps.

To keep things simple, both the applications will just expose 2 APIs

1.  Get Card by Id - Returns a card for the corresponding id
2.  Get All Cards - Returns list of 5000 cards

Since Card Service is a core domain application, we will be quite curious to know how will Card Client behave if

*   Card Service is either down
*   Card Service is responding too slowly

Hence we will be performing chaos engineering experiments with Card Service application as blast radius.

As per my earlier post, most fundamental prerequisite for performing Chaos Engineering experiments is to have state of the art Monitoring available in order to understand system behavior by capturing required metrics. This is the key as it will help in identifying vulnerable areas of system which needs to be further hardened. So in order to enable required monitoring stack we will need some additional infrastructure components as mentioned below -

1.  [Consul](https://www.consul.io/) - It is a [service mesh](https://en.wikipedia.org/wiki/Microservices#Service_mesh) solution providing features like service discovery, health check, secure service communication etc
2.  [Prometheus](https://prometheus.io/) - It is an open source monitoring and alerting tool built by [SoundCloud](https://soundcloud.com/)
3.  [Grafana](https://grafana.com/) - It is a visualizing tool that allows us to view different types of metrics by formulating queries and alerts. It has an excellent looking aesthetic UI.

Note - You can refer to my older [blog](http://dhaval-shah.com/monitoring-spring-boot-application/)  for understanding how Spring Boot's Actuator, [Micrometer](http://micrometer.io/),  Prometheus and Grafana works in unison to provide us required system metrics.

In order to generate chaos we will inject failures and for that we will make use of some tool. As on date there are myriad tools available in market viz. [Chaos IQ](https://chaosiq.io/), [Gremlin](https://www.gremlin.com/), [Simian Army](https://github.com/Netflix/SimianArmy/wiki) etc. Since we need to inject failures within application, we will be using [Chaos Monkey for Spring Boot (CM4SB)](https://codecentric.github.io/chaos-monkey-spring-boot/)

# Internals of Chaos Monkey for Spring Boot (CM4SB)

At a high level Chaos monkey for Spring Boot basically consists of **Watchers **and **Assaults.**

### Watchers

It will basically scan Spring Boot app for specific annotation (as per the configured values). It supports all the Spring annotation -

*   @Controllers
*   @RestControllers
*   @Service
*   @Repository
*   @Component

By using [AOP](https://en.wikipedia.org/wiki/Aspect-oriented_programming), CM4SB will identify the public method on which configured assaults need to be applied. One can even customize behavior of Watcher by using _watchedCustomService _property and thereby decide which classes and their public methods need to be assaulted

### Assaults

They are the most important component of CM4SB. They are basically categorized into -

1.  Latency Assault - Adds latency to the request. Number of requests can be controlled by level
2.  Exception Assault - Enables throwing of RuntimeException as per the configured value
3.  Appkiller Assault - Shuts down the application. The only caveat with this assault is, once the application is shut down, it needs manual step to restart the application.

### Metrics Emitted by CM4SB

| Type of Metric   | Metric name     |
| --------  | -------- |
| Chaos Monkey metric request count | *chaos_monkey_application_request_count_total* *chaos_monkey_application_request_count_assaulted* *chaos_monkey_assault_component_watcher_total* *chaos_monkey_assault_controller_watcher_total* *chaos_monkey_assault_repository_watcher_total* *chaos_monkey_assault_restController_watcher_total* *chaos_monkey_assault_service_watcher_total* |
| Chaos Monkey metric latency count in ms | *chaos_monkey_assault_latency_count_gauge* *chaos_monkey_assault_latency_count_total* |
| Chaos Monkey metric exception | *chaos_monkey_assault_exception_count* |

Note - We will be viewing each of this metric via Grafana dashboard as we perform chaos engineering experiments with our working example

### Key implementation aspects

#### 1. Adding Maven Dependency

{{< highlight java >}}
<dependency>
	<groupId>de.codecentric</groupId>
	<artifactId>chaos-monkey-spring-boot</artifactId>
	<version>2.0.2</version>
</dependency>
{{< /highlight >}}

#### 2. Activate Chaos Monkey for Spring Boot and Watcher related properties within application configurations

{{< highlight java >}}
spring:
  profiles:
    active: chaos-monkey

chaos:
  monkey:
    watcher:
      component: false
      controller: false
      repository: false
      rest-controller: true
      service: true
{{< /highlight >}}

#### 3. Enabling Chaos Monkey endpoints for monitoring

{{< highlight java >}}
management:
  endpoints:
    web:
      exposure:
        include: \["\*"\]
  #        include: \["info", "health", "prometheus", "chaosmonkey"\]
  metrics:
    tags:
      application: ${spring.application.name}
    distribution:
      percentiles:
        http.server.requests: 0.5, 0.9, 0.95, 0.99

  endpoint:
    chaosmonkey:
      enabled: true
{{< /highlight >}}

### List of HTTP Endpoints

| HTTP URI   | Description     | HTTP Method   |
| --------  | -------- | ------ |
| */chaosmonkey* | Running Chaos Monkey configuration | GET |
| */chaosmonkey/status* | Is Chaos Monkey enabled or disabled? | GET |
| */chaosmonkey/enable* | Enable Chaos Monkey | POST |
| */chaosmonkey/disable* | Disable Chaos Monkey | POST |
| */chaosmonkey/watcher* | Running Watcher configuration.                            NOTE: Watcher cannot be changed at runtime, they are Spring AOP components that have to be created when the application starts. | GET |
| */chaosmonkey/assaults* | Running Assaults configuration | GET |
| */chaosmonkey/assaults* | Change Assaults configuration | POST |

 

# Demonstration of working example

Once we have our both the applications i.e. Card Client and Card Service along with Consul, Prometheus and Grafana up and running, we can inject failure using CM4SB. For monitoring CM4SB metrics, we have imported [Grafana Dashboard](https://grafana.com/dashboards?search=spring%20boot) with id as 9845. Before initiating chaos engineering experiments lets understand current configurations of Chaos Monkey by invoking '**_/actuator/chaosmonkey_**' with HTTP GET method

{{< highlight java >}}
{
    "chaosMonkeyProperties": {
        "enabled": false
    },
    "assaultProperties": {
        "level": 5,
        "latencyRangeStart": 1000,
        "latencyRangeEnd": 3000,
        "latencyActive": true,
        "exceptionsActive": false,
        "exception": {},
        "killApplicationActive": false,
        "frozen": false,
        "proxyTargetClass": true,
        "proxiedInterfaces": \[\],
        "preFiltered": false,
        "advisors": \[
            {
                "order": 2147483647,
                "advice": {},
                "pointcut": {
                    "classFilter": {},
                    "methodMatcher": {
                        "runtime": false
                    }
                },
                "perInstance": true
            }
        \],
        "targetSource": {
            "target": {
                "level": 5,
                "latencyRangeStart": 1000,
                "latencyRangeEnd": 3000,
                "latencyActive": true,
                "exceptionsActive": false,
                "exception": {},
                "killApplicationActive": false
            },
            "static": true,
            "targetClass": "de.codecentric.spring.boot.chaos.monkey.configuration.AssaultProperties"
        },
        "exposeProxy": false,
        "targetClass": "de.codecentric.spring.boot.chaos.monkey.configuration.AssaultProperties"
    },
    "watcherProperties": {
        "controller": false,
        "restController": true,
        "service": true,
        "repository": false,
        "component": false
    }
}
{{< /highlight >}}

As we can see from above response, it mainly depicts key configurations pertaining to Chaos Monkey for the corresponding Spring Boot application -

*   Is Chaos Monkey for Spring Boot enabled
*   Assault Configurations for Latency, Exception and Kill Application
*   Watcher Configurations which mainly indicates Spring annotations on which configured assaults will be applied

Next we need to enable Chaos Monkey. So we will be using _'**/actuator/chaosmonkey/enable**'_ URI. As soon as it is enabled, we can clearly see it in Grafana dashboard (under 'Chaos Monkey Status') as shown below

[![](http://dhaval-shah.com/wp-content/uploads/2019/06/enable-CM4SB.png)](http://dhaval-shah.com/wp-content/uploads/2019/06/enable-CM4SB.png)

### First Chaos Experiment

As part of the first experiment, we will be generating chaos by applying **Latency Assault** via '**_/actuator/chaosmonkey/assaults_**' URI. Latency induced will be ranging from 4 - 7 seconds with level configured as 5. Request payload for configuring assault related settings will be as shown below

{{< highlight java >}}
{
"level": 5,
"latencyRangeStart": 4000,
"latencyRangeEnd": 7000,
"latencyActive": true,
"exceptionsActive": false,
"killApplicationActive": false,
"exception": {
    "type": "java.lang.IllegalArgumentException",
    "arguments": \[{
      "className": "java.lang.String",
      "value": "custom illegal argument exception"}\] }
}
{{< /highlight >}}

After running 250 requests with 4 concurrent threads for fetching all cards using [Apache Bench](https://httpd.apache.org/docs/2.4/programs/ab.html)

```
> ./ab.exe -n 250 -c 4 http://localhost:8090/cards
```

we can clearly see below metrics within the imported dashboard

1.  Total number of incoming requests for Card Service application (as per selected time frame)
2.  Total number of requests for which latency was induced
3.  Actual latency induced (in seconds)

[![](http://dhaval-shah.com/wp-content/uploads/2019/06/latency-req-count-graphs-latest.png)](http://dhaval-shah.com/wp-content/uploads/2019/06/latency-req-count-graphs-latest.png)

Since we have also configured distribution percentiles for server request we can also see metrics pertaining to response time. As we can see from below metrics, that response time of Get all Cards API is taking 7 seconds.

[![](http://dhaval-shah.com/wp-content/uploads/2019/06/latency-response-time-and-percentiles.png)](http://dhaval-shah.com/wp-content/uploads/2019/06/latency-response-time-and-percentiles.png)

### Second Chaos Experiment

As part of this experiment we will be applying **Exception Assault** to understand Card Client behavior in case Card Service ends up with Runtime exceptions. So we will modify the assault configuration by applying below request payload with same URI as above i.e. '**_/actuator/chaosmonkey/assaults_**'

{{< highlight java >}}
{
"level": 5,
"latencyRangeStart": 4000,
"latencyRangeEnd": 6000,
"latencyActive": false,
"exceptionsActive": true,
"killApplicationActive": false,
"exception": {
    "type": "java.lang.IllegalArgumentException",
    "arguments": \[{
      "className": "java.lang.String",
      "value": "custom illegal argument exception"}\] }
}
{{< /highlight >}}

After running the same tests (i.e. Fetch all Cards using the same above Apache Bench command) we can clearly see metrics within Exception Section of dashboard (as shown below). What we are able to see here is

1.  Total number of requests received by Card Service app
2.  Number of exceptions thrown
3.  Error Rate

[![](http://dhaval-shah.com/wp-content/uploads/2019/06/error-count.png)](http://dhaval-shah.com/wp-content/uploads/2019/06/error-count.png)

Within HTTP code section of dashboard we can also see number of requests that have ended up with HTTP code as 500 (due to Runtime Exception thrown by CM4SB assault)

[![](http://dhaval-shah.com/wp-content/uploads/2019/06/error-http-status-code.png)](http://dhaval-shah.com/wp-content/uploads/2019/06/error-http-status-code.png)

### Third Chaos Experiment

As part of this chaos experiment we will be applying kill application Assault to understand Card Client behavior in case Card Service inadvertently goes down. So we will change the assault configuration by applying below request payload with same URI as above i.e. '**_/actuator/chaosmonkey/assaults_**'

{{< highlight java >}}
{
     "level": 5,
     "latencyRangeStart": 4000,
     "latencyRangeEnd": 6000,
     "latencyActive": false,
     "exceptionsActive": false,
     "killApplicationActive": true, 
     "exception": { 
          "type": "java.lang.IllegalArgumentException", 
          "arguments": \[{ 
               "className": "java.lang.String", 
               "value": "custom illegal argument exception"
          }\] 
     }
}
{{< /highlight >}}

After running the same tests (i.e. Fetch all Cards using the same above Apache Bench command) we can clearly see that there is not a single failure ! So we can infer that our fallback mechanism that we have configured within Card Client application is working as per its expected behavior in case Card Service becomes unresponsive.

[![](http://dhaval-shah.com/wp-content/uploads/2019/07/fallback-apache-bench-image.png)](http://dhaval-shah.com/wp-content/uploads/2019/07/fallback-apache-bench-image.png)

# Conclusion

With this 2 part series on Chaos Engineering we were able to understand the fundamentals of Chaos Engineering along with following points -

1.  Significance of Chaos Engineering in this distributed world
2.  How to perform chaos in Spring Boot applications
3.  How to monitor application behavior and derive metrics from it

Due to  inevitability of failures in software world, I am sure that you are convinced about the need of a disciplined and an organized way to proactively test system by injecting failures - that discipline is none other than CHAOS ENGINEERING! I would like to end this 2 part series with an apt and famous axiom :

> _**The MORE you SWEAT in PEACE,**_
> 
> _**the LESS you BLEED in WAR**_<br>
> — <cite>Norman Schwarzkopf</cite>

which completely resonates with 'Why Chaos Engineering' should be formally inducted as part of (distributed) application development and its production release.