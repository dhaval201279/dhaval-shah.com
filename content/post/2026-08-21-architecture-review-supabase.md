---
title: "The Architect's Dilemma: Validating a Vibe-Coded MVP for Regulated Industries"
author: Dhaval Shah
type: post
date: 2026-08-20T01:00:50+00:00
url: /architecture-review-supabase/
categories:
  - architecture
  - vibe-coding
tags:
  - architecture
  - vibe-coding
thumbnail: "images/wp-content/uploads/2026/08/architect-dilemma-vibe-coded-app.png"
---

[![](https://www.dhaval-shah.com/images/wp-content/uploads/2026/08/architect-dilemma-vibe-coded-app.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2026/08/architect-dilemma-vibe-coded-app.png)
-----------------------------------------------------------------------------------------------------------------------------------------

> **What happens when senior leadership ships a "soft launch" to production without architectural review - and why every architect needs a battle-tested framework for saying no.**


# Background

A product company - has been building software for a industry that operates in highly regulated environment for more than twenty years. Their legacy platform is battle-hardened, regulation-compliant, and technologically obsolete. Business logic residing in _Stored procedures_ & _EJBs_ written during 2005-2008. A database schema that has grown like coral - layer upon layer, undocumented, but somehow still working!

Senior leadership decides it's time to modernize. They hire a team. The team vibe-codes an MVP in 6-8 weeks with latest tech stack : _React_, _Vite_, _TypeScript_, _Tailwind_, _shadcn/ui_ on the frontend. [_Supabase_](https://supabase.com/) i.e. managed PostgreSQL, Auth, Storage, Realtime, and Deno Edge Functions for business logic in the backend. It looks beautiful in demos. The CI/CD pipeline is green. The [_Vercel_](https://vercel.com/) deploy is instant.

Then someone in leadership says the quiet part out loud: *"Let's soft-launch this to production."*

And someone else - perhaps a board member, perhaps a nervous engineering manager — says: *"Has an architect reviewed this?"*

That's when I was reached out!

# The First Look: What You See in Hour One

I opened the architecture diagram. It looked clean & elegant. The kind of diagram that wins startup pitch competitions but falls apart under the first audit.

The stack is textbook modern web development:

| Layer | Technology |
|-------|-----------|
| Frontend | React  + Vite + TypeScript + Tailwind + shadcn/ui |
| API/Transport | HTTPS + REST (PostgREST) + Edge Function invoke |
| Backend | Supabase Cloud + Deno Edge Functions |
| Data | PostgreSQL (managed) + RLS policies |
| Security | Supabase Auth (JWT), role tables |

The product serves construction industries workflow. But here's the complication: their customers build **defence installations** and **nuclear facilities**. Their target markets include the **UK, EU and Middle East**. The legacy system they're replacing has been in production for two decades because it passed every audit & regulatory review.

And no architect was involved in designing the replacement!

# The Framework: Seven Dimensions of Production Readiness

When I validate an architecture for products pertaining to regulated industries, I don't start with code quality, test coverage - certainly they are important aspects of Product architecture but not now atleast!

## Dimension 1: Architecture & Vendor Strategy
**Question:** Can we exit this platform in future if the vendor changes terms, pricing, or availability?

**What I found:** The entire backend - auth, database, storage, edge compute — is tightly coupled to Supabase's proprietary APIs. There is no industry standard architecture. No engineering pattern abstractions. No event sourcing that would allow a backend swap. The code imports `supabase-js` directly into React components.

**The Risk:** This isn't just vendor lock-in. It's *platform capture*. If Supabase discontinues a feature, changes their MFA flow, or gets acquired, the migration path is a full backend rewrite. For a twenty-year-old product serving strategic customers, that's not a risk - it's an existential threat.

**Leadership's typical response:** *"We'll cross that bridge when we come to it."*

**The architect's reply:** *"In regulated industries, there is no bridge. There's a huge gap, and you fall into it during compliance audit."*


## Dimension 2: Implementation & Business Logic Placement
**Question:** Where does the critical business logic live, and **_can it execute deterministically under load?_**

**What I found:** Entire business logic living inside [Deno Edge Functions](https://supabase.com/docs/guides/functions). These are ephemeral, stateless, resource-constrained isolates with 2-second CPU limits, 256MB memory caps, and cold starts that range from ~200 ms to 800 ms depending on idle time.

**The Risk:** Edge functions are designed for lightweight transformations - webhooks, API proxies, auth middleware. They are not designed for deterministic financial calculations or safety-critical workflow orchestration. A cold start during a nuclear compliance report generation is not a performance issue. It's a safety issue.

**Supabase's own documentation confirms this:** *"Design for short-lived, idempotent operations. Heavy long-running jobs should be moved to background workers."*


## Dimension 3: Operations & Control
**Question:** Can we operate this system at 3 AM during an incident without vendor intervention?

**What I found:** No infrastructure-as-code for edge functions. No ability to tweak memory allocation, restart a stuck function, or inspect thread dumps. Deployments happen via CLI (`supabase functions deploy`) with no canary releases, no blue-green roll-outs, and no automated rollback.

**The Risk:** In regulated operations, you need comprehensive runbooks, not prayers. When an edge function times out during a defence contractor's end-of-quarter financial close, your on-call engineer cannot SSH into the runtime, cannot restart it, and cannot even reliably determine which geographic region executed the failing request. You cannot build a compliant runbook around an opaque runtime.

**Brutal Truth:** Supabase Edge Functions are **developer-centric**, **not ops-centric**


## Dimension 4: Resiliency & Fault Tolerance
**Question:** What happens when components fail? Do we fail safe, or do we fail catastrophically?

**What I found:** No circuit breakers. No dead-letter queues. No cross-region disaster recovery. No graceful degradation paths. If the AI Gateway edge function fails, the entire project dashboard fails. If compute intensive calculation exceeds its 2-second CPU limit, the request returns a 546 error with no retry mechanism.

**The Risk:** Complicance standards require *fail-safe* behaviour - systems must degrade into a known-safe state. This architecture fails with *fail-closed* (hard errors) or *fail-unsafe* (partial data writes) behaviour.

**The client's proposed mitigation:** *"We'll self-host Supabase and then we'll have control."*

**The reality:** Self-hosting gives you Linux VMs, not resilience. You still have to bake in circuit breakers, DLQs, and DR - and that too all by yourself.


## Dimension 5: Scalability & Performance
**Question:** Will this architecture handle 10x growth without architectural changes?

**What I found:** Row Level Security (RLS) policies are the primary access control mechanism. Supabase's own benchmarks show that unoptimized RLS policies can degrade query performance. Edge functions have hard concurrency limits. Database connection pools are tier-capped.

**The Risk:** A platform for large defence projects handles thousands of work packages, millions of cost records, and complex multi-tenant queries. RLS policies that work fine in a demo with 100 rows become query killers at production scale. And because RLS is invisible to application code, developers don't know they've hit the cliff until users start timing out.

**Brutal Truth:** Supabase’s RLS capability is powerful for security but dangerous for performance if not engineered carefully. The operational hazard is real: developers can unknowingly ship implementations that scale poorly, and the **failure only surfaces under production load.**


## Dimension 6: Data Localization & Sovereignty
**Question:** Can we prove, with documentation, that data never leaves approved jurisdictions?

**What I found:** Supabase Cloud has no Middle East region. Edge functions execute on a distributed global network — the region of execution is opaque and not guaranteed. Auth metadata, realtime events, and logging infrastructure may transit through non-approved regions even if the database is pinned to an approved region.

**The Risk:** If your domain can be blacklisted because someone else on the platform violated terms, you are not in control of your destiny. Since you don’t control DNS, IP ranges, or failover. Your destiny is tied to Supabase’s operational maturity.

**Brutal Truth:** Supabase Cloud is developer‑friendly but sovereignty‑weak. You cannot guarantee data locality beyond the database region. For regulated workloads (defense, finance, healthcare), this is a business continuity risk. Business continuity cannot hinge on a shared SaaS domain that you don’t control.


## Dimension 7: Observability & Traceability
**Question:** Can we trace a user's request end-to-end and retain audit logs for regulatory inspection?

**What I found:** No distributed tracing. No metrics collection. No centralized logging with tamper-proof retention. No alerting. No SLO/SLI definitions. Log retention is 1 day (Free) to 90 days (Enterprise). Correlating an edge function invocation with the specific database transactions it triggered requires manual SQL queries across separate log tables.

**The Risk:** NIST 800-171 requires continuous monitoring. NIS2 mandates 24-hour incident reporting. IAEA standards require audit trails for safety system performance. GDPR requires breach detection. Supabase doesn't provide these capabilities natively.

**The client's proposed mitigation:** *"We'll use Log Drains to ship logs to our own SIEM."*

**Brutal Truth:** Supabase Cloud is developer‑friendly but observability‑weak. For regulated workloads, it is non‑compliant out of the box. Log Drains are a partial fix, but they only address retention. You must build tracing, metrics, alerting, and compliance monitoring yourself - at significant cost and complexity.


# General Considerations for Architects Evaluating Vibe-Coded Products

If you're an architect brought in to validate an MVP for regulated industries, here's what you need to keep in mind:

## 1. Vibe-Coding is Prototyping, Not Product Engineering
[Vibe-coding](https://en.wikipedia.org/wiki/Vibe_coding) - rapid assembly of modern frameworks without architectural guardrails — produces excellent demos and terrible production grade systems. In regulated industries, the cost of fixing architecture in production is 10–100x the cost of fixing it in design. Treat vibe-coded MVPs as **throwaway prototypes** unless they've been explicitly architected for production with all required guardrails.

## 2. "Soft Launch" is an Oxymoron in Regulated Sectors
There is no such thing as a soft launch for software that touches defence, nuclear, healthcare, or financial data. Every production deployment is a compliance event. Every data breach is a regulatory incident. Every incorrect calculation is a potential safety risk. **If the system isn't ready for a full audit, it isn't ready for production.**

## 3. Managed Services are Not Free of Compliance Burden
Using a managed service doesn't outsource your compliance obligations. It outsources the infrastructure management. You are still responsible for data classification, access controls, audit trails, incident response, and vendor due diligence. **If your managed service provider lacks FedRAMP, ISO 27001, or SOC 2 certifications for your tier, you cannot comply by proxy.**

## 4. Data Residency is About both - Storage & Compute
Most teams think data residency means "our database is in EU." It's not. It's about where auth tokens are validated, where edge functions execute, where logs are stored, where backups are replicated, and where support staff can access the data. If any of these touch a non-approved region, you are breaching data residency requirements.

## 5. Observability is Not a Production Add-On
You cannot bolt observability onto an architecture after it's built. Distributed tracing, metrics, alerting, and audit logging must be designed into the system as first class requirements.

## 6. The Best Architecture is Boring
In regulated industries, the best systems are boring. They use well-understood patterns. They have clear abstraction layers. They prioritize maintainability over fancy / hyped technologies. They choose containerized services over edge functions for business logic.

The legacy system lasted twenty years because it was boring. The new system needs to last twenty years too.

## 7. Saying No is the Architect's Most Important Job
Everyone else on the team is incentivized to ship. Product wants features. Engineering wants to finish the sprint. Leadership wants revenue. The architect is the only person whose job is to say: *"This isn't ready."* - of course with required justifications and trade-off analysis

## 8. Self-Hosting is Not a Silver Bullet
When leadership hears "the cloud vendor doesn't meet our requirements," their response is often: "We'll self-host it then." This is rarely the right answer. Self-hosting a complex platform (Supabase, Firebase, etc.) means you become the platform team. You patch CVEs. You scale databases. You debug distributed systems at 2 AM. If you don't have platform engineering expertise, self-hosting increases risk rather than reducing it.

**The right question isn't "Can we self-host it?" The right question is "Should we be using this platform at all for our use case?"**


# The Decision Framework: A Checklist for Regulated Production

Use this framework when evaluating any backend platform for regulated production:

| Dimension | Pass Criteria | Red Flags |
|-----------|--------------|-----------|
| **Architecture** | Abstraction layer allows backend swap within scheduled timelines; no proprietary API lock-in | Direct imports of vendor SDKs into UI; no architecture / implementation patterns |
| **Implementation** | Business logic in version-controlled, testable services with deterministic performance | Core logic in serverless functions with timeouts and cold starts |
| **Operations** | Infrastructure-as-code; documented runbooks; 24/7 on-call with vendor escalation | Manual CLI based deployments; no runbooks; opaque runtime |
| **Resiliency** | Circuit breakers; DLQs; cross-region DR tested quarterly; graceful degradation | Single points of failure; hard errors on dependency outage |
| **Scalability** | Load tested at 2x projected peak; RLS policies benchmarked | No load testing; RLS as primary and only access control |
| **Data Sovereignty** | Proven geo-fencing for compute, storage, auth, logs, and backups | "Database is in region X" as sole residency proof |
| **Observability** | Distributed tracing; metrics dashboards; alerting on SLOs; multi-year log retention | Console logs only; reactive debugging; log retention < 1 year |
| **Maintenance** | ADRs; data flow diagrams; legacy parity mapping | Knowledge in one person's head; no documentation; orphaned code |

If you have more than two red flags, the architecture is not production-ready for regulated workloads.


# Conclusion: The Architect's Responsibility

I told the client:

> *"This architecture is suitable for a prototype or an internal demo. It is not suitable for production in defence, nuclear, or sovereign data environments. The eight dimensions I've outlined are not negotiable — they are the minimum bar for regulated production. Self-hosting Supabase doesn't lower the bar; it just moves the work to your team. If you don't have a platform engineering organization, you're not ready to self-host. And if you don't have an architect reviewing this, you're not ready to launch."*

The client didn't want to hear this. They had invested eight weeks and significant budget. The demos were beautiful. The team was proud. Leadership had promised a launch date to the board.

But here's the thing about architecture: 
> **it's not about what works in a demo. It's about what survives an audit, an outage, and twenty years of maintenance.**

The legacy system is bit ugly, slow, and outdated. But it has survived because an architect, twenty years ago, made boring choices that prioritized compliance over convenience, maintainability over bleeding edge technology, and safety over speed.


# Further Reading

- [NIST SP 800-171 Rev. 2 — Protecting Controlled Unclassified Information](https://csrc.nist.gov/pubs/sp/800/171/r2/upd1/final)
- [IAEA Safety Standards Series — Software for Computer Based Systems Important to Safety in NPPs](https://www.iaea.org/publications/6019/software-for-computer-based-systems-important-to-safety-in-nuclear-power-plants)
