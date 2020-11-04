---
title: Monitoring Spring Boot application using Actuator, Micrometer, Prometheus and Grafana
author: Dhaval Shah
type: post
date: 2018-11-02T10:37:46+00:00
url: /monitoring-spring-boot-application/
categories:
  - Cloud Native Architecture
  - Spring Boot
tags:
  - Actuator
  - Grafana
  - Prometheus
  - spring boot

---

[![](http://dhaval-shah.com/wp-content/uploads/2018/11/final-main-image.png)](http://dhaval-shah.com/wp-content/uploads/2018/11/final-main-image.png)

[Spring Boot](http://spring.io/projects/spring-boot) helps developers to implement enterprise grade applications which can be pushed to production in no time. Once application gets into production and if we strongly believe in [vedic](https://en.wikipedia.org/wiki/Vedas) philosophy of [Karma](https://en.wikipedia.org/wiki/Karma) :), we are bound to experience [Murphy's Law](https://en.wikiquote.org/wiki/Murphy%27s_law).

> Whatever can go wrong, will go wrong<br>

Considering nature of Distributed Architecture, Observability and Monitoring of application and infrastructure is of paramount importance. Thanks to the [Spring Boot](http://spring.io/projects/spring-boot) team, who has shipped Spring Boot Actuator from the very first release.

In order to understand Spring Boot Actuator and its usage w.r.t observability and monitoring in real world application, lets construct a system with hypothetical requirements - We will implement an application which consists of two applications -

1.  Reservation Endpoints - which consists of public APIs for managing Reservation workflows
2.  Reservation Service - responsible for catering to Command operations (as a part of [CQS](https://en.wikipedia.org/wiki/Command%E2%80%93query_separation)). It is primarily entrusted with responsibility of consuming events / messages pertaining to Save Reservation operation

**Application Architecture**
----------------------------

[![Sample App Architecture](http://dhaval-shah.com/wp-content/uploads/2018/10/spring-boot-actuator-monitoring.jpg)](http://dhaval-shah.com/wp-content/uploads/2018/10/spring-boot-actuator-monitoring.jpg)

Source code of both the applications can be referred at -

1.  [Reservation Endpoint](https://github.com/dhaval201279/spring-boot-micrometer-prometheus-grafana-demo)
2.  [Reservation Service](https://github.com/dhaval201279/spring-boot-rabbitmq-consumer-micrometer-demo)

Below are the APIs exposed by Reservation Endpoints

| Business Operation   | URI     | HTTP Method   | Description   |
| --------  | -------- | ------ |
| Add Reservation | */reservation* | POST | Allows to add Reservation with user details |
| Fetch all Reservations | */reservations* | GET | Returns list of all the users whose reservation has been done |
| Fetch all Reservations by first name | */reservation/firstName/{firstName}* | GET | Returns list of all the users with given first name whose reservation has been done |
| Fetch all Reservations by last name | */reservation/lastName/{lastName}* | GET | Returns list of all the users with given last name whose reservation has been done |

Before we delve into nuances of Monitoring and Observability of distributed systems, lets understand how to get required metrics

Spring Boot Team has shipped Spring Boot Actuator from the very first release.

So Spring Boot actuator exposes set of endpoints over HTTP or JMX. It can be enabled by adding 'spring-boot-starter-actuator' dependency within build file -

**Maven Configurations**

{{< highlight java >}}
<dependencies>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-actuator</artifactId>
	</dependency>
</dependencies>
{{< /highlight >}}

**Gradle Configurations**
{{< highlight java >}}
dependencies {
	compile("org.springframework.boot:spring-boot-starter-actuator")
}
{{< /highlight >}}

When you run your application, it exposes set of endpoints relative to '_/actuator'_ URI which are as shown below

[![](http://dhaval-shah.com/wp-content/uploads/2018/10/acutator-URIs.png)](http://dhaval-shah.com/wp-content/uploads/2018/10/acutator-URIs.png)

The same can be viewed over JMX

One can see the complete list of actuator endpoints at the [official documentation](https://docs.spring.io/spring-boot/docs/current/reference/html/production-ready-endpoints.html).

Spring Boot properties that can be used for enabling / disabling actuator endpoints -

{{< highlight java >}}
########################## Actuator Configurations
# Use "\*" to expose all endpoints, or a comma-separated list to expose selected ones i.e. health,info
management.endpoints.web.exposure.include = \*

# Use "\*" to expose all endpoints, or a comma-separated list to expose selected ones
management.endpoints.jmx.exposure.include = \*
{{< /highlight >}}

# Customization of Management Endpoints

Since its inception, Spring has always preferred giving customization and extension hook points at appropriate places within its framework. Same holds good for Actuator endpoints - which is apparent from below table

| Customiation Type   | Key within *application.properties*     | Example   | Description     |
| --------  | -------- | ------ | ---------- |
| Customizing management endpoint | *management.endpoints.web.base-path* | *management.endpoints.web.base-path=/abc* | Changes the endpoint from /actuator/{id} to /abc/{id} |
| Customizing management server address | *management.server.address* | *management.server.address=127.1.2.3* | Customizes management server's address to 127.1.2.3. This may be required if one wants to listen only on ops monitored network |
| Customizing management server port | *management.server.port* | *management.server.port=8181* | Changes default port of management server to 8181. This may be needed if one has specific need of exposing endpoint on separate HTTP port (governed by the rules of its data centre) |
| Disable HTTP endpoints | *management.server.port* OR *management.endpoints.web.exposure.exclude* | *management.server.port=-1* OR *management.endpoints.web.exposure.exclude=/** | Disables all the HTTP endpoints exposed as a part of Actuator framework |

Note - Management endpoints can be secured in terms of its access by configuring required ssl configuration within application.properties file

# Health Information

In today's contemporary world of distributed architecture, gathering health information about application and infrastructure is imperative - as it not only helps to monitor the system but also helps to setup alerts for proactive actions that need to be taken. With actuator one can get detailed health information about application along with its infrastructure by configuring below property within _application.properties_ file

{{< highlight java >}}
########################## Actuator Configurations

# To get the complete details including the status of every health indicator that was checked as 
# part of the health check-up process
management.endpoint.health.show-details=always
{{< /highlight >}}

By enabling health endpoints within our application, we will not only be able to see application's health information but we will also get to see infrastructure's health i.e. MongoDB and Rabbitmq as they are auto-configured

[![](http://dhaval-shah.com/wp-content/uploads/2018/10/app-health-endpoints.png)](http://dhaval-shah.com/wp-content/uploads/2018/10/app-health-endpoints.png)

# Metrics

Spring Boot mainly provides below metrics via Actuator
*   JVM metrics, report utilization of:
    *   Various memory and buffer pools
    *   Statistics related to garbage collection
    *   Threads utilization
    *   Number of classes loaded/unloaded 
*   CPU metrics
*   File descriptor metrics
*   Logback metrics: record the number of events logged to Logback at each log level
*   Uptime metrics: report a gauge for uptime and a fixed gauge representing the application’s absolute start time
*   Tomcat metrics
*   [Spring Integration](https://docs.spring.io/spring-integration/docs/current/reference/html/system-management-chapter.html#micrometer-integration) metrics
*   Spring MVC metrics
*   Spring Webflux metrics
*   RestTemplate metrics

So with this understanding of basic capabilities of Spring Boot's Actuator, can we perform enterprise monitoring? Tentatively YES, but lot of things need to be built from scratch viz. Application that constantly fetches data from Actuator and then some how renders the captured data in human readable form. Spring framework community once again absolves software fraternity from doing such heavy lifting - Actuator provides dependency management and auto configuration of [Micrometer](https://micrometer.io/). Micrometer is

**"vendor-neutral interfaces for timers, gauges, counters, distribution summaries, and long task timers with a dimensional data model that, when paired with a dimensional monitoring system, allows for efficient access to a particular named metric with the ability to drill down across its dimensions"**

And Micrometer in principle -

*   is metrics supporting library for JVM application
*   supports plethora of monitoring tools viz. [Atlas](https://github.com/Netflix/atlas), Datadog, Graphite, [Prometheus](https://prometheus.io/) etc.

We will be using Prometheus to demonstrate its capabilities using above example. In order to configure Prometheus, we just need to add below gradle dependency

{{< highlight java >}}
dependencies {
	compile('io.micrometer:micrometer-registry-prometheus')
}
{{< /highlight >}}

When we access */actuator/prometheus* endpoint we can see lot of information pertaining to -
*   JVM Memory
*   Garbage Collection
*   Disk Space
*   Rabbitmq
*   Spring Integration channels

so on and so forth. Looking at the screen shot below one can realize that it is certainly not that user friendly in terms of inferring from data that is being emitted

[![](http://dhaval-shah.com/wp-content/uploads/2018/10/actuator_prometheus_url.png)](http://dhaval-shah.com/wp-content/uploads/2018/10/actuator_prometheus_url.png)

So we will be using [Prometheus](https://prometheus.io/) and [Grafana](https://grafana.com/) to easily infer from the data emitted as a part of Spring Boot Actuator. Prometheus is an open source monitoring tool developed by [SoundCloud](https://soundcloud.com/).  Grafana is an open platform for beautiful monitoring and analytics of time series data.

# Setting up Prometheus

Download Prometheus for your corresponding environment. Update _prometheus.yml _within its root as per below configurations
{{< highlight java >}}
global:
  scrape\_interval:     15s # By default, scrape targets every 15 seconds.
scrape\_configs:
  - job\_name: 'prometheusJN'
    scrape\_interval: 5s
    static\_configs:
      - targets: \['localhost:9090'\]
  - job\_name: 'springJN1'
    scrape\_interval: 5s
    metrics\_path: '/actuator/prometheus'
    static\_configs:
      - targets: \['localhost:8090'\]
  - job\_name: 'springJN2'
    scrape\_interval: 5s
    metrics\_path: '/actuator/prometheus'
    static\_configs:
      - targets: \['localhost:8091'\]
{{< /highlight >}}

Note - 8090 and 8091 are the 2 ports on which my applications i.e. Reservation Endpoint and Reservation Service are running respectively

Run Prometheus by executing following command

{{< highlight java >}}
_.\\prometheus.exe --config.file=prometheus.yml_.
{{< /highlight >}}

Once Prometheus is successfully up and running you can see below console which means that it is not only running on 9090 but also scrapping metrics for you as per above configurations

[![](http://dhaval-shah.com/wp-content/uploads/2018/10/prometheus-console-9090.png)](http://dhaval-shah.com/wp-content/uploads/2018/10/prometheus-console-9090.png)

# Setting up Grafana

Download Grafana for your corresponding environment. Setting up Grafana will be a 3 step process :

## 1. Datasource Configuration

Update *grafana-datasource.yml* (at *<GRAFANA\_HOME>/conf/provisioning/datasources*) file with below configurations - 

{{< highlight java >}}
apiVersion: 1

datasources:
- name: prometheusDS
  type: prometheus
  access: direct
  url: http://localhost:9090
{{< /highlight >}}

## 2. Dashboard Configuration

Update *grafana-dashboard.yml* (at *<GRAFANA\_HOME>/conf/provisioning/dashboards*) file with below configurations

{{< highlight java >}}
apiVersion: 1

providers:
- name: 'default'
  folder: ''
  type: file
  options:
    path: H:\\\\softwares\\\\Tech\\\\grafana-5.1.3\\\\public\\\\dashboards
{{< /highlight >}}

# 3. Adding Dashboard

Copy *mygrafana-dashboard.json* file within *<GRAFANA\_HOME>/public/dashboards*

Start Grafana server by executing following command

`> .\\grafana-server.exe`

Note -  All the above configuration files of Prometheus and Grafana are available [here](https://github.com/dhaval201279/spring-boot-micrometer-prometheus-grafana-demo/tree/master/src/main/resources)

Login to Grafana console with username and password. Configure Prometheus data source so that it can get data from Prometheus to render it to the UI. Along with data source, we will add some pre-configured dashboards to visualize health of our applications viz.

*   [Java Micrometer](https://grafana.com/dashboards/3308) Dashboard
*   [Spring Boot Statistics](https://grafana.com/dashboards/6756) Dasboard
*   Spring Throughput Dashboard

So with this entire setup we are equipped with necessary infrastructure to perform enterprise monitoring of our application.

We will now simulate load comprising of all 3 API calls -

1.  Save Reservation
2.  Fetch all Reservations
3.  Fetch specific Reservation

Note - We can even simulate load using [Apache Bench](https://httpd.apache.org/docs/2.4/programs/ab.html) which provides finer control in terms of performance testing and benchmarking

Below are some of the screen shots taken from Grafana dashboards, which are giving some insights about the state of application when it was under load

## Basic Statistics and CPU usage

[![](http://dhaval-shah.com/wp-content/uploads/2018/11/spring-boot-stats-basic-stats.png)](http://dhaval-shah.com/wp-content/uploads/2018/11/spring-boot-stats-basic-stats.png)

## Logback Statistics

[![](http://dhaval-shah.com/wp-content/uploads/2018/11/spring-boot-stats-logback-stats.png)](http://dhaval-shah.com/wp-content/uploads/2018/11/spring-boot-stats-logback-stats.png)

## JVM Statistics

[![](http://dhaval-shah.com/wp-content/uploads/2018/11/jvm-micrometer-jvm2.png)](http://dhaval-shah.com/wp-content/uploads/2018/11/jvm-micrometer-jvm2.png)

[![](http://dhaval-shah.com/wp-content/uploads/2018/11/jvm-micrometer-jvm3.png)](http://dhaval-shah.com/wp-content/uploads/2018/11/jvm-micrometer-jvm3.png)

## Application Throughput

[![](http://dhaval-shah.com/wp-content/uploads/2018/11/throughput-stats.png)](http://dhaval-shah.com/wp-content/uploads/2018/11/throughput-stats.png)

# Conclusion
So we saw how Spring Boot provides exhaustive time series metrics about different facets of health of an application which can help team to infer lot of insights from the available data by using monitoring and rendering tools like Prometheus / Grafana. Having said that, usage of Grafana is not just about setting up and configuring dashboards - one can create his own dashboard based on type of insights one would really like to have. One should also consider infrastructure (MongoDB and Rabbitmq here) as key aspect of the application and hence they should also be monitored closely via monitoring tools - this might be a candidate of another blog sometime later.