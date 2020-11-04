---
title: Unraveling the magic behind Spring Boot
author: Dhaval Shah
type: post
date: 2017-05-14T18:30:00+00:00
url: /unraveling-the-magic-behind-spring-boot/
categories:
  - Spring Boot
tags:
  - microservice
  - spring boot

---

Considering the extensive usage of [Spring Boot](https://projects.spring.io/spring-boot/) for building [Cloud Native Architecture](http://dhaval-shah.com/understanding-cloud-native-architecture-with-an-example/), I embarked on the journey of utilizing it in my reference [Cloud Native application](https://github.com/dhaval201279/cloud-native-java-demo). When I ran my first application I was literally flabbergasted with the magic [Spring Boot](https://projects.spring.io/spring-boot/) does under the hood, using which it camouflages the complexity and challenges of building enterprise applications. In order to understand capabilities of Spring Boot and thereby have its justifiable usage along with its troubleshooting skills I felt a dire need of demystifying the magic behind it!

The thing that makes an app a Spring boot application is the annotation *@SpringBootApplication*. This is basically applied to the class having main method i.e. entry point of the application. It is a meta annotation which internally comprises of 3 annotations -

[![](http://dhaval-shah.com/wp-content/uploads/2017/05/annotation-2-spring-boot-internals.jpg)](http://dhaval-shah.com/wp-content/uploads/2017/05/annotation-2-spring-boot-internals.jpg)

1.  *@SpringBootConfiguration* - Its a specialization of spring framework's configuration annotation and it mainly helps in automatically discovering Spring configurations
2.  *@ComponentScan* - Its a core spring framework standard component scan annotation.
3.  *@EnableAutoConfigurations* - It primarily switches on Spring boot's intellectual configurations based on class path, properties that has been configured

{{< highlight java >}}
package com.its.hello;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class HelloApp {

	public static void main(String\[\] args) {
		SpringApplication.run(HelloApp.class,args);
	}
}
{{< /highlight >}}

# @ComponentScan

So in this example we have *HelloApp* within *com.its.hello* package annotated with *@SpringBootApplication*. So all the components (i.e. classes annotated with *@Component*) underneath package *com.its.hello* or its sub packages (e.g. *com.its.hello.alpha* etc.) will be picked up via component scanning. So in nutshell, this gets triggered as a part of *@SpringBootApplication*

{{< highlight java >}}
package com.its.hello;

import org.springframework.boot.SpringApplication;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;

@Configuration
@ComponentScan
public class HelloApp {

	public static void main(String\[\] args) {
		SpringApplication.run(HelloApp.class,args);
	}
}
{{< /highlight >}}

If application does not require auto configuration, one can still create an application without *@SpringBootApplication* i.e. by using *@Configuration* and *@ComponentScan*

So within a package you not only find *Component* but you will also find other annotations which are specializations of component i.e.

1.  *@Bean*
2.  *@Repository*
3.  *@Configuration*
4.  *@Controller*
5.  *@Service*
6.  *@RestController*

All these annotations will mainly create beans within your application context.

When any spring application bootstraps, it will be able to scan all the specializations of *Component* (as indicated above). i.e. Apart from *@Component* and *@Bean*, it will also look for *@Configuration* classes and bean methods declared within *Component* and its specialization. So all of these together constitute to form beans pertaining to user configurations within application context. What's in for migrating legacy Spring projects - There is a provision regarding usage Import annotation which will allow direct reference to xml file. And by this one can still keep its wiring within configurations file i.e. xml and refer it within Spring boot application. This facilitates gradual migration instead of a big bang migration.

### Guidelines
*   It is strongly recommend to have dedicated package for each application and thereafter put your Spring Boot application at the root of application.
*   For any spring boot application its very common gotcha when a new component or package is created and it is not picked up by spring boot application. So its worth checking whether the component created is in the same package where spring boot application is created or any of its sub packages.
*   It not only supports scanning components with @Configuration but it can also support Spring's XML based configuration i.e. *<context:component-scan>*. So it mainly provides user configuration i.e. beans which are created as a part of business workflow can be

# @EnableAutoConfigure

Auto configurations are switched on using *@EnableAutoConfiguration* annotation, which will be implicitly done if we are using *@SpringBootApplication*. So this in a way will also create set of beans within application context. It is an auto configuration which intellectually configures beans that are likely to be required by the application.

Important point to keep in mind is these are i.e User Configuration and Auto Configuration are two disparate steps / processes that happen whilst creation of application context and that too in a chronological order i.e. initially beans related to user configuration gets created first and then the beans related to auto configuration gets created.

Lets create a Configuration class which is neither going to be scanned, nor going to be referred via *import* statement. Instead we are going to register it with specific file, that Spring Boot will look on our classpath during application bootstrap.

{{< highlight java >}}
package hello.autoconfigure;

import hello.ConsoleHelloService;
import hello.HelloService;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;


@Configuration
public class HelloAutoConfiguration {

    @Bean
    public HelloService helloService() {
        return new ConsoleHelloService();
    }
}
{{< /highlight >}}

For enabling auto configuration one needs to add a value for the key named `org.springframework.boot.autoconfigure.EnableAutoConfiguration` within *src/main/resources/META-INF/spring.factories* file. Value is a comma separated list of fully qualified name of auto configuration classes that you want to switch on by enabling it via *@EnableAutoConfigure*

{{< highlight java >}}
org.springframework.boot.autoconfigure.EnableAutoConfiguration=hello.autoconfigure.HelloAutoConfiguration
{{< /highlight >}}

This is Spring Boot's standard mechanism for enabling auto configuration.

# Auto configuration with condition(s)

{{< highlight java >}}
package hello.autoconfigure;

import hello.ConsoleHelloService;
import hello.HelloService;
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
@ConditionalOnClass(HelloService.class)
public class HelloAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public HelloService helloService() {
        System.out.println("Entering HelloAutoConfiguration : helloService");
        System.out.println("Instantiating ConsoleHelloService and leaving HelloAutoConfiguration : helloService");
        return new ConsoleHelloService();

    }
}
{{< /highlight >}}

*@ConditionalOnMissingBean :* It indicates that this bean will only be created only when there is no other bean of type *HelloService* within application context

*@ConditionalOnClass (HelloService.class) :* Before instantiation of any bean one can always validate whether all the prerequisites are present. Lets take a hypothetical scenario wherein auto configuration code is separate from services. So in that case one can still safe guard auto configuration instantiation by putting *@ConditionalOnClass*

Within Spring, there can be multiple conditions on a given class / method, these conditions are evaluated in a specific order (as mentioned below) :

1.  **_@ConditionalOnClass_**
2.  **_@ConditionalOnMissingClass_**
3.  **_@ConditionalOnResource_**
4.  **_@ConditionalOnJNDI_**
5.  **_@ConditionalOnWebApplication_**
6.  **_@ConditionalOnProperty_**
7.  **_@ConditionalOnExpression_**
8.  **_@ConditionalOnSingleCandidate_**
9.  **_@ConditionalOnBean_**
10.  **_@ConditionalOnMissingBean_**

Bottom three i.e. *@ConditionalOnSingleCandidate*, *@ConditionalOnBean* and *@ConditionalOnMissingBean* are performance intensive as their corresponding validations need to be done at application context level i.e. they are dependent on the state of application context; Hence one needs to be extremely cautious in terms of its usage. Rest all of them are related to infrastructure / environment.

# Auto configuration with property file(s)

{{< highlight java >}}
package hello.autoconfigure;

import org.springframework.boot.context.properties.ConfigurationProperties;

@ConfigurationProperties("hello")
public class HelloProperties {
	private String prefix;
	private String suffix = "!";

	public String getPrefix() {
		return prefix;
	}

	public void setPrefix(String prefix) {
		this.prefix = prefix;
	}

	public String getSuffix() {
		return suffix;
	}

	public void setSuffix(String suffix) {
		this.suffix = suffix;
	}

}
{{< /highlight >}}

Configuration properties are simple POJOs which asks spring boot to bind environment with POJO instance via certain key. In our above example *hello* is the key through which we can refer *HelloProperties* and thereby set its attributes.

# Ordering within Auto Configuration
If there is a need to maintain order in terms of instantiating multiple Auto Configurations, Spring Boot allows to ensure its chronological sequence by using **_@AutoConfigureAfter_** and _**@AutoConfigureBefore**_

# Understanding auto configuration report

For getting auto configuration report, we need to run our spring boot app in debug mode. And on successful start of an application, spring boot will emit quiet exhaustive logging which will in turn help us to understand the configurations of the application. It not only tells us about positive matches but also conveys about negative matches i.e. the one for which dependencies are not present. Look at the snapshot of auto configuration report extracted from the console logs

{{< highlight java >}}
=========================
AUTO-CONFIGURATION REPORT
=========================


Positive matches:
-----------------

   GenericCacheConfiguration matched:
      - Cache org.springframework.boot.autoconfigure.cache.GenericCacheConfiguration automatic cache type (CacheCondition)

   HelloAutoConfiguration#helloService matched:
      - @ConditionalOnMissingBean (types: hello.HelloService; SearchStrategy: all) did not find any beans (OnBeanCondition)
      - ValidHelloPrefix valid prefix ('Howdy') is available (OnValidHelloPrefixCondition)

   JmxAutoConfiguration matched:
      - @ConditionalOnClass found required class 'org.springframework.jmx.export.MBeanExporter'; @ConditionalOnMissingClass did not find unwanted class (OnClassCondition)
      - @ConditionalOnProperty (spring.jmx.enabled=true) matched (OnPropertyCondition)

   JmxAutoConfiguration#mbeanExporter matched:
      - @ConditionalOnMissingBean (types: org.springframework.jmx.export.MBeanExporter; SearchStrategy: current) did not find any beans (OnBeanCondition)

   JmxAutoConfiguration#mbeanServer matched:
      - @ConditionalOnMissingBean (types: javax.management.MBeanServer; SearchStrategy: all) did not find any beans (OnBeanCondition)

   JmxAutoConfiguration#objectNamingStrategy matched:
      - @ConditionalOnMissingBean (types: org.springframework.jmx.export.naming.ObjectNamingStrategy; SearchStrategy: current) did not find any beans (OnBeanCondition)

   NoOpCacheConfiguration matched:
      - Cache org.springframework.boot.autoconfigure.cache.NoOpCacheConfiguration automatic cache type (CacheCondition)

   PropertyPlaceholderAutoConfiguration#propertySourcesPlaceholderConfigurer matched:
      - @ConditionalOnMissingBean (types: org.springframework.context.support.PropertySourcesPlaceholderConfigurer; SearchStrategy: current) did not find any beans (OnBeanCondition)

   RedisCacheConfiguration matched:
      - Cache org.springframework.boot.autoconfigure.cache.RedisCacheConfiguration automatic cache type (CacheCondition)

   SimpleCacheConfiguration matched:
      - Cache org.springframework.boot.autoconfigure.cache.SimpleCacheConfiguration automatic cache type (CacheCondition)

   SpringApplicationAdminJmxAutoConfiguration matched:
      - @ConditionalOnProperty (spring.application.admin.enabled=true) matched (OnPropertyCondition)

   SpringApplicationAdminJmxAutoConfiguration#springApplicationAdminRegistrar matched:
      - @ConditionalOnMissingBean (types: org.springframework.boot.admin.SpringApplicationAdminMXBeanRegistrar; SearchStrategy: all) did not find any beans (OnBeanCondition)


Negative matches:
-----------------

   ActiveMQAutoConfiguration:
      Did not match:
         - @ConditionalOnClass did not find required classes 'javax.jms.ConnectionFactory', 'org.apache.activemq.ActiveMQConnectionFactory' (OnClassCondition)

   AopAutoConfiguration:
      Did not match:
         - @ConditionalOnClass did not find required classes 'org.aspectj.lang.annotation.Aspect', 'org.aspectj.lang.reflect.Advice' (OnClassCondition)

   ArtemisAutoConfiguration:
      Did not match:
         - @ConditionalOnClass did not find required classes 'javax.jms.ConnectionFactory', 'org.apache.activemq.artemis.jms.client.ActiveMQConnectionFactory' (OnClassCondition)

   BatchAutoConfiguration:
      Did not match:
         - @ConditionalOnClass did not find required classes 'org.springframework.batch.core.launch.JobLauncher', 'org.springframework.jdbc.core.JdbcOperations' (OnClassCondition)
...
..
.
{{< /highlight >}}

Looking at the positive matches we are able to see custom auto configuration we have implemented for our application along with its custom conditions.

# Event Life Cycle of Spring Boot applications

  [![](http://dhaval-shah.com/wp-content/uploads/2017/05/event-life-cycle-spring-boot-internals.jpg)](http://dhaval-shah.com/wp-content/uploads/2017/05/event-life-cycle-spring-boot-internals.jpg) 

Lets try understanding different events that gets published once spring boot application bootstraps -

1.  When spring boot application runs, first thing that happens is _**ApplicationStartedEvent**_ is published. This is the point where Logging initialization begins
2.  Next thing that happens is, _**ApplicationEnvironmentPreparedEvent**_ is published. This means that environment is created which consists of default set of property sources. This is the standard list of property sources one can get from spring framework. As one observes till here, nothing specific to Spring boot has happened within the environment. So there will be 2 property source in there - One for environment variables and other for system properties. So far just the environment has been created. Neither the application context nor the beans have been created so far; and not even the context is refreshed yet. Next step within this will be to read config files. This is where spring boot specific things get started. i.e. it start reading _application.properties_  / _application.yml_ files and the profile specific variants if any. Within spring boot you can make profile specific property files - e.g. _application.properties_, _application-<profile name>.properties_. So profile specific application.properties will be read only when profile is active. So this in a way allows to construct beans based on a specific profile. So this is useful in scenarios where one wants certain set of configurations for certain environments - e.g. In dev environment one want to enable debug logs, or tells hibernate to display sql statements  etc. So now all the config files are read and also one or more property sources are read. This is where ordering of property sources helps. So in layman's terms profile specific property source (e.g. application-dev.properties) will always take precedence against your general property source (e.g. application.properties). At this point, environment has been setup and now its in default form for spring boot; so it has property source for configured environment and also property source for system properties. It also has got config files that it has read in. And this environment is passed to all the configured post processors within _META-INF/spring.factories. _Now the logging initialization formally would get completed. Rationale behind not completing logging initialization earlier is due to the fact that logging settings (i.e. log level, lot pattern etc.) can be configured within configuration files - kind of a chicken and egg problem :)
3.  Next thing that happens is we fire _**ApplicationPreparedEvent**_ i.e. spring boot says, I have done all the setup as a part of environment preparation. And at this point, application context is refreshed i.e. bean starts getting created, dependency injection happens etc.
4.  Now spring framework's _**ContextRefreshedEvent**_ is fired. All the previous events were spring boot specific. Embedded servlet container connectors are started  now.
5.  **EmbeddedServletContainerInitializedEvent** gets fired next. So now once the container along with its connectors are up and running, this event is published which mainly encapsulates port (local.server.port variable) on which container has started. It also updates the environment with above variable (i.e. local.server.port)
6.  Last of all is the _**ApplicationReadyEvent**_ is published - it just says application is up and running - Context is refreshed, server is bootstrapped, connectors are started, published local server port property.

This is just an article that emits my learnings and understanding that I was able to do as a part of viewing and following the [talk](https://youtu.be/uof5h-j0IeE) given by [Stephen Nicoll](https://twitter.com/snicoll) and [Andy Wilkinson](https://twitter.com/ankinson) 

Source Code - [here](https://github.com/dhaval201279/spring-boot-magic).