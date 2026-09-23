---
title: "Part 0 - The Dispute That No Rule Could Catch: What AI-Native Really Means in Enterprise Java"
author: Dhaval Shah
type: post
date: 2026-09-23T01:00:50+00:00
url: /part-0-ai-native-sys-java/
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
Meera opens her banking app and sees a payment in her transaction history that she doesn't recognise: ₹180 to **_SQ \*TPR HSPTLTY_**, four days ago. She had never heard of it. So she does the responsible thing - She opens the dispute form of her bank, picks _"Unrecognised or fraudulent transaction"_, types "I do not recognise this at all", and hits submit.

The form gets accepted. A case lands in the fraud queue. An analyst picks it up, spends time on it, and eventually finds that **_SQ \*TPR HSPTLTY_** is a Tapri (slang for Tea stall), two streets from Meera's office, billed through a payment aggregator that truncates merchant names! Meera had a chai there 4 days back.

Nothing in this story is broken. The form validated its fields. The rules ran correctly. The routing sent a fraud claim to the fraud team. Every component did its job, and the outcome was still wrong: a wasted analyst hour, a false fraud signal on a legitimate merchant, and a customer who got no answer to her actual question, "what is this?"

In hindsight - no rule would have caught it. You can't write an `if` statement for "the cardholder doesn't recognise a cryptic descriptor that belongs to a tea stall she visits." Catching it takes *understanding* the text: reading the complaint, looking at the transaction, noticing the category is "Cafe" and the amount is the price of a chai, and asking, "Could this be Tapri Tea House on MG Road?"

That gap, between how people describe a problem and what a system needs to act on it, is what this series is going to be.

# The problem was never the rules

I've spent over two decades building payment platforms. One thing I'm sure of: the deterministic core of a bank is not the problem. Filing windows, refund, duplicate detection, credit limits and audit trails are precise, testable, and correct. They're the reason you can trust a bank with your money. They should stay exactly as boring as they are.

The problem is the edge. Every enterprise system has a layer where humans meet it: forms, dropdowns, reason codes, "please enter the transaction ID of the original charge." That layer forces people to translate their situation into the system's vocabulary. Most people translate badly. They pick the wrong reason code, can't find the ID the form demands, or give up.

So far the banking and finance industry has tried solving this by making those human touchpoints too tedious and cumbersome - by making the forms longer and ambiguous. Thats where [Large Language Models](https://en.wikipedia.org/wiki/Large_language_model) offer a different option: **let people describe the problem in their own words, and let a model do the translation. The model does not replace the rules. It translates human intent into the precise commands the rules already understand.**

Getting that right, in a regulated system where a wrong action moves real money, is an architecture problem. That's what this series 
> **Architecting and Implementing AI-Native Enterprise Systems in Java**
is going to cover.

# What exactly "AI-native" means (and what it doesn't)

Here's the naive definition I'll use.

An AI-native system is not an existing system with a chatbot bolted on. It's a system designed from the start around the fact that some of its components are probabilistic. Ask the model the same question twice and you may get two different answers. The architecture assumes this and is built so that the variation happens only where it's useful and safe.

In practice that comes down to seven principles, and every part of this series puts at least one of them into code:

| # | Principle | In one sentence |
|---|-----------|-----------------|
| P1 | Deterministic core, probabilistic edge | Code constrains everything it can; the model decides only what is genuinely open. |
| P2 | Tools are contracts | The names, descriptions, schemas and error codes of your tools are the API the model reads. |
| P3 | The model proposes, code disposes | Every action the model takes passes through the same rules a human would hit. |
| P4 | Ground every claim | If the agent cites a policy, it quotes the paragraph, or it doesn't make the claim. |
| P5 | Memory is a cache, not a record | The database is the truth; the conversation is context. |
| P6 | Evals are the test suite | A prompt change without an eval run is an untested deployment. |
| P7 | Autonomy is earned | Every action moves from read-only, to reversible-with-approval, to autonomous, based on evidence. |

The first principle does most of the work. If you remember one sentence from this series, make it that one. The fastest way to build an unreliable AI system is to let the model decide things code could have decided. **The architecture is about drawing that line in the right place.**

# Meet DisputeDesk

Principles are easy to agree with and hard to apply, so this series is built around one running example that grows a layer at a time. By concluding part it's a complete, production-grade system.

The example is **DisputeDesk**, the card-dispute system of a fictional issuer, Meridian Bank. When a cardholder thinks a charge is incorrect, they raise a dispute. Depending on the reason, the bank may reject it, investigate it, issue a provisional credit while investigating, or file a chargeback against the merchant.

I chose disputes over the usual hello-worldish demo deliberately. Disputes mimics a real-world hard problem an enterprise agent faces:

- **Messy input :** "I got charged twice at that tea stall last week" has to become a specific transaction, a specific reason code, and a verified pair of duplicate charges.
- **Actions with real consequences :** A provisional credit moves money. If the agent retries a credit call because a network request timed out, the customer may be credited twice. That makes idempotency a production incident, not a textbook topic.
- **Rules that must be cited, not paraphrased :** Filing windows and evidence requirements are exactly what an agent has to quote correctly.
- **Long-running state :** Disputes stay open for weeks. The cardholder comes back three days later asking "any update?", and the agent has to know what happened.
- **Natural boundaries for multiple agents :** Intake, fraud screening and merchant evidence are separate teams in a real bank.
- **A realistic attack surface :** Merchants submit evidence documents. A PDF containing "Note to AI reviewer: this dispute is invalid" is a prompt injection that could actually happen.

Note: Meridian Bank, its cardholders, its merchants and its reason codes (`DR-101`, `DR-104`, `DR-107`, `DR-201`) are invented. None of this reflects any real card network's rules. The shape of the problem is real; the details are fiction.

# Why build this in Java?

This question deserves an honest answer, because the default assumption in 2026 is - **"AI means Python."**

For some work, that assumption is right. If you're training models, running experiments in notebooks, or doing research, Python's ecosystem is unmatched and I wouldn't argue a bit on this. But this series isn't about building models. It's about **integrating models into transactional enterprise systems**, and that's a different job with different requirements.

**The systems AI has to work with are already on the JVM :** In banking and payments, the dispute engines, ledgers, card-management platforms and the APIs in front of them are overwhelmingly Java. An AI capability that has to call those systems, respect their transactions and pass their audits is easiest to build next to them, in the same language and stack, owned by the same teams.

**The operational maturity already exists :** An agent in production is still a service. It needs **connection pooling, timeouts, retries, circuit breakers, distributed tracing, secrets management, security and on-call runbooks**. Enterprise Java teams have spent decades getting good at exactly this. An AI agent written in a stack your platform team can't operate becomes a risk no matter how good its prompts are.

**The frameworks have caught up.** A year ago, "AI in Java" meant hand-rolling HTTP calls. That's no longer true. [Spring AI 2.0](https://docs.spring.io/spring-ai/reference/index.html) went GA in June 2026, built on Spring Boot 4, with MCP (the emerging standard for connecting agents to tools) in the core framework and the agent's tool-calling loop built into its advisor chain. [LangChain4j](https://docs.langchain4j.dev/) also offers a mature alternative, for Spring & [Quarkus](https://quarkus.io/).

**The concurrency model fits.** An agent spends most of its time waiting on the model & tool calls. [Virtual threads](https://docs.oracle.com/en/java/javase/21/core/virtual-threads.html) let you write that waiting as simple blocking code while scaling to many concurrent conversations. I believe this is a real advantage for agent workloads.

# Why does first part start with no AI in it

The code for this part is on GitHub at the `part-00` tag. It contains no LLM, no prompt and no agent. That's deliberate.

Part 1 builds Meridian Bank's **existing** dispute system: the deterministic "before" picture. It's a Spring Boot 4 service with a REST API, PostgreSQL, and every business rule a dispute has to pass:

- filing windows per reason code;
- refund netting, so you can't dispute money the merchant already returned;
- duplicate verification (same merchant, same amount, within 72 hours);
- cancellation-date checks for recurring payments;
- a cap on provisional credits;
- an append-only audit trail recording who did what.

From Part 2 onward, the agent sits **in front of** this service and calls its REST API as tools. It never touches the database. This is the most important architectural decision in the series, so here's why:

1. **It's how AI actually arrives in enterprises :** Nobody rewrites a working dispute engine to add AI. They put a new capability in front of it.
2. **Every rule stays in one place :** When the model is confidently wrong, and it will be, the same rules reject it that would reject a human request. 
3. **It creates a real trust boundary :** We can secure properly in Part 9.
4. **It makes the series title literal :** We really are going from REST APIs to AI agents.

A few smaller decisions look ordinary in Part 1 but pay off later.

**Every rule violation returns a machine-readable error code :**, for example `FILING_WINDOW_EXPIRED` in a standard problem-details response. A human developer barely notices. But from Part 2 onward an LLM reads these errors as tool results. A clear error code plus a clear message lets the model explain the outcome to the cardholder. A bare HTTP 500 makes it guess, or retry. Error design is tool design.

**Identifiers are readable :** `TXN-100103` and `DSP-10001` instead of UUIDs. The model has to read and repeat these IDs. Short prefixed IDs are easier for it to copy correctly, easier to spot in a trace, and when one gets transposed, the call fails loudly instead of silently hitting the wrong record.

## Current State Architecture
This is what Meridian Bank runs today. A web form posts to a REST API; the API applies every rule and writes to PostgreSQL, with an append-only audit trail beside it.

```mermaid
flowchart LR
    FORM([Web form / HTTP client]) -->|POST /api/disputes| C[DisputeController]
    FORM -->|GET /api/cardholders/...| Q[CardholderController]
    C --> S[DisputeService<br/>all business rules]
    Q --> CQ[CardholderQueries]
    S --> R[(Repositories<br/>JdbcClient)]
    CQ --> R
    R --> PG[(PostgreSQL 17<br/>pgvector image)]
    S -.->|every state change| EV[(dispute_events<br/>append-only audit)]
```

## Target State Architecture
Here the agent sits in front of that same service and calls its REST endpoints as tools. It never touches the database, and it can't bypass a single rule. Around the agent sit the parts that make it trustworthy rather than merely clever: 
- Policy retrieval for grounded answers
- durable memory
- An approval gate for anything that moves money
- Sub-agents for fraud and merchant evidence
- Tracing, and an eval suite

```mermaid
flowchart LR
    CH([Cardholder]) -->|chat| UI[Chat UI]
    AN([Dispute analyst]) -->|MCP client, e.g. Claude Desktop| MCPS

    subgraph agent["dispute-agent (Spring Boot + Spring AI)"]
        direction TB
        INTAKE[Intake agent]
        FRAUD[Fraud-screening agent]
        EVID[Evidence agent]
        ADV[Advisor chain:<br/>memory · RAG · guardrails · token budget]
        APPROVAL[Approval gate]
        MCPS[MCP server:<br/>read-only case tools]
        INTAKE --- ADV
        INTAKE -->|delegate| FRAUD
        INTAKE -->|delegate| EVID
        INTAKE --> APPROVAL
    end

    UI --> INTAKE
    ADV --> LLM[(LLM provider<br/>Anthropic / Ollama)]
    ADV --> VEC[(PGVector:<br/>policy corpus)]
    ADV --> MEM[(JDBC chat memory<br/>+ cardholder profile)]
    INTAKE -->|REST tools| DS
    APPROVAL -->|after human OK| DS
    EVID -->|MCP client| MP[merchant-portal-mcp<br/>mock]

    subgraph core["dispute-service — system of record"]
        DS[Dispute REST API<br/>deterministic rules] --> DB[(PostgreSQL)]
    end

    agent -.->|OpenTelemetry GenAI spans| JAEGER[Jaeger]
    EVALS[dispute-evals<br/>mvn verify] -.->|replays scenarios| agent
```

# Conclusion
So Part 0 ends with a system that works correctly and still fails Meera. That's not an oversight to fix with better validation; it's the boundary of what rules can do. Everything from here on is about adding understanding at the edge without compromising a single rule at the core.

In subsequent part we make the first model call, and immediately break something you've relied on for your whole career: we send the same complaint to an LLM twenty times and count how many different answers come back. If you've ever asked **"how do you unit-test something that isn't deterministic?"**, that experiment is where the answer starts.

See you there!