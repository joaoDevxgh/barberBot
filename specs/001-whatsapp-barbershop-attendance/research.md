# Phase 0 Research: Intelligent WhatsApp Barbershop Attendance Monorepo

**Feature**: `001-whatsapp-barbershop-attendance`  
**Date**: 2026-10-15

## 1. Monorepo Workspace Management & Concurrent Development

- **Decision**: Use **npm workspaces** (`"workspaces": ["apps/*"]` in root `package.json`) with `tsx` for TypeScript development execution and `concurrently` (or native npm workspace scripts) to start all 3 daemons (`barbershop-mock:3001`, `whatsapp-gateway:3002`, `bot-orchestrator:3000`) via `npm run dev`.
- **Rationale**: `npm workspaces` is built into Node.js 20+ out of the box without requiring global installation of external package managers, while maintaining strict per-app `package.json` isolation (`apps/barbershop-mock`, `apps/whatsapp-gateway`, `apps/bot-orchestrator`).
- **Alternatives Considered**:
  - *Turborepo / Nx*: Adds heavy build-cache orchestration overhead unnecessary for 3 focused Fastify microservices.
  - *Single Node process with internal modules*: Rejected because Constitution v3.0.0 explicitly mandates 3 independent Node.js daemons under `apps/*` communicating strictly over HTTP/REST.

## 2. HTTP Server Framework & Runtime Validation

- **Decision**: **Fastify v4/v5** paired with **Zod** (`zod`) across all three services (`3000`, `3001`, `3002`), using native `fetch` (Node 20+) for inter-service HTTP calls.
- **Rationale**: Fastify provides ultra-low overhead, native `fastify.inject()` support for fast HTTP route integration testing without binding TCP sockets during unit/integration tests, and seamless TypeScript strict mode compatibility. Node 20+ includes global `fetch`, removing the need for `axios` while allowing simple `vi.spyOn(globalThis, 'fetch')` mocking in Vitest.
- **Alternatives Considered**:
  - *Express.js*: Lacks built-in `inject()` testing and has weaker TypeScript ergonomics.
  - *Axios*: Unnecessary third-party dependency when Node.js 20+ native `fetch` + Zod response parsing satisfies all HTTP client requirements.

## 3. System 1 Intent Classification with Jev (TypeSafe SDK)

- **Decision**: Encapsulate **Jev (TypeSafe SDK)** inside `apps/bot-orchestrator/src/services/jev-classifier.ts` behind a strict TypeScript contract returning `{ choice: IntentEnum, score: number, extracted?: { serviceId?: string, date?: string, time?: string } }`.
- **Rationale**: Constitution Principle II mandates exclusive use of Jev for System 1 classification into `AGENDAR`, `CANCELAR`, `VER_HORARIOS`, `TABELA_PRECOS`, `LOCALIZACAO`, `ATENDIMENTO_HUMANO`, `OUTRO` with a strict `< 0.70` confidence fallback gate. Wrapping the Jev SDK call behind an injectable classifier service ensures deterministic local execution and full testability in Vitest (testing exact boundary scores like `0.69` vs `0.70`).
- **Alternatives Considered**:
  - *Direct LLM tool-calling for routing*: Strictly prohibited by Constitution Principle II.

## 4. System 2 Lightweight LLM Adapter (`OUTRO` Intent Only)

- **Decision**: Implement `LlmContextualService` in `apps/bot-orchestrator/src/services/llm-service.ts` that is invoked **exclusively** when `choice === 'OUTRO'` and `score >= 0.70`. It uses a friendly barbershop receptionist system prompt and calls OpenAI/Anthropic via HTTP (or falls back to a safe polite receptionist message if no API key is provided in local test/dev).
- **Rationale**: Satisfies FR-018 and Constitution Principle II while ensuring local development and automated CI tests never fail due to missing external LLM credentials.
- **Alternatives Considered**:
  - *Calling LLM on low confidence (`< 0.70`)*: Prohibited by the specification and constitution; `< 0.70` must immediately transfer to human attendance (`ATENDIMENTO_HUMANO`).

## 5. WhatsApp Gateway Adapter (`WhatsAppClientInterface`)

- **Decision**: In `apps/whatsapp-gateway` (Port `3002`), define `WhatsAppClientInterface` with `sendMessage(to: string, text: string): Promise<void>` and an inbound event handler `onIncomingMessage(event: { sender: string, messageText: string, messageId: string, fromMe: boolean }): Promise<void>`. When `fromMe === false`, the handler sends `POST http://localhost:3000/webhook/whatsapp`. Provide an HTTP simulation/webhook trigger route (`POST /webhook/inbound` or event emitter trigger) alongside `POST /send-message` so external community APIs (Evolution API / Z-API) or local end-to-end tests can inject inbound WhatsApp events.
- **Rationale**: Completely decouples the WhatsApp Web protocol from `bot-orchestrator` while allowing both unit testing and live E2E simulation across ports `3002 -> 3000 -> 3001 -> 3002`.
- **Alternatives Considered**:
  - *Coupling Baileys/Evolution directly inside `bot-orchestrator`*: Violates Constitution Principle IV (`apps/whatsapp-gateway` isolation).
