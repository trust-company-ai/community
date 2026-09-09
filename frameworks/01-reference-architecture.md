# Framework 01 — Reference Architecture for AI at a Trust Company

**Status:** Draft v0.1 · **Last updated:** 2026-09-05 · **Maintainer:** open

This document describes the layers a trust company's AI infrastructure needs, where client data is permitted to flow, and the controls that sit between layers. It is deliberately vendor-neutral: the same shape works whether you build on AWS, Azure, Google Cloud, or a vendor platform.

The full diagram is in [`architecture/reference-architecture.md`](../architecture/reference-architecture.md).

---

## 1. Why trust companies need their own reference architecture

Generic "enterprise AI" architectures assume the main risk is a bad answer. For a trust company the main risks are different:

| Risk | Why it matters more here |
| --- | --- |
| **Client data leaving the firm's control** | Fiduciary duty of confidentiality; GLBA safeguards; state privacy law. A leak is a breach of duty, not just a security incident. |
| **AI output being treated as a decision** | Discretionary fiduciary decisions (distributions, investment changes, beneficiary questions) must be made by a person who can be held accountable. |
| **Inability to explain to an examiner** | Examiners apply model-risk expectations to quantitative models (interagency guidance revised April 2026, OCC Bulletin 2026-13). That guidance explicitly leaves generative and agentic AI *out of scope* — meaning there is no supervisory roadmap yet, and firms must be able to explain their AI controls on their own terms. |
| **Third-party dependence** | Most firms will buy, not build. Interagency third-party risk guidance (OCC Bulletin 2023-17, June 2023) applies to AI vendors like any other critical vendor. |
| **Small teams** | A 3-person IT department cannot run a 12-layer MLOps stack. The architecture has to be operable by the people who actually work there. |

---

## 2. The five layers

Every deployment, from a single document-summarisation tool to a firm-wide assistant, maps onto these layers. If a layer is missing, that is a finding waiting to happen.

### Layer 1 — Identity & access
Who is allowed to use the system, and as whom.

- Single sign-on tied to the firm's existing directory. No separate AI logins.
- Role-based access that mirrors the firm's existing entitlements: a trust officer sees their accounts; a relationship manager sees theirs; nobody sees everything by default.
- Every AI request carries the identity of the human who made it, end to end.

### Layer 2 — Data boundary
What the AI is allowed to see, and where that data physically is.

- **Client data stays inside the firm's tenant.** Whether that is your own cloud account or a vendor's dedicated environment, it must be isolated from other customers and from the vendor's own training pipelines.
- **Retrieval, not training.** The model reads documents at query time (retrieval-augmented generation). Client data is never used to fine-tune or train a model — the firm's own or a vendor's.
- **Zero-retention inference.** Prompts and responses are not stored by the model provider beyond the request. Get this in the contract and verify it technically (e.g. provider's data-processing terms, logging configuration).
- **Classification before ingestion.** Documents are tagged (public / internal / client-confidential / restricted) before they reach any AI pipeline, and the pipeline enforces the tag.

### Layer 3 — Model & orchestration
The model(s) and the software that decides what to send them.

- Prefer models hosted in a region and environment the firm controls (e.g. a cloud provider's managed model service inside your own account) over public consumer endpoints.
- An orchestration layer sits between users and models. It assembles prompts, attaches only the documents the user is entitled to, applies guardrails, and records everything. Users never call a model directly.
- Model choice is a configuration setting, not an architectural commitment. You will change models.

### Layer 4 — Human review & workflow
Where AI output meets a decision.

- **Drafting use cases** (meeting summaries, first-draft letters, document summaries): AI output is clearly labelled as a draft and edited by a professional before use.
- **Decision-adjacent use cases** (distribution request analysis, investment policy review, beneficiary communications): AI output is one input; a named person records the decision and their reasoning in the system of record.
- **Prohibited use cases** are listed explicitly (see §4) and blocked at the orchestration layer, not just by policy.

### Layer 5 — Logging, monitoring & audit
Proof, for you and for the examiner.

- Every request: who, when, which documents were retrieved, which model, what came back, and what the human did with it.
- Logs live in the firm's environment, retained per the firm's records policy, and are searchable by account as well as by user.
- Periodic sampling review: a compliance or risk function reads a sample of interactions each month and records findings.
- Vendor attestations (SOC 2 Type II, penetration tests, data-processing agreements) collected and dated in the vendor file.

---

## 3. Controls between the layers

The layers are only useful because of what sits between them.

| Boundary | Control |
| --- | --- |
| User → Orchestration | Authentication, session logging, acceptable-use acknowledgement |
| Orchestration → Data | Entitlement check on every retrieval; classification enforcement; PII/PHI detection on outbound prompts |
| Orchestration → Model | Zero-retention endpoint; region pinning; prompt-injection filtering; output content filters |
| Model → User | "AI-generated draft" labelling; citation of retrieved sources; confidence / abstention where the model has nothing to cite |
| Everything → Logs | Immutable, append-only, in the firm's tenant |

---

## 4. Use-case tiers

Firms have found it useful to classify use cases into tiers and gate the architecture accordingly.

| Tier | Examples | Minimum requirements |
| --- | --- | --- |
| **Tier 0 — Internal, no client data** | Drafting internal policies, summarising public regulatory guidance, IT helpdesk | Layers 1, 3, 5 |
| **Tier 1 — Client data, drafting only** | Summarising a trust instrument for the officer, drafting a client letter, meeting notes | All five layers; output labelled as draft |
| **Tier 2 — Client data, decision-adjacent** | Analysing a discretionary distribution request against the instrument and history, flagging investment-policy drift | All five layers; named decision-maker; reasoning recorded; periodic model-risk review |
| **Prohibited (for now)** | Autonomous approval of distributions; unattended client communication; investment execution | Blocked at orchestration layer |

Tiers move over time. A firm's first deployment should be Tier 0 or Tier 1.

---

## 5. Build vs. buy — what the architecture tells you to ask

Whether you build in your own cloud or buy a vendor platform, the five layers give you the questions:

1. **Where does client data physically sit, and who else's data sits with it?** (Layer 2)
2. **Is any of our data used to train or improve any model? Show me the contract clause and the technical control.** (Layer 2)
3. **Can we see and export every log, and does it live in our environment?** (Layer 5)
4. **How is user entitlement enforced at retrieval time — not just at login?** (Layers 1–2)
5. **Can we switch the underlying model without a migration project?** (Layer 3)
6. **What is blocked, and where is the block enforced?** (Layer 4)

A vendor that cannot answer these six questions clearly is not ready for a trust company, regardless of how good the demo is.

---

## 6. What this framework does not cover (yet)

- Model risk management documentation templates — planned for `templates/`
- Data classification scheme — planned as Framework 02
- Internal due-diligence checklist for evaluating AI platforms — planned for `templates/`
- Incident response for AI-specific failures (hallucinated facts in client communications, prompt injection via uploaded documents)

Contributions welcome on any of these. See [CONTRIBUTING.md](../CONTRIBUTING.md).

---

## References

- OCC Bulletin 2026-13, *Model Risk Management: Revised Guidance* (interagency, April 17, 2026). Replaces OCC 2011-12 / SR 11-7; generative and agentic AI expressly out of scope.
- OCC Bulletin 2023-17, *Third-Party Relationships: Interagency Guidance on Risk Management* (June 2023)
- 12 CFR Part 9, *Fiduciary Activities of National Banks*
- Gramm-Leach-Bliley Act, Title V (privacy and safeguards)

*Regulatory references are provided for orientation, dated as of the "last updated" date above, and are not legal advice. Verify against current guidance and your own counsel.*
