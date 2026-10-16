# HTTP/REST Service Contracts

**Feature**: `001-whatsapp-barbershop-attendance`  
**Protocol**: HTTP/REST (`application/json`) validated via **Zod**

---

## 1. `apps/barbershop-mock` (`http://localhost:3001`)

### `GET /services`
Lists all barbershop services in memory.
- **Response `200 OK`**:
  ```json
  [
    { "id": "barba", "name": "Barba", "price": 45, "duration": 30 },
    { "id": "cabelo", "name": "Cabelo", "price": 55, "duration": 45 },
    { "id": "combo", "name": "Combo", "price": 90, "duration": 60 }
  ]
  ```

### `GET /slots?date=YYYY-MM-DD`
Returns available (unbooked) slots for the given date (defaults to today's `YYYY-MM-DD` if omitted).
- **Query Parameters**:
  - `date` (optional, format `YYYY-MM-DD`)
- **Response `200 OK`**:
  ```json
  {
    "date": "2026-10-15",
    "slots": ["09:00", "10:00", "11:00", "14:00", "15:00", "16:00", "17:00"]
  }
  ```

### `POST /bookings`
Creates a new booking in memory.
- **Request Body**:
  ```json
  {
    "customerPhone": "5511999999999",
    "serviceId": "cabelo",
    "time": "14:00",
    "date": "2026-10-15"
  }
  ```
  *(Note: `date` is optional if `time` is a slot for today or if `time` includes the date).*
- **Response `201 Created`**:
  ```json
  {
    "id": "bk_1",
    "customerPhone": "5511999999999",
    "serviceId": "cabelo",
    "serviceName": "Cabelo",
    "price": 55,
    "date": "2026-10-15",
    "time": "14:00"
  }
  ```
- **Response `400 Bad Request`**: Invalid payload or unknown `serviceId`.
- **Response `409 Conflict`**: Requested slot is already occupied.

### `DELETE /bookings/:customerPhone`
Cancels an active booking for `:customerPhone` and releases the time slot.
- **Path Parameters**:
  - `customerPhone`: string
- **Response `200 OK`**:
  ```json
  {
    "cancelled": true,
    "customerPhone": "5511999999999"
  }
  ```
- **Response `404 Not Found`**: No active booking found for `:customerPhone`.

### `GET /info`
Returns barbershop location and opening hours.
- **Response `200 OK`**:
  ```json
  {
    "name": "Barbearia NovoBot",
    "address": "Rua das Tesouras, 123 - Centro, São Paulo - SP",
    "openingHours": "Segunda a Sábado, das 09:00 às 19:00"
  }
  ```

---

## 2. `apps/whatsapp-gateway` (`http://localhost:3002`)

### `POST /send-message`
Receives an outbound message from `bot-orchestrator` and dispatches it to the WhatsApp Web client adapter.
- **Request Body**:
  ```json
  {
    "to": "5511999999999",
    "text": "Olá! Segue nossa tabela de preços:\n- Barba: R$ 45 (30 min)\n- Cabelo: R$ 55 (45 min)\n- Combo: R$ 90 (60 min)"
  }
  ```
- **Response `200 OK`**:
  ```json
  {
    "sent": true,
    "to": "5511999999999"
  }
  ```

### Outbound Webhook Emitter (Gateway → Orchestrator)
Triggered when `whatsapp-gateway` receives a message from WhatsApp Web with `fromMe === false`:
- **Target**: `POST http://localhost:3000/webhook/whatsapp`
- **Payload**:
  ```json
  {
    "sender": "5511999999999",
    "messageText": "Qual o valor do corte?",
    "messageId": "msg_abc123"
  }
  ```

---

## 3. `apps/bot-orchestrator` (`http://localhost:3000`)

### `POST /webhook/whatsapp`
Ingests customer messages from `whatsapp-gateway`, classifies via **Jev SDK** (`Choice` and `Score`), executes HTTP calls to `barbershop-mock` (`3001`) or System 2 LLM (`OUTRO`), and dispatches the reply to `whatsapp-gateway` (`POST http://localhost:3002/send-message`).
- **Request Body**:
  ```json
  {
    "sender": "5511999999999",
    "messageText": "Quero saber a tabela de preços",
    "messageId": "msg_001"
  }
  ```
- **Response `200 OK`**:
  ```json
  {
    "processed": true,
    "intent": "TABELA_PRECOS",
    "score": 0.95,
    "replyText": "Confira nossos serviços e valores:\n- Barba: R$ 45 (30 min)\n- Cabelo: R$ 55 (45 min)\n- Combo: R$ 90 (60 min)"
  }
  ```
- **Response `400 Bad Request`**: Zod validation error on malformed webhook payload.
