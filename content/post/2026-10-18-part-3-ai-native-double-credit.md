---
title: "Part 3 - The Customer Got Credited Twice: Tools Are the New Endpoints"
author: Dhaval Shah
type: post
date: 2027-11-30T02:00:50+00:00
url: /part-3-ai-native-idempotency/
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
thumbnail: "images/wp-content/uploads/2026/10/part2-agents-springai.png"
---

[![](https://www.dhaval-shah.com/images/wp-content/uploads/2026/10/part2-agents-springai.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2026/10/part2-agents-springai.png)
-----------------------------------------------------------------------------------------------------------------------------------------
# Background
Everything the agent has done so far has been safe, because everything it has done has been read-only. It looked things up, formed an opinion, and told us about it. If it got something wrong - and in Part 2 it did, calling a fraud case a duplicate - the cost was a wrong answer on my screen.

This part changes that. The agent gets tools that file disputes and move money.

The moment an agent can act, three things stop being optional: the tools have to be designed rather than exposed, the write operations have to survive being retried, and "it seemed to work" has to become a number. This post does all three, and the middle one produces the best story - a bug that credits a customer twice, left in Part 0's code on purpose, finally triggered.

## Why a retry is not a corner case

Here is the whole problem in one sequence.

The agent decides a cardholder is owed a provisional credit of ₹4,375 while their dispute is investigated. It calls the tool. The bank's service receives the request, writes the credit, and starts sending the response. The connection drops.

From the agent's side, nothing happened. No response, no confirmation. So it does the only reasonable thing: it tries again.

The cardholder now has ₹8,750.

Nothing in that story is unusual, and critically, **none of it requires the model to be wrong**. The agent behaved correctly at every step. A load balancer, a framework retry policy, or a human clicking twice would produce the same result. An LLM just adds a new way to arrive there, because "let me try that again to be sure" is a thing models genuinely do.

Part 0 shipped this bug deliberately. The credit endpoint had no idempotency key, and a test documented it:

```java
service.issueProvisionalCredit(d.id(), new BigDecimal("4375.00"), "analyst:test");
var after = service.issueProvisionalCredit(d.id(), new BigDecimal("4375.00"), "analyst:test");

assertThat(after.provisionalCreditTotal()).isEqualByComparingTo("8750.00"); // doubled, silently
```

What makes it nasty is the cap. `DisputeService` refuses credits that exceed the disputed amount, so a doubled *full* credit gets rejected and looks safe. A doubled *partial* credit stays under the cap and sails through. The guard that was supposed to protect the money is exactly what hides the bug.

## The fix is a database constraint

The instinct is to check before inserting: look for an existing credit, and skip if you find one. That doesn't work. Two retries arriving together can both run the check, both find nothing, and both insert. The race is small but it's precisely the window a retry storm lives in.

So the fix goes in the schema:

```sql
ALTER TABLE provisional_credits ADD COLUMN idempotency_key VARCHAR(80);

-- Scoped per dispute, not globally: two different disputes may legitimately be
-- credited with the same caller-generated key.
CREATE UNIQUE INDEX ux_credit_idempotency
    ON provisional_credits (dispute_id, idempotency_key);
```

and the insert becomes unconditional, with the database deciding:

```java
int rows = jdbc.sql("""
        INSERT INTO provisional_credits
               (dispute_id, idempotency_key, amount, currency, issued_by, issued_at)
        VALUES (:id, :key, :amount, :currency, :by, :at)
        ON CONFLICT (dispute_id, idempotency_key) DO NOTHING
        """)
        ...
        .update();
return rows > 0;
```

Two details in the service are worth more than the SQL.

**The replay check runs first, before the business rules.** A retry of a credit that already succeeded must not be re-validated against current state - the dispute may since have been resolved, and rejecting the replay would leave the caller believing the credit never landed, which is how you get a *third* attempt.

**Same key with a different amount is rejected, not replayed.** That combination means the caller has a bug, and silently returning the old result would hide it:

```java
if (existing.get().compareTo(amount) != 0) {
    throw new DisputeRuleException(IDEMPOTENCY_KEY_REUSED,
            "Idempotency key %s was already used on %s for %s, not %s"
                    .formatted(idempotencyKey, d.id(), existing.get(), amount));
}
```

And the header is required, not optional. An optional safety mechanism is one the default path skips, and defaults are what callers actually ship.

## The part that looks fixed and isn't

Here's the failure mode I'd put money on being live in production systems right now.

```java
@Test
void retriedCreditWithAFreshKeyEachTimeStillDoubles() {
    Dispute d = service.fileDispute(goodsNotReceived());

    service.issueProvisionalCredit(d.id(), UUID.randomUUID().toString(), new BigDecimal("4375.00"), "analyst:test");
    var after = service.issueProvisionalCredit(d.id(), UUID.randomUUID().toString(), new BigDecimal("4375.00"), "analyst:test");

    assertThat(after.provisionalCreditTotal()).isEqualByComparingTo("8750.00");
}
```

The schema has a unique index. The endpoint demands a key. The ticket says "added idempotency". And the customer is still credited twice, because **the key identifies the attempt instead of the intent**.

A UUID generated inside the retry loop is a new key every time. It satisfies every check and protects nothing. The key has to be derived from the thing you're trying to do - here, the dispute being credited - so that a retry naturally produces the same one.

That test lives in the suite permanently. It's not testing a bug; it's testing that the distinction stays understood.

I've marked this as a test rather than prose deliberately. "Use an idempotency key" is advice everyone has heard. "Your idempotency key is probably wrong in this specific way" is the part that actually prevents the incident.

## Tool design: now an experiment, not an opinion

In Part 2 I claimed tool descriptions are an API contract the model reads, and that coarse tools beat fine-grained ones. That was intuition. Here it's measured.

Two tool sets, **identical capability**, same REST API underneath, same model, same temperature, same ten scenarios. The only variable is how the capability is presented.

**Set A - five coarse tools.** Each completes one decision, and the descriptions carry domain knowledge rather than type information:

```java
@Tool(description = """
        Credit the cardholder provisionally while the dispute is investigated. The money is
        returned now and reversed later if the dispute is decided in the merchant's favour.
        idempotencyKey must identify this CREDIT, not this attempt: use the dispute id, for
        example "credit-DSP-10007". If the call times out, retry with the SAME key - a repeat
        with the same key is safe and will not credit the cardholder twice. Never generate a
        new key for a retry.""")
public Map<String, Object> issueProvisionalCredit(String disputeId, String idempotencyKey, BigDecimal amount)
```

Read that again as what it is: **the idempotency rule from the previous section, written where the model will read it every single time**. Not in a wiki. Not in a code review comment. In the tool.

**Set B - twelve narrow tools** with accurate, useless descriptions:

```java
@Tool(description = "Returns the merchant descriptor string for a transaction.")
public String getMerchantDescriptor(String transactionId)

@Tool(description = "Posts a provisional credit on a dispute.")
public Map<String, Object> postCredit(String disputeId, BigDecimal amount)
```

This is not a straw man. It's what you get when tools are generated from an existing API one accessor at a time, or when someone reasons that smaller tools are more composable. Every description is true. None tells the model anything the schema didn't already say.

Note what `postCredit` lacks: a key parameter. So the implementation invents one per call - the exact bug from the last section, now structural rather than accidental.

The most consequential difference is filing. Set A takes all the evidence in one call. Set B splits it into `setEvidence` then `createDispute`, which means evidence that should be atomic with the filing becomes a separate step the model can simply forget.

### Results

> **FILL: run the eval and paste your numbers.** Fresh database first:
> `docker compose down -v; docker compose up -d`, then
> `mvn -pl dispute-agent spring-boot:run "-Dspring-boot.run.profiles=eval,groq"`

| Tool set | Passed | Score | Avg tool calls | Avg time |
|---|---|---|---|---|
| coarse | ?/10 | ?% | ? | ? ms |
| fine-grained | ?/10 | ?% | ? | ? ms |

What I expect, stated in advance so I can be wrong in public: coarse should pass more cases, use far fewer tool calls, and run faster. The fine-grained set should fail hardest on cases needing evidence, because that's where its boundaries force a step that can be skipped.

If the gap turns out to be small, that's worth reporting too - it would mean modern models are more robust to bad tool boundaries than I think, and that the real cost is latency and tokens rather than correctness.

## What the eval actually checks

Part 1's experiment scored one thing: did the model produce the right label. That's not enough once the agent acts.

```java
public record EvalCase(String id, String cardholderId, String complaint,
                       String expectedReasonCode,
                       boolean shouldFile,          // did a dispute actually get created?
                       Boolean shouldEscalate,      // null = either answer is defensible
                       List<String> mustMention,
                       String note) { }
```

**`shouldFile` is the check that matters**, and it's weighted double. A reason code is an opinion; a filed dispute is a row in the bank's system of record. Four of the ten cases expect *no* filing - outside the filing window, already refunded, transaction still pending, and too vague to act on. An agent that cheerfully files all ten is not a good agent that needs tuning; it's a liability.

Scoring is mechanical - string and boolean comparisons, no model judging another model. That keeps it reproducible, cheap, and specific: a failure points at a broken behaviour rather than a vague quality drop. Part 7 adds an LLM judge for the one thing this can't check, which is whether the explanation to the cardholder is any good.

There's partial credit, because "wrong code but correctly refused to file" is genuinely better than "wrong code and filed anyway":

```java
public double points() {
    double earned = (reasonCodeCorrect ? 1 : 0)
            + (filingCorrect ? 2 : 0)
            + (escalationCorrect ? 1 : 0)
            + (mentionsCorrect ? 1 : 0);
    return earned / 5.0;
}
```

Two cases carry forward from earlier parts as permanent regression tests. **E7** is Meera's tea shop, the complaint that defeated Part 1's classifier; it expects UNKNOWN, no filing, escalation, and the word "Tapri" in the explanation - because naming the merchant proves the agent resolved the descriptor rather than guessing. **E8** is the fraud case Part 2 got wrong, where two identical charges look like a duplicate and the cardholder explicitly denies making either. Both now fail loudly if they regress.

## The thing nobody tells you about eval suites

My first run of the suite scored well. My second run, five minutes later, collapsed.

Nothing had changed. The cases that file disputes had filed them, and Part 0's rules correctly refuse a second active dispute on the same transaction. Every case that worked the first time failed the second with `DISPUTE_ALREADY_ACTIVE`.

That's the eval suite discovering, the hard way, that **an agent that writes needs deterministic fixtures**. Part 1's experiment was read-only, so it could run a thousand times against the same data. This one can't. Every run needs a reseeded database:

```powershell
docker compose down -v; docker compose up -d
```

It's a boring operational detail that becomes load-bearing in Part 7, when the suite runs in CI on every prompt change. A test suite that passes only on a clean database is fine. A test suite that *silently* depends on a clean database is a trap, so the runner says so at startup.

## Seeing the bug without a model

The double-credit demo deliberately has no LLM in it:

```powershell
mvn -pl dispute-agent spring-boot:run "-Dspring-boot.run.profiles=credit-demo"
```

Three sequences: a retry with a stable key, a retry with a fresh key each attempt, and a key reused with a different amount. It's free to run, identical for every reader, and makes the point cleanly - the bug has nothing to do with intelligence. It's what happens when an operation that moves money gets retried, and something will always retry.

> **FILL: paste the demo output.** The three `ledger total` lines are the whole story.

If you'd rather poke at it by hand, `http/part-03.http` walks the same ground as HTTP requests, including the 400 you get for omitting the key and the 409 for reusing one with a different amount.

## What I'd take from this

**Tool descriptions are where operational knowledge belongs.** The idempotency rule exists in three places now: the database constraint, the service, and the tool description. The first two stop the bug. The third is the only one the model can read.

**Idempotency keys are easy to add and easy to get wrong.** The unique index is the simple part. Deriving the key from the intent rather than the attempt is the part that takes a test to defend.

**An eval suite for an acting agent is mostly about restraint.** Four of my ten cases check that the agent does *nothing*. That ratio surprised me, and I now think it's roughly right for any agent with write access.

**Writes make your test fixtures load-bearing.** Read-only experiments are idempotent by nature. The moment the agent files something, your eval suite needs the same reset discipline as a database integration test.

## Next

The agent can now investigate and act, but ask it when a dispute must be filed by and it will improvise a plausible-sounding number from training data. Filing windows, evidence requirements and scheme rules are exactly the content it must quote rather than invent.

Part 4 adds grounding: a policy corpus in PGVector, hybrid search so an exact code like `DR-104` matches, and citations in every policy answer. Including the case that bites everyone - two versions of the same policy, one superseded, and an agent confidently citing the one that was withdrawn.

Code for this part is at the `part-03` tag.