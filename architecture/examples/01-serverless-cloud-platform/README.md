# Example 01 — AI Integration Platform (in production)

**Contributed by a Canadian trust company that designed, built and now operates this platform under Board governance.** It is in live use for cross-system estate-file review and risk identification, quarterly trust reporting, and parts of anti-money-laundering compliance. Shared as a *starting point*: alternate approaches and suggestions for improvement are welcome — open an [issue](../../../../../issues) or see [Contributing](../../../CONTRIBUTING.md).

Client identifiers have been removed. Original diagrams (prepared during the proof-of-concept phase, before the platform went into production): [architecture (PDF)](architecture-diagram.pdf) · [data flow (PDF)](data-flow-diagram.pdf). The Mermaid versions below are redrawn from them so they can be edited as text.

## What it does

Three use cases, all inside the company's own cloud account in a single in-country region. The firm treats the AI layer as just another regulated system: it sits inside the firm's own infrastructure, it logs everything, and it is never the last word on anything that leaves the building.

| # | Use case | Trigger | Uses an LLM? |
| --- | --- | --- | --- |
| 1 | **Cross-system estate-file review and risk identification** — staff ask a plain-language question and get a source-cited summary across practice management, document store, estate administration and accounting systems, including flags for items that need attention | On demand, by staff | Yes (inference only) |
| 2 | **Quarterly matter status & financial report drafts** — per-matter draft reports placed in a review folder | Scheduled | Yes (drafts only) |
| 3 | **Monthly AML / sanctions screening name export** — deterministic extraction of active client and party names for the existing screening tool | Scheduled, 1st of month | **No** |

## Design choices worth copying

- **Read-only everywhere.** Source systems are systems of record; nothing is written back. Existing user permissions are respected.
- **Data residency is explicit — including the one exception.** All compute, storage, secrets, logs and outputs stay in one in-country region. The *only* cross-border path is *cross-region inference* on the managed model service: when enabled, prompts and responses can transit another region even though nothing is stored there. Rather than pretending it did not exist, the firm treated it as a **named, documented exception**: transient, encrypted in transit, covered by the cloud provider's data-processing agreement and zero-data-retention / no-training terms, with formal risk acceptance recorded. The diagram marks it in amber so nobody has to go looking for it. If you tell clients or a regulator that nothing crosses the border, this is the detail that will get tested.
- **Staff use the sign-in they already have.** Company identity provider federated into the platform; MFA enforced; browser only, nothing installed.
- **Every request is audited.** Who asked what, when, which sources were read, what was generated. Outputs are stored immutably (versioning / object lock). When the regulator asks how a given output came to exist, the answer comes from the logs, not from memory.
- **Humans decide.** Report drafts are generated in region and land in a review folder for staff to verify and approve. Nothing is auto-sent to a client, ever. The AML export has a *reconciliation condition* — each run must reconcile line by line to its memo; if it does not, automation suspends and staff revert to the manual process — and a *human approval* step before upload.
- **No LLM where determinism matters.** The AML / sanctions export uses no LLM at all: deterministic extraction and rules, because a hallucinated or dropped name on a screening list is a regulatory problem, not a quality problem. Names and dates of birth never leave the country; national ID numbers are excluded from the export entirely. In the contributor's words: knowing where the model is *not* allowed to be turned out to be as important as knowing where it helps.
- **Secrets and least privilege.** All tokens and keys in a managed secrets store; least-privilege roles throughout.

## Architecture (redrawn)

```mermaid
flowchart TB
    STAFF["Trust company staff<br/>browser only, nothing installed"]
    IDP["Company identity provider<br/>MFA enforced"]
    STAFF -- sign-in --> IDP

    subgraph ACCT["Company cloud account — single in-country region"]
        WEB["Web app (static SPA + CDN)<br/>chat interface for matter search"]
        AUTH["Auth service<br/>SAML federation to company IdP"]
        API["API gateway<br/>authenticated calls only"]
        ORCH["Serverless orchestration<br/>read-only fan-out · assemble · call model · cite sources"]
        SCHED["Scheduler<br/>quarterly report drafts · monthly AML export"]
        SEC["Secrets manager<br/>tokens & keys · least-privilege roles"]
        AUDIT["Audit log<br/>who asked what, when, sources read, output"]
        OUT["Immutable output store<br/>versioning / object lock"]
        LLM["Managed LLM endpoint<br/>zero retention · no training"]
        WEB --> AUTH --> API --> ORCH
        ORCH --> SCHED
        ORCH <--> LLM
        ORCH -.-> AUDIT
        ORCH -.-> SEC
        ORCH --> OUT
    end

    XB["Cross-region inference — NAMED EXCEPTION<br/>transient · encrypted · DPA · zero retention · risk-accepted"]
    LLM <-.-> XB

    REVIEW["Review folders (in-country document store)<br/>HUMAN REVIEW — nothing client-facing without approval"]
    OUT --> REVIEW

    subgraph SRC["Source systems — read-only, nothing written back"]
        S1["Practice management"]
        S2["Office 365 / document store"]
        S3["Estate administration"]
        S4["Accounting"]
    end
    ORCH -- read-only --> SRC

    STAFF -- HTTPS --> WEB
    IDP -- SAML --> AUTH

    style XB fill:#fff4e0,stroke:#e08a00,stroke-dasharray: 5 5
    style ORCH fill:#e8f3e8
    style LLM fill:#e8f3e8
```

## Data flows (redrawn)

```mermaid
flowchart LR
    subgraph F1["Flow 1 — Global matter search (on demand)"]
        Q["Staff query<br/>SSO + MFA"] --> O1["Orchestration<br/>parallel read-only fan-out"] --> S["Source systems<br/>existing permissions respected"]
        O1 --> M["LLM synthesis<br/>(cross-border, inference only)"] --> A["Source-cited answer<br/>every fact tied to system + record"]
        O1 -.-> L1["Audit trail"]
    end
```

```mermaid
flowchart LR
    subgraph F2["Flow 2 — Quarterly report drafts (scheduled)"]
        T2["Scheduler"] --> J2["Job reads sources<br/>(read-only)"] --> D2["Draft reports<br/>(.docx / PDF)"] --> R2["Review folder<br/>(in-country)"] --> H2["HUMAN REVIEW GATE<br/>verify · correct · approve"]
    end
```

```mermaid
flowchart LR
    subgraph F3["Flow 3 — Monthly AML / sanctions name export (no LLM)"]
        T3["Scheduler<br/>1st of month"] --> J3["Extraction job<br/>active clients & parties"] --> RULES["Deterministic rules engine<br/>dedupe · exclusions with reasons · exception flags"] --> X3["Workbook<br/>Summary · Parties · Exclusions · Exceptions · Missing-DOB worklist"] --> ST["Immutable store + review folder"]
        ST --> GATE["RECONCILIATION CONDITION<br/>line-by-line vs run memo<br/>variance ⇒ suspend automation, revert to manual"] --> APPR["HUMAN APPROVAL"] --> SCR["Existing sanctions screener"]
    end
```

## Governance around the platform

The technology was the smaller part of the work. Alongside the build, the firm produced Board-level governance covering: privacy, cybersecurity, third-party risk, risk acceptance (including the cross-region exception above), data flows, incident response, business continuity, and its response to a review by its provincial financial-services regulator. The firm has offered to share genericised versions of these materials; they will be added under [`templates/`](../../../templates/) as they arrive.

## Advice from the contributor

- **Draw the one-page picture early.** The architecture and data-flow diagrams above were originally prepared for a supervisory review. Having a single page showing exactly where data sits and where it moves was by far the most useful artifact the firm produced, because it forces the confidentiality answer to be concrete.
- **Read-only was an easy sell.** Because nothing is ever written back to a system of record, the platform cannot corrupt a matter file even if something goes wrong.
- **Put a hard human review gate on anything with consequences,** and decide explicitly where the model is not allowed to operate at all.

## Open questions for peers

Things the contributing firm — and this forum — would like alternate views on:

1. Is an inference-only cross-border call acceptable to your regulator, or do you require in-country model hosting?
2. How do you handle retrieval permissions when the source system's entitlement model is coarser than the question being asked?
3. What evidence do examiners actually ask for from the audit trail — and does immutable storage of *outputs* suffice, or must prompts be retained too?
4. Would you put the AML export on a fully separate pipeline (different account / roles) from the LLM workloads?

Reply via an issue, or propose a change to this file.
