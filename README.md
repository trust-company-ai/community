# Trust Company AI Frameworks

**Shared best practices and lessons learned for building AI infrastructure at trust companies.**

Trust companies face a specific combination of constraints when adopting AI: fiduciary duties to clients, confidentiality of client and beneficiary data, confidentiality that may include solicitor-client / attorney-client privilege, regulatory oversight (federal, state or provincial banking and financial-services regulators, plus anti-money-laundering obligations), and small technology teams. Most firms are solving the same problems independently. This repository exists so they don't have to.

## What's here

| Folder | What you'll find |
| --- | --- |
| [`frameworks/`](frameworks/) | Structured guidance for a specific topic — e.g. reference architecture, data governance, platform due diligence. Each is a living document. |
| [`architecture/`](architecture/) | Architecture diagrams and technical design notes, plus [worked examples](architecture/examples/) contributed by firms. Diagrams are written in [Mermaid](https://mermaid.js.org/) so they render directly on GitHub and can be edited as text. |
| [`lessons-learned/`](lessons-learned/) | Short, candid write-ups from firms that have built or bought something: what worked, what didn't, what they'd do differently. |
| [`templates/`](templates/) | Reusable documents — due-diligence checklists, policy outlines, worksheets — that firms can copy and adapt. |

A plain-English companion website is in preparation. This repository is the technical source of record.

## Start here

1. **[Reference Architecture for AI at a Trust Company](frameworks/01-reference-architecture.md)** — the core framework: the layers every deployment needs, where client data is allowed to flow, and the controls that sit between them.
2. **[Contributing](CONTRIBUTING.md)** — how to add a lesson learned, propose a change, or submit a template. No git experience required.

## Principles

These guide everything in this repository:

- **Fiduciary first.** An AI system that helps the firm but exposes a client is a failure, not a trade-off.
- **Client data never trains any model outside the firm.** Contractually, technically, and verifiably.
- **Humans decide; AI drafts.** AI output is an input to a professional's judgment, never the decision itself, for anything affecting a client or beneficiary.
- **Explainable enough for an examiner.** If you can't describe to a regulator what the system does, what data it saw, and who reviewed the output, it isn't ready.
- **Share the pattern, not the secret.** Contributors share architecture and process. Nobody is asked to share client data, proprietary code, or commercially sensitive detail.

## Who this is for

- Trust officers and fiduciary leaders evaluating AI
- Chief compliance / risk officers who need to sign off
- The (often very small) technology teams who have to actually build or integrate it

## Status

Early stage. The first framework and a first [worked example](architecture/examples/) — a Canadian trust company's platform in live use — are in; lessons learned and templates are being collected. Open an [issue](../../issues) to ask a question, correct something, or volunteer a contribution.

## License

[Apache License 2.0](LICENSE). Reuse freely, with attribution.
