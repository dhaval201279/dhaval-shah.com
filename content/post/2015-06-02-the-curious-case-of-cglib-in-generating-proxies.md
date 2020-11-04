---
title: The curious case of CGLIB in generating Proxies !
author: Dhaval Shah
type: post
date: 2015-06-02T18:42:08+00:00
url: /the-curious-case-of-cglib-in-generating-proxies/
categories:
  - Performance
tags:
  - cglib
  - OOM
  - performance
  - spring

---

Recently I was required to identify a memory leak in one of the enterprise application running in production.

Fortunately we were able to have[heap dumps](http://www.wikiconsole.com/wiki/introduction-to-heap-dump-core-dump/)from the production environment. After analyzing few heap dumps I was able to trace a uniform pattern; which I felt might be one of the potential root causes for the memory leak. Within all the heap dumps, the same object was holding almost 25 % - 35 % of heap memory.

So the next step that I took was to review the code responsible for its instantiation. Needless to say that code includes Java and XML, as in today’s highly flexible and configurable world quiet a bunch of things are being done via XML i.e. Spring’s application context file J.

Since instantiation of potential suspect (for memory leak) was being done via XML configuration, I was not required to do any frantic search!

Spring’s application context filed look like –

{{< highlight java >}}
<bean class="org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator" />
{{< /highlight >}}

Generally speaking, Spring creates proxies either by using JDK Dynamic proxy or [CGLIB](https://github.com/cglib/cglib). As per the above configuration, Spring will create proxies using CGLIB, and as a result the proxy classes of potential suspect seems to hold strong references to its advice and target object, through holding the ProxyCallbackFilter instance. And its cumulative effect was leading to Out of Memory issue.

Reason being all the [advice](http://docs.spring.io/spring/docs/current/spring-framework-reference/html/aop.html) or target objects will occupy a large amount of memory, as none of those objects will get garbage collected as long as the CGLIB generated proxy classes are still hanging around in the class loader.

So updating above _AutoProxyCreator_ by setting _proxyTargetClass=’false', it will ensure that _JDKDynamicProxy_ is used for creating proxies for advice and target objects.

{{< highlight java >}}
<bean class="org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator"/> 
     <property name="proxyTargetClass" value="false'"/> 
</bean>
{{< /highlight >}}

Another approach of solving the same problem is by using _BeanNameAutoProxyCreator_ But this would require additional effort against the previous one, as the team will have to keep on updating list of beans that should be auto proxied.

{{< highlight java >}}
<bean class="org.springframework.aop.framework.autoproxy.BeanNameAutoProxyCreator"> 
     <property name="beanNames"> 
          <list> 
               <value>\*Service</value> 
          </list> 
     </property> 
     <property name="interceptorNames"> 
          <list> 
               <value>customerAdvisor</value> 
          </list> 
     </property> 
</bean>
{{< /highlight >}}

So the memory leak issue was resolved by forcing Spring to use JDK Dynamic Proxy instead of CGLIB to create proxies of advices and its related target objects !