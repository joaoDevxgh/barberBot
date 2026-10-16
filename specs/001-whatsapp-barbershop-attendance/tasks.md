# Tasks: Intelligent WhatsApp Barbershop Attendance Monorepo (3 Services)

**Input**: Design documents from `/specs/001-whatsapp-barbershop-attendance/`
**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md), [data-model.md](./data-model.md), [contracts/http-contracts.md](./contracts/http-contracts.md)

**Tests**: Included per Constitution Principle V (Isolated Workspace Testing with Vitest + `fastify.inject()` + network mocking).

**Organization**: Tasks are grouped by user story so each story can be implemented, tested, and verified independently across the 3 monorepo apps (`apps/barbershop-mock`, `apps/whatsapp-gateway`, `apps/bot-orchestrator`).

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files/apps, no dependencies on incomplete tasks)
- **[Story]**: Maps task to specific user story (`[US1]`, `[US2]`, `[US3]`, `[US4]`)
- Exact file paths are included in every task description

---

## Phase 1: Setup (Monorepo & Workspace Initialization)

**Purpose**: Initialize the npm workspaces monorepo, shared strict TypeScript configuration (`strict: true`, `noImplicitAny: true`), environment templates, and per-app package manifests.

- [ ] T001 Create root workspace manifest with centralized `dev`, `build`, `test`, and `typecheck` scripts in `package.json`
- [ ] T002 Create shared strict TypeScript compiler configuration (`strict: true`, `noImplicitAny: true`, `target: "ES2022"`) in `tsconfig.base.json`
- [ ] T003 [P] Create environment variables template for ports `3000`, `3001`, `3002` and service URLs in `.env.example` and `.gitignore`
- [ ] T004 [P] Initialize `apps/barbershop-mock` package manifest, Vitest config, and TypeScript config in `apps/barbershop-mock/package.json` and `apps/barbershop-mock/tsconfig.json`
- [ ] T005 [P] Initialize `apps/whatsapp-gateway` package manifest, Vitest config, and TypeScript config in `apps/whatsapp-gateway/package.json` and `apps/whatsapp-gateway/tsconfig.json`
- [ ] T006 [P] Initialize `apps/bot-orchestrator` package manifest, Vitest config, and TypeScript config in `apps/bot-orchestrator/package.json` and `apps/bot-orchestrator/tsconfig.json`

---

## Phase 2: Foundational (Core Schemas, Configs, and Server Scaffolds)

**Purpose**: Establish environment validation (`Zod`), core domain contracts, Fastify app factories, and the Jev System 1 classifier contract before implementing user story flows.

**⚠️ CRITICAL**: No user story work can begin until this phase is complete.

- [ ] T007 [P] Implement Zod environment config (`PORT=3001`) and domain schemas (`ServiceItemSchema` with non-empty `id`/`name`, positive `price`/`duration`; `BarbershopInfoSchema`; `TimeSlotQuerySchema` matching `/^\d{4}-\d{2}-\d{2}$/`; `CreateBookingInputSchema` with non-empty `customerPhone`, `serviceId`, `time`) in `apps/barbershop-mock/src/config/env.ts` and `apps/barbershop-mock/src/domain/schemas.ts`
- [ ] T008 [P] Implement Zod environment config (`PORT=3002`, `ORCHESTRATOR_WEBHOOK_URL="http://localhost:3000/webhook/whatsapp"`) and gateway schemas (`SendMessageRequestSchema` with non-empty `to`/`text`; `InboundWhatsAppEventSchema` with `sender`, `messageText`, `messageId`, `fromMe: boolean`) in `apps/whatsapp-gateway/src/config/env.ts` and `apps/whatsapp-gateway/src/domain/schemas.ts`
- [ ] T009 [P] Implement Zod environment config (`PORT=3000`, `BARBERSHOP_API_URL="http://localhost:3001"`, `WHATSAPP_GATEWAY_URL="http://localhost:3002"`) and orchestrator schemas (`WhatsAppWebhookPayloadSchema` with non-empty `sender`, `messageText`, `messageId`; `IntentEnum` strictly typed as `'AGENDAR' | 'CANCELAR' | 'VER_HORARIOS' | 'TABELA_PRECOS' | 'LOCALIZACAO' | 'ATENDIMENTO_HUMANO' | 'OUTRO'`; `JevClassificationResultSchema` with `choice`, `score` 0.0–1.0, and optional `extracted`) in `apps/bot-orchestrator/src/config/env.ts` and `apps/bot-orchestrator/src/domain/schemas.ts`
- [ ] T010 [P] Implement in-memory store (`MockBarbershopStore`) pre-seeded with `Barba` (R$ 45, 30m), `Cabelo` (R$ 55, 45m), `Combo` (R$ 90, 60m), store info, and daily time slots in `apps/barbershop-mock/src/domain/store.ts`
- [ ] T011 [P] Implement `WhatsAppClientInterface` adapter and outbound HTTP client in `apps/whatsapp-gateway/src/services/whatsapp-client.ts` and `apps/bot-orchestrator/src/services/whatsapp-http.client.ts`
- [ ] T012 [P] Implement Jev (TypeSafe SDK) System 1 intent classifier (`Choice` and `Score` evaluation + entity extraction for service/date/time) in `apps/bot-orchestrator/src/services/jev-classifier.ts`
- [ ] T013 [P] Create Fastify application factories and HTTP daemon entrypoints in `apps/barbershop-mock/src/app.ts`, `apps/barbershop-mock/src/server.ts`, `apps/whatsapp-gateway/src/app.ts`, `apps/whatsapp-gateway/src/server.ts`, `apps/bot-orchestrator/src/app.ts`, and `apps/bot-orchestrator/src/server.ts`

**Checkpoint**: Foundation ready — all 3 Fastify apps have validated configs, Zod contracts, and server entrypoints.

---

## Phase 3: User Story 1 - Consult Barbershop Catalog, Available Slots, and Store Info (Priority: P1) 🎯 MVP

**Goal**: Enable customers to query prices (`TABELA_PRECOS`), available time slots (`VER_HORARIOS`), and location/opening hours (`LOCALIZACAO`) end-to-end across `bot-orchestrator` (3000), `barbershop-mock` (3001), and `whatsapp-gateway` (3002).

**Independent Test**: Send `POST /webhook/whatsapp` to port `3000` with messages asking for prices, slots, or address; verify `barbershop-mock` (`GET /services`, `GET /slots`, `GET /info`) is queried and the formatted response is dispatched to `POST http://localhost:3002/send-message`.

### Tests for User Story 1 ⚠️

> **NOTE: Write these tests FIRST and verify they cover all read-only endpoints and orchestrator routing**

- [ ] T014 [P] [US1] Write Vitest route tests for `GET /services`, `GET /slots?date=YYYY-MM-DD`, and `GET /info` using `fastify.inject()` in `apps/barbershop-mock/tests/barbershop.routes.test.ts`
- [ ] T015 [P] [US1] Write Vitest unit and integration tests for `TABELA_PRECOS`, `VER_HORARIOS`, and `LOCALIZACAO` routing (score `>= 0.70`) with mocked HTTP calls in `apps/bot-orchestrator/tests/webhook.routes.test.ts`

### Implementation for User Story 1

- [ ] T016 [P] [US1] Implement `GET /services`, `GET /slots`, and `GET /info` endpoints in `apps/barbershop-mock/src/routes/barbershop.routes.ts`
- [ ] T017 [P] [US1] Implement HTTP client methods `getServices()`, `getSlots(date)`, and `getInfo()` with Zod response validation in `apps/bot-orchestrator/src/services/barbershop-http.client.ts`
- [ ] T018 [US1] Implement orchestrator routing and message formatting for `TABELA_PRECOS`, `VER_HORARIOS`, and `LOCALIZACAO` in `apps/bot-orchestrator/src/services/orchestrator.service.ts` and `apps/bot-orchestrator/src/routes/webhook.routes.ts`

**Checkpoint**: User Story 1 is fully functional and independently testable.

---

## Phase 4: User Story 2 - Create and Cancel Bookings Across Services (Priority: P1)

**Goal**: Enable customers to book appointments (`AGENDAR`) and cancel existing bookings (`CANCELAR`) via WhatsApp, enforcing required parameter checks (`serviceId`, `time`) and slot conflict prevention.

**Independent Test**: Test `POST /bookings` and `DELETE /bookings/:customerPhone` on `barbershop-mock` (3001) and test `AGENDAR` (complete vs missing date/time) and `CANCELAR` via `POST /webhook/whatsapp` on `bot-orchestrator` (3000).

### Tests for User Story 2 ⚠️

- [ ] T019 [P] [US2] Add Vitest tests for `POST /bookings` (`201 Created`, `409 Conflict` on double-booking) and `DELETE /bookings/:customerPhone` (`200 OK`, `404 Not Found`) in `apps/barbershop-mock/tests/barbershop.routes.test.ts`
- [ ] T020 [P] [US2] Add Vitest tests for `AGENDAR` (complete booking, missing date/time prompt, `409` slot conflict) and `CANCELAR` (`200` cancelled, `404` no active booking) in `apps/bot-orchestrator/tests/webhook.routes.test.ts`

### Implementation for User Story 2

- [ ] T021 [P] [US2] Implement `POST /bookings` and `DELETE /bookings/:customerPhone` endpoints in `apps/barbershop-mock/src/routes/barbershop.routes.ts`
- [ ] T022 [P] [US2] Implement `createBooking({ customerPhone, serviceId, time })` and `cancelBooking(customerPhone)` HTTP client methods in `apps/bot-orchestrator/src/services/barbershop-http.client.ts`
- [ ] T023 [US2] Implement `AGENDAR` (missing parameter solicitation + booking creation) and `CANCELAR` orchestration branches in `apps/bot-orchestrator/src/services/orchestrator.service.ts`

**Checkpoint**: User Stories 1 and 2 (`barbershop-mock` 5 endpoints + transactional orchestrator flows) work independently and together.

---

## Phase 5: User Story 3 - Confidence Gate (`< 0.70`), Human Handoff, and `OUTRO` LLM Fallback (Priority: P2)

**Goal**: Enforce the non-negotiable confidence threshold (`Score < 0.70` → immediate human handoff), explicit `ATENDIMENTO_HUMANO` routing, and System 2 lightweight LLM invocation strictly for `OUTRO` (`Score >= 0.70`).

**Independent Test**: Send webhook messages to port `3000` yielding Jev confidence `< 0.70`, `ATENDIMENTO_HUMANO` (`>= 0.70`), and `OUTRO` (`>= 0.70`); verify `< 0.70` never calls `barbershop-mock` or the LLM, and `OUTRO` calls the lightweight LLM service with the friendly barbershop system prompt.

### Tests for User Story 3 ⚠️

- [ ] T024 [P] [US3] Write Vitest unit tests for Jev intent classification and confidence scores across all 7 intent enums in `apps/bot-orchestrator/tests/jev-classifier.test.ts`
- [ ] T025 [P] [US3] Add Vitest integration tests for confidence `< 0.70` fallback, `ATENDIMENTO_HUMANO` routing, and `OUTRO` System 2 LLM invocation in `apps/bot-orchestrator/tests/webhook.routes.test.ts`

### Implementation for User Story 3

- [ ] T026 [P] [US3] Implement lightweight System 2 LLM service (`LlmContextualService` supporting OpenAI/Anthropic HTTP API with friendly barbershop system prompt and safe local fallback) in `apps/bot-orchestrator/src/services/llm-contextual.service.ts`
- [ ] T027 [US3] Implement `< 0.70` confidence gate, `ATENDIMENTO_HUMANO` handoff response, and `OUTRO` LLM delegation in `apps/bot-orchestrator/src/services/orchestrator.service.ts`

**Checkpoint**: All 7 intents and the `< 0.70` confidence gate in `apps/bot-orchestrator` are complete and verified.

---

## Phase 6: User Story 4 - WhatsApp Gateway Inbound Filtering & Outbound Dispatch (Priority: P2)

**Goal**: Complete `apps/whatsapp-gateway` (Port `3002`) with `POST /send-message` and the inbound WhatsApp Web webhook emitter that filters `fromMe === false` and posts to `http://localhost:3000/webhook/whatsapp`.

**Independent Test**: Test `POST /send-message` and inbound event emission (`fromMe: false` forwards to `http://localhost:3000/webhook/whatsapp`; `fromMe: true` is ignored) in `apps/whatsapp-gateway/tests/gateway.routes.test.ts`.

### Tests for User Story 4 ⚠️

- [ ] T028 [P] [US4] Write Vitest unit and integration tests for `POST /send-message` and inbound webhook emitter (`fromMe === false` vs `fromMe === true`) in `apps/whatsapp-gateway/tests/gateway.routes.test.ts`

### Implementation for User Story 4

- [ ] T029 [P] [US4] Implement inbound WhatsApp event listener and HTTP webhook emitter (`fromMe === false` → `POST http://localhost:3000/webhook/whatsapp`) in `apps/whatsapp-gateway/src/services/webhook-emitter.ts`
- [ ] T030 [US4] Implement `POST /send-message` (and inbound event simulation endpoint `POST /webhook/inbound`) in `apps/whatsapp-gateway/src/routes/gateway.routes.ts`

**Checkpoint**: All 3 applications (`3000`, `3001`, `3002`) are complete and independently tested.

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: Finalize production containerization, documentation, full monorepo typecheck, and end-to-end verification across all 3 services.

- [ ] T031 [P] Create production multi-stage `Dockerfile` and `.dockerignore` in `Dockerfile` and `.dockerignore`
- [ ] T032 [P] Create comprehensive root documentation with architecture overview, ports (`3000`, `3001`, `3002`), and cURL/PowerShell examples in `README.md`
- [ ] T033 Run full workspace strict compilation (`npm run typecheck`), test suite (`npm test`), and end-to-end validation from `specs/001-whatsapp-barbershop-attendance/quickstart.md`

---

## Dependencies & Execution Order

### Phase Dependencies

- **Phase 1 (Setup)**: No dependencies — can start immediately.
- **Phase 2 (Foundational)**: Depends on Phase 1 — **BLOCKS** all user stories.
- **Phase 3 (US1 - P1 Catalog/Slots/Info)**: Depends on Phase 2.
- **Phase 4 (US2 - P1 Bookings/Cancellations)**: Depends on Phase 2 (extends `barbershop-mock` and `bot-orchestrator` routes/services from US1).
- **Phase 5 (US3 - P2 Confidence Gate & `OUTRO` LLM)**: Depends on Phase 2.
- **Phase 6 (US4 - P2 WhatsApp Gateway)**: Depends on Phase 2 — can run in parallel with US1–US3.
- **Phase 7 (Polish)**: Depends on Phases 3–6 completion.

### Parallel Opportunities

- **Setup (Phase 1)**: `T003`, `T004`, `T005`, `T006` can run in parallel across the 3 app directories.
- **Foundational (Phase 2)**: `T007`, `T008`, `T009`, `T010`, `T011`, `T012`, `T013` operate on separate files across the 3 apps and can be executed in parallel.
- **User Stories**: `apps/whatsapp-gateway` (US4: `T028`–`T030`), `apps/barbershop-mock` endpoints (`T014`, `T016`, `T019`, `T021`), and `apps/bot-orchestrator` services (`T017`, `T022`, `T026`) can be developed in parallel.

---

## Parallel Example: Foundational & User Story 1

```bash
# Parallel Foundational Schemas across all 3 apps:
Task: "T007 [P] Implement Zod env and domain schemas in apps/barbershop-mock/src/domain/schemas.ts"
Task: "T008 [P] Implement Zod env and gateway schemas in apps/whatsapp-gateway/src/domain/schemas.ts"
Task: "T009 [P] Implement Zod env and orchestrator schemas in apps/bot-orchestrator/src/domain/schemas.ts"

# Parallel User Story 1 Tests & Endpoints:
Task: "T014 [P] [US1] Write Vitest route tests in apps/barbershop-mock/tests/barbershop.routes.test.ts"
Task: "T015 [P] [US1] Write Vitest routing tests in apps/bot-orchestrator/tests/webhook.routes.test.ts"
```

---

## Implementation Strategy

### MVP First (User Story 1 + User Story 2)

1. Complete **Phase 1: Setup** (`T001`–`T006`)
2. Complete **Phase 2: Foundational** (`T007`–`T013`)
3. Complete **Phase 3: User Story 1** (`T014`–`T018`) & **Phase 4: User Story 2** (`T019`–`T023`)
4. Complete **Phase 5: User Story 3** (`T024`–`T027`) & **Phase 6: User Story 4** (`T028`–`T030`)
5. Complete **Phase 7: Polish & Validation** (`T031`–`T033`)
