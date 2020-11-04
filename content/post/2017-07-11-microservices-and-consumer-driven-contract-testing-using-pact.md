---
title: Microservices and Consumer Driven Contract testing using Pact
author: Dhaval Shah
type: post
date: 2017-07-10T20:37:52+00:00
url: /microservices-and-consumer-driven-contract-testing-using-pact/
categories:
  - Spring Boot
  - Testing
tags:
  - microservice
  - spring boot
  - TDD

---
# Background
As per the current trends, Microservice Architecture has become a common paradigm using which enterprise applications are built. With this paradigm shift, an application is going to have myriad set of independent and autonomous (micro)services. So how does a developer do testing within Microservice Architecture? Answer is very obvious -
1.  Create integration tests that invokes microservice under test which will internally call dependent microservices
2.  Get all the services under test up and running
3.  Start executing integration test which invokes microservice under test

With this approach, the entire developer level integration testing will have following disadvantages -

*   Extremely slow execution of integration tests
*   Fragile tests as they are directly dependent on successful execution of other micorservices
*   Obscures ability to debug outcome of tests as it will be ambiguous
*   Combinatorial increase in number of tests (depending on number of classes and alternate paths within them)

And hence the overall intent behind having integration tests gets defeated.

So Martin Fowler gave an interesting perspective called [Consumer Driven Contract](https://martinfowler.com/articles/consumerDrivenContracts.html) which is nothing but a contract between consumer of service and producer of service. In this style of testing, format of contract is defined by Consumer and then shared with the corresponding Producer of service.

[![](http://dhaval-shah.com/wp-content/uploads/2017/07/Consumer-Driven-Contract-Testing-1.jpg)](http://dhaval-shah.com/wp-content/uploads/2017/07/Consumer-Driven-Contract-Testing-1.jpg)

Lets take an example to understand this. We have *ReservationClient* which is an end consumer facing API; this in turn invokes *ReservationService* which is responsible for managing business workflows pertaining to Reservation. For the sake of simplicity, we will just try to retrieve Reservation using both the services.

# Pact
We will be using Pact for realizing Consumer Driven Contract (CDC) testing. [PACT](https://docs.pact.io/) is an open source CDC testing framework which also supports multiple languages like _Ruby, Java, Scala, .NET, Javascript, Swift/Objective-C_. It comprises of 2 main steps for performing CDC testing. One may want to look at the jargons to understand Pact specific [terminology](https://docs.pact.io/documentation/how_does_pact_work.html)
1.  Pact generation on consumer side service
2.  Pact verification on provider side service

[![](http://dhaval-shah.com/wp-content/uploads/2017/07/Consumer-Driven-Contract-Testing-pact.jpg)](http://dhaval-shah.com/wp-content/uploads/2017/07/Consumer-Driven-Contract-Testing-pact.jpg)

## 1. Pact Generation on Consumer side
### 1.1 Define expected result
We first start by writing JUnit test case *TestReservationClient* for *ReservationClient*, where we specify the expected result from the Provider i.e. *ReservationService*. Test case can be implemented in 2 ways -
1.  Extend the base class *ConsumerPactTest*
2.  Use annotations

We will be implementing test case using second approach.

{{< highlight java >}}
@Rule
public PactProviderRule pactProviderRule = new PactProviderRule("reservation-provider-demo", this);

	@Pact(consumer = "reservation-consumer-demo")
	public PactFragment createFragment(PactDslWithProvider pactDslWithProvider) {
		Map<String, String> headers = new HashMap<>();
		headers.put("Content-Type", "application/json;charset=UTF-8");
		return pactDslWithProvider.given("test demo first state")
				.uponReceiving("ReservationDemoTest interaction")
					.path("/producer/reservation/names")
					.method("GET")
				.willRespondWith()
					.status(200)
					.headers(headers)
					.body("{" + 
								"\\"name\\" : \\"Dhaval\\"" + 
					"}")
					.toFragment();
	}

	@Test
	@PactVerification
	public void runTest() throws Exception {
		String url = pactProviderRule.getConfig().url();
		Reservation fetchedReservation = new ReservationAPIGateway(url+"/producer/reservation/names").fetchOne();
		assertEquals("Dhaval", fetchedReservation.getName());
	}
{{< /highlight >}}

### 1.2 Generate the Pact file

Lets run the test case after implementing test case with required Pact contracts. If the test runs successfully, a JSON file will be created within a new folder *pacts* underneath */target* folder

{{< highlight java >}}
{
    "provider": {
        "name": "reservation-provider-demo"
    },
    "consumer": {
        "name": "reservation-consumer-demo"
    },
    "interactions": \[
        {
            "description": "ReservationDemoTest interaction",
            "request": {
                "method": "GET",
                "path": "/producer/reservation/names"
            },
            "response": {
                "status": 200,
                "headers": {
                    "Content-Type": "application/json;charset=UTF-8"
                },
                "body": {
                    "name": "Dhaval"
                }
            },
            "providerState": "test demo first state"
        }
    \],
    "metadata": {
        "pact-specification": {
            "version": "2.0.0"
        },
        "pact-jvm": {
            "version": "3.3.3"
        }
    }
}
{{< /highlight >}}

### 1.3 Share the generated pact file with Producer

Last step of consumer should be to share the generated contract with provider. This can be done either by a file sharing service or [Pact Broker](https://docs.pact.io/documentation/sharings_pacts.html)

## 2. Pact Verification on Provider side
### 2.1 Bootstrap the Provider service
After getting the pact file from consumer, provider service i.e. *ReservationServiceController* should be bootstrapped first

### 2.2 Execute JUnit-Pact test against the
Implement a JUnit test which will refer to the pact file and also match its state with that of the state configured at Consumer's end

{{< highlight java >}}
@RunWith(PactRunner.class)
@Provider("reservation-provider-demo")
@PactFolder("../reservation-client/target/pacts")
@VerificationReports({"console", "markdown"})
public class ReservationServiceControllerContractTest {
	@TestTarget
    public final Target target = new HttpTarget(8080);

    @BeforeClass
    public static void setUpProvider() {

    }

    @State("test demo first state")
    public void demoState() {
        System.out.println("Reservation Service is in demo state");
    }
}
{{< /highlight >}}

### 2.3 Check verification result
After executing verification test by Provider JUnit i.e. *ReservationServiceControllerContractTest* we get the output as shown below

{{< highlight java >}}
Reservation Service is in demo state

Verifying a pact between reservation-consumer-demo and reservation-provider-demo
  Given test demo first state
  ReservationDemoTest interaction
01:29:28.559 \[main\] DEBUG au.com.dius.pact.provider.ProviderClient - Making request for provider au.com.dius.pact.provider.ProviderInfo(http, localhost, 8080, /, reservation-provider-demo, null, null, null, null, null, false, null, changeit, null, true, false, null, \[\], \[\]):
01:29:28.617 \[main\] DEBUG au.com.dius.pact.provider.ProviderClient - 	method: GET
	path: /producer/reservation/names
	query: null
	headers: null
	matchers: \[:\]
	body: au.com.dius.pact.model.OptionalBody(MISSING, null)

...
..
.
Reservation Service is in demo state

Verifying a pact between reservation-consumer-demo and reservation-provider-demo
  Given test demo first state
  ReservationDemoTest interaction
01:29:28.559 \[main\] DEBUG au.com.dius.pact.provider.ProviderClient - Making request for provider au.com.dius.pact.provider.ProviderInfo(http, localhost, 8080, /, reservation-provider-demo, null, null, null, null, null, false, null, changeit, null, true, false, null, \[\], \[\]):
01:29:28.617 \[main\] DEBUG au.com.dius.pact.provider.ProviderClient - 	method: GET
	path: /producer/reservation/names
	query: null
	headers: null
	matchers: \[:\]
	body: au.com.dius.pact.model.OptionalBody(MISSING, null)
{{< /highlight >}}

Lets understand the nature of output in case contract fails. Lets create a separate JUnit for ReservationClient which returns user's name along with its id. We then follow the same steps as mentioned above and then verify the generated pact with Provider's JUnit test case. Resulting output is as shown below

{{< highlight java >}}
Reservation Service is in demo state

Verifying a pact between reservation-consumer-demo and reservation-provider-demo
  Given test demo first state
  ReservationDemoTest interaction
01:45:42.938 \[main\] DEBUG au.com.dius.pact.provider.ProviderClient - Making request for provider au.com.dius.pact.provider.ProviderInfo(http, localhost, 8080, /, reservation-provider-demo, null, null, null, null, null, false, null, changeit, null, true, false, null, \[\], \[\]):
01:45:43.078 \[main\] DEBUG au.com.dius.pact.provider.ProviderClient - 	method: GET
	path: /producer/reservation/names
	query: null
	headers: null
	matchers: \[:\]
	body: au.com.dius.pact.model.OptionalBody(MISSING, null)
01:45:44.729 \[main\] DEBUG org.apache.http.client.protocol.RequestAddCookies - CookieSpec selected: default

...
..
.

01:45:46.963 \[Finalizer\] DEBUG org.apache.http.impl.conn.PoolingHttpClientConnectionManager - Connection manager shut down
01:45:47.374 \[main\] DEBUG au.com.dius.pact.model.Matching$ - Found a matcher for application/json -> Some((application/.\*json,au.com.dius.pact.matchers.JsonBodyMatcher@71e5f61d))
01:45:47.423 \[main\] DEBUG au.com.dius.pact.matchers.JsonBodyMatcher - compareValues: No matcher defined for path List($, body, name), using equality
    returns a response which
      has status code 200 (OK)
      includes headers
        "Content-Type" with value "application/json;charset=UTF-8" (OK)
      has a matching body (FAILED)

Failures:

0) ReservationDemoTest interaction returns a response which has a matching body
      $.body -> Expected id='100' but was missing

      Diff:

      @1
      -    "name": "Dhaval",
      -    "id": "100"
      +    "name": "Dhaval"
      }

{{< /highlight >}}

It is quiet evident from the above error that Consumer has broken the contract as it is sending id attribute of user.

# Summary
As we saw above, with CDC we can make integration testing easy to manage and strive to get faster feedback. This in a way helps us to realize [Test Pyramid](http://dhaval-shah.com/anatomy-of-test-driven-development-part-1/). In subsequent posts, I will try to introduce usage of [Spring Cloud Contract](http://cloud.spring.io/spring-cloud-contract/spring-cloud-contract.html) for realizing CDC testing.

[Github code](https://github.com/dhaval201279/spring-boot-pact-cdc) for your perusal.