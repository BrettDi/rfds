---
title: "Model Runtime Fingerprint & Registrable Identity"
number: "043"
author: "Brett Di Lello"
state: "idea"
tags: ["attestation", "cryptography", "inference", "verification"]
created: "2026-11-22"
---

## The Idea

A software mechanism surfacing a runtime fingerprint from a model and registering it as a known identity. The fingerprint is derived via black-box access and uses ordinary inference requests, with no access to weights or hardware. 

The key insight: the evidence of which model is running is produced by the model itself as it runs, and not from a claim made by the model, the host or attested hardware. A later check derives a fingerprint from an endpoint in the same way and compares it with the registration. See Current State and References.

## Why It Matters

Model Fingerprint Attestation (RFD 039) proves what's loaded now. Model Tenancy Ledger Attestation (RFD 040) proves what's ever been loaded. Both are from the inside out. A Model Runtime Fingerprint would prove which model is actually operating, from the outside in. It would establish a registered identity for a model in its generative state from the position of the endpoint consumer, and address key areas such as:

- **Security or safety claims that cannot be held to account.** Evaluations, compliance and approvals attach to a named model as a canonical reference. If the models identity in it's generative state cannot be established, any claim of the model tested being the one in use rests on trust alone, and those relying on it have no recourse.
- **Silent substitution.** An approved model can be swapped for another, quantised, or stripped of safety fine-tuning behind an unchanged name or endpoint, and the change surfaces only through its consequences.
- **Unattributable incidents.** When an action results in harm or adverse impact, establishing true cause and the attributed model is not possible from outside the hardware boundary, so investigation and accountability stall.
- **Controls with nothing to act on.** Approved lists and bans on unknown models only work where the model's identity can be established. At an endpoint without attested hardware they cannot be enforced.

## The Gap

Most of the RFDs within the collection are hardware focused. However, many models are consumed through endpoints where the checker has neither the hardware operator's cooperation nor attested hardware, whether that is a third-party host, a vendor's closed-weight model or a model served without attestation.

Existing AI Safety and Security categories recently provided by Ankur Pandey [AI Safety Tools document](https://docs.google.com/spreadsheets/d/e/2PACX-1vTxko20RW37hL3qRLdAQ19xWUKcUnCRv_tqc71MdCd1RzBOzegOLt-9PdS59SsDO2guRmYHV2_JbzxC/pubhtml) cover elements nearby, yet do not state the fundamental of capturing a model's runtime identity. Model signing proves a file is genuine, agent identity says which agent is acting, and hardware attestation says where something ran. None of them cover which model is actually answering behind an endpoint.

This RFD proposes a software-layer route to closing this gap in establishing a model's canonical identity in its generative state and contributing to the same objectives and outcomes of RFD 039 and 040.

## Architecture

```
┌──────────────────────────────────────────┐           ┌────────────────────────┐
│ VERIFIER, outside in (RFD 043)           │  look up  │ REGISTRAR              │
│ derives FP(E) from ordinary requests     │◄────┐     │ derives FP(M) from a   │
│ and compares it with the registered FP   │     │     │ known model's endpoint │
└──────────┬───────────────────┬───────────┘     │     └──────────┬─────────────┘
           │                   ▲                 │                │ register
           │ requests          │ observations    │                │
           ▼                   │                 │                ▼
┌──────────┴───────────────────┴───────────┐     │     ┌──────────┴─────────────┐
│ ENDPOINT UNDER TEST, generative state    │     └─────├ REGISTRY               │
└────────────────────┬─────────────────────┘           │ known fingerprints     │
                     ▲                                 │ provenance and lineage │
                     │ serves                          │ FP(M1) FP(M2) ...      │
                     │                                 │                        │
┌────────────────────┴─────────────────────┐           │                        │
│ HARDWARE, inside out (RFD 039 and 040)   │ known FPs │                        │
│ 039 attests what is loaded now           │◄┈┈┈┈┈┈┈┈┈┈├                        │
│ 040 records what has ever been loaded    │ envisioned│                        │
└──────────────────────────────────────────┘           └────────────────────────┘
```
Dotted lines show links that are envisioned and not yet demonstrated.

- Registration happens once per model and checks repeat as often as needed.
- Neither the Registrar nor Verifier has access to weights or hardware.
- The hardware layer is shown for context. The check does not require it, and it applies only where attested hardware is present.
- Registration is repeated when a registered model is updated or changed and each entry carries provenance metadata linking it to its predecessor, with registry retaining model lineage as well as identity.

## Protocol Sketch

```
REGISTRATION (once per model, repeated when the model is updated)
1. REGISTRAR obtains access to a known model M through its inference endpoint
2. REGISTRAR sends ordinary inference requests and records what it observes
3. REGISTRAR derives fingerprint FP(M) and stores it in the REGISTRY against
   a model identifier, with provenance and a link to its predecessor

CHECK (outside in, repeated as often as needed)
4. VERIFIER sends the same kind of requests to the endpoint under test,
   with black-box access only
5. VERIFIER derives FP(E) from its observations
6. VERIFIER looks up the claimed model in the REGISTRY and compares FP(E) with FP(M)
7. VERIFIER reports
   - match to the claimed model
   - mismatch against the claimed model
   - no registered model matches

CROSS-CHECK (inside out, envisioned and not yet demonstrated)
8. Where hardware attestation is present, the result is set against the model
   that 039 attests as loaded, and the REGISTRY supplies the known fingerprints
   that 039 attests against and the approved list that 040 draws on
9. Agreement strengthens the claim, and disagreement flags it for investigation
```

The abstract fingerprint derivation within this sketch has to satisfy the following requirements.

| Property | Requirement |
| --- | --- |
| **Canonical** | One agreed form per model, so independent parties register and check against the same reference |
| **Black-box obtainable** | Derived from ordinary inference requests alone |
| **Stable** | The same model produces a matching fingerprint |
| **Model-dependent** | Different models produce unique fingerprints |
| **Registrable** | Can be stored and later compared independently without the model being present |

## Current State

The evidence is existence of a runtime fingerprint on open-weight models and their extraction as a canonical artefact. Open-weight models were used because they could be accessed and registered to replicate interrogation in the same way for any model served behind an endpoint, with the method not dependent on the weights being open. See ["Open-Weight Models Carry a Runtime Fingerprint: Evidence for a Registrable Identity"](https://www.lesswrong.com/posts/7DJGoG7FJwDT5Jb9z/open-weight-models-carry-a-runtime-fingerprint-evidence-for)

It has not yet been replicated on independently captured sessions, other inference engines, tensor-parallel layouts, batching loads or other hardware, but demonstrates that such a fingerprint exists and can be extracted. Of which, Independent validation and relevant expertise would establish it as a viable gateway for identification and registration of deployed models.

The wider proposal goes beyond this evidence. Closed-weight endpoints, open community services, attestation cross checks, baseline infrastructure and alike are areas envisaged, but yet to be cost measured, demonstrated or formalised.

## Open Questions

- What does the fingerprint capture beyond architecture, size and quantisation, and how well do closely related variants separate?
- Does it carry over from open-weight models to closed-weight vendor endpoints?
- What sits between the checker and the model on the network? Proxies, gateways, routers, latency, load, buffering and caches could mask or reroute what the check observes, so how is a result bound to a specific endpoint or registered infrastructure?
- How costly is it for a provider to imitate a registered fingerprint, and can a host tell check traffic from ordinary traffic and answer the two differently?
- How is a model first registered as known, and by whom? What establishes that the registered model is the genuine release, how does registration relate to model signing, and is there an accepted baseline environment with a traceable supply chain to deployed infrastructure that could also offer registration openly to the community?
- Should a black-box result be tied to hardware activity at the time of a request, such as the fingerprint 039 attests or 040 records for the device serving it? Can a verifier know which device served its request, where should trust be placed if the two disagree, and how should a registry identifier relate to the weights hash used in 039 and 040?
- What does a checkable identity reveal about the hosts and vendors being checked, and how does that sit with the privacy concerns in 040?
- What does a check cost in requests and time, and how often must it repeat to stay meaningful?
- Who runs a registry, and can an open community service backed by a transparent log such as Sigstore, sharing registrations across operators, provide the governance and security foundation?

## References

- Brett Di Lello, ["Open-Weight Models Carry a Runtime Fingerprint: Evidence for a Registrable Identity"](https://www.lesswrong.com/posts/7DJGoG7FJwDT5Jb9z/open-weight-models-carry-a-runtime-fingerprint-evidence-for) (LessWrong)
- RFD 039, Model Fingerprint Attestation
- RFD 040, Model Tenancy Ledger Attestation
- Davidson, Wofford and Sounder, ["The Missing Product Layers in AI Safety"](https://www.lesswrong.com/posts/hAozFAsKhsr9HhyGi/the-missing-product-layers-in-ai-safety) (LessWrong), citing Ankur Pandey's [AI Safety Tools document](https://docs.google.com/spreadsheets/d/e/2PACX-1vTxko20RW37hL3qRLdAQ19xWUKcUnCRv_tqc71MdCd1RzBOzegOLt-9PdS59SsDO2guRmYHV2_JbzxC/pubhtml).
