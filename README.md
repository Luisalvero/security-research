# security-research

Technical writeups on systems I have built, broken, or studied. Each
subfolder is a self-contained report.

## Index

| Area | Target | Status | Skills |
|---|---|---|---|
| System analysis | [Zennvoi](./zennvoi-analysis/) | Complete (v1.0) | Architecture review, RLS-as-authorization, Stripe webhook security, threat modeling, defense-in-depth at the database layer |
| Reverse engineering | [reports/](./reverse-engineering/) | Planned | Static analysis, Ghidra |
| Embedded security | [reports/](./embedded-security/) | Planned | Telemetry, attack surface analysis |
| Practical labs | [reports/](./practical-security-labs/) | Planned | Web and network labs |

## About

Luis Alvero — Electrical and Computer Engineering at FIU. I build production
systems and write up how they work and where they break.

More work: [luisalvero.com](https://luisalvero.com) · [LinkedIn](https://www.linkedin.com/in/luis-alvero)

## Reading order

Start with [`zennvoi-analysis/report.md`](./zennvoi-analysis/report.md) — a
self-conducted security and architecture assessment of a production
invoicing SaaS I build and operate ([zennvoi.com](https://www.zennvoi.com)).
The companion threat model is at
[`zennvoi-analysis/threat-model.md`](./zennvoi-analysis/threat-model.md).

## Conventions

- Each subfolder contains a README describing scope and status.
- Reports are self-contained: prose, diagrams, and supporting artifacts in
  the same directory.
- Status values: Complete, In Progress, Planned, Archived.

## License

MIT — see [LICENSE](./LICENSE).
