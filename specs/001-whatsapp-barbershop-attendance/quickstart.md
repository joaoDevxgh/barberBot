# Quickstart & Validation Guide: Intelligent WhatsApp Barbershop Attendance Monorepo

**Feature**: `001-whatsapp-barbershop-attendance`  
**References**: [data-model.md](./data-model.md) | [contracts/http-contracts.md](./contracts/http-contracts.md)

## 1. Prerequisites

- **Node.js**: v20+ LTS
- **Package Manager**: `npm` (v10+, with native Workspaces support) or `pnpm`
- **Environment Configuration**: Copy `.env.example` to `.env` in root (or per app) with:
  ```env
  ORCHESTRATOR_PORT=3000
  BARBERSHOP_MOCK_PORT=3001
  WHATSAPP_GATEWAY_PORT=3002
  ORCHESTRATOR_WEBHOOK_URL=http://localhost:3000/webhook/whatsapp
  BARBERSHOP_API_URL=http://localhost:3001
  WHATSAPP_GATEWAY_URL=http://localhost:3002
  ```

## 2. Setup & Installation

From the repository root (`C:\novoBot`):

```powershell
npm install
```

## 3. Running Typechecks & Automated Tests (Vitest)

Verify strict TypeScript (`strict: true`, `noImplicitAny: true`, zero `any`) and run isolated Vitest unit/integration suites across all 3 apps:

```powershell
npm run typecheck
npm test
```

## 4. Starting All 3 Services Simultaneously

Start `apps/barbershop-mock` (3001), `apps/whatsapp-gateway` (3002), and `apps/bot-orchestrator` (3000) in development mode:

```powershell
npm run dev
```

## 5. End-to-End Validation Scenarios

With `npm run dev` running, execute the following HTTP checks to validate each service and the end-to-end orchestration pipeline:

### Scenario A: Verify `barbershop-mock` (Port `3001`) Directly
1. **List Services (`GET /services`)**:
   ```powershell
   Invoke-RestMethod -Uri "http://localhost:3001/services" -Method Get
   ```
   *Expected*: Returns `Barba` (R$ 45), `Cabelo` (R$ 55), and `Combo` (R$ 90).

2. **Check Available Slots (`GET /slots?date=2026-10-15`)**:
   ```powershell
   Invoke-RestMethod -Uri "http://localhost:3001/slots?date=2026-10-15" -Method Get
   ```
   *Expected*: Returns `{ date: "2026-10-15", slots: [...] }`.

3. **Check Location & Info (`GET /info`)**:
   ```powershell
   Invoke-RestMethod -Uri "http://localhost:3001/info" -Method Get
   ```
   *Expected*: Returns barbershop name, address, and opening hours.

### Scenario B: Verify `bot-orchestrator` (Port `3000`) Webhook & Jev Routing
1. **Price Table Query (`TABELA_PRECOS`)**:
   ```powershell
   Invoke-RestMethod -Uri "http://localhost:3000/webhook/whatsapp" -Method Post -ContentType "application/json" -Body '{"sender":"5511999999999","messageText":"Qual a tabela de preços?","messageId":"m1"}'
   ```
   *Expected*: Jev classifies `TABELA_PRECOS` (`>= 0.70`), fetches `http://localhost:3001/services`, dispatches the formatted table to `http://localhost:3002/send-message`, and returns `200 OK`.

2. **Booking Flow (`AGENDAR`) & Cancellation (`CANCELAR`)**:
   ```powershell
   # Book Cabelo at 14:00
   Invoke-RestMethod -Uri "http://localhost:3000/webhook/whatsapp" -Method Post -ContentType "application/json" -Body '{"sender":"5511999999999","messageText":"Quero agendar cabelo às 14:00","messageId":"m2"}'

   # Cancel booking
   Invoke-RestMethod -Uri "http://localhost:3000/webhook/whatsapp" -Method Post -ContentType "application/json" -Body '{"sender":"5511999999999","messageText":"Quero cancelar meu agendamento","messageId":"m3"}'
   ```
   *Expected*: Creates booking via `POST http://localhost:3001/bookings`, then cancels via `DELETE http://localhost:3001/bookings/5511999999999`, dispatching confirmations to `http://localhost:3002/send-message`.

3. **Human Handoff / Low Confidence Fallback (`< 0.70` or `ATENDIMENTO_HUMANO`)**:
   ```powershell
   Invoke-RestMethod -Uri "http://localhost:3000/webhook/whatsapp" -Method Post -ContentType "application/json" -Body '{"sender":"5511999999999","messageText":"Quero falar com um atendente humano","messageId":"m4"}'
   ```
   *Expected*: Immediately dispatches human transfer message to `http://localhost:3002/send-message`.
