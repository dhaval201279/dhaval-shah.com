---
title: "Part 1 - Multiple Runs - Why Deterministic Thinking Breaks!"
author: Dhaval Shah
type: post
date: 2026-09-26T01:00:50+00:00
url: /part-1-ai-native-multiple-runs/
categories:
  - java
  - spring-ai
  - ai-agents
  - architecture
tags:
  - java
  - spring-ai
  - ai-agents
  - architecture
thumbnail: "images/wp-content/uploads/2026/09/part-0-ai-native-sys-java.png"
---

[![](https://www.dhaval-shah.com/images/wp-content/uploads/2026/09/part-0-ai-native-sys-java.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2026/09/part-0-ai-native-sys-java.png)
-----------------------------------------------------------------------------------------------------------------------------------------
# Background
In [Part 0](https://www.dhaval-shah.com/part-0-ai-native-sys-java/) we built the system that Meridian Bank already has as an existing banking system: a **_REST API_** over **_PostgreSQL_** that applies every dispute rule exactly, every time. **Run the same request twice, get the same answer twice.** That property of building **deterministic systems and testing** them out comes very naturally to us!

However, with advent of AI we're about to change our understanding.

Rules are defined in Part 0 and dispute classification is done based on these rules. But at the edge, where **Meera's "I don't recognise SQ \*TPR HSPTLTY"** has to become a reason code, we're going to call something that may answer differently each time we ask.

Before building that intelligence, I want to know exactly how much it moves. **So this part is an experiment, not a feature**. We send ten real-sounding complaints to a model, multiple times each, and count the answers.

If you've spent your career writing assertions that either pass or fail, the results will be uncomfortable. They're also the subsequent parts of this series are shaped the way they are.

# The task

One job: read a complaint, return a reason code.

```
DR-101   The same purchase was charged more than once
DR-104   Goods or services were paid for but never received
DR-107   A recurring payment continued after the cardholder cancelled
DR-201   The cardholder does not recognise the transaction at all
UNKNOWN  The complaint does not clearly match any reason code
```

That fifth option matters more than the other four. A classifier that can't say "I don't know" will always guess, and a guess here becomes a wrongly routed case and a wasted analyst hour. We're explicitly giving the model permission to decline - and then measuring whether it takes it.

The ten fixtures are written the way people actually write:

| ID | Complaint (abbreviated) | Analyst's label |
|----|------------------------|-----------------|
| C1 | "i got charged twice for the same order at that furniture place" | DR-101 |
| C2 | "streamflix is still taking 649 every month even though i cancelled ages ago" | DR-107 |
| C3 | "ordered a chair from quickcart 25 days back, it never came" | DR-104 |
| C4 | "two charges to some games company i have never heard of… my card is in my wallet" | DR-201 |
| C5 | "what is SQ \*TPR HSPTLTY on my statement? i dont recognise it at all" | **UNKNOWN** |
| C6 | "the gym charged me again after i cancelled my membership in august" | DR-107 |
| C7 | "there's a wrong charge on my card please help" | **UNKNOWN** |
| C8 | "the same amount deducted twice from my account on the same day, its the same shop" | DR-101 |
| C9 | "i cancelled my subscription and they charged me again, its basically fraud" | DR-107 |
| C10 | "my flight was cancelled by the airline months ago and i still havent got my money back" | DR-104 |

Three complaind ids deserve attention -
1. **C5 is Meera's complaint from Part 0 :** The correct answer is UNKNOWN. She isn't denying the purchase; she can't place the merchant name. You cannot tell those apart from the text alone - you need to look up the transaction first. Classifying this as fraud is the exact failure Part 0 opened with.

2. **C7 is deliberately hopeless :** "There's a wrong charge, please help" could be any of the four. The only correct behaviour is to ask a follow-up question.

3. **C9 contains a trap :** The cardholder says "basically fraud", but describes a cancelled subscription. The model has to classify what happened, not the label the customer reached for. Most humans get this wrong too, which is why intake teams are trained on it.

# The Experiment

The experiment runs each complaint through two implementations of the same task.

**Unbounded** is how most first attempts look. Ask in plain language, take whatever comes back, extract a code from it.

```java
    public Classification classifyUnbounded(String complaint, double temperature) {
        String answer = chatClient.prompt()
                .system(UNBOUNDED_SYSTEM)     // "tell me which reason code applies"
                .user(complaint)
                .options(ChatOptions.builder().temperature(temperature).build())
                .call()
                .content();
        return new Classification(ReasonCode.parse(answer), null, false, answer);
    }
```

`ReasonCode.parse` is lenient on purpose, because free-text answers arrive in every shape a sentence can take: `DR-101`, `dr_101`, "This looks like DR-107 to me.", and sometimes a paragraph explaining the bank's dispute process with no code at all.

**Bounded** constrains both ends. The prompt states rules the code owns, and the response is mapped into a record instead of a string:

```java
    Classification result = chatClient.prompt()
            .system(BOUNDED_SYSTEM)
            .user(complaint)
            .options(ChatOptions.builder().temperature(temperature).build())
            .call()
            .entity(Classification.class);
```

```java
    public record Classification(ReasonCode reasonCode,
                                String merchantHint,
                                boolean needsMoreInfo,
                                String reasoning) { }
```

The bounded system prompt adds the domain distinctions an analyst would know:

```
    - "I don't recognise this" means DR-201 ONLY when the cardholder denies making the
    purchase. If they might have made it but cannot place the merchant name, that is
    not DR-201: set needsMoreInfo to true and use UNKNOWN.
    - A subscription that keeps billing after cancellation is DR-107, not DR-101,
    even when the amounts are identical.
    - If the complaint is too vague to place, use UNKNOWN and set needsMoreInfo to true.
    An honest UNKNOWN is better than a confident guess.
```

Note what just happened. Those lines aren't prompt engineering tricks. They're **business rules written in English because they can't be written as code** - the first appearance of the split that runs through this whole series.

# What we measure

Three numbers per complaint, per mode, per temperature:

- **Distinct answers** - how many different reason codes came back from twenty identical requests. One means perfectly stable.
- **Stability** - the share of runs that agreed with the most common answer.
- **Accuracy** - the share of runs that matched the analyst's label.

Keeping the last two separate is the whole point. They aren't the same thing, and the gap between them is where the interesting failures live.

The aggregation is plain arithmetic in `VarianceReport`, which means it's unit-tested - worth doing when the numbers end up in a blog post:

```java
    @Test
    void stabilityAndAccuracyAreDifferentThings() {
        // Every run agrees on the wrong answer: the failure mode that looks like success.
        var r = VarianceReport.of("C5", "unbounded", 0.0, ReasonCode.UNKNOWN,
                List.of(ReasonCode.DR_201, ReasonCode.DR_201, ReasonCode.DR_201, ReasonCode.DR_201));

        assertThat(r.stability()).isEqualTo(1.0);
        assertThat(r.accuracy()).isEqualTo(0.0);
        assertThat(r.stablyWrong()).isTrue();
    }
```

# Run it yourself

```bash
    export GROQ_API_KEY=sk-...
    mvn -pl dispute-agent spring-boot:run \
        -Dspring-boot.run.profiles=experiment \
        -Dspring-boot.run.arguments=--experiment.runs=20
```

Ten complaints × 20 runs × 2 temperatures × 2 modes is 800 calls, so start with `--experiment.runs=5` while you're adjusting prompts. Results land in `target/variance.csv`.

Your numbers will not match mine exactly! Different model version, different day, possibly different answers - which is, after all, the thesis.

## The results

Model: **Groq - openai/gpt-oss-20b**. 10 complaints × 2 temperatures × 2 modes.

| Complaint | Mode | Temp | Expected | Most common | Distinct | Stability | Accuracy |
|---|---|---|---|---|---|---|---|
| C1 | unbounded | 0.0 | DR-101 | DR-101 | 1 | 100% | 100% |
| C1 | bounded | 0.0 | DR-101 | DR-101 | 1 | 100% | 100% |
| C1 | unbounded | 0.7 | DR-101 | DR-101 | 1 | 100% | 100% |
| C1 | bounded | 0.7 | DR-101 | DR-101 | 1 | 100% | 100% |
| C2 | unbounded | 0.0 | DR-107 | DR-107 | 1 | 100% | 100% |
| C2 | bounded | 0.0 | DR-107 | DR-107 | 1 | 100% | 100% |
| C2 | unbounded | 0.7 | DR-107 | DR-107 | 1 | 100% | 100% |
| C2 | bounded | 0.7 | DR-107 | DR-107 | 1 | 100% | 100% |
| C3 | unbounded | 0.0 | DR-104 | DR-104 | 1 | 100% | 100% |
| C3 | bounded | 0.0 | DR-104 | DR-104 | 1 | 100% | 100% |
| C3 | unbounded | 0.7 | DR-104 | DR-104 | 1 | 100% | 100% |
| C3 | bounded | 0.7 | DR-104 | DR-104 | 1 | 100% | 100% |
| C4 | unbounded | 0.0 | DR-201 | DR-201 | 1 | 100% | 100% |
| C4 | bounded | 0.0 | DR-201 | DR-201 | 1 | 100% | 100% |
| C4 | unbounded | 0.7 | DR-201 | DR-201 | 1 | 100% | 100% |
| C4 | bounded | 0.7 | DR-201 | DR-201 | 1 | 100% | 100% |
| C5 | unbounded | 0.0 | UNKNOWN | DR-201 | 1 | 100% | 0% |
| C5 | bounded | 0.0 | UNKNOWN | DR-201 | 1 | 100% | 0% |
| C5 | unbounded | 0.7 | UNKNOWN | DR-201 | 1 | 100% | 0% |
| C5 | bounded | 0.7 | UNKNOWN | DR-201 | 2 | 50% | 50% |
| C6 | unbounded | 0.0 | DR-107 | DR-107 | 1 | 100% | 100% |
| C6 | bounded | 0.0 | DR-107 | DR-107 | 1 | 100% | 100% |
| C6 | unbounded | 0.7 | DR-107 | DR-107 | 1 | 100% | 100% |
| C6 | bounded | 0.7 | DR-107 | DR-107 | 1 | 100% | 100% |
| C7 | unbounded | 0.0 | UNKNOWN | DR-201 | 1 | 100% | 0% |
| C7 | bounded | 0.0 | UNKNOWN | UNKNOWN | 1 | 100% | 100% |
| C7 | unbounded | 0.7 | UNKNOWN | DR-201 | 2 | 50% | 50% |
| C7 | bounded | 0.7 | UNKNOWN | UNKNOWN | 1 | 100% | 100% |
| C8 | unbounded | 0.0 | DR-101 | DR-101 | 1 | 100% | 100% |
| C8 | bounded | 0.0 | DR-101 | DR-101 | 1 | 100% | 100% |
| C8 | unbounded | 0.7 | DR-101 | DR-101 | 1 | 100% | 100% |
| C8 | bounded | 0.7 | DR-101 | DR-101 | 1 | 100% | 100% |
| C9 | unbounded | 0.0 | DR-107 | DR-107 | 1 | 100% | 100% |
| C9 | bounded | 0.0 | DR-107 | DR-107 | 1 | 100% | 100% |
| C9 | unbounded | 0.7 | DR-107 | DR-107 | 1 | 100% | 100% |
| C9 | bounded | 0.7 | DR-107 | DR-107 | 2 | 50% | 50% |
| C10 | unbounded | 0.0 | DR-104 | DR-104 | 1 | 100% | 100% |
| C10 | bounded | 0.0 | DR-104 | DR-104 | 1 | 100% | 100% |
| C10 | unbounded | 0.7 | DR-104 | DR-104 | 1 | 100% | 100% |
| C10 | bounded | 0.7 | DR-104 | DR-104 | 1 | 100% | 100% |

**Overall accuracy 86%.** Bounded 90%, unbounded 83%.
Every case where the same input produced two different answers occurred at temperature 0.7.

3 things to look for when you read your own table. I'd expect all four, but check rather than assume.

**1. The unambiguous cases are boring, and that's the finding :** C1, C2, C3 and C4 describe exactly one thing. They are highly stable and accurate in both the modes. If a model can't handle them, nothing that follows matters.

**2. The ambiguous cases are where the variance lives :** C5, C7 and C9 have ambiguity in the text. This is also where the two modes should diverge: the bounded prompt gives the model a legitimate way to decline (UNKNOWN plus `needsMoreInfo`), and the unbounded one doesn't.

**3. Watch specifically for "stably wrong".** That's my name for stability at 100% and accuracy at 0% - every run agrees, and every run is wrong. C5 is the candidate: a model that confidently returns DR-201 for each run, because "I don't recognise it" sounds like fraud if you only read the words. This is the most dangerous result in the table, because it's invisible to any check based on consistency. **A stable classifier can be confidently useless.** 

# What this means for how to test non-deterministic systems

This in a way tell us - there is a major shift in how non-deterministic systems are tested!

You cannot write `assertEquals(DR_101, classify(complaint))` and call it a test. Not because the model is bad, but because a single run tells you nothing. Pass or fail, you learned one sample from a distribution.

So the nature of testing changest in three ways -

**1. Assert on distributions, not on single answers :** "This complaint classifies as DR-101 at least 90% of the time over multiple runs" is a meaningful assertion. "This complaint classifies as DR-101" is a coin toss you wrote down.

**2. Separate stability from correctness, always.** They fail differently and they're fixed differently. Instability usually means the prompt is underspecified. Stable wrongness means the model doesn't have the information it needs - which is exactly C5's problem, and no amount of prompt tuning fixes it. The model needs to *look at the transaction*. That's coming in [Part-2]().

**3. Treat the prompt as code under test.** Those three English rules in the bounded prompt are business logic. They deserve version control, review, and a test suite that runs when they change. In subsequent parts, this will be converted into a CI gate: **a prompt change without an eval run is an untested deployment.**

**None of this replaces your existing testing approach and methodologies**. Part 0's rules still get ordinary deterministic tests, because they're ordinary deterministic code.

> You’ve got two styles pulling in different directions, and it's the **Architect** that keeps the boundary in the right spot so the whole design stays coherent.