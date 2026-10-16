<!--
Sync Impact Report:
- Version change: 2.0.0 → 3.0.0
- Modified principles:
  - I. Strict Type Safety & Runtime Validation (100% TypeScript + Zod) → I. Strict Type Safety & Zod Network Contract Validation
  - II. Hybrid System 1 (Jev) & System 2 (Lightweight LLM) Architecture (NON-NEGOTIABLE) → II. Deterministic Decision Rules with Jev in `bot-orchestrator` (NON-NEGOTIABLE)
  - III. Zero Hallucination, Strict Confidence Gate & Contract-Driven Domain → III. Strict HTTP/REST Inter-Service Communication & Zero Hallucination
  - IV. Decoupled Adapter Architecture & Canonical Folder Structure → IV. Modular Monorepo Architecture (`apps/*`) & Long-Running Daemons
  - V. Test-First Critical Decision Flows (Vitest + Fastify Inject) → V. Isolated Workspace Testing (Vitest + Network Mocking) & Unified Dev Workflow
- Added sections:
  - Monorepo Service Topology (`apps/bot-orchestrator`, `apps/whatsapp-gateway`, `apps/barbershop-mock`)
- Removed sections:
  - Single-process `src/routes`, `src/services`, `src/domain`, `src/config` layout and in-process `MockBarbershopClient` (replaced by 3 independent Fastify HTTP daemons in `apps/` communicating strictly over HTTP/REST)
- Follow-up TODOs: None
-->
# novoBot Constitution

## Core Principles

### I. Strict Type Safety & Zod Network Contract Validation
All source code across every workspace MUST be written in 100% pure TypeScript with `strict: true` and `noImplicitAny: true` enabled. The use of `any` is strictly prohibited. All external and inter-service payload contracts (`bot-orchestrator` ↔ `whatsapp-gateway`, `bot-orchestrator` ↔ `barbershop-mock`, incoming webhooks, environment variables, and **Jev** schemas) MUST be strictly validated at runtime using **Zod**.
*Rationale*: In a distributed multi-service topology, compile-time TypeScript combined with runtime Zod boundary validation prevents contract drift and unhandled serialization failures between HTTP services.

### II. Deterministic Decision Rules with Jev in `bot-orchestrator` (NON-NEGOTIABLE)
All intent classification and structured routing inside `apps/bot-orchestrator` MUST be executed exclusively via **Jev (TypeSafe SDK)** as the System 1 decision engine:
- **No Generative LLM for Routing/Parsing**: Using a generative LLM to decide routes, classify intents, or parse structured entities that Jev can resolve is strictly prohibited.
- **Strictly Typed Intent Enums**: Intent classification MUST strictly use the typed enum: `AGENDAR`, `CANCELAR`, `VER_HORARIOS`, `TABELA_PRECOS`, `LOCALIZACAO`, `ATENDIMENTO_HUMANO`, `OUTRO`.
- **Confidence Threshold (`< 0.70`)**: If the Jev confidence score is below `0.70`, the orchestrator MUST immediately execute a deterministic fallback to human handoff (`ATENDIMENTO_HUMANO`).
- **Restricted Generative LLM Invocation (System 2)**: A generative LLM MAY ONLY be invoked when the classified intent is explicitly `OUTRO` or an open-ended question not covered by deterministic flows.
*Rationale*: Enforcing Jev for all System 1 decisions guarantees deterministic routing, predictable latency, and zero hallucination in transactional barbershop workflows.

### III. Strict HTTP/REST Inter-Service Communication & Zero Hallucination
Communication between services MUST occur strictly over the network via **HTTP/REST**. Direct cross-application code imports of runtime state or in-process memory sharing between `apps/*` are prohibited. `apps/bot-orchestrator` MUST communicate with `apps/whatsapp-gateway` and `apps/barbershop-mock` exclusively via base URLs configured through environment variables (`.env`). The orchestrator MUST NEVER fabricate services, prices, time slots, or reservations; all domain data MUST come from HTTP responses returned by `apps/barbershop-mock`.
*Rationale*: Enforcing HTTP/REST boundaries with environment-configured URLs ensures each service is genuinely decoupled and can be deployed, scaled, or replaced by a production backend independently.

### IV. Modular Monorepo Architecture (`apps/*`) & Long-Running Daemons
The repository MUST be structured as a modular **Monorepo** managed via **npm workspaces** or **pnpm workspaces**, containing three independent Node.js applications under `apps/`:
- `apps/bot-orchestrator`: The brain of the system. Receives webhook events, executes deterministic decisions via **Jev (TypeSafe SDK)**, orchestrates business data via HTTP calls to `barbershop-mock`, and dispatches outbound messages via `whatsapp-gateway`.
- `apps/whatsapp-gateway`: Isolated service encapsulating the community WhatsApp Web API integration (inbound event receiver and outbound message sender).
- `apps/barbershop-mock`: Independent HTTP REST API simulating the real barbershop backend (services, pricing table, available time slots, and appointment reservations).
Fullstack frameworks and Serverless/Vercel dependencies are strictly prohibited; all three applications MUST run as pure TypeScript long-running Node.js HTTP daemons powered by **Fastify**.
*Rationale*: Separating the orchestrator, WhatsApp transport gateway, and barbershop domain API into standalone Fastify daemons mirrors real-world microservice boundaries while keeping development unified in a single repository.

### V. Isolated Workspace Testing (Vitest + Network Mocking) & Unified Dev Workflow
Each application in `apps/*` MUST maintain strict dependency isolation in its own `package.json` and include its own **Vitest** test suite covering both unit tests and HTTP integration tests (using Fastify injection and mocked outbound network calls). The monorepo root MUST provide centralized scripts (e.g., `npm run dev` / `pnpm dev`) to launch and run all three services simultaneously during local development.
*Rationale*: Per-app test suites with network mocking allow each service to be verified in isolation without requiring live external processes during unit/integration test runs, while root scripts keep local multi-service development frictionless.

## Technology Stack & Constraints

- **Repository Topology**: Modular Monorepo (`npm workspaces` or `pnpm workspaces`) with `apps/bot-orchestrator`, `apps/whatsapp-gateway`, and `apps/barbershop-mock`.
- **Runtime & Process Model**: Node.js 20+ long-running HTTP daemons (zero Serverless/Vercel or fullstack framework dependencies).
- **Language**: Pure TypeScript (`strict: true`, `noImplicitAny: true`; `any` is strictly prohibited).
- **HTTP Framework**: **Fastify** across all three HTTP services.
- **Validation**: **Zod** for all HTTP request/response payloads, inter-service contracts, Jev schemas, and `.env` configurations.
- **Inter-Service Protocol**: HTTP/REST using service URLs injected via `.env` variables.
- **Decision Engine (`bot-orchestrator`)**: **Jev** (TypeSafe SDK) for System 1 classification; generative LLM restricted exclusively to `OUTRO` / open-ended queries.
- **Testing**: **Vitest** in every app for unit and integration testing with mocked network calls.

## Development Workflow & Quality Gates

1. **Workspace Dependency Isolation Gate**: Each app under `apps/*` MUST declare its own dependencies explicitly; no app may import internal implementation files from another app's directory.
2. **Zero `any` & Strict TypeScript Gate**: All workspaces MUST pass `tsc --noEmit` with `strict: true` and `noImplicitAny: true`.
3. **Zod Contract Gate**: Every HTTP endpoint and inter-service client call MUST validate request and response payloads through Zod schemas.
4. **Jev Determinism & Confidence Gate**: Tests in `apps/bot-orchestrator` MUST verify that Jev handles all intent routing (`AGENDAR`, `CANCELAR`, `VER_HORARIOS`, `TABELA_PRECOS`, `LOCALIZACAO`, `ATENDIMENTO_HUMANO`, `OUTRO`), that scores `< 0.70` immediately trigger human handoff, and that the generative LLM is only called for `OUTRO`/open questions.
5. **Centralized Root Scripts & Semantic Commits**: The root `package.json` MUST maintain working scripts to run all apps concurrently in development (`dev`), build (`build`), and test (`test`). All Git commits MUST follow Semantic Commits (`feat:`, `fix:`, `docs:`, `test:`, `refactor:`, `chore:`).

## Governance

This Constitution is the supreme architectural and engineering reference for `novoBot`.
- **Compliance Review**: Every Spec Kit artifact (`spec.md`, `plan.md`, `tasks.md`) and implementation MUST respect the 3-app monorepo boundaries (`apps/bot-orchestrator`, `apps/whatsapp-gateway`, `apps/barbershop-mock`), HTTP/REST communication rules, and Jev decision constraints.
- **Amendment Procedure**: Changes to service boundaries, intent enums, confidence thresholds, or stack rules require explicit user approval, a Sync Impact Report update, and a semantic version bump in `.specify/memory/constitution.md`.
- **Versioning Policy**: Semantic Versioning (`MAJOR.MINOR.PATCH`):
  - `MAJOR`: Changes to monorepo service topology, inter-service protocols, or non-negotiable Jev decision rules.
  - `MINOR`: New principles, new intent enums, or additional workspace quality gates.
  - `PATCH`: Wording clarifications, typo fixes, or non-semantic refinements.

**Version**: 3.0.0 | **Ratified**: 2026-10-15 | **Last Amended**: 2026-10-15
