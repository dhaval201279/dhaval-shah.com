---
title: "Part 2 - Teaching It to Look Things Up: Tools, Spring AI and Conversations at Once"
author: Dhaval Shah
type: post
date: 2026-10-10T02:00:50+00:00
url: /part-2-ai-native-agent-runs/
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
[Part 1](https://www.dhaval-shah.com/part-1-ai-native-multiple-runs/) ended with a mistake worth fixing.

Meera’s complaint was: **“what is SQ \TPR HSPTLTY on my statement? I don’t recognise it at all.”**
The system flagged it as fraud every single time, with full confidence. And honestly, based on the words alone, that seemed fair. But the real issue wasn’t the prompt - it was that we were asking the model to judge a transaction it had never seen.

**No amount of prompt tweaking can solve that. What fixes it is more information.**

So this part gives the model a way to look things up, which is the step that turns a classifier into an **agent**.

# Anatomy of an Agent
In Part-0 we had REST APIs which were mainly about resources and response - you send request and you get back response (as per defined contract).
With [LLM](https://en.wikipedia.org/wiki/Large_language_model), it adds layer of reading and reasoning - it just interprets text and produces response.

> An agent goes one step further: it doesn’t just read or respond, **it can act**. It observes the world (through tools or context), reacts to what it sees, and decides the next move. In other words, REST is about data exchange, LLMs are about understanding, and agents are about doing.

Forget frameworks for time being. The idea is simple:

You give the model a conversation and a list of tools it can use. It can reply in two ways:

1. “Call this tool with these arguments.”
2. “Here’s my answer.”

If it asks for a tool, you run it, add the result back into the conversation, and send it again. If it gives an answer, you’re done. You repeat until one of those happens or you hit a step limit.

That’s the whole loop. Memory, retries, sub‑agents, approval gates - all of that sits around this loop.

One detail matters most: **the model never executes anything itself.** It only suggests a function call in JSON with arguments. Your code decides whether to run it. 
> The model proposes, code disposes. That’s not a policy, it’s how the structure works.

You could write this loop yourself in about a hundred lines. It’s worth trying once, because it makes the whole concept less mysterious. But for production, we let [Spring AI](https://docs.spring.io/spring-ai/reference/index.html) handle the loop and focus on the parts that are critical from business standpoint.

## The tool

Here is the agent's main tool. Read the description first, because the description is the part that matters:

```java
@Tool(description = """
        List the cardholder's recent card transactions, newest first. Each entry includes the
        merchant's real name, the cryptic statement descriptor the cardholder actually sees on
        their statement, the merchant category, the amount, and how much has already been
        refunded by the merchant. Use this whenever the cardholder mentions a charge, a merchant
        or a date, and always before concluding anything about a transaction you have not seen.""")
public List<TransactionSummary> findTransactions(
        @ToolParam(description = "Cardholder ID, for example CH-1001") String cardholderId,
        @ToolParam(description = "How many days back to search. Use 90 if unsure.", required = false)
        Integer days) {

    return restClient.get()
            .uri("/api/cardholders/{id}/transactions?days={days}", cardholderId, days == null ? 90 : days)
            .retrieve()
            .body(new ParameterizedTypeReference<List<TransactionSummary>>() {});
}
```

**That description is the API contract, and the model is its only reader.** The model doesn’t see your Javadoc, OpenAPI spec, or method signature. It only sees the description. From that, it decides if the tool is relevant, what to pass, and what it will get back.

Notice how the description does two unusual things:
1. It explains what a statement descriptor is, because that’s domain knowledge the model doesn’t have.
2. It tells when to call the tool - “always before concluding anything about a transaction you have not seen.” That kind of guidance belongs in the tool description, not bloating the system prompt.

The second tool carries a rule of a different kind:

```java
@Tool(description = """
        List disputes this cardholder has raised before, including how each was resolved.
        Useful background for an analyst. Never refuse or discourage a new dispute because of
        what this returns: past outcomes do not decide the present one.""")
public List<DisputeSummary> findDisputeHistory(String cardholderId)
```

It lists past disputes for a cardholder. The description adds a fairness constraint: “Never refuse or discourage a new dispute because of what this returns.”

That’s important. Karan, for example, has lost three disputes (scenario S9 in the repo). That history is useful context, but it must never be grounds to deny him a fourth. A rules engine can’t express “may inform but must not decide.” But a human intake team understands that, and now the tool description makes sure the model does too.

## The agent

With the tools written, the agent is almost nothing:

```java
public DisputeAgent(ChatClient.Builder builder, DisputeTools tools) {
    this.chatClient = builder
            .defaultSystem(SYSTEM_PROMPT)
            .defaultTools(tools)                        // the loop is wired here
            .defaultAdvisors(new SimpleLoggerAdvisor())  // every request and response, logged
            .defaultOptions(ChatOptions.builder().temperature(0.0).build())
            .build();
}

public DisputeAssessment assess(String cardholderId, String complaint) {
    return chatClient.prompt()
            .user("Cardholder " + cardholderId + " says: " + complaint)
            .call()
            .entity(DisputeAssessment.class);
}
```

Everything in the protocol I described above happens inside `call()`: the message list, the tool-call branch, executing the method, appending the result, going round again. `defaultTools(tools)` is the line that does it - [Spring AI]() reflects over the `@Tool` annotations, generates the JSON schema the model needs, and dispatches calls back to your methods.

`.entity(DisputeAssessment.class)` is the other line worth noticing. The agent's output is a Java record, not a paragraph:

```java
public record DisputeAssessment(String reasonCode,
                                String transactionId,
                                String relatedTransactionId,
                                String explanation,
                                boolean needsHuman) { }
```

This is the boundary between the probabilistic edge and the deterministic core. Everything downstream - [Part 0's](https://www.dhaval-shah.com/part-0-ai-native-sys-java/) rules

**What the framework gave us:** The loop, schema generation, dispatch, structured output mapping, logging, and portability come from the framework. 
**What stays ours, and should:** the tools, their descriptions, the system prompt & the output type. That’s the right division.

## What the first run actually produced

Four complaints, four assessments. Three were right. The fourth was wrong in a way I didn’t anticipate, and it’s the most useful thing in this post.

**1. C5 worked, which is the whole point of this part :**
App logs - 
```text
    tool findTransactions(CH-1003, 90)
    C5 [2542 ms] reasonCode=UNKNOWN, transactionId=TXN-100301
  "The statement descriptor SQ *TPR HSPTLTY corresponds to a purchase at Tapri Tea House,
   a cafe in India... currently unrecognized but not necessarily fraudulent."
```

The complaint that [Part 1’s](https://www.dhaval-shah.com/part-1-ai-native-multiple-runs/) classifier called fraud twenty times out of twenty is now handled correctly - not because of a better prompt, but because the agent looked it up.

**2. C1 got the pairing right :** 
`transactionId=TXN-100103` (the duplicate being disputed), `relatedTransactionId=TXN-100102` (the original). That’s the exact ordering [Part 0’s](https://www.dhaval-shah.com/part-0-ai-native-sys-java/) rules demand, and it’s the step the legacy web form forced a confused human to do.

**3. C4 got it wrong, and the reason matters :**

The complaint was “two charges to some games company i never bought anything from”. That’s an explicit denial - DR-201, fraud, straight to the fraud queue. The agent returned DR-101, duplicate charge. App Logs -

```text
    C4 [1167 ms] reasonCode=DR-101, transactionId=TXN-100303, relatedTransactionId=TXN-100302
    "...two identical charges of $59.99 to the same merchant... This pattern indicates
    a duplicate charge."
```

### Observations
Couple of observations based on 1st run -

**1. C5 came back with reasonCode=UNKNOWN and needsHuman=false :** UNKNOWN means “I couldn’t decide”, so the case has nowhere to go - the agent declined to classify and declined to escalate. In production that’s a case that silently stalls business!

**2. C4 flagged as duplicate transaction :** Two posted charges, $59.99 each, same merchant, ten minutes apart. As a data pattern, that is a duplicate. A duplicate means the cardholder bought something once and was billed twice. Here they say they bought nothing.

## The fix
Both problems live in one place - `SYSTEM_PROMPT` in `DisputeAgent.java`. Two rules added :

```text
    If the cardholder denies making the transaction at all ("I never bought this",
    "I don't have an account there"), that is DR-201 - even when the transactions look
    like duplicates of each other. Denial outranks any pattern you see in the data.

    If reasonCode is UNKNOWN, needsHuman must be true. An undecided case goes to an
    analyst; it never stops here.
```
**1. The first is a precedence rule between two kinds of evidence :** what the customer says versus what the data shows. 
**2. The second is a routing invariant :** Neither can be written as code, because neither has anything to check until the model has already formed an interpretation. They are business rules in English, sitting in the one place the model reads every single time.

# Conclusion
Agents aren’t magic - they’re a disciplined way of letting models act with context. REST APIs give you data, LLMs help you read and reason, but agents stitch those together into outcomes. The real power lies not in prompts but in the tools you expose and the rules you write down.

What I’d take from this:

- Tools don’t just add knowledge, they shift what the model pays attention to. That means you need to re‑check cases that already worked once you add a new tool. I skipped that, and one case quietly regressed until I caught it by reading the output line by line.

- Looking at four cases by hand isn’t a test. I only spotted the issue because I happened to read an explanation closely. With twenty cases I’d have missed it, and with a prompt change next week I’d have no way to know if it came back. That’s not a discipline failure - it’s a tooling gap. Part 4 is where we fix that.

- Write down your precedence rules before the model has to guess them. Experienced intake staff know that “denial beats pattern” because it’s drilled into them. But nobody had written it down, so the model invented its own ordering and got it wrong. Much of making agents work in real domains is about surfacing rules that humans follow instinctively but never document.

> The bottom line: agents succeed when you stop treating them as clever guessers and start treating them as structured systems. The more clearly you define the tools and rules, the more reliably they deliver.

Source Code - [Part-2 Release](https://github.com/dhaval201279/dispute-service/releases/tag/part-02)