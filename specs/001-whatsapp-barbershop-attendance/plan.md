# Implementation Plan: Intelligent WhatsApp Barbershop Attendance Monorepo (3 Services)

**Branch**: `001-whatsapp-barbershop-attendance` | **Date**: 2026-10-15 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/001-whatsapp-barbershop-attendance/spec.md`

## Summary

Build a modular TypeScript Monorepo containing 3 independent Fastify HTTP daemons under `apps/` (`apps/barbershop-mock` on port `3001`, `apps/whatsapp-gateway` on port `3002`, and `apps/bot-orchestrator` on port `3000`) communicating strictly via HTTP/REST validated with **Zod**. `apps/bot-orchestrator` uses **Jev (TypeSafe SDK)** exclusively for System 1 intent classification (`AGENDAR`, `CANCELAR`, `VER_HORARIOS`, `TABELA_PRECOS`, `LOCALIZACAO`, `ATENDIMENTO_HUMANO`, `OUTRO`) with a strict `0.70` confidence threshold gate, orchestrates schedule and catalog operations against `apps/barbershop-mock`, invokes a lightweight System 2 LLM only for `OUTRO` queries, and dispatches replies via `apps/whatsapp-gateway`.

## Technical Context

**Language/Version**: Node.js 20+ LTS, pure TypeScript (`strict: true`, `noImplicitAny: true`, zero `any`)

**Primary Dependencies**: Fastify (HTTP server in all 3 apps), Zod (runtime validation for all inputs/outputs/contracts), Jev TypeSafe SDK (System 1 classification in `bot-orchestrator`), OpenAI/Anthropic lightweight HTTP client (System 2 for `OUTRO` intent in `bot-orchestrator`)

**Storage**: In-memory domain store (`MockBarbershopStore`) inside `apps/barbershop-mock`

**Testing**: Vitest in each app (`fastify.inject()` for HTTP routes + mocked `fetch` for inter-service calls)

**Target Platform**: Long-running Node.js HTTP daemons (no Serverless/Vercel dependencies)

**Project Type**: Modular Monorepo (`npm workspaces` / `pnpm workspaces`) with 3 HTTP microservices

**Performance Goals**: `< 50ms` p95 for deterministic System 1 + mock ERP queries; `< 3s` end-to-end webhook response

**Constraints**: Strict HTTP/REST inter-service communication via `.env` URLs; Jev confidence `< 0.70` immediately triggers human handoff; generative LLM forbidden for routing/parsing

**Scale/Scope**: 3 independent Fastify apps, 7 HTTP endpoints total, 7 strict intent enums, full unit and integration test suites per app

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

- **[PASS] I. Strict Type Safety & Zod Network Contract Validation**: All 3 apps enforce `strict: true` and `noImplicitAny: true` in `tsconfig.json` with zero `any`, and validate every request/response and `.env` config with **Zod**.
- **[PASS] II. Deterministic Decision Rules with Jev in `bot-orchestrator` (NON-NEGOTIABLE)**: System 1 classification uses **Jev** exclusively across the 7 strict enums (`AGENDAR`, `CANCELAR`, `VER_HORARIOS`, `TABELA_PRECOS`, `LOCALIZACAO`, `ATENDIMENTO_HUMANO`, `OUTRO`), enforces `< 0.70` fallback to human handoff, and restricts System 2 LLM strictly to `OUTRO`.
- **[PASS] III. Strict HTTP/REST Inter-Service Communication & Zero Hallucination**: No cross-app runtime imports; `bot-orchestrator` calls `http://localhost:3001` (`barbershop-mock`) and `http://localhost:3002` (`whatsapp-gateway`) over HTTP using `.env` URLs.
- **[PASS] IV. Modular Monorepo Architecture (`apps/*`) & Long-Running Daemons**: Repository uses workspaces under `apps/barbershop-mock`, `apps/whatsapp-gateway`, and `apps/bot-orchestrator` running as Fastify Node.js daemons.
- **[PASS] V. Isolated Workspace Testing (Vitest + Network Mocking) & Unified Dev Workflow**: Each app has isolated dependencies, its own Vitest suite with mocked network calls, and centralized root scripts (`npm run dev`, `npm test`, `npm run build`).

## Project Structure

### Documentation (this feature)

```text
specs/001-whatsapp-barbershop-attendance/
├── plan.md              # This file (/speckit-plan command output)
├── research.md          # Phase 0 output (/speckit-plan command)
├── data-model.md        # Phase 1 output (/speckit-plan command)
├── quickstart.md        # Phase 1 output (/speckit-plan command)
├── contracts/           # Phase 1 output (/speckit-plan command)
│   └── http-contracts.md
└── tasks.md             # Phase 2 output (/speckit-tasks command)
```

### Source Code (repository root)

```text
package.json             # Root workspace configuration & scripts (dev, build, test, typecheck)
tsconfig.base.json       # Shared strict TypeScript compiler configuration (strict: true, noImplicitAny: true)
.env.example             # Default ports & service URLs (3000, 3001, 3002)

apps/
├── barbershop-mock/     # Port 3001 - Independent Barbershop ERP Mock REST API
│   ├── package.json
│   ├── tsconfig.json
│   ├── src/
│   │   ├── config/
│   │   │   └── env.ts
│   │   ├── domain/
│   │   │   ├── schemas.ts
│   │   │   └── store.ts
│   │   ├── routes/
│   │   │   └── barbershop.routes.ts
│   │   ├── app.ts
│   │   └── server.ts
│   └── tests/
│       └── barbershop.routes.test.ts
│
├── whatsapp-gateway/    # Port 3002 - Isolated WhatsApp Web HTTP Adapter & Webhook Emitter
│   ├── package.json
│   ├── tsconfig.json
│   ├── src/
│   │   ├── config/
│   │   │   └── env.ts
│   │   ├── domain/
│   │   │   └── schemas.ts
│   │   ├── services/
│   │   │   ├── whatsapp-client.ts
│   │   │   └── webhook-emitter.ts
│   │   ├── routes/
│   │   │   └── gateway.routes.ts
│   │   ├── app.ts
│   │   └── server.ts
│   └── tests/
│       └── gateway.routes.test.ts
│
└── bot-orchestrator/    # Port 3000 - Intelligent Orchestrator (Jev System 1 + System 2 LLM)
    ├── package.json
    ├── tsconfig.json
    ├── src/
    │   ├── config/
    │   │   └── env.ts
    │   ├── domain/
    │   │   └── schemas.ts
    │   ├── services/
    │   │   ├── jev-classifier.ts
    │   │   ├── barbershop-http.client.ts
    │   │   ├── whatsapp-http.client.ts
    │   │   ├── llm-contextual.service.ts
    │   │   └── orchestrator.service.ts
    │   ├── routes/
    │   │   └── webhook.routes.ts
    │   ├── app.ts
    │   └── server.ts
    └── tests/
        ├── jev-classifier.test.ts
        └── webhook.routes.test.ts
```

**Structure Decision**: Modular Monorepo (`apps/*`) with 3 completely independent Fastify applications (`apps/barbershop-mock`, `apps/whatsapp-gateway`, `apps/bot-orchestrator`), each containing its own `src/config`, `src/domain`, `src/services` (where applicable), `src/routes`, and `tests/` directories in strict compliance with Constitution v3.0.0.

## Complexity Tracking

> No Constitution Check violations. All 5 Core Principles are satisfied directly by the 3-app monorepo design.
