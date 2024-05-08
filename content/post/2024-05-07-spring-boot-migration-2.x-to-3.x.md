---
title: Migrating Spring Boot application from 2.x to 3.x
author: Dhaval Shah
type: post
date: 2024-05-07T00:00:50+00:00
url: /spring-boot-migration-from-2.x-to-3.x/
categories:
  - Spring Boot
  - Microservices
tags:
  - spring boot
  - microservice
  - cloud native
thumbnail: "images/wp-content/uploads/2024/05/spring-boot-image-with-text-nd-birds.png"
---


[![](https://www.dhaval-shah.com/images/wp-content/uploads/2024/05/spring-boot-image-with-text-nd-birds.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2024/05/spring-boot-image-with-text-nd-birds.png)
-----------------------------------------------------------------------------------------------------------------------------------------
# Background
Lot of my pet projects have been built using [Spring Boot 2.x version](https://docs.spring.io/spring-boot/docs/2.2.0.RELEASE/reference/htmlsingle/). Same might be applicable for all the enterprises and organizations who have been building [Microservices](https://en.wikipedia.org/wiki/Microservices) based applications for their products / services using [Spring Boot](https://spring.io/projects/spring-boot).
During last year, Spring community made a major version upgrade and thereafter released [Spring Boot 3.2](https://spring.io/projects/spring-boot) version with bunch of new features.

First thing that comes to our mind is 
**Why should I migrate my application from 2.x to 3.x**

# Why migrate to Spring Boot 3.x
Below are some of the compelling reasons to help you convince yourself for migrating your application from 2.x to 3.x
1. **[Java 17](https://www.oracle.com/java/technologies/javase/jdk17-archive-downloads.html) Support:** Spring Boot 3.x supports Java 17 and its higher versions, allowing you to leverage the latest language features and performance improvements
2. **Reduced Memory Footprint:** Spring Boot 3.x has been optimized to reduce the memory footprint of your applications
3. **Support for [Virtual Threads](https://openjdk.org/jeps/444):** JDK feature that allows to overcome limitation of platform threads
4. **Native Images:** Spring Boot 3.x provides better support for building native executables using GraalVM, leading to significant improvements in application bootstrap time
5. **End of Support for Spring Boot 2.x:** Support for Spring Boot 2.x has ended in 2023. Migrating to 3.x ensures that your applications continue to receive updates and security patches

Now that we are convinced, lets take the plunge and upgrade our Spring Boot application from 2.x to the latest and greatest 3.x version. But amidst excitement, migration process may seem to be daunting.
Fear not, for [OpenRewrite](https://docs.openrewrite.org/) is here to make your upgrade relatively smooth and enjoyable :smile:

# What is OpenRewrite
[OpenRewrite](https://docs.openrewrite.org/) is a powerful code-transformation library that automates the process of migrating your Spring Boot application. It analyzes your codebase, identifies outdated patterns and dependencies, and suggests changes to bring your app up to speed with the latest Spring Boot version. Think of it as your personal migration assistant, guiding you through the process with precision and efficiency.

# Application to be migrated
[WebClient Demo Project](https://github.com/dhaval201279/RxWebClientDemo.git)

## Pre-requisites
1. **Java 17 or any other higher JDK version :** Ensure you have [JDK 17](https://openjdk.org/projects/jdk/17/) or any other higher version. I will be using [JDK 21](https://openjdk.org/projects/jdk/21/)

# Steps performed for migrating above application from 2.2 to 3.2
1. Since my WebClient Demo project was on 2.2.7, I first upgraded my project to 2.7.18 by modifying *build.gradle* file

``` java
  plugins {
    . .
    id 'org.springframework.boot' version '2.7.18'
    . .
  }
```
2. As per [OpenRewrite recipes](https://docs.openrewrite.org/recipes/java/spring/boot3/upgradespringboot_3_2), 3 steps are to be performed
   1. Add *org.openrewrite.rewrite* as a plugin
  
    ``` java
      plugins {
        . . . 
        id("org.openrewrite.rewrite") version("6.12.0")
        . . .
      }
    ```

   2. Add *activeRecipe* i.e. ***UpgradeSpringBoot_3_3*** as *rewrite* task

    ``` java
      rewrite {
	      activeRecipe("org.openrewrite.java.spring.boot3.UpgradeSpringBoot_3_2")
      }
    ```    

   3. Add OpenRewrite's *rewrite-spring* as project dependency
    ``` java
      dependencies {
	      . . . 
        rewrite("org.openrewrite.recipe:rewrite-spring:5.8.0")
        . . .
      }
    ```
3. Load all the [Gradle](https://gradle.org/) changes
4. You may want to trigger *rewriteDryRun* to understand list of changes that are required for this migration. It will be listed within a file with *.patch* extension
5. Add JDK 17 or any higher version to your project settings and module settings (within your [Intellij Idea](https://www.jetbrains.com/idea/)project)
6. Update *gradle-wrapper.properties* to ensure that your project uses JDK compatible version of Gradle. Since I am using JDK 21, I have added *gradle-8.5-bin.zip* as value to *distributionUrl* key
``` properties
  distributionBase=GRADLE_USER_HOME
  distributionPath=wrapper/dists
  distributionUrl=https\://services.gradle.org/distributions/gradle-8.5-bin.zip
  zipStoreBase=GRADLE_USER_HOME
  zipStorePath=wrapper/dists
```
7. Sync the Gradle project and thereafter follow it up with *Clean* and *Build*
8. Address all the compilation issues
9. Validate and make required changes as per its need and applicability
10. Test your application exhaustively

# Conclusion
Migrating to Spring Boot 3.x opens the door to a world of exciting new features and performance enhancements. With OpenRewrite as your guide, you can confidently navigate the migration process and reap the benefits of the latest Spring Boot version. So, buckle up, start your engines, and let OpenRewrite take you on a smooth and enjoyable migration journey!

# Source Code for reference
[RxWebClientDemo](https://github.com/dhaval201279/RxWebClientDemo)

**Happy Spring Boot 3.x Migration!**