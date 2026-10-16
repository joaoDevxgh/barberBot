# Phase 1 Data Model: Intelligent WhatsApp Barbershop Attendance Monorepo

**Feature**: `001-whatsapp-barbershop-attendance`  
**Date**: 2026-10-15

All entities across the three services (`apps/barbershop-mock`, `apps/whatsapp-gateway`, `apps/bot-orchestrator`) are strictly validated at runtime using **Zod** with TypeScript `strict: true` and `noImplicitAny: true`.

---

## 1. Domain Entities in `apps/barbershop-mock` (Port `3001`)

### 1.1 `ServiceItem`
Represents a barbershop catalog service.
- **Fields**:
  - `id`: `string` (non-empty, e.g., `"barba"`, `"cabelo"`, `"combo"`)
  - `name`: `string` (non-empty, e.g., `"Barba"`, `"Cabelo"`, `"Combo (Cabelo + Barba)"`)
  - `price`: `number` (positive number in BRL, e.g., `45`, `55`, `90`)
  - `duration`: `number` (positive integer in minutes, e.g., `30`, `45`, `60`)
- **Initial Seed Data**:
  1. `{ id: "barba", name: "Barba", price: 45, duration: 30 }`
  2. `{ id: "cabelo", name: "Cabelo", price: 55, duration: 45 }`
  3. `{ id: "combo", name: "Combo", price: 90, duration: 60 }`

### 1.2 `BarbershopInfo`
Represents static barbershop metadata returned by `GET /info`.
- **Fields**:
  - `name`: `string` (e.g., `"Barbearia NovoBot"`)
  - `address`: `string` (e.g., `"Av. Paulista, 1000 - Bela Vista, São Paulo - SP"`)
  - `openingHours`: `string` (e.g., `"Segunda a Sábado, das 09:00 às 19:00"`)

### 1.3 `TimeSlotQuery` & `SlotsResponse`
Represents daily schedule availability on `GET /slots?date=YYYY-MM-DD`.
- **Query Schema (`TimeSlotQuerySchema`)**:
  - `date`: `string` matching `/^\d{4}-\d{2}-\d{2}$/` (optional in query; defaults to current date `YYYY-MM-DD` if omitted by caller, or required `YYYY-MM-DD`).
- **Response Schema (`SlotsResponseSchema`)**:
  - `date`: `string` (`YYYY-MM-DD`)
  - `slots`: `string[]` (array of `"HH:mm"` strings representing unbooked slots, e.g., `["09:00", "10:00", "11:00", "14:00", "15:00", "16:00", "17:00"]`)

### 1.4 `Booking` & `CreateBookingInput`
Represents an appointment reservation in memory.
- **Input Schema (`CreateBookingInputSchema`)** for `POST /bookings`:
  - `customerPhone`: `string` (non-empty phone identifier)
  - `serviceId`: `string` (must match an existing `ServiceItem.id`)
  - `time`: `string` (must match `"HH:mm"` or `"YYYY-MM-DD HH:mm"` / valid slot)
  - `date`: `string` (`YYYY-MM-DD`, optional — defaults to current date if `time` is `"HH:mm"`)
- **Stored Entity (`BookingSchema`)**:
  - `id`: `string` (UUID or generated booking ID)
  - `customerPhone`: `string`
  - `serviceId`: `string`
  - `serviceName`: `string`
  - `price`: `number`
  - `date`: `string` (`YYYY-MM-DD`)
  - `time`: `string` (`HH:mm`)
  - `createdAt`: `string` (ISO timestamp)
- **State Transitions**:
  - **Created**: `POST /bookings` verifies the slot `(date, time)` is not already booked. If free, saves booking and removes `time` from available slots for `date`. If already booked, returns `409 Conflict`.
  - **Cancelled**: `DELETE /bookings/:customerPhone` removes active booking(s) for `customerPhone` and restores `time` to the available slots for `date`. If no active booking exists for `customerPhone`, returns `404 Not Found`.

---

## 2. Gateway Entities in `apps/whatsapp-gateway` (Port `3002`)

### 2.1 `SendMessageRequest`
Payload accepted by `POST /send-message`.
- **Fields**:
  - `to`: `string` (non-empty recipient phone/JID)
  - `text`: `string` (non-empty message text)

### 2.2 `InboundWhatsAppEvent`
Internal/adapter event representing a raw message from the WhatsApp Web API.
- **Fields**:
  - `sender`: `string` (non-empty customer phone/JID)
  - `messageText`: `string` (non-empty text content)
  - `messageId`: `string` (non-empty unique message ID)
  - `fromMe`: `boolean` (when `false`, triggers webhook emission to `http://localhost:3000/webhook/whatsapp`; when `true`, ignored)

---

## 3. Orchestrator Entities in `apps/bot-orchestrator` (Port `3000`)

### 3.1 `WhatsAppWebhookPayload`
Payload accepted by `POST /webhook/whatsapp`.
- **Fields**:
  - `sender`: `string` (non-empty string)
  - `messageText`: `string` (non-empty string)
  - `messageId`: `string` (non-empty string)

### 3.2 `IntentEnum`
Strictly typed Zod enum (`z.enum([...])`):
- `'AGENDAR'`
- `'CANCELAR'`
- `'VER_HORARIOS'`
- `'TABELA_PRECOS'`
- `'LOCALIZACAO'`
- `'ATENDIMENTO_HUMANO'`
- `'OUTRO'`

### 3.3 `JevClassificationResult`
Output contract of the Jev (System 1) classifier service.
- **Fields**:
  - `choice`: `IntentEnum`
  - `score`: `number` (`0.0` to `1.0`)
  - `extracted`: optional object `{ serviceId?: string, date?: string, time?: string }`
- **Routing Rule**:
  - If `score < 0.70` OR `choice === 'ATENDIMENTO_HUMANO'` → route immediately to human handoff message.
  - Otherwise, execute the deterministic branch for `choice` (`TABELA_PRECOS`, `LOCALIZACAO`, `VER_HORARIOS`, `AGENDAR`, `CANCELAR`) or System 2 lightweight LLM (`OUTRO`).
