---
title: Anatomy of Test Driven Development – Part 1
author: Dhaval Shah
type: post
date: 2017-01-05T19:48:47+00:00
url: /anatomy-of-test-driven-development-part-1/
categories:
  - Best of the Week
tags:
  - TDD
  - XP

---

Since we being one of the most intellectual and so called logical species on this earth, we need proper rationale behind each and every action that we do in our personal and professional life. Hence I thought to pen down a two post series highlighting rationale behind following one off the most important and underrated [XP](http://www.extremeprogramming.org/) practice of software devlopment i.e. [Test Driven Development](http://agiledata.org/essays/tdd.html)

# What is TDD
Lets try to understand definition of TDD and the primary motive behind TDD; this was outlined by none other than one of the protagonist of TDD i.e. [Nat Pryce](http://www.natpryce.com/) and [Steve Freeman](http://higherorderlogic.com/) given during [XP Day 2006](http://www.xpday.org/) conference -

>_"Everybody knows that TDD stands for Test Driven Development. However, people too often concentrate on the words "Test" and "Development" and don't consider what the word "Driven" really implies. For tests to drive development they must do more than just test that code performs its required functionality: they must clearly express that required functionality to the reader. That is, they must be clear specifications of the required functionality. Tests that are not written with their role as specifications in mind can be very confusing to read. The difficulty in understanding what they are testing can greatly reduce the velocity at which a codebase can be changed"_

The most common notion prevalent in our industry when we hear 'We are doing TDD' is – we are asked to do TDD for ensuring that functionality works. But the irony is, the same functionality is going to be Quality Assured by a dedicated team !

The term TDD is often **misinterpreted** and used as a **synonym** for "unit testing". And hence in a conventional developer-based testing approach, tests are employed to discover defects in code, whereas the primary focus of TDD is that **tests are used for specification and driving/validating design**.

So a question to ponder - is there any relationship between TDD and unit testing. The relationship between the two is that unit testing forms part of TDD, but there is more to TDD than just unit testing.

Unit testing in general can be employed and practiced in many ways. One may still be doing unit testing religiously without following TDD :)

Before we look into HOW part of TDD, lets first understand WHY Test Driven Development is an indispensable practice that needs to be religiously followed

# Why Test Driven Development
Before we understand the advantages of following TDD, lets try to understand actual purpose of Unit Testing. [Brian Marick](https://en.wikipedia.org/wiki/Brian_Marick) who is leading Agile Testing guru and one of the authors of [Agile Manifesto](http://agilemanifesto.org/) came up with categorization of tests – which test fits into which quadrant and primary criteria behind this classification is to help people understand purpose of each type of tests that are implemented within a software application

[![brian-maricks-test-quadrant-tdd](http://dhaval-shah.com/wp-content/uploads/2016/12/Brian-Maricks-Test-Quadrant-TDD-1024x576.jpg)](http://dhaval-shah.com/wp-content/uploads/2016/12/Brian-Maricks-Test-Quadrant-TDD.jpg)

4 axes against which categorization is done

1.  Top : Business facing
2.  Bottom : Tech/Implementation facing
3.  Left : Supports Programming – Helps developers and programmers
4.  Right : Critique Product – These are the tests that after a given functionality has been implemented, validates whether it works as per it's expected behavior

If you take any of 4 quadrants, you can put a given test in 1 of the 4 quadrants and thereby understand what the significance of those tests is. One thing which is quiet apparent from above test quadrant - Unit Tests helps in development

On a side note if we try to super impose [Test Pyramid](https://www.mountaingoatsoftware.com/blog/the-forgotten-layer-of-the-test-automation-pyramid), most of the test cases i.e. Unit and Services will fall into 1st quadrant.

# Advantages of TDD

## TDD aids in deriving loosely coupled and highly cohesive design

[![loosely-coupled-design](http://dhaval-shah.com/wp-content/uploads/2017/01/loosely-coupled-design.jpg)](http://dhaval-shah.com/wp-content/uploads/2017/01/loosely-coupled-design.jpg)

Lets take an example to understand this -

We're writing code for an application which needs to depend on a database or external API. As an initial thought there's nothing really stopping us from adding those low level dependencies viz. Call to database or an external API in the code we are writing. Separating out these responsibilities into separate components may not seem entirely beneficial at first. However when we are allowing the tests to inform about the design of implementation, the issues with this approach will manifest itself into -

*   Hard to write Unit Test cases
*   Slow Unit Tests

Reason is obvious - if we're test driving some application logic, and during each test we are going to call database or to an external API, then implementing those test cases would be a challenge as writing test setup and test fixtures would be too difficult because of multiple dependencies. This in a way is kind of a [test smell](http://xunitpatterns.com/Test%20Smells.html) which is giving an indication about the design which is violating few of the OO principles

Three main aspects of TDD that help us achieve clean and elegant design :

1.  When we start with a test, we have to describe ‘what we want to achieve’ before identifying ‘how’. This in a way helps us in :
    *   identifying right level of abstraction for the target object i.e. Class Under Test (CUT)
    *   right level of information hiding as we need to decide what will be visible from outside of the object
2.  We would have seen tests which are few 100 lines/ 1000 lines; such test tell us that the target object is too large and needs to broken up further for achieving better separation of concerns.
3.  Since target object needs to be instantiated before test starts executing actual API, whilst construction of target object we may identify that if there are too many dependencies it is too painful to construct and test

## Helps create live specification

  [![live-specification](http://dhaval-shah.com/wp-content/uploads/2017/01/live-specification.jpg)](http://dhaval-shah.com/wp-content/uploads/2017/01/live-specification.jpg)

How many times we have ended up in situations where in we are newly allocated to a project and size of the application is such that team is still clueless in terms of design of key features/functionalities within application.

TDD will implicitly help us to create appropriate and just enough specification using which one can clearly understand business features and functionalities. Most importantly this specification is a live specification, hence it gets updated as and when changes are incorporated; since test cases would also have to be updated along with code :)

## Promotes Refactoring

[![refactoring](http://dhaval-shah.com/wp-content/uploads/2017/01/refactoring.jpg)](http://dhaval-shah.com/wp-content/uploads/2017/01/refactoring.jpg)

As we are required to do reflexive design throughout the life cycle of product, refactoring is a quintessential aspect of product development - needless to say it is also equally important from XP perspective. Hence having right granular level of test cases immensely helps team members to do incremental refactoring. And by the virtue of this we can constantly strive for clean, elegant and maintainable software.

Stay tuned for part 2 of this post !