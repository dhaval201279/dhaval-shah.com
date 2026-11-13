---
title: "Security Design - JWT or No JWT"
author: Dhaval Shah
type: post
date: 2026-11-13T02:00:50+00:00
url: /security-jwt-or-no-jwt/
categories:
  - Architecture
  - Security
tags:
  - api-security
  - fintech
  - oauth2
thumbnail: "images/wp-content/uploads/2026/11/api-security-models-1.png"
---

[![](https://www.dhaval-shah.com/images/wp-content/uploads/2026/11/api-security-models-1.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2026/11/api-security-models-1.png)
-----------------------------------------------------------------------------------------------------------------------------------------

**_Written in my capacity as a Consulting Architect for a fintech who is currently designing its API security model._**

# Background
Recently I have been partnering with a Fintech to design their API security model. Initial design draft (prepared by their engineering team) read: **"TLS 1.2+ and API keys."**. In a production grade system that moves money, its just authentication by possession and nothing else - no message integrity, no non-repudiation, no protection once TLS terminates at the gateway.

A security model for a Fintech platform is not just about technology decision or selection of a fancy acronym from information security world. Its more about set of answers to a few specific questions, against a specific attacker and operational cost the Fintech would incur every year for as long as the platform lives.

My Architecture and Design philosophy for crafting enterprise applications has always been problem driven. So following the same principle, I started with answering -

> What does PROBLEM SPACE look like?

# Start with the Attacker
On the other side of a B2B fintech platform is an institution - a bank, an NBFC, a payment aggregator. The relationship is contractual and long-lived. Credentials sit on servers, not on phones. Each one can move money! When something breaks, the argument is settled with logs and contracts.

These are the six threats that actually matter :

**1. The leaked credential :** Secrets in Git history, CI logs, .env files on a jump host, a vendor support ticket etc. This is the most common real failure by a wide margin. With only what leaked, what can an attacker do, and for how long?

**2. TLS that ends early :** Your TLS session doesn't run client-app to your-app. It ends at a CDN, WAF, load balancer, or API gateway - and on the client side, at an outbound proxy. What still protects the request after the first termination point?

**3. The chatty intermediary :** Gateways, APM agents, request loggers, and "temporary" debug filters write headers and bodies to log stores with broader access than your production database - including the Authorization header.

**4. Replay and duplication :** A valid request sent again. Sometimes malicious, sometimes a genuine retry after a timeout. Same symptom, different controls - confusing the two produces designs that handle neither.

**5. The insider :** At your end or theirs. Anyone who can read a shared secret can forge requests.

**6. The counterparty who disputes :** "We never sent that ₹5 lac payout instruction." Not a network attack - the situation your controls have to stand up in, months later, with a lawyer reading along.

# The five properties every API security decision is really trading off

## 1. Authentication: who is calling?

- **TLS alone :** authenticates the *server* to the client. The client stays anonymous unless you use mutual TLS.
- **Mechanisms :** API key, client ID + secret, HMAC request signing, `private_key_jwt`, mTLS client certificate.
- **Common mistake :** treating possession of a static string as identity. An API key proves that the caller has the key. Nothing more.

## 2. Authorization: what is this caller allowed to do, on which resources?

- **TLS alone :** nothing.
- **Mechanisms :** scopes and claims in access tokens, restricted or scoped keys, server-side policy.
- **Common mistake :** one credential that unlocks every endpoint.

## 3. Integrity and non-repudiation: was the request altered, and can the sender later deny it?

These two are related, but they're not the same property.

- **Integrity :** means the bytes you received are the bytes the sender produced. A shared key (HMAC) gives you that.
- **Non-repudiation :** means you can show a third party that *only* the sender could have produced those bytes. That needs an asymmetric key that only the sender holds, plus the operational discipline to make that claim credible.
- **TLS alone :** integrity in transit, one hop at a time. Nothing provable afterwards. 
- **Mechanisms :** HMAC (integrity only); JWS or HTTP Message Signatures with asymmetric keys (integrity, plus non-repudiation if key custody holds up).
- **Common mistake :** believing that a signed JWT access token protects the request body. **It doesn't.**

## 4. Confidentiality: who can read it?

- **TLS alone :** only the endpoints of each hop, only while the data is moving.
- **Mechanisms :** JWE over the full payload, field-level JWE.
- **Common mistake :** encrypting entire payloads by default, then finding out the WAF, the gateway policies, the logs and your on-call engineers are all blind. The second-most-common: carefully encrypting fields you never needed to receive.

## 5. Sender-constraint and replay protection: can a stolen credential, or a captured request, be used by someone else or used again?

- **TLS alone :** no.
- **Mechanisms :** tokens bound to an mTLS certificate ([RFC 8705](https://www.rfc-editor.org/rfc/rfc8705.html)) or a DPoP key ([RFC 9449](https://www.rfc-editor.org/rfc/rfc9449.html)); timestamps and nonces inside request signatures; short credential lifetimes. Idempotency keys belong next to these, but they solve the *reliability* half of the problem, not the security half.
- **Common mistake:** "our tokens expire in five minutes" offered as replay protection.

# The two axes that decide most outcomes

Mostly every B2B API security model is a position on two axes.

## Axis 1: shared keys vs asymmetric keys

**Shared secrets** (API keys, client secrets, HMAC keys) are held by both parties. That has consequences:

- Either side can produce a valid request, so non-repudiation is impossible by construction.
- A leak at *either* party compromises the credential.

**Asymmetric keys** keep the private key with the sender, and you hold only the public key. Leak your database of public keys and nothing happens. That's the basis for non-repudiation. The cost is **Key Management Lifecycle** across your integrators' stacks, and debugging signature mismatches whose cause is invisible.

## Axis 2: where trust lives

- **Direct (bilateral) trust.** Each pair of parties exchanges keys or certificates. It's easy to reason about and auditable. Onboarding work grows linearly with the number of counterparties, and every rotation is a coordinated event. It fits a small number of long-lived institutional relationships.
- **Delegated trust.** An authorization server issues tokens and APIs trust the issuer. Policy, scopes and revocation are centralised. In exchange, you've added a component whose availability is now your API's availability.
- **Ecosystem trust.** A scheme operator runs a directory or certificate authority that vouches for participants, as in open banking. You inherit its rules, its tooling and its pace of change.

# The centerpiece: "JWT" means two different things

Half the confusion in "JWT or no JWT" comes from one acronym covering two jobs.

**JWT as an access token.** An authorization server issues it and signs it. It carries claims like `sub`, `scope`, `aud` and `exp`. A valid token proves *the authorization server issued this token to this client, with these permissions, until this time*. That's all.

**JOSE as a message container.** JWS signs *your payload* with the sender's key. JWE encrypts it to the recipient's key. With asymmetric keys, a valid JWS proves *this sender produced these exact bytes*.

The difference shows up in a single request:

```http
POST /v1/payouts HTTP/1.1
Host: api.example-fintech.in
Authorization: Bearer eyJhbGciOiJQUzI1NiIsImtpZCI6ImFzLTIwMjYtMTEifQ...
Content-Type: application/json

{
        "beneficiaryAccount":"XXXXXXXX4521",
        "ifsc":"ABCD0001234",
        "amount":500000
}
```

Suppose something after the TLS termination point changes `amount` to `5000000`. The access token still validates. Its signature covers the token, not the body. Your resource server checks issuer, audience, expiry and scope, everything passes, and the money moves.

So "JWT or no JWT" is really two independent decisions:

1. **How do we do authorization:** static credentials, or tokens from an authorization server?
2. **Do we need message-level protection:** signatures over the request, encryption of the payload, or neither?

# How to evaluate security models
To compare available security models, it can be vetted through below criteria - 

- **Security -** Which of the five properties it delivers, against which of the six threats, and what happens when a key leaks.
- **Operations -** The recurring work per tenant per year: rotations, certificate renewals, onboarding steps, incident handling.
- **Overhead -** Extra infrastructure and moving parts: authorization server, HSM or KMS, PKI, JWKS endpoints, nonce and idempotency stores.
- **Performance -** CPU cycles spent on cryptography per request, extra round trips (token fetches, handshakes) and payload growth.
- **Scalability -** The state each request depends on and how that state behaves across regions and under failover.
- **Implementation -** What you must build and operate correctly.
- **Developer experience -** How long an integrator takes to get from receiving credentials to the first successful call, and how debuggable a failure is. In B2B this matters more than people admit. Your integrator may be an FI's outsourced vendor on a stack you don't control, and "401 signature invalid" with no further detail is a tax you pay on every onboarding.

# The map of various security mechanisms and what it achieves

Here's how the common mechanisms line up against the five properties.

| Mechanism | AuthN | AuthZ | Integrity | Non-repudiation | Confidentiality beyond TLS | Sender-constraint / replay |
|---|---|---|---|---|---|---|
| Static API key / client secret | Possession only | Coarse | No | No | No | No |
| HMAC request signing | Yes | No | Yes | No | No | Partial (if timestamp/nonce signed) |
| OAuth 2.0 client credentials + JWT, `client_secret` auth | Yes | Yes (scopes) | No | No | No | No (bearer token) |
| OAuth 2.0 client credentials + JWT, `private_key_jwt` | Stronger | Yes (scopes) | No | No | No | No (token still bearer) |
| mTLS (transport) | Yes, at terminating hop | No | Per hop | No | No | Connection-level only |
| Certificate-bound (RFC 8705) or DPoP (RFC 9449) tokens | Adds proof of possession | No | Partial (DPoP binds method + URI, not body) | No | No | Yes |
| Asymmetric request signing (JWS / RFC 9421) | Yes | No | Yes | Yes, with sender-only key custody | No | Yes, if timestamp/nonce covered |
| JWE (full or field-level) | No | No | Of ciphertext, not of sender | No | Yes | No |
| Tokenization / vaulting | No | No | No | No | Yes, by not holding the data | No |
| IP allowlisting / private connectivity | Network-level only | No | No | No | No | Reduces exposure |

Two rows deserve a second look.

**JWE gives integrity of the ciphertext, not authenticity of the sender.** JWE uses authenticated encryption, so tampering with the ciphertext is detected. But anyone holding your public key can produce a perfectly valid JWE addressed to you. Encryption without a signature tells you nobody changed the message. It doesn't tell you who wrote it.

**`private_key_jwt` strengthens client authentication at the token endpoint, but the access token it produces is still a bearer token.**

# Conclusion
What this means before you write a single line of spec
- **If the security section of your API spec is "TLS 1.2+ and an API key", you have authentication by possession and nothing else.**
- **Adding JWT access tokens without fixing client authentication and sender-constraint mostly adds a network hop.** You gain scopes and expiry. You don't gain resistance to the most common attack, which is a leaked credential.
- **If disputes are a real business risk (payouts, disbursements, standing instructions, anything that moves money on instruction), you need asymmetric request signing, plus the key-custody discipline that makes it mean something.**
- **If you're about to mandate payload encryption, first ask whether you need to receive that data at all.** The cheapest field to protect is the one you never collect.