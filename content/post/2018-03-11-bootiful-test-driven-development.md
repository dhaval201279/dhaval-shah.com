---
title: Bootiful Test Driven Development
author: Dhaval Shah
type: post
date: 2018-03-11T14:28:47+00:00
url: /bootiful-test-driven-development/
categories:
  - Spring Boot
  - Testing
tags:
  - spring
  - spring boot
  - TDD

---

[![](http://dhaval-shah.com/wp-content/uploads/2018/03/bootiful-TDD.jpg)](http://dhaval-shah.com/wp-content/uploads/2018/03/bootiful-TDD.jpg)

Software engineers have been ardently following [Test Driven Development](https://en.wikipedia.org/wiki/Test-driven_development) (TDD) as an [XP](https://en.wikipedia.org/wiki/Extreme_programming) practice for having necessary safety nets. I have even tried covering different [schools of TDD](http://dhaval-shah.com/anatomy-of-test-driven-development-part-1/) with an [example](https://github.com/dhaval201279/tdd-with-spring_mvc/tree/master/Spring_MVC_TDD) in one of my previous posts. Considering recent surge in using [Spring Boot](https://projects.spring.io/spring-boot/) for developing [Microservice](https://en.wikipedia.org/wiki/Microservices) applications, I felt a need to understand and learn how to do TDD whilst implementing [Spring Boot](https://projects.spring.io/spring-boot/) application.

In order to understand how [Spring Boot](https://projects.spring.io/spring-boot/) simplifies the overall process of doing [TDD](https://en.wikipedia.org/wiki/Test-driven_development), we will consider a very simple use case -

1.  Given a reservation system, when user enters details of the user which needs to be searched, it fetches required user details i.e. First name and Last Name
2.  Extend above use case to ensure that caching is being used to optimize lookup operation

# End to End Test - 1st Version

By following [Outside-In / Mockist](http://www.growing-object-oriented-software.com/) school of TDD, we will start with an end to end test - Needless to say that it will fail initially. Of course we will need to ensure that it is compilation error free.

{{< highlight java >}}
package com.its.reservation;

import org.assertj.core.api.Assertions;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.context.SpringBootTest.WebEnvironment;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.test.context.junit4.SpringRunner;

import com.its.reservation.repository.Reservation;

@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
public class ReservationEndToEndTest {
	@Autowired
	TestRestTemplate testRestTemplate;
	
	@Test
	public void getReservation_shouldReturnReservationDetails() {
		// Arrange
		
		// Act
		ResponseEntity<Reservation> response = testRestTemplate.getForEntity("/reservation/{name}", Reservation.class, "Dhaval");
		
		// Assert
		Assertions.assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
		Assertions.assertThat(response.getBody().getName()).isEqualTo("Dhaval");
		Assertions.assertThat(response.getBody().getId()).isNotNull();
	}
}
{{< /highlight >}}


##### [@SpringBootTest](https://docs.spring.io/spring-boot/docs/current/api/org/springframework/boot/test/context/SpringBootTest.html) 
Annotation that can be used for testing Spring Boot application. Along with Spring's *TestContext* framework it does following -
*   Registers *TestRestTemplate* / *WebTestClient* that can be used for running application
*   Looks for *@SpringBootConfiguration* for loading spring configuration from the source. One can not only have custom configuration but also alter the order of configurations
*   Supports different *WebEnvironment* modes

# Presentation Layer

Next we start with [unit tests](https://en.wikipedia.org/wiki/Unit_testing) which will eventually help us in successfully executing the above end to end test. Since we have adopted Outside - In approach, we will first start with unit testing of [REST](https://en.wikipedia.org/wiki/Representational_state_transfer) endpoint. We will be strictly adhering to one of the fundamental rule of TDD

> **"No Production code without any test"**

So in order to make end to end test pass, we will need to have missing pieces. The first missing piece that we will require is the API endpoint i.e. REST controller. We will first start with the controller test

{{< highlight java >}}
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.test.context.junit4.SpringRunner;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders;
import org.springframework.test.web.servlet.result.MockMvcResultMatchers;

import com.its.reservation.web.ReservationController;

@RunWith(SpringRunner.class)
@WebMvcTest(ReservationController.class)
public class ReservationControllerTest {
	@Autowired
	private MockMvc mockMvc;
	
	@Test
	public void getReservation_shouldReturnReservationInfo() {		
		try {
			mockMvc.perform(MockMvcRequestBuilders.get("/reservation/Dhaval"))
				.andExpect(MockMvcResultMatchers.status().isOk())
				.andExpect(MockMvcResultMatchers.jsonPath("firstName").value("Dhaval"))
				.andExpect(MockMvcResultMatchers.jsonPath("lastName").value("Shah"));
		} catch (Exception e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
	}
}
{{< /highlight >}}

In order to run the test, above test has to be made free of any compilation error. So this leads us to creation of actual endpoint.

{{< highlight java >}}
package com.its.reservation.web;

import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RestController;

import com.its.reservation.repository.Reservation;

@RestController
@RequestMapping("/reservation")
public class ReservationController {
	@RequestMapping(method = RequestMethod.GET, value = "/{name}")
	private Reservation getReservation(@PathVariable String name) {
		return null;
	}
}
{{< /highlight >}}

Now that we have an endpoint, we will be able to execute test case and we are sure that it is going to fail :) as it is not returning null currently.

Since the *Controller* is going to further delegate its task of business processing to the underlying Service. We are able to see that *Outside-In* / *Mockist* approach of doing TDD is helping us to determine collaborators required by class under test. Hence we will be creating *ReservationService* with required API which will be devoid of any behavior for time being - As its actual behavior will be discovered when we start writing unit test case for *ReservationService*. So we create the collaborator to get our class under test i.e. *ReservationController* compilation error free

{{< highlight java >}}
package com.its.reservation.service;

import org.springframework.stereotype.Service;

import com.its.reservation.repository.Reservation;

@Service
public class ReservationService {
	public Reservation getReservationDetails(String name) {
		return null;
	}
}
{{< /highlight >}}

We also update *ReservationController* by wiring required dependency

{{< highlight java >}}
@RestController
@RequestMapping("/reservation")
public class ReservationController {
	
	private ReservationService reservationService;
	
	public ReservationController(ReservationService reservationService) {
		this.reservationService = reservationService;
	}

	@RequestMapping(method = RequestMethod.GET, value = "/{name}")
	private Reservation getReservation(@PathVariable String name) {
		return reservationService.getReservationDetails(name);
	}

}
{{< /highlight >}}

However, as per the rule of Mockist approach - dependencies (within application) for a Class Under Test should be mocked whilst performing Unit Testing. So we need to update *ReservationControllerTest* :

{{< highlight java >}}
@RunWith(SpringRunner.class)
@WebMvcTest(ReservationController.class)
public class ReservationControllerTest {
	@Autowired
	private MockMvc mockMvc;
	
	@MockBean
	ReservationService reservationService;
	
	@Test
	public void getReservation_shouldReturnReservationInfo() {
		BDDMockito.given(reservationService.getReservationDetails(ArgumentMatchers.anyString()))
					.willReturn(new Reservation(Long.valueOf(1), "Dhaval", "Shah"));
		try {
			mockMvc.perform(MockMvcRequestBuilders.get("/reservation/Dhaval"))
				.andExpect(MockMvcResultMatchers.status().isOk())
				.andExpect(MockMvcResultMatchers.jsonPath("firstName").value("Dhaval"))
				.andExpect(MockMvcResultMatchers.jsonPath("lastName").value("Shah"));
		} catch (Exception e) {
			e.printStackTrace();
		}
	}
}
{{< /highlight >}}

When we run above test . . . . . It passes - which means that Controller implementation is as per its expected behavior. So now we have some kind of safety net for our _ReservationController_ . . Vola !

However, one might still feel that test driving happy path flows is relatively easier than business exceptions being thrown during workflow execution. So we will write a test case for a scenario which might end up in a scenario where in user is not available in our system i.e. *ReservationNotFoundException*. Also our unit test case will have to mock this exception

{{< highlight java >}}
@RunWith(SpringRunner.class)
@WebMvcTest(ReservationController.class)
public class ReservationControllerTest {
	@Autowired
	private MockMvc mockMvc;
	
	@MockBean
	ReservationService reservationService;
	
	@Test
	public void getReservation_shouldReturnReservationInfo() {
		BDDMockito.given(reservationService.getReservationDetails(ArgumentMatchers.anyString()))
					.willReturn(new Reservation(Long.valueOf(1), "Dhaval", "Shah"));
		try {
			mockMvc.perform(MockMvcRequestBuilders.get("/reservation/Dhaval"))
				.andExpect(MockMvcResultMatchers.status().isOk())
				.andExpect(MockMvcResultMatchers.jsonPath("firstName").value("Dhaval"))
				.andExpect(MockMvcResultMatchers.jsonPath("lastName").value("Shah"));
		} catch (Exception e) {
			e.printStackTrace();
		}
	}
	
	@Test
	public void getReservation_NotFound() throws Exception {
		BDDMockito.given(reservationService.getReservationDetails(ArgumentMatchers.anyString()))
					.willThrow(new ReservationNotFoundException());
		
		mockMvc.perform(MockMvcRequestBuilders.get("/reservation/Dhaval"))
			.andExpect(MockMvcResultMatchers.status().isNotFound());
	}
}
{{< /highlight >}}

In order to pass the newly added above test case, *ReservationController* also needs to be updated by having required behavior for handling exceptions

{{< highlight java >}}
@RestController
@RequestMapping("/reservation")
public class ReservationController {
	private ReservationService reservationService;
	
	public ReservationController(ReservationService reservationService) {
		this.reservationService = reservationService;
	}

	@RequestMapping(method = RequestMethod.GET, value = "/{name}")
	private Reservation getReservation(@PathVariable String name) {
		System.out.println("Entering and leaving ReservationController : getReservation after fetching service");
		return reservationService.getReservationDetails(name);
	}
	
	@ExceptionHandler()
	@ResponseStatus(HttpStatus.NOT_FOUND)
	public void userNotFoundHandler(ReservationNotFoundException rnfe) {
		System.out.println("Entering and leaving ReservationController : userNotFoundHandler");
	}
}
{{< /highlight >}}

Within the purview of our feature, this completes the unit testing of REST endpoint i.e Controller. Now we move to business layer i.e. *ReservationService* which has been discovered as a collaborator of *ReservationController*.

# Business / Service Layer

Since *ReservationService*  is a plain Spring bean, we just need to write a plain unit test which is devoid of any Spring dependency - and of course it will fail as our class under test i.e.*ReservationService* is returning null.

{{< highlight java >}}
package com.its.reservation;

import static org.assertj.core.api.Assertions.assertThat;

import org.junit.Test;
import org.springframework.beans.factory.annotation.Autowired;

import com.its.reservation.repository.Reservation;
import com.its.reservation.service.ReservationService;

@RunWith(MockitoJUnitRunner.class)
public class ReservationServiceTest {
	ReservationService reservationservice;
	
	@Before
	public void setUp() throws Exception {
		reservationService = new ReservationService();
	}
	
	@Test
	public void getReservationDetails_returnsReservationInfo() {
		Reservation aReservation = reservationservice.getReservationDetails("Dhaval");
		
		assertThat(aReservation.getFirstName()).isEqualTo("Dhaval");
		assertThat(aReservation.getFirstName()).isEqualTo("Shah");
	}
}
{{< /highlight >}}

In real world application, business layer i.e. Service class will just be an orchestrator of business workflow, which needs to be executed for returning back required response to Controller. For the sake of simplicity our service layer will just be responsible for fetching required information from database. Hence it will need *ReservationRepository* as a collaborator, which will be able to fetch required data from database. So we will be creating corresponding Repository which just ensures that my class under test i.e. *ReservationService* is free from any compilation errors

{{< highlight java >}}
package com.its.reservation.repository;

import org.springframework.data.repository.CrudRepository;

public interface ReservationRepository extends CrudRepository<Reservation, Long> {
	Reservation findByFirstName(String name);
}
{{< /highlight >}}

So with the introduction of new above collaborator, *ReservationService* would look like

{{< highlight java >}}
package com.its.reservation.service;

import org.springframework.stereotype.Service;

import com.its.reservation.repository.Reservation;
import com.its.reservation.repository.ReservationRepository;

@Service
public class ReservationService {
	private ReservationRepository reservationRepository;
	
	public ReservationService(ReservationRepository reservationRepository) {
		this.reservationRepository = reservationRepository;
	}

	public Reservation getReservationDetails(String name) {
		return reservationRepository.findByFirstName(name);
	}
}
{{< /highlight >}}

Now coming back to *ReservationServiceTest* which is now free of compilation errors; we will mock collaborator of *ReservationService* i.e. *ReservationRepository*

{{< highlight java >}}
@RunWith(MockitoJUnitRunner.class)
public class ReservationServiceTest {
	ReservationService reservationService;
	
	@Mock
	ReservationRepository reservationRepository;
	
	@Before
	public void setUp() throws Exception {
		reservationService = new ReservationService(reservationRepository);
	}
	
	@Test
	public void getReservationDetails_returnsReservationInfo() {
		BDDMockito.given(reservationRepository.findByFirstName("Dhaval"))
					.willReturn(new Reservation(Long.valueOf(1), "Dhaval", "Shah"));
		
		Reservation aReservation = reservationService.getReservationDetails("Dhaval");
		
		assertThat(aReservation.getFirstName()).isEqualTo("Dhaval");
		assertThat(aReservation.getLastName()).isEqualTo("Shah");
	}
}
{{< /highlight >}}

Lets also verify exceptional flows within this Service class. So we implement test case for the scenario where in *ReservationRepository* is returning null

{{< highlight java >}}
@RunWith(MockitoJUnitRunner.class)
public class ReservationServiceTest {
	ReservationService reservationService;
	
	@Mock
	ReservationRepository reservationRepository;
	
	@Before
	public void setUp() throws Exception {
		reservationService = new ReservationService(reservationRepository);
	}
	
	@Test
	public void getReservationDetails_returnsReservationInfo() {
		BDDMockito.given(reservationRepository.findByFirstName("Dhaval"))
					.willReturn(new Reservation(Long.valueOf(1), "Dhaval", "Shah"));
		
		Reservation aReservation = reservationService.getReservationDetails("Dhaval");
		
		assertThat(aReservation.getFirstName()).isEqualTo("Dhaval");
		assertThat(aReservation.getLastName()).isEqualTo("Shah");
	}
	
	@Test(expected = ReservationNotFoundException.class)
	public void getReservationDetails_whenNotFound() {
		BDDMockito.given(reservationRepository.findByFirstName("Dhaval")).willReturn(null);
		Reservation aReservation = reservationService.getReservationDetails("Dhaval");
	}
}
{{< /highlight >}}

Of course this test case will fail as our current implementation of *ReservationService is not having required logic for handling no results returned by *ReservationRepository*. Hence we update our *ReservationService*

{{< highlight java >}}
@Service
public class ReservationService {
	private ReservationRepository reservationRepository;
	
	public ReservationService(ReservationRepository reservationRepository) {
		this.reservationRepository = reservationRepository;
	}

	public Reservation getReservationDetails(String name) {
		System.out.println("Entering and leaving ReservationService : getReservationDetails "
				+ "after calling reservationRepository.findByFirstName");
		Reservation aReservation = reservationRepository.findByFirstName(name);
		if (aReservation == null) {
			throw new ReservationNotFoundException();
		}
		return aReservation;
	}
}
{{< /highlight >}}

Within the purview of our feature, this completes the unit testing of *ReservationService*.

# Repository

Now we move to the respository layer i.e. *ReservationRepository* which has been discovered as a collaborator of *ReservationService*.

Even though from implementation standpoint it might seem trivial to test - but its other way round. Reason being, lot of complexity is camouflaged behind [CrudRepository](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/repository/CrudRepository.html) which needs to be considered whilst implementing test case for a [Repository](https://msdn.microsoft.com/en-us/library/ff649690.aspx). In addition we will also need a mechanism to generate test data without database - Many thanks to [H2 Database](https://en.wikipedia.org/wiki/H2_(DBMS)) which can be used for our development purpose and also to [Spring Boot starter](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#using-boot-build-systems) which helps in getting required dependencies of H2 Database. Just to reiterate, we will start with a failing test and then add required implementation to pass this test.

{{< highlight java >}}
package com.its.reservation;

import org.assertj.core.api.Assertions;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.boot.test.autoconfigure.orm.jpa.TestEntityManager;
import org.springframework.test.context.junit4.SpringRunner;

import com.its.reservation.repository.Reservation;
import com.its.reservation.repository.ReservationRepository;

/**
 * @author Dhaval
 *
 */
@RunWith(SpringRunner.class)
@DataJpaTest
public class ReservationRepositoryTest {
	@Autowired
	TestEntityManager entityManager;
	
	@Autowired
	ReservationRepository reservationRepository;
	
	@Test
	public void getReservation_returnReservationDetails() {
		Reservation savedReservation = entityManager.persistAndFlush(new Reservation("Dhaval","Shah"));
		
		Reservation aReservation = reservationRepository.findByFirstName("Dhaval");
		
		Assertions.assertThat(aReservation.getFirstName()).isEqualTo(savedReservation.getFirstName());
		Assertions.assertThat(aReservation.getLastName()).isEqualTo(savedReservation.getLastName());
	}
}
{{< /highlight >}}

[@DataJpaTest](https://docs.spring.io/spring-boot/docs/current/api/org/springframework/boot/test/autoconfigure/orm/jpa/DataJpaTest.html) - Spring annotation that can be used for a JPA test. This will ensure that *AutoConfiguration* are disabled and JPA specific configurations are applied. By default it will use H2 database which can be changed by using *@AutoConfigureTestDatabase* and *application.properties*. One can change database according to environment by using Spring profile

[TestEntityManager](https://docs.spring.io/spring-boot/docs/current/api/org/springframework/boot/test/autoconfigure/orm/jpa/TestEntityManager.html) - Along with few methods of *EntityManager* it provides helper methods for testing JPA implementation. We can customize *TestEntityManager* by using *@AutoConfigureTestEntityManager*

# End to End Test - Final Version

We are still remaining to get our *ReservationEndToEndTest* pass :) So we will update this test such that H2 database is populated with required data which can be used for asserting post invocation of API endpoint. And by this we are able to complete our feature implementation with Outside - In / Mockist style of TDD

{{< highlight java >}}
@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
public class ReservationEndToEndTest {
	@Autowired
	TestRestTemplate testRestTemplate;
	
	@Test
	public void getReservation_shouldReturnReservationDetails() {
		// Arrange
		
		// Act
		ResponseEntity<Reservation> response = testRestTemplate.getForEntity("/reservation/{name}", 
		
		// Assert
		Assertions.assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
		Assertions.assertThat(response.getBody().getFirstName()).isEqualTo("Dhaval");
		Assertions.assertThat(response.getBody().getLastName()).isEqualTo("Shah");
		Assertions.assertThat(response.getBody().getId()).isNotNull();
	}
}

@Component
class SampleDataCLR implements CommandLineRunner {
	private ReservationRepository reservationRepository;
	
	public SampleDataCLR(ReservationRepository reservationRepository) {
		this.reservationRepository = reservationRepository;
	}

	@Override
	public void run(String... args) throws Exception {
		System.out.println("@@@@@@@@@@@@@@ Entering SampleDataCLR : run");
		reservationRepository.save(new Reservation("Dhaval","Shah"));
		reservationRepository.save(new Reservation("Yatharth","Shah"));
		reservationRepository.findAll().forEach(System.out :: println);
		System.out.println("@@@@@@@@@@@@@@ Leaving SampleDataCLR : run");
	}
}
{{< /highlight >}}

# Testing Non Functional Requirement - Caching

In order to test caching, first we will need to enable caching within our Spring Boot application via *@EnableCaching* i.e.

{{< highlight java >}}
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cache.annotation.EnableCaching;

@SpringBootApplication
@EnableCaching
public class BootifulTddApplication {

	public static void main(String[] args) {
		SpringApplication.run(BootifulTddApplication.class, args);
	}
}
{{< /highlight >}}

[@EnableCaching](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/cache/annotation/EnableCaching.html) - Responsible for registering necessary Spring components based on [_@Cacheable_](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/cache/annotation/Cacheable.html) annotation

Next thing that needs to be done is - we annotate our business API i.e *getReservationDetails()* within *ReservationService* with *@Cacheable*

{{< highlight java >}}
@Service
public class ReservationService {
	
	private ReservationRepository reservationRepository;
	
	public ReservationService(ReservationRepository reservationRepository) {
		this.reservationRepository = reservationRepository;
	}

	@Cacheable("reservation")
	public Reservation getReservationDetails(String name) {
		System.out.println("Entering and leaving ReservationService : getReservationDetails "
				+ "after calling reservationRepository.findByFirstName");
		Reservation aReservation = reservationRepository.findByFirstName(name);
		if (aReservation == null) {
			throw new ReservationNotFoundException();
		}
		return aReservation;
	}

}
{{< /highlight >}}

With this NFR the key question is, how can we test Caching implementation without any actual cache? We can still test this implementation with *ReservationCachingTest* as shown below

{{< highlight java >}}
package com.its.reservation;

import org.junit.Test;
import org.junit.runner.RunWith;
import org.mockito.ArgumentMatchers;
import org.mockito.BDDMockito;
import org.mockito.Mockito;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.jdbc.AutoConfigureTestDatabase;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.test.context.junit4.SpringRunner;

import com.its.reservation.repository.Reservation;
import com.its.reservation.repository.ReservationRepository;
import com.its.reservation.service.ReservationService;

@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment=SpringBootTest.WebEnvironment.NONE)
@AutoConfigureTestDatabase
public class ReservationCachingTest {
	@Autowired
	ReservationService reservationService;
	
	@MockBean
	ReservationRepository reservationRepository;
	
	@Test
	public void caching_reducesDBCall() {
		BDDMockito.given(reservationRepository.findByFirstName(ArgumentMatchers.anyString()))
			  .willReturn(new Reservation(Long.valueOf(1),"Dhaval","Shah"));
		
		reservationService.getReservationDetails("Dhaval");
		reservationService.getReservationDetails("Dhaval");
		
		Mockito.verify(reservationRepository, Mockito.times(1)).findByFirstName("Dhaval");
	}
}
{{< /highlight >}}

Finally we have covered all aspects of the feature that we were suppose to implement. I do know it has got bit too long, but the nature of topic and relevance it has with chronological order of evolution of implementation does not allow me to convert it into two parts !

# Conclusion

As we saw throughout the implementation, Spring Boot has lot of features to easily test Spring Boot application. We can use them as per our need to ensure that we are not only able to have required safety nets whilst implementing them but can also verify / validate design decisions.

Testing is an indispensable practice of software development. I hope this article will be of some help to get you started with Test Driven Development of Spring Boot applications.

Source Code : [Bootiful TDD](https://github.com/dhaval201279/bootiful-TDD)