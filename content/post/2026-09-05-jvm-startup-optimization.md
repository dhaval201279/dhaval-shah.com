---
title: "The 280-Millisecond Lie: How Ready Pods Were Quietly Costing a Fintech Client"
author: Dhaval Shah
type: post
date: 2026-09-05T01:00:50+00:00
url: /jvm-startup-optimization/
categories:
  - cloud-native-architecture
  - performance-engineering
tags:
  - cloud-native-architecture
  - performance-engineering
thumbnail: "images/wp-content/uploads/2026/08/architect-dilemma-vibe-coded-app.png"
---

[![](https://www.dhaval-shah.com/images/wp-content/uploads/2026/08/architect-dilemma-vibe-coded-app.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2026/08/architect-dilemma-vibe-coded-app.png)
-----------------------------------------------------------------------------------------------------------------------------------------
# The Scene
A CTO of a small FinTech company had scheduled a meeting with me along with senior folks from his engineering team.

**_CTO :_** (shares screen) "Look at this. Every 1st and 15th of month - 9:00 to 9:30 AM - Our _p99_ latency spikes from **50 ms** to **800 ms**. After 4 mins -  back to normal. Surprisingly, No 5xx errors, No pods crashing. Kubernetes says everything is green. Our APM says everything is green. But our enterprise customers? They're calling account managers and asking why settlement APIs feels so 'sluggish.'"

**_Me :_** "1st and 15th - are these settlement days?"

**_CTO :_** "Bingo - Thousands of our customers hit our settlement APIs. [HPA]() scales us from 10 pods to 40. The new pods come online. The load balancer sends them traffic. And somehow, for those first four minutes, the newest pods in the fleet are the slowest ones."

**_Me :_** "How do you know it's the new pods specifically?"

**_CTO :_** (switches to another dashboard) "Because I stayed up last night and traced request IDs of last month. Look - requests landing on pods older than 5 minutes: 50ms p99. Requests landing on pods younger than 60 seconds: 780ms p99. Same code. Same image. Same node pool."

He sat back. Rubbed his eyes.

**_CTO :_** "Fix provided — We have kept a warm pool of 15 pods running 24/7. We never let the cluster drop below that. And our HPA scale-up stabilization window is so conservative that we're basically pre-paying for capacity we need maybe 4% of the time. Do you know what 15 idle pods cost us per month?
Its 8,200 $ / month - and that too for avoiding a 4 min latency hiccup twice a month."

The call went quiet. Someone from the platform team unmuted, then muted again without speaking.

**_Me :_** "Show me your readiness probe."

**_CTO :_** "It's standard. HTTP GET on /health. Returns 200, pod is Ready, load balancer routes traffic. Why?"

**_Me :_** "Because I think your pods are lying to Kubernetes. They're saying 'I'm open for business' about 280 milliseconds before they've actually finished starting up. And in a 30-minute traffic spike, 280 milliseconds of 'sort of ready' multiplied across 30 new pods is exactly what a 15x latency spike looks like."

**_CTO :_** (pause) "That's... specific. Two hundred and eighty milliseconds?"

**_Me :_** "That's my hypothesis. Let me prove it to you."

# The Measurement
I don't recommend solutions until I've measured the behavior with actual data points. So I instrumented the service with two independent timestamps:

1. **T1 (serverStartedAt):** The moment the embedded web server finishes starting and the port is accepting TCP connections. I captured this using a WebServerInitializedEvent
2. **T2 (initializationCompletedAt):** The moment all business-logic initialization is actually done - captured via custom **_StartupMetrics_**

``` java
@Component
public class StartupMetrics {
    private volatile long serverStartedAt;
    private volatile long initializationCompletedAt;
    private final AtomicBoolean trulyReady = new AtomicBoolean(false);

    @EventListener(WebServerInitializedEvent.class)
    public void onServerStarted(WebServerInitializedEvent event) {
        this.serverStartedAt = System.currentTimeMillis();
        log.info("SERVER_STARTED:  port = {} timestamp = {}", 
                 event.getWebServer().getPort(), this.serverStartedAt);
    }

    public void markInitializationComplete() {
        this.initializationCompletedAt = System.currentTimeMillis();
        this.trulyReady.set(true);
        long gap = this.initializationCompletedAt - this.serverStartedAt;
        log.info("INIT_COMPLETED : gapMs = {} serverStartedAt = {} initCompletedAt = {}", 
                 gap, this.serverStartedAt, this.initializationCompletedAt);
    }
}
```

To ensure I was measuring cold starts, I ran 20 fresh container instances, not 20 restarts inside the same pod. 

I compared medians across those 20 runs & not single samples. The gap was consistent:

``` text
Baseline (port-open vs. actually-ready): median 280.5 ms | stdev 23.2 ms
```

The 23.2 ms standard deviation was operationally not acceptable - as it was too high to tune **_initialDelaySeconds_** . Some pods were "almost ready" around ~260 ms. Others needed ~310 ms.

My first instinct was [AppCDS](). It's well-understood, zero code changes, with just two commands:

``` bash
  java -XX:ArchiveClassesAtExit=set-app-cds.jsa -jar set-app.jar
  java -XX:SharedArchiveFile=set-app-cds.jsa -jar set-app.jar
```

I ran the same readiness-gap test. The result were:

``` text
  AppCDS: median 280.5 ms | stdev 30.7 ms -> 0.0% improvement on this metric
```

AppCDS made overall JVM startup faster, but it did nothing for the specific window between **port open and actually ready**. My hypothesis - AppCDS's benefit (faster parsing of already-loaded JDK classes) is spent entirely during the JVM's early bootstrap phase, before the HTTP port ever opens. The remaining work after port-open is dominated by application logic and framework initialization.

# The Mechanism
[JDK 25]() ships with [Project Leyden's]() AOT cache. While AppCDS pre-parses classes into a shared archive, Leyden's AOT cache pre-loads and links them. The commands look like this:

``` bash
  java -XX:AOTCacheOutput=set-app.aot -jar set-app.jar   # training run
  java -XX:AOTCache=set-app.aot -jar set-app.jar         # production run
```

When I ran the identical readiness-gap test:

``` text
  Leyden AOT: median 254.0 ms | stdev 7.2 ms -> 9.4% improvement, far more consistent
```

The median improvement was not that substantial, but the standard deviation dropping from 23.2 ms to 7.2 ms was more valuable. A pod that starts in more predictable manner is always easy to tune from [HPA]() standpoint

## Training at Build Time
The AOT cache is only as good as the training run that produced it. If your training run exercises a different code path than production, you're caching the wrong thing. I worked with client's platform team to implement a multi-stage Docker build that performed the training run at image build time:

``` docker
  # syntax=docker/dockerfile:1
  FROM eclipse-temurin:25-jdk AS build
  WORKDIR /app
  COPY . .
  RUN ./mvnw clean package -DskipTests

  # Training run — exercises real startup paths once, during the build
  RUN java -XX:AOTCacheOutput=set-app.aot -jar target/set-app.jar --training-mode

  FROM eclipse-temurin:25-jre
  WORKDIR /app
  COPY --from=build /app/target/set-app.jar /app/set-app.aot ./
  ENTRYPOINT ["java", "-XX:AOTCache=set-app.aot", "-jar", "set-app.jar"]
```

The `--training-mode` flag is a convention - your application should interpret it to mean "exercise your real startup code paths, then exit cleanly," so the cache captures genuinely representative behavior equivalent to that in production.

### CI / CD Pipeline
```yaml
# .github/workflows/build.yml
name: Build and Package
on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '25'

      - name: Compile and package
        run: |
          javac -d out $(find src -name "*.java")
          jar cfe set-app.jar com.sti.Settlement -C out .

      - name: Build AOT cache from a representative training run
        run: |
          java -XX:AOTCacheOutput=set-app.aot -jar set-app.jar --training-mode

      - name: Verify the cache actually loads before shipping it
        run: |
          java -Xlog:aot=info -XX:AOTCache=set-app.aot -jar set-app.jar 2>&1 | \
            grep -q "Opened AOT cache" || (echo "AOT cache failed to load — failing build" && exit 1)

      - name: Build and push container image
        run: |
          docker build -t <> .
          docker push <>
```

The verification step is the one most real pipelines skip, and it's the single highest-leverage addition here: it turns "the cache silently didn't load" from a production mystery into a **failed build**.

# The Outcome
Key benefits of deploying realistically trained Leyden-enabled image:

1. **Warm pool reduction -** Because new pods were consistent and predictably ready, Client dropped their over-provisioned warm pool from 20 pods to 5. That alone saved huge amount in compute costs.
2. **Aggressive HPA tuning -** With startup standard deviation dropping from ~23 ms to ~7 ms, they tightened their HPA scale-up stabilization window. The cluster scaled faster and more predictable during the settlement window.
3. **Latency stability -** No p99 spikes during scale-up events. The 280 ms invisible gap was still there (now ~254 ms), but it was more predictable and no longer catching the load balancer off-guard.

# Conclusion
The most important finding in this entire engagement wasn't that **Leyden** was faster. It was that AppCDS - the obvious, easy, zero-code-change answer didn't solve client's problem. Had I not adopted data driven approach of identifying hotspots - client would still be having **P99 latency spikes on Settlement day along with wastage of few thousand of dollars every month on warm pool.**

That's the standard I hold myself to

> **_Measure the thing that hurts, not the thing that's easy to benchmark_**

P.S - If your platform is facing any Performance / Resiliency / Scalability / High Availability issues, [reach out]().
