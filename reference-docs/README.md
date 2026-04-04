# Retail Agentic Commerce — Reference Documentation

> Comprehensive architecture, design, and functional documentation for the Agentic Commerce Protocol (ACP) reference implementation.

## Document Index

| Document | Audience | Description |
|----------|----------|-------------|
| [High-Level Design (HLD)](high-level-design.md) | All stakeholders | System overview, capabilities, deployment topology, and integration landscape |
| [Low-Level Design (LLD)](low-level-design.md) | Engineers, Architects | Detailed component internals, data models, algorithms, and API contracts |
| [Architecture Decision Records (ADR)](architecture-decision-records.md) | Architects, Tech Leads | Key design choices, rationale, trade-offs, and alternatives considered |
| [C4 and TOGAF Diagrams](c4-togaf-diagrams.md) | Architects, Program Managers | Context, Container, Component, and Logical diagrams in Mermaid format |
| [Functional Documentation](functional-documentation.md) | Business Analysts, PMs, QA | Business logic, user journeys, and workflow descriptions in plain language |
| [Technical Documentation](technical-documentation.md) | Engineers, DevOps, SREs | Service internals, configuration, deployment, and operational runbooks |

## How to Read This Documentation

- **Non-technical stakeholders**: Start with the [High-Level Design](high-level-design.md), then read the [Functional Documentation](functional-documentation.md) for business logic.
- **Solution architects**: Read [HLD](high-level-design.md) then [C4 and TOGAF Diagrams](c4-togaf-diagrams.md) and [ADRs](architecture-decision-records.md).
- **Engineers / Contributors**: Start with [LLD](low-level-design.md) and [Technical Documentation](technical-documentation.md).
- **DevOps / SREs**: Focus on the deployment sections in [Technical Documentation](technical-documentation.md).

## System at a Glance

The Retail Agentic Commerce platform is a reference implementation demonstrating how AI agents can participate in e-commerce checkout flows using standardized protocols. It features a three-panel UI (Client Agent, Merchant Server, Agent Activity), four specialized AI agents (Promotion, Recommendation, Search, Post-Purchase), and dual-protocol support (ACP REST and UCP A2A JSON-RPC).

```
 Client Agent (UI)               Merchant Server               AI Agents
 ┌────────────────┐             ┌──────────────────┐          ┌──────────────┐
 │ Next.js 15     │  ◄─ ACP ──►│ FastAPI + SQLite  │◄─ NAT ─►│ Promotion    │
 │ React 19       │  ◄─ UCP ──►│ Dual Protocol     │          │ Recommendation│
 │ Kaizen UI      │             │ Engine            │          │ Search       │
 └────────────────┘             └──────────────────┘          │ Post-Purchase│
                                        │                     └──────────────┘
                                        ▼
                                ┌──────────────────┐
                                │ PSP (Payment)    │
                                │ Vault + Intents  │
                                └──────────────────┘
```
