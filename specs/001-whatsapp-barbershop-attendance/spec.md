# Feature Specification: Intelligent WhatsApp Barbershop Attendance Monorepo (3 Services)

**Feature Branch**: `001-whatsapp-barbershop-attendance`

**Created**: 2026-10-15

**Status**: Draft

**Input**: User description: "Monorepo com 3 serviços independentes: apps/barbershop-mock (Porta 3001), apps/whatsapp-gateway (Porta 3002) e apps/bot-orchestrator (Porta 3000) com classificação determinística via Jev SDK, roteamento HTTP/REST e validação Zod."

## Service Architecture & Endpoints Overview

### 1. `apps/barbershop-mock` (Port `3001`)
**Objective**: Independent HTTP REST server simulating the barbershop's ERP/management system using in-memory data storage.
- `GET /services`: Lists all available barbershop services containing `id`, `name`, `price`, and `duration` (e.g., Barba R$ 45, Cabelo R$ 55, Combo R$ 90).
- `GET /slots?date=YYYY-MM-DD`: Returns available (unbooked) time slots for the specified date.
- `POST /bookings`: Creates a new appointment booking with payload `{ customerPhone: string, serviceId: string, time: string }`.
- `DELETE /bookings/:customerPhone`: Cancels an existing active appointment associated with `:customerPhone` and releases the slot.
- `GET /info`: Returns barbershop metadata including physical address and operating hours.

### 2. `apps/whatsapp-gateway` (Port `3002`)
**Objective**: Isolated HTTP adapter for the WhatsApp Web API integration.
- `POST /send-message`: Accepts `{ to: string, text: string }` and dispatches the outbound text message to the WhatsApp Web API/library.
- **Webhook Emitter**: Whenever the WhatsApp Web connection receives an inbound text message where `fromMe == false`, it dispatches an HTTP `POST` request to `http://localhost:3000/webhook/whatsapp` (configurable via `.env`) with payload `{ sender: string, messageText: string, messageId: string }`.

### 3. `apps/bot-orchestrator` (Port `3000`)
**Objective**: Intelligent message orchestrator and decision brain.
- `POST /webhook/whatsapp`: Receives `{ sender: string, messageText: string, messageId: string }` from `whatsapp-gateway`.
- **Decision & Execution Pipeline**:
  1. **Input Validation**: Strictly validates the incoming webhook payload via **Zod**.
  2. **System 1 Classification (Jev SDK)**: Evaluates `messageText` using Jev (`Choice` and `Score`) across the strict intent enum (`TABELA_PRECOS`, `LOCALIZACAO`, `VER_HORARIOS`, `AGENDAR`, `CANCELAR`, `ATENDIMENTO_HUMANO`, `OUTRO`).
  3. **Intent Routing**:
     - `TABELA_PRECOS` (Score `>= 0.70`): Calls `GET http://localhost:3001/services` and formats a customer-friendly pricing table response.
     - `LOCALIZACAO` (Score `>= 0.70`): Calls `GET http://localhost:3001/info` and formats the barbershop's address and opening hours.
     - `VER_HORARIOS` (Score `>= 0.70`): Calls `GET http://localhost:3001/slots?date=YYYY-MM-DD` and formats the list of available slots.
     - `AGENDAR` (Score `>= 0.70`): Checks if service and date/time are present; if missing, replies asking the customer for the missing details; if complete, calls `POST http://localhost:3001/bookings` with `{ customerPhone: sender, serviceId, time }` and formats the confirmation.
     - `CANCELAR` (Score `>= 0.70`): Calls `DELETE http://localhost:3001/bookings/:sender` and formats the cancellation confirmation.
     - `ATENDIMENTO_HUMANO` or **Confidence Score `< 0.70`**: Immediately generates a handoff message informing the customer that they are being transferred to a human attendant.
     - `OUTRO` (Score `>= 0.70`): Invokes the lightweight LLM (OpenAI/Anthropic) with a friendly barbershop system prompt to generate a contextual response.
  4. **Final Response Dispatch**: Sends `POST http://localhost:3002/send-message` with `{ to: sender, text: finalResponseText }`.

---

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Consult Barbershop Catalog, Available Slots, and Store Info (Priority: P1)

A customer sends a WhatsApp message asking for service prices, open time slots, or the barbershop's location. `whatsapp-gateway` (3002) forwards the message to `bot-orchestrator` (3000), which classifies the intent via Jev (`>= 0.70`), fetches live data from `barbershop-mock` (3001), and dispatches the formatted reply back through `whatsapp-gateway` (3002).

**Why this priority**: Delivers the core read-only self-service flows across all 3 microservices end-to-end.

**Independent Test**: Send a POST request to `http://localhost:3000/webhook/whatsapp` (or simulate an inbound WhatsApp message on 3002) asking for prices, slots, or address, and verify that 3001 is queried and 3002 `POST /send-message` receives the formatted reply.

**Acceptance Scenarios**:

1. **Given** `barbershop-mock` (3001) has registered services (`Barba` R$ 45, `Cabelo` R$ 55, `Combo` R$ 90), **When** `bot-orchestrator` (3000) receives `{ sender, messageText: "Qual o preço do corte e barba?", messageId }` classified by Jev as `TABELA_PRECOS` (`>= 0.70`), **Then** it calls `GET http://localhost:3001/services` and dispatches the formatted price list to `POST http://localhost:3002/send-message`.
2. **Given** `barbershop-mock` (3001) has open slots for `YYYY-MM-DD`, **When** `bot-orchestrator` (3000) receives a message classified as `VER_HORARIOS` (`>= 0.70`), **Then** it calls `GET http://localhost:3001/slots?date=YYYY-MM-DD` and sends the formatted available times to `POST http://localhost:3002/send-message`.
3. **Given** `barbershop-mock` (3001) serves store details on `GET /info`, **When** `bot-orchestrator` (3000) receives a message classified as `LOCALIZACAO` (`>= 0.70`), **Then** it fetches `GET http://localhost:3001/info` and sends the address and opening hours to `POST http://localhost:3002/send-message`.

---

### User Story 2 - Create and Cancel Bookings Across Services (Priority: P1)

A customer messages via WhatsApp to book an appointment (`AGENDAR`) or cancel an existing one (`CANCELAR`). `bot-orchestrator` (3000) validates required parameters, calls `POST /bookings` or `DELETE /bookings/:customerPhone` on `barbershop-mock` (3001), and sends the confirmation via `whatsapp-gateway` (3002).

**Why this priority**: Implements the core transactional booking lifecycle across the orchestrator, mock ERP, and WhatsApp gateway.

**Independent Test**: Test `POST /bookings` and `DELETE /bookings/:customerPhone` directly on port 3001, and test the orchestrated flow via `POST /webhook/whatsapp` on port 3000 for both complete and incomplete booking requests.

**Acceptance Scenarios**:

1. **Given** a customer sends an `AGENDAR` message (`>= 0.70`) containing both the desired service and time, **When** `bot-orchestrator` (3000) processes the webhook, **Then** it calls `POST http://localhost:3001/bookings` with `{ customerPhone: sender, serviceId, time }` and sends a booking confirmation message via `POST http://localhost:3002/send-message`.
2. **Given** a customer sends an `AGENDAR` message (`>= 0.70`) missing the date/time or service, **When** `bot-orchestrator` (3000) processes the webhook, **Then** it does NOT call `POST /bookings` and instead sends a message via `POST http://localhost:3002/send-message` requesting the missing details.
3. **Given** a customer has an active booking in `barbershop-mock` (3001), **When** `bot-orchestrator` (3000) receives a message classified as `CANCELAR` (`>= 0.70`), **Then** it calls `DELETE http://localhost:3001/bookings/:sender` and sends a cancellation confirmation via `POST http://localhost:3002/send-message`.

---

### User Story 3 - Confidence Gate (`< 0.70`), Human Handoff, and `OUTRO` LLM Fallback (Priority: P2)

When a customer requests human assistance (`ATENDIMENTO_HUMANO`) or sends a message whose Jev confidence score is `< 0.70`, `bot-orchestrator` (3000) immediately replies with a human handoff notification. When the intent is `OUTRO` with confidence `>= 0.70`, `bot-orchestrator` invokes the lightweight LLM with a friendly barbershop system prompt and dispatches the reply via `whatsapp-gateway` (3002).

**Why this priority**: Enforces the non-negotiable System 1 / System 2 boundary and prevents low-confidence hallucinations.

**Independent Test**: Simulate webhook payloads on port 3000 where Jev returns score `< 0.70`, `ATENDIMENTO_HUMANO` (`>= 0.70`), and `OUTRO` (`>= 0.70`), verifying that `< 0.70` and `ATENDIMENTO_HUMANO` never call the LLM or mutate bookings, and `OUTRO` invokes the LLM adapter and forwards the text to port 3002.

**Acceptance Scenarios**:

1. **Given** Jev classifies an incoming message with any intent but returns `score < 0.70`, **When** `bot-orchestrator` (3000) evaluates the result, **Then** it immediately generates the human transfer message and sends it to `POST http://localhost:3002/send-message` without calling `barbershop-mock` or the LLM.
2. **Given** Jev classifies an incoming message as `ATENDIMENTO_HUMANO` with `score >= 0.70`, **When** `bot-orchestrator` (3000) evaluates the result, **Then** it sends the human transfer message to `POST http://localhost:3002/send-message`.
3. **Given** Jev classifies an incoming message as `OUTRO` with `score >= 0.70`, **When** `bot-orchestrator` (3000) evaluates the result, **Then** it calls the lightweight LLM (OpenAI/Anthropic) with a friendly barbershop system prompt and dispatches the generated text via `POST http://localhost:3002/send-message`.

---

### User Story 4 - WhatsApp Gateway Inbound Filtering & Outbound Dispatch (Priority: P2)

`whatsapp-gateway` (3002) listens to events from the WhatsApp Web integration, filters out self-sent messages (`fromMe == true`), forwards inbound customer text messages (`fromMe == false`) to `http://localhost:3000/webhook/whatsapp`, and exposes `POST /send-message` to deliver replies back to WhatsApp.

**Why this priority**: Isolates the WhatsApp Web protocol/library inside a dedicated service on port 3002 so the orchestrator remains a pure HTTP service.

**Independent Test**: Simulate an inbound event in `whatsapp-gateway` with `fromMe: false` and `fromMe: true`, verifying that only `fromMe: false` triggers `POST http://localhost:3000/webhook/whatsapp`, and call `POST http://localhost:3002/send-message` with `{ to, text }` to verify outbound delivery.

**Acceptance Scenarios**:

1. **Given** `whatsapp-gateway` (3002) receives a WhatsApp Web text event with `fromMe == false`, **When** the event is processed, **Then** it sends an HTTP `POST` to `http://localhost:3000/webhook/whatsapp` with `{ sender, messageText, messageId }`.
2. **Given** `whatsapp-gateway` (3002) receives a WhatsApp Web event with `fromMe == true`, **When** the event is processed, **Then** it ignores the event and does NOT call `bot-orchestrator`.
3. **Given** `bot-orchestrator` sends `POST http://localhost:3002/send-message` with `{ to: string, text: string }`, **When** `whatsapp-gateway` validates the payload, **Then** it dispatches the message through the WhatsApp Web client and returns HTTP 200.

---

### Edge Cases

- **Time Slot Already Booked (`POST /bookings` Conflict)**: If a customer tries to book a `time` slot that is already reserved in `barbershop-mock` (3001), `POST /bookings` returns a conflict error (`409 Conflict`), and `bot-orchestrator` informs the customer that the slot is unavailable and suggests checking free horarios.
- **Cancel Non-Existent Booking (`DELETE /bookings/:customerPhone` Not Found)**: If `DELETE /bookings/:sender` returns `404 Not Found` because the customer has no active appointment, `bot-orchestrator` replies politely informing the customer that no active booking was found for their number.
- **Invalid Webhook or Inter-Service Payload**: Any request failing Zod validation on `3000`, `3001`, or `3002` immediately returns `400 Bad Request` with structured validation details without crashing the daemon.
- **Missing Date Parameter on `VER_HORARIOS`**: When a user asks for available slots without specifying a date, `bot-orchestrator` defaults the `date` query parameter (`YYYY-MM-DD`) to the current date when calling `GET http://localhost:3001/slots?date=YYYY-MM-DD`.

## Requirements *(mandatory)*

### Functional Requirements

#### `apps/barbershop-mock` (Port `3001`)
- **FR-001**: `apps/barbershop-mock` MUST run on port `3001` (configurable via `.env`) and maintain in-memory state pre-seeded with default services (`Barba` R$ 45, `Cabelo` R$ 55, `Combo` R$ 90), daily time slots, and store info.
- **FR-002**: `GET /services` MUST return a JSON array of services, each validated with `id` (string), `name` (string), `price` (number), and `duration` (number in minutes).
- **FR-003**: `GET /slots?date=YYYY-MM-DD` MUST validate the `date` query parameter and return the list of available (unbooked) time strings (e.g., `["09:00", "10:00", ...]`) for that date.
- **FR-004**: `POST /bookings` MUST validate `{ customerPhone: string, serviceId: string, time: string }`, verify that the service exists and the slot is available, store the booking in memory, mark the slot as occupied, and return the created booking (`201 Created` or `409 Conflict` if occupied).
- **FR-005**: `DELETE /bookings/:customerPhone` MUST remove the active booking for `:customerPhone`, free the corresponding time slot, and return confirmation (`200 OK` or `404 Not Found` if no booking exists).
- **FR-006**: `GET /info` MUST return the barbershop's address, opening hours, and contact metadata.

#### `apps/whatsapp-gateway` (Port `3002`)
- **FR-007**: `apps/whatsapp-gateway` MUST run on port `3002` (configurable via `.env`) and encapsulate the WhatsApp Web API/library behind a clean adapter interface (`WhatsAppClientInterface`).
- **FR-008**: `POST /send-message` MUST validate `{ to: string, text: string }` via Zod and dispatch the text message to the recipient via the WhatsApp Web adapter.
- **FR-009**: When an inbound text message event occurs on WhatsApp Web with `fromMe === false`, `whatsapp-gateway` MUST emit an HTTP `POST` to `http://localhost:3000/webhook/whatsapp` (URL configurable via `.env`) with payload `{ sender: string, messageText: string, messageId: string }`.

#### `apps/bot-orchestrator` (Port `3000`)
- **FR-010**: `apps/bot-orchestrator` MUST run on port `3000` (configurable via `.env`) and expose `POST /webhook/whatsapp` accepting `{ sender: string, messageText: string, messageId: string }` validated via Zod.
- **FR-011**: `bot-orchestrator` MUST classify `messageText` using the **Jev SDK** (`Choice` and `Score`) into the strict enum: `AGENDAR`, `CANCELAR`, `VER_HORARIOS`, `TABELA_PRECOS`, `LOCALIZACAO`, `ATENDIMENTO_HUMANO`, `OUTRO`.
- **FR-012**: If Jev returns a confidence `Score < 0.70` OR `Choice === 'ATENDIMENTO_HUMANO'`, `bot-orchestrator` MUST immediately generate a human handoff message without invoking the LLM or mutating bookings.
- **FR-013**: For `TABELA_PRECOS` (`>= 0.70`), `bot-orchestrator` MUST request `GET http://localhost:3001/services` and format the response text with the service names and values.
- **FR-014**: For `LOCALIZACAO` (`>= 0.70`), `bot-orchestrator` MUST request `GET http://localhost:3001/info` and format the response text with the address and operating hours.
- **FR-015**: For `VER_HORARIOS` (`>= 0.70`), `bot-orchestrator` MUST request `GET http://localhost:3001/slots?date=YYYY-MM-DD` and format the available time slots.
- **FR-016**: For `AGENDAR` (`>= 0.70`), `bot-orchestrator` MUST check if the service and date/time are identified; if missing, it MUST reply asking for the missing information; if complete, it MUST send `POST http://localhost:3001/bookings` with `{ customerPhone: sender, serviceId, time }` and format the confirmation message.
- **FR-017**: For `CANCELAR` (`>= 0.70`), `bot-orchestrator` MUST send `DELETE http://localhost:3001/bookings/:sender` and format the cancellation response.
- **FR-018**: For `OUTRO` (`>= 0.70`), `bot-orchestrator` MUST dispatch the message to a lightweight LLM (OpenAI/Anthropic) configured with a friendly barbershop system prompt to generate the reply.
- **FR-019**: At the end of every webhook execution flow, `bot-orchestrator` MUST send `POST http://localhost:3002/send-message` with `{ to: sender, text: responseText }`.

### Key Entities *(include if feature involves data)*

- **WebhookPayload (`bot-orchestrator` ↔ `whatsapp-gateway`)**: `{ sender: string, messageText: string, messageId: string }`.
- **SendMessagePayload (`bot-orchestrator` → `whatsapp-gateway`)**: `{ to: string, text: string }`.
- **ServiceItem (`barbershop-mock`)**: `{ id: string, name: string, price: number, duration: number }`.
- **SlotsResponse (`barbershop-mock`)**: `{ date: string, slots: string[] }`.
- **BookingRequest (`bot-orchestrator` → `barbershop-mock`)**: `{ customerPhone: string, serviceId: string, time: string, date?: string }`.
- **BarbershopInfo (`barbershop-mock`)**: `{ name: string, address: string, openingHours: string, phone?: string }`.
- **JevDecision (`bot-orchestrator`)**: `{ intent: 'AGENDAR' | 'CANCELAR' | 'VER_HORARIOS' | 'TABELA_PRECOS' | 'LOCALIZACAO' | 'ATENDIMENTO_HUMANO' | 'OUTRO', score: number, extracted?: { serviceId?: string, date?: string, time?: string } }`.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: All 3 apps (`barbershop-mock` on 3001, `whatsapp-gateway` on 3002, and `bot-orchestrator` on 3000) start cleanly via a single root development command and communicate strictly via HTTP/REST.
- **SC-002**: 100% of webhook requests with Jev confidence `< 0.70` result in immediate human handoff without triggering `POST /bookings`, `DELETE /bookings`, or the generative LLM.
- **SC-003**: All 5 endpoints of `barbershop-mock` (`GET /services`, `GET /slots`, `POST /bookings`, `DELETE /bookings/:customerPhone`, `GET /info`), the gateway endpoint (`POST /send-message`), and the orchestrator endpoint (`POST /webhook/whatsapp`) pass automated Vitest integration tests using `fastify.inject()` and mocked network calls.
- **SC-004**: 100% of codebase compiles with `tsc --noEmit` (`strict: true`, `noImplicitAny: true`, zero `any`) and validates all HTTP boundaries with Zod.

## Assumptions

- Service base URLs default to `http://localhost:3000` (`bot-orchestrator`), `http://localhost:3001` (`barbershop-mock`), and `http://localhost:3002` (`whatsapp-gateway`), all overrideable via `.env` variables.
- When `AGENDAR` or `VER_HORARIOS` does not explicitly specify a date string (`YYYY-MM-DD`), the current date (or a configured default date) is used so single-turn booking and slot queries work seamlessly in tests and demos.
- If an external LLM API key (`OPENAI_API_KEY` / `ANTHROPIC_API_KEY`) is not configured in local/test environments when routing to `OUTRO`, the LLM client adapter provides a deterministic safe fallback response so local execution never crashes.
