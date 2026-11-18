# Everna Platform

**The business system of [Everna](https://everna.lat/en), a solar-installation company in Costa Rica: quoting, engineering design, drone-based site measurement and client portal, built end to end by one engineer.**

> This repository is a **case study**. The source code is private; this page documents what the system does, how it is built and how it is verified.
> Everna is co-founded by three engineers. I am the co-founder who designed and built the software.

---

## What it is

Everna designs and installs photovoltaic systems for mid-size commercial and industrial customers. Selling that kind of system needs sizing, tariff analysis, electrical design, drawings and a defensible price, normally spread across spreadsheets and CAD files.

The platform puts that whole process behind one system:

- a **quoting and design engine** that turns a customer's demand and roof geometry into a sized system, a price, a client PDF, DXF drawings and a calculation memo;
- a **public website** and **internal management panel** for leads, quotes, pricing and roadmap;
- a **client portal** for system monitoring;
- a **drone and photogrammetry pipeline** that measures roofs from photos;
- an **agent layer** (custom agents, reusable skills and a Slack-connected agent) used to operate and extend the system.

## Numbers (counted from the repository)

| | |
|---|---|
| Code | ~175k lines of Python, TypeScript and SQL |
| Automated tests | 5,000+ test functions across engine, API and drone tooling, plus Playwright end-to-end |
| API | 24 routers, ~174 route handlers |
| Database | 36 tables, 101 migrations, row-level security with three roles |
| History | 1,400+ commits between May and November 2026 |

## Architecture

<p align="center"><img src="docs/architecture.svg" alt="Architecture: the public site and the platform call the FastAPI routers; the routers use Supabase and the quoting and design engine, which feeds PDF and DXF generation; the drone pipeline and the workers also write to Supabase." width="560"></p>

<details><summary>Diagram source (Mermaid)</summary>

```mermaid
flowchart LR
  subgraph Client
    W[Public site<br/>Next.js 16 · React 19]
    P[Platform<br/>/gestion · /socios · /panel]
  end
  subgraph API["API (FastAPI · Railway)"]
    R[Routers + auth]
    PDF[PDF / DXF generation]
    WK[Workers<br/>reminders · retention · inverter polling]
  end
  E[Quoting & design engine<br/>deterministic, staged]
  DB[(Supabase<br/>Postgres · RLS · Auth · Storage)]
  D[Drone pipeline<br/>WebODM · OpenCV · Blender]
  W --> R
  P --> R
  R --> DB
  R --> E
  E --> PDF
  D --> DB
  WK --> DB
```

</details>

| Layer | Stack |
|---|---|
| Web | Next.js 16 (App Router), React 19, TypeScript, Tailwind v4, react-three-fiber for 3D site views |
| API | Python 3.12, FastAPI, Pydantic v2, WeasyPrint, ezdxf, shapely |
| Data | Supabase (Postgres, Auth, private Storage with signed URLs), audited migrations |
| Delivery | Docker, Vercel (web), Railway (API, infrastructure as code), GitHub |
| Geoprocessing | WebODM, OpenCV, scipy, shapely, Blender (headless) |

## The engine

The core of the system is a **deterministic, staged calculation engine**: demand → sizing → conductors → conduit → losses → battery → tariff and billing → pricing → surplus. Every result can be traced and reproduced, which makes the numbers auditable end to end.

- Sizing follows **NEC 2020** and local tariff structures; yield uses **PVGIS** data.
- Outputs: client PDF, DXF drawing sheets, calculation memo and partner report.
- Verified by **golden cases** (frozen catalog and solar data), **Python/JavaScript parity tests** for the quick calculator and **monotonicity tests** (more load never produces a smaller system).

## Drone and photogrammetry

Mission planning for a consumer drone, photo processing with WebODM, georeferencing, roof and neighbour extraction with OpenCV and shapely, mesh simplification with Blender, and publication of orthophoto, height map and 3D model into the design workflow. The same tooling was extended into a rule-based lot-subdivision tool with earthwork estimation.

## Engineering practices

- **Test-gated delivery:** changes reach `main` only through pull requests with a green `verify` run; code owners review every change.
- **Tiered local verification:** `verify` chooses what to run (unit tests, Docker image checks, Playwright, smoke tests) from the files that changed, with quotas shared across parallel worktrees.
- **Security:** row-level security with three roles, private storage with short-lived signed URLs, secret scanning (gitleaks), personal access tokens without DDL access, a weekly security-posture workflow, a pre-push hook that blocks direct pushes to `main`.
- **Data safety:** transactional migrations with a prior backup and a `schema_migrations` ledger; a disposable Postgres to test RLS.
- **Documentation as code:** a plan with status and hand-off for each feature, and a checker that keeps docs consistent with the code.

## AI-native engineering capabilities

The system was built with an agent-first workflow, which required these capabilities:

- **Agent orchestration:** specialist agents (quoting, finance, orchestration) working in parallel isolated worktrees, one pull request per change, with written plans that carry status and hand-off between sessions.
- **Context engineering:** versioned repository rules, a sourced domain knowledge base and a single source of truth for catalogs, so agents act on facts instead of guesses.
- **Reusable skills and tool design:** packaged procedures for drone route planning, photogrammetry processing, quote reporting and data integration.
- **Agent operations:** a Slack-connected agent with identity-based permissions: full environment for the engineer, read-only for everyone else.
- **Guardrails and safety:** permission rules, protected branches, a pre-push hook, secret scanning, least-privilege tokens and audit logging.
- **Verification of generated work:** golden cases, parity and monotonicity tests, end-to-end and visual regression checks; tests make the final call on every change.
- **Architecture judgment:** deciding where deterministic, auditable logic is the right tool and where automation or agents add leverage.

## Links

- Company site: <https://everna.lat/en>
- Author: [Andrey Quesada](https://github.com/andreyques41)

---

© Everna. All rights reserved. No source code is published in this repository.
