# Reference Architecture — Diagram

Companion to [Framework 01](../frameworks/01-reference-architecture.md). The diagram is Mermaid; GitHub renders it automatically. Edit the text to change the diagram.

## Logical view

```mermaid
flowchart TB
    subgraph USERS["Layer 1 — Identity & access"]
        U1["Trust officer"]
        U2["Relationship manager"]
        U3["Compliance / risk"]
        SSO["Firm SSO / directory<br/>(roles & entitlements)"]
        U1 & U2 & U3 --> SSO
    end

    subgraph FIRM["Firm-controlled tenant (own cloud account or dedicated vendor environment)"]
        direction TB

        subgraph ORCH["Layer 3 — Orchestration"]
            GW["AI gateway<br/>auth check · prompt assembly · guardrails · logging"]
            PI["Prompt-injection &<br/>PII / PHI filters"]
            GW --- PI
        end

        subgraph DATA["Layer 2 — Data boundary"]
            CLS["Classification<br/>public / internal / client-confidential / restricted"]
            IDX["Retrieval index<br/>(entitlement-aware)"]
            SRC["Systems of record<br/>trust accounting · document mgmt · CRM"]
            SRC --> CLS --> IDX
        end

        subgraph MODEL["Layer 3 — Model"]
            LLM["Managed model endpoint<br/>zero retention · region-pinned · no training on firm data"]
        end

        subgraph REVIEW["Layer 4 — Human review & workflow"]
            DRAFT["Draft labelled<br/>'AI-generated'"]
            DECIDE["Named decision-maker<br/>records decision + reasoning"]
            BLOCK["Prohibited use cases<br/>blocked here"]
        end

        subgraph AUDIT["Layer 5 — Logging, monitoring & audit"]
            LOG["Immutable request log<br/>who · when · docs retrieved · model · output · action"]
            SAMPLE["Monthly sampling review"]
            VEND["Vendor attestations<br/>SOC 2 · pen test · DPA"]
            LOG --> SAMPLE
        end
    end

    SSO -->|"authenticated request<br/>carries user identity"| GW
    GW -->|"retrieve only what<br/>this user may see"| IDX
    IDX -->|"entitled documents"| GW
    GW -->|"prompt + context"| LLM
    LLM -->|"response"| GW
    GW --> DRAFT
    GW --> BLOCK
    DRAFT -->|"Tier 1"| U1
    DRAFT -->|"Tier 2"| DECIDE
    DECIDE --> SRC
    GW -.->|"every request"| LOG
    DECIDE -.-> LOG

    classDef boundary stroke-dasharray: 5 5
    class FIRM boundary
```

## Reading the diagram

- **Everything inside the dashed box is the firm's.** Even if a vendor operates it, the tenant is isolated and the logs are the firm's.
- **The model never talks to users directly.** All traffic goes through the gateway, which is where entitlements, guardrails and logging live.
- **Retrieval is entitlement-aware.** The index knows which user may see which document; the gateway asks on the user's behalf.
- **Two exits from the gateway:** a labelled draft (Tier 1), or a draft that goes to a named decision-maker who writes the decision back to the system of record (Tier 2). Prohibited cases are stopped at the gateway.
- **Logs are dotted lines** because they are a side effect of everything, not a step in the flow.

## Deployment variants

The logical view is the same in each; only who operates the boxes changes.

| Variant | Who runs Layers 2–3 | Typical fit |
| --- | --- | --- |
| **Own cloud** | Firm's IT, using a cloud provider's managed model service inside the firm's account | Firms with an existing cloud footprint and at least one engineer |
| **Dedicated vendor tenant** | Vendor, in an environment isolated per customer | Most trust companies; requires strong contract terms on data, logs and model training |
| **Hybrid** | Firm keeps data boundary and logs; vendor provides orchestration + model | Firms that want control of data without building a platform |

Shared-tenant SaaS where client data is commingled with other customers' data is **not** a variant of this architecture.
