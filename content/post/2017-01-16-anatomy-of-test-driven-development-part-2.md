---
title: Anatomy of Test Driven Development – Part 2
author: Dhaval Shah
type: post
date: 2017-01-16T18:14:52+00:00
url: /anatomy-of-test-driven-development-part-2/
categories:
  - Uncategorized
tags:
  - TDD
  - XP
thumbnail: "images/wp-content/uploads/2016/12/TDD-lifecycle-217x300.jpg"
---

This is the concluding blog of 2 part series on 'Anatomy of Test Driven Development'. Continuing from where we left in [first part](https://dhaval-shah.com/anatomy-of-test-driven-development-part-1/) of this series, this will mainly talk about 'How' part of TDD along with its avatars.

# How to do TDD

[![tdd-lifecycle](https://www.dhaval-shah.com/images/wp-content/uploads/2016/12/TDD-lifecycle-217x300.jpg)](https://www.dhaval-shah.com/images/wp-content/uploads/2016/12/TDD-lifecycle.jpg)

We start with a Test. To get this to work we need to frame the test. We need to make sure that there is enough code in place to make it compile or to make it to work and not throw up run time error. Thereafter when we run the test it will fail; when it fails what is expected is to quickly write some code and make some change to the application for making the test pass. Primary focus is to get test pass as quickly as possible. We certainly do not intend to damage and hack the existing system. Now that test passes successfully, REFACTORING needs to be done for making the implementation look more elegant and clean. Since we have test cases as a part of TDD, we have enough safety nets to do refactoring and validate its impact w.r.t functionality. This cycle of doing refactoring and running the tests should be performed till 4 rules of Simple Design (outlined by [Kent Beck](https://en.wikipedia.org/wiki/Kent_Beck)) are met in the below mentioned priority order

1.  **All the tests pass** - all test should pass. Not just the test you wrote but all the test that existed before your implementation should pass.
2.  **There is no [code smell](https://sourcemaking.com/refactoring/smells)** - like duplication, speculative generality, primitive obsession etc.
3.  **The code expresses the intent of the programmer** - it should not look as if a cat came and just walked over your keyboard. It has to make some logical sense as it will be referred and used by your fellow team members
4.  **Classes, and methods are minimalist**- you want to be as minimalist as possible without compromising on communication and quality of design

Before we actually delve into how part of TDD, first lets understand avatars of Test Driven Development -

1.  **London School of TDD / Mockist Approach**
2.  **Chicago School of TDD / Classicist Approach**

# London School of TDD / Mockist approach

[![outside-in-tdd-lifecycle](https://www.dhaval-shah.com/images/wp-content/uploads/2017/01/outside-in-tdd-lifecycle.jpg)](https://www.dhaval-shah.com/images/wp-content/uploads/2017/01/outside-in-tdd-lifecycle.jpg)

For Mockist approach, the golden rule tells us that we need to start with a failing test. When we’re implementing a feature, we generally start by writing an acceptance test, which exercises the functionality we want to build. While it’s failing, an acceptance test demonstrates that the system does not yet implement that feature; when it passes, we’re done. When working on a feature, we use acceptance test to guide us as to whether we really need the code we’re about to write; as with this approach we only write code that’s relevant to the underlying functionality. Underneath the acceptance test, we basically follow the unit level test/implement/refactor cycle (as discussed earlier) to develop the feature.

If we notice characteristic of this school of TDD, we will realize that outer test loop i.e. Acceptance Tests is a measure of demonstrable progress, and the growing suite of unit tests underneath it, not only helps in driving/validating design but it also protects us against regression failures when we change the system.

## Example
Lets understand this approach with a semi realistic example - lets say we are implementing an e-commerce application for which price of shopping basket need to be calculated. So adopting Mockist approach for TDD, we start with an acceptance test which will fail initially as implementation is missing. In order to make this test pass, we create a suite of unit test for the given functionality i.e. calculating price of shopping basket So we will test the target object and whilst implementing target object via Mockist approach we would also come to know about its neighbors (also known as **collaborators**) and what should be their role within the purview of test case being executed. So in a way test is helping us to carve out supporting roles of target object and they are defined as Java interfaces. We call this process as ‘**interface discovery**’ Most important aspect of Mockist style of TDD is those collaborators don’t need to exist when we’re writing a unit test; sounds quiet surprising ! Real implementations of collaborators will evolve with its own tests as we develop the rest of the system. Hence, we will need to create mock instances of the discovered collaborators, define expectations on how they’re called and then check them, and implement any stub behavior we need to get through the test. In practice, the runtime structure of a test with mock objects usually looks like.

[![](https://www.dhaval-shah.com/images/wp-content/uploads/2017/01/outside-in-tdd-basket-price-example-2.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2017/01/outside-in-tdd-basket-price-example-2.png)

Underlying principle is – Every class will have its own test and Test will be responsible for testing only 1 target object i.e. CUT and none of the classes/collaborators target object/CUT depends on. Each of those collaborators will have their own test. With above approach test case for the given target object would look like -

{{< highlight java >}}
Promotion aMockPromotion = Mockito.mock(Promotion.class);

// Apply promotion of 1 Rs. discount on purchased video
when(aMockPromotion.applyTo(isA(Video.class))).
     thenReturn(aVideo.getPrice().
     subtract(new BigDecimal("1.00")));

// Do not apply promotion purchased book
when(aMockPromotion.applyTo(isA(Book.class))).
     thenReturn(aVideo.getPrice());
{{< /highlight >}}

The London school of TDD has a different emphasis, focusing on roles, responsibilities and interactions, as opposed to algorithms. An important offshoot emerged from this mockist style of doing TDD and is known as [Behavior Driven Development](http://dannorth.net/introducing-bdd/) (BDD). BDD was originally developed by [Dan North](https://dannorth.net/about/) as a technique to better help people learn Outside-In TDD by focusing on how TDD operates as a design technique.

# Chicago School of TDD / Classicist approach**
In this approach a test may be written for target object i.e. Shopping Cart object and test assumes that all its collaborators are having right value/state and are properly wired with Shopping Cart. Hence this test will test the Shopping Cart with all its **actual dependencies**. [![inside-out-tdd](https://www.dhaval-shah.com/images/wp-content/uploads/2017/01/inside-out-tdd.jpg)](https://www.dhaval-shah.com/images/wp-content/uploads/2017/01/inside-out-tdd.jpg)With this approach, we generally work in a low granularity, meaning every **graph** of classes has its own test fixture. As a result, each test fixture covers a graph of classes implicitly by testing the graph's root. Usually one might not require to test the inner classes of the graph explicitly since they are already tested implicitly by the tests of their root class, and thus one can avoid coverage duplication.

# Conclusion
After understanding 'how' aspects of Test Driven Development along with its Avataars we can say that there is a deep synergy between good design and testability. And so is the case with feedback and testability. Hence all of the three are inextricably linked !