---
title: Agile India 2019 Conference – Learning Experience
author: Dhaval Shah
type: post
date: 2019-04-14T19:42:30+00:00
url: /agile-india-2019-conference-learning-experience/
categories:
  - Testing
thumbnail: "images/wp-content/uploads/2019/04/agile-conf-2019-logo.png"
---
[![](https://www.dhaval-shah.com/images/wp-content/uploads/2019/04/agile-conf-2019-logo.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2019/04/agile-conf-2019-logo.png)

Personally speaking, I am of the belief that only way to excel (professionally) is to collaborate and learn from experts. Since I am constantly striving to improve professionally as [Software Craftsman](https://8thlight.com/blog/micah-martin/2008/09/21/definition-of-software-craftsman.html), I try my best to exercise available options. One such option is attending some good Technology Conferences - as they are the platforms from where the so called experts of industry share their distilled experience to the larger community.

I was extremely fortunate to attend Agile India 2019 conference - Special thanks to my employer [Mastercard](https://www.mastercard.co.in/en-in.html) which strongly believes in fostering learning and development environment for all of its employees. This not only helps its employees to shape their career path but also assist them in meeting their career aspirations. I would like to share key take aways from Agile 2019 conference.

I along with my other colleague had opted for 2 themes-

1.   Agile Mindset
2.   Continuous Delivery and DevOps

Agile Mindset theme started with an amazing key note about [Holocracy](https://www.holacracy.org/constitution) by [Brian Robertson](https://www.holacracy.org/team/brian-robertson/)

# Holocracy

It is a set of management principles and practices for being -

*   Agile, Adaptive and responsive to the ever changing needs of business that is moving at brisk pace
*   Distributed Authority
*   Disciplined and data driven

It primarily talks about new approach to organization structure, meeting formats, autonomy and decision making process. The basic premise of its philosophy lies in its promise - "Processing any tension sensed by anyone and anywhere in the Organisation, into increasing clarity and evolution towards purpose, rapidly, reliably and continuously"

# Effective Retrospectives

This talk highlighted about how to convert the mundane routine [Agile](https://en.wikipedia.org/wiki/Agile) ceremony of Retrospective into a more pleasant, insightful and action oriented by talking about innovative and engaging ways to do the retros viz. Mad - Glad - Sad, Discussing about Sprint with [De Bono's](https://en.wikipedia.org/wiki/Six_Thinking_Hats) hats etc. Another interesting part of the talk was how to gather actionable insights viz. 3Ls (Liked, Learned, Lacked), Snakes and ladders, Starfish (Keep doing, Stop doing, Start Doing, do More off, do Less off). It definitely added a new perspective towards performing Agile ceremonies - i.e from just following [Agile](https://en.wikipedia.org/wiki/Agile) ceremonies for the sake of it to justifiable, purpose oriented following of Agile ceremonies.

# SLICE

It was quite an unconventional topic which mainly talked about steps to be followed for running successful experiments. SLICE is an acronym which stands for Select, Learn, Implement, Chronicle and Expand. It is mainly about having right engineering and experimentation mindset which mainly teaches us to -

*   Learn to fail
*   Don't hate what you don't understand
*   Thought experiment

It indeed was an eye opener to me as it was mainly focused on experimentation and how to do it in cost effective way.

# Mob Programming

To me this was another bit unconventional concept which I have never tried or seen in my professional career so far. To me this concept is an extension to [Pair Programming](https://en.wikipedia.org/wiki/Pair_programming) and it definitely takes it to a new level. The speaker mainly talked about the approaches and styles he has been following to do Mob Programming. He even had statistics to show that it does not jeopardizes the productivity, efficiency and effectiveness of team; on the contrary it fared relatively better - which was bit surprising to me at-least!

This was the last session of Day-1 and the by the end of this session we were literally drained out mentally :) There were couple of more talks  - but they could not create significant impact, hence I am not mentioning over here.

After tinkering our mind with Agile principles, CI / CD and Devops theme started with a key note from none other than  [Jez Humble](https://twitter.com/jezhumble) - author of [Continuous Delivery](https://www.amazon.com/gp/product/0321601912), [Lean Enterprise](https://www.amazon.com/gp/product/1449368425) and [Accelerate](https://www.amazon.com/gp/product/1942788339) and the topic was

# Building and Scaling High Performing Technology Organizations

It mainly talked about software delivery performance metrics responsible for high performing organizations. Key yardsticks for any organization to be high performing are-

*   Deployment frequency
*   Lead time to change
*   Time to restore service
*   Change failure rate.

Jez mainly talked about capabilities that organizations should have viz. Lead Product Development, Lean Management and Continuous Delivery for being high performing organizations. He had special mention about Software Engineering practices - Continuous Testing, Observability and Monitoring,  Disaster Recovery Testing along with Deployment Automation, CI, Trunk Based Dev etc.

I believe most of the topics he covered during his session would definitely be present in his recently launched book [Accelerate](https://www.amazon.com/gp/product/1942788339) - which is one of to be read book in my list as on date :)

# Strategic DDD

This talk was mainly focused on how to design 'sociotechnical' systems to maximize quality and profit of organizations. It mainly highlighted how Domain Driven Design can be correctly done to achieve the same. Speaker walked us through set of practices he has been following for extracting out [Bounded Context](https://martinfowler.com/bliki/BoundedContext.html) and Context Map. He also highlighted effective ways to do [event storming](https://www.eventstorming.com/) and how to use it effectively to extract boundaries. The talk presented pragmatic ways to model software viz. Decouple high and low value parts of system to maximize iteration speed and [ROI](https://en.wikipedia.org/wiki/Return_on_investment). He also highlighted importance of assessing bounded context against 5 pillars - Value, Domain, Social, Technical and UX. Last part of his talk focused on established patterns for realizing dependencies across bounded context i.e. [Shared Kernel](http://ddd.fed.wiki.org/view/shared-kernel), Open Host Service, ACL, Conformist etc

# Acceptance Testing for Continuous Delivery

This was a 90 minute talk (longest of all the sessions I had attended so far). And needless to say it was extremely enriching as it talked about one of my favorite topic i.e. Testing. It mainly talked about whys', whats and hows of Acceptance Testing. Most important ones that were highlighted and discussed at length -

Acceptance testing is primarily about :

*   Asserting user flows
*   Verifying code works in prod like test environment
*   Testing deployment and config
*   Provides quick and timely feedback

Properties of good Acceptance Testing :

*   Should focus on 'what' rather than 'how'
*   Isolated from other tests
*   Repeatable
*   Stubs system external to SUT (System Under Test)

Anti-patterns of Acceptance Testing :

*   Having separate QA team focusing solely on tests
*   Adding wait() / delay() to solve sporadic test failures
*   Including systems outside SUT within scope of AT

_Personally speaking I am a big fan of testing. For all the testing aficionados, I had penned down following posts:_

1.  _[Bootiful TDD](https://dhaval-shah.com/bootiful-test-driven-development/) - How to do outside - in TDD whilst implementing Spring Boot Application_
2.  _A brief article on doing [Consumer Driven Contract Testing](https://dhaval-shah.com/microservices-and-consumer-driven-contract-testing-using-pact/) using [Pact](https://docs.pact.io/)_
3.  _A 2 part series for understanding [Anatomy of TDD](https://dhaval-shah.com/anatomy-of-test-driven-development-part-1/) - an oldie but good enough to rehash the fundamentals ;)_

# Reactive Systems

This talk mainly covered 4 pillars of [reactive manifesto](https://www.reactivemanifesto.org/) and thereby explained key properties of reactive systems. It put lot of emphasis on realizing [eventual consistency](https://en.wikipedia.org/wiki/Eventual_consistency) using asynchronous messaging. However, as we all know it has its own limitations and pitfalls in terms of [observability](https://medium.com/observability/microservices-observability-26a8b7056bb4) and resiliency - but it can definitely be addressed by implementing it in a correct way. Session was full of real world examples that speaker had worked upon. It also talked about isolation, as that is main force behind implementing [Microservices](https://en.wikipedia.org/wiki/Microservices) within [cloud native](https://dhaval-shah.com/understanding-cloud-native-architecture-with-an-example/) world. Concluding part of session was mainly about [Backpressure](https://www.reactivemanifesto.org/glossary#Back-Pressure) by explaining ways to handle it when a component within application is producing stream of data at a rate faster than the component who is going to consume stream of data emitted by producer of data

So this is the synopsis of concepts / topics I was able to learn from most impactful sessions that I had attended during Agile 2019 conference. I hope you would have enjoyed it; to such an extent that next year you will try your level best to make a point of attending 2020 Agile India Conference. 

HAPPY LEARNING!