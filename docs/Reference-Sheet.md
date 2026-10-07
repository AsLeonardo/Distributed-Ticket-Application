# Ticket System; Complete Command & Test Reference

A full reference sheet for exercising every endpoint, every profile, every payment
path, and every failure mode of the distributed ticket-sale system. This is not a
tutorial; it is an exhaustive list of commands you can copy, run, and verify, with
an explanation of what each piece does and what the expected response is.

> **Convention in this document**
> Commands are written for **Windows `cmd.exe`** (your shell). Where syntax differs
> on PowerShell / Linux / macOS, a note is included. JSON bodies on Windows `cmd`
> require **escaped double quotes** (`\"`), which is why every body looks like
> `-d "{\"key\":\"value\"}"`.

---
| Requirement            | Mechanism                                | Verification                       |
| ---------------------- | ---------------------------------------- | ---------------------------------- |
| `Load balancer`        | Nginx round-robin                        | /whoami alternates instances       |
| `API`                  | Spring Cloud Gateway                     | all traffic via :8000              |
| `Multiple instances`   | event-service-1/-2                       | both healthy, both serve           |
| `Async communication`  | RabbitMQ topic exchange                  | order.confirmed -> notification    |
| `Concurrency control`  | atomic DB UPDATE                         | 2nd reserve on qty=1 -> 409        |
| `Resilience`           | Resilience4j CB+retry                    | breaker opens at failureRate=1.0   |
| `Idempotency`          | gateway key + consumer PK + status guard | kill-test: one charge/ticket/email |
| `Observability`        | JSON logs + Prometheus + Grafana         | dashboard panels + structured logs |
| `Mocked payment/email` | standalone mock + log/DB notification    | admin/mode flips, /notifications   |
---


## Table of Contents

1. [Anatomy of a `curl` command](#1-anatomy-of-a-curl-command)
2. [The request/response model](#2-the-requestresponse-model)
3. [Topology; who lives where](#3-topology-who-lives-where)
4. [Ports & URLs cheat-sheet](#4-ports--urls-cheat-sheet)
5. [Lifecycle commands (Docker)](#5-lifecycle-commands-docker)
6. [Token & shell-variable handling](#6-token--shell-variable-handling)
7. [The Standard Happy Path (run this first)](#7-the-standard-happy-path-run-this-first)
8. [Endpoint-by-endpoint reference](#8-endpoint-by-endpoint-reference)
9. [Authorization / profile failure tests](#9-authorization--profile-failure-tests)
10. [Payment-method behavior matrix](#10-payment-method-behavior-matrix)
11. [Resilience: circuit breaker & retry tests](#11-resilience-circuit-breaker--retry-tests)
12. [Concurrency / oversell tests](#12-concurrency--oversell-tests)
13. [Idempotency tests](#13-idempotency-tests)
14. [Async / RabbitMQ / DLQ tests](#14-async--rabbitmq--dlq-tests)
15. [Load-balancing tests](#15-load-balancing-tests)
16. [Observability checks](#16-observability-checks)
17. [Full error-code catalogue](#17-full-error-code-catalogue)
18. [Reset & teardown](#18-reset--teardown)

---

## 1. Anatomy of a `curl` command

Every test in this sheet is an HTTP request made with `curl`. A request has four
moving parts. Here is one command fully dissected:

```
curl  -X POST  http://localhost:8000/orders  -H "Content-Type: application/json"  -H "Authorization: Bearer %USER_TOKEN%"  -d "{\"eventId\":1}"
└┬─┘  └──┬──┘  └──────────┬──────────────┘   └───────────────┬───────────────┘   └──────────────┬───────────────────┘   └──────┬──────┘
curl    method           URL                                header 1                            header 2                          body
```

| Piece                              | Meaning                                                                                                                                                                                                                                                                                                                     |
| ---------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `curl`                           | The command-line HTTP client. It opens a TCP connection, sends an HTTP request, prints the response.                                                                                                                                                                                                                        |
| `-X POST`                        | **`-X`** sets the HTTP **method** (verb). `POST` = create / perform an action. Other verbs: `GET` (read), `PATCH` (partial update), `PUT` (replace), `DELETE` (remove). If you omit `-X`, curl defaults to `GET`; *unless* `-d` is present, in which case it implicitly becomes `POST`. |
| `http://localhost:8000/orders`   | The**URL**. `http` = scheme, `localhost` = this machine, `8000` = the port the API Gateway listens on, `/orders` = the path that selects which controller handles the request.                                                                                                                                |
| `-H "..."`                       | **`-H`** adds a **header**; metadata about the request. You can repeat `-H` as many times as needed.                                                                                                                                                                                                      |
| `Content-Type: application/json` | Tells the server the**body** is JSON, so it parses it as JSON instead of form data or plain text. Required on every request that sends a JSON body.                                                                                                                                                                   |
| `Authorization: Bearer `         | Carries your identity.`Bearer` is the token *type*; the long string after it is the JWT. The gateway reads this to decide who you are and what you may do.                                                                                                                                                              |
| `-d "{...}"`                     | **`-d`** (data) is the **request body**; the payload you send. The presence of `-d` flips the default method to `POST`.                                                                                                                                                                                 |

### Other `curl` flags used in this sheet

| Flag                  | Meaning                                                                                                                                                                               |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `-i`                | **Include** the response headers and status line in the output (e.g. `HTTP/1.1 401 Unauthorized`). Use this when you care about the **status code**, not just the body. |
| `-v`                | **Verbose**; prints the entire request *and* response, including the TLS handshake and every header sent. Best debugging tool when something behaves unexpectedly.          |
| `-s`                | **Silent**; hides the progress meter. Useful when scripting loops.                                                                                                            |
| `-o file`           | Write the response**body** to a file instead of the screen.                                                                                                                     |
| `-w "%{http_code}"` | After the response, print a chosen variable (here, just the numeric status code). Handy for terse pass/fail checks.                                                                   |

---

## 2. The request/response model

Every interaction is **request -> response** over HTTP.

**What you SEND (request):**

- a **method** (`GET`/`POST`/`PATCH`); the intent
- a **URL path**; the resource
- **headers**; `Content-Type` (what you're sending) and `Authorization` (who you are)
- optionally a **body**; JSON payload

**What you RECEIVE (response):**

- a **status code**; the single most important field:
  - `2xx` success (`200 OK`, `201 Created`, `202 Accepted`)
  - `4xx` *your* fault (`400` bad body, `401` no/invalid token, `403` wrong role, `404` not found, `409` conflict / sold out)
  - `5xx` *server* fault (`500` unhandled, `503` dependency down)
- **headers**; metadata (content type, length)
- a **body**; usually JSON; the data or an `{"error": "..."}` object

By default `curl` prints only the body. Add `-i` to see the status code. **A blank
response with no body very often means a non-2xx status the gateway produced before
reaching a service**; always re-run with `-i` to see what actually happened.

### How a request flows through the system

```
you (curl)
       │  http://localhost:8000/...
ㅤ  ㅤ \/
┌──────────────┐   JWT validated here, role checked here,
│ API Gateway  │   X-User-Id / X-User-Role stamped here
│   :8000      │
└──────┬───────┘
       │ routed by path
       ├──────────────> /events/**  ─> Nginx LB :8080 ─> event-service-1 / event-service-2  ─> PostgreSQL
       ├──────────────> /orders/**  ─> order-service :8081  ─> PostgreSQL, RabbitMQ, payment-gateway-mock
       ├──────────────> /users/**   ─> order-service :8081
       ├──────────────> /auth/**    ─> order-service :8081
       └──────────────> /notifications/** ─> notification-service :8082

   payment-gateway-mock :8090  ──(webhook, async)──>  order-service /webhooks/payment-callback
   order-service ──(order.confirmed via RabbitMQ)──>  notification-service
```

Two things never go through the gateway:

- **`/admin/mode`** on the payment mock (`:8090`); you hit it directly; it's a test knob, not part of the product.
- **`/webhooks/payment-callback`**; the mock calls order-service directly by container name, server-to-server, inside the Docker network.

---

## 3. Topology; who lives where

| Service              | Container name           | Internal port | Published to host?    | Role                                        |
| -------------------- | ------------------------ | ------------- | --------------------- | ------------------------------------------- |
| API Gateway          | `api-gateway`          | 8000          | **Yes -> 8000** | Single entry point, JWT auth, routing       |
| Nginx LB             | `nginx-lb`             | 8080          | No                    | Round-robins event-service instances        |
| Event service #1     | `event-service-1`      | 8080          | No                    | Event CRUD, inventory                       |
| Event service #2     | `event-service-2`      | 8080          | No                    | Second instance (load-balanced)             |
| Order service        | `order-service`        | 8081          | No*                   | Users, auth, orders, payments, webhook      |
| Notification service | `notification-service` | 8082          | No*                   | Consumes `order.confirmed`, "sends" email |
| Payment gateway mock | `payment-gateway-mock` | 8090          | **Yes -> 8090** | Fake gateway + fault injection knobs        |
| PostgreSQL           | `postgres`             | 5432          | optional              | Single source of truth                      |
| RabbitMQ             | `rabbitmq`             | 5672 / 15672  | **15672 (UI)**  | Message broker                              |
| Prometheus           | `prometheus`           | 9090          | **Yes -> 9090** | Scrapes metrics                             |
| Grafana              | `grafana`              | 3000          | **Yes -> 3000** | Dashboards                                  |

\* If you published 8081/8082 in your compose for direct testing, you can hit them
directly too; the canonical path is through the gateway on 8000.

> **Why services aren't published:** only the gateway (and the mock + observability
> UIs) are reachable from your host. The services talk to each other by **container
name** over the internal Docker network. This is what makes the gateway-as-the-only-
> authenticator design sound; there's no public door into a service that bypasses auth.

---

## 4. Ports & URLs cheat-sheet

| What you want                    | URL                                            |
| -------------------------------- | ---------------------------------------------- |
| All app traffic (the front door) | `http://localhost:8000`                      |
| Payment mock admin knobs         | `http://localhost:8090/admin/mode`           |
| Prometheus targets               | `http://localhost:9090/targets`              |
| Prometheus query UI              | `http://localhost:9090`                      |
| Grafana                          | `http://localhost:3000` (admin / admin)      |
| RabbitMQ management UI           | `http://localhost:15672` (tickets / tickets) |

---

## 5. Lifecycle commands (Docker)

| Command                                       | What it does                                                                           |
| --------------------------------------------- | -------------------------------------------------------------------------------------- |
| `docker compose up --build`                 | Build images and start everything in the foreground (logs stream to your terminal).    |
| `docker compose up --build -d`              | Same, but**detached**; runs in the background, returns your prompt.            |
| `docker compose ps`                         | List all containers and their health (`healthy` / `starting` / `unhealthy`).     |
| `docker compose logs -f order-service`      | **Follow** (`-f`) the live logs of one service. Swap the name for any service. |
| `docker compose logs --tail 50 api-gateway` | Last 50 log lines of one service.                                                      |
| `docker compose restart prometheus`         | Restart a single container (e.g. if a scrape target started late).                     |
| `docker compose stop`                       | Stop all containers but keep them and their volumes.                                   |
| `docker compose down`                       | Stop and**remove** containers and the network (volumes survive).                 |
| `docker compose down -v`                    | Stop, remove,**and wipe volumes**; a completely fresh database next start.     |

> **The ID-reset trap.** Because the services use `ddl-auto: create-drop`, every
> `down -v` (and every rebuild that recreates the DB) wipes all data and restarts ID
> sequences at 1. After any reset, the seeded admin is re-created, but all events,
> users, and orders are gone. **Always re-create your test data and re-check actual
IDs with `GET /events` before assuming an ID.**

---

## 6. Token & shell-variable handling

The system uses JWT bearer tokens. You log in once, capture the token, and reuse it.

**Windows `cmd.exe`:**


- store a token in an environment variable
```cmd
set USER_TOKEN=eyJhbGciOiJIUzUxMiJ9....
```

- use it (cmd expands %VAR%)
```cmd
curl http://localhost:8000/orders/1 -H "Authorization: Bearer %USER_TOKEN%"
```

**PowerShell:**

```powershell
$USER_TOKEN = "eyJhbGc..."
```
```powershell
curl http://localhost:8000/orders/1 -H "Authorization: Bearer $USER_TOKEN"
```

**Linux / macOS (bash):**

```bash
USER_TOKEN="eyJhbGc..."
```
```bash
curl http://localhost:8000/orders/1 -H "Authorization: Bearer $USER_TOKEN"
```

> **Token expiry.** Tokens expire after 1 hour (`jwt.expiration-ms: 3600000`). If a
> previously-working call suddenly returns `401`, your token expired; log in again
> and re-`set` the variable. The token is a signed string containing your user id,
> email, and role; the gateway validates the signature with a shared secret, so no
> database lookup happens on each request.

### Decoding a token (to see what's inside)

A JWT is three base64url parts separated by dots: `header.payload.signature`. The
payload is readable (not encrypted, only signed). The middle segment of your USER
token decodes to something like:

```json
{"sub":"2","email":"leo@test.com","role":"USER","iat":1781212174,"exp":1781215774}
```

`sub` = subject (your user id), `role` = what the gateway authorizes against,
`iat`/`exp` = issued-at / expiry as Unix timestamps. **Never trust a token whose
signature doesn't verify**; that's exactly what the gateway checks.

---

## 7. The Standard Happy Path (run this first)

This is the canonical end-to-end run. Execute top to bottom on a fresh stack. Each
step notes the expected response.


#### 0. Bring the stack up (separate terminal, leave it running) 
```cmd
docker compose up --build -d
```
```cmd
docker compose ps
```
- wait until every row shows "healthy"



#### 1. Register a user (PUBLIC; no token needed) 
```cmd
curl -X POST http://localhost:8000/users -H "Content-Type: application/json" -d "{\"email\":\"leo@test.com\",\"name\":\"Leonardo\",\"password\":\"pass123\"}"
```
- EXPECT 201: {"email":"leo@test.com","name":"Leonardo","role":"USER","id":...}
- NOTE: role is forced to USER; you cannot self-register as ADMIN

#### 2. Log in as the user, capture the token 
```cmd
curl -X POST http://localhost:8000/auth/login -H "Content-Type: application/json" -d "{\"email\":\"leo@test.com\",\"password\":\"pass123\"}"
```
- EXPECT 200: {"token":"eyJ..."}
```cmd
set USER_TOKEN=<paste the token value here>
```

#### 3. Log in as the SEEDED admin, capture that token 
```cmd
curl -X POST http://localhost:8000/auth/login -H "Content-Type: application/json" -d "{\"email\":\"admin@tickets.com\",\"password\":\"admin123\"}"
```
- EXPECT 200: {"token":"eyJ..."}
```cmd
set ADMIN_TOKEN=<paste the admin token value here>
```

#### 4. Admin creates an event (ADMIN-only) 
```cmd
curl -X POST http://localhost:8000/events -H "Content-Type: application/json" -H "Authorization: Bearer %ADMIN_TOKEN%" -d "{\"name\":\"Show\",\"eventDate\":\"2026-12-01T20:00:00\",\"price\":100.00,\"availableQuantity\":5}"
```
- EXPECT 201: {"name":"Show",...,"availableQuantity":5,"id":1}

#### 5. Confirm the event id the DB actually assigned 
```cmd
curl http://localhost:8000/events -H "Authorization: Bearer %USER_TOKEN%"
```
- EXPECT 200: [ { ... "id":1 } ]   <- use this id below


#### 6. Admin adjusts quantity, then price (the other two admin ops) 
```cmd
curl -X PATCH http://localhost:8000/events/1/quantity -H "Content-Type: application/json" -H "Authorization: Bearer %ADMIN_TOKEN%" -d "{\"quantity\":10}"
```
```cmd
curl -X PATCH http://localhost:8000/events/1/price    -H "Content-Type: application/json" -H "Authorization: Bearer %ADMIN_TOKEN%" -d "{\"price\":120.00}"
```
- EXPECT 200 each, with the field updated in the returned object



#### 7. User creates an order (reserves a ticket). Note: NO userId in body 
```cmd
curl -X POST http://localhost:8000/orders -H "Content-Type: application/json" -H "Authorization: Bearer %USER_TOKEN%" -d "{\"eventId\":1}"
```
- EXPECT 201: {"userId":2,"eventId":1,...,"status":"PENDING","paymentMethod":null,"id":1}
- userId came from the TOKEN, not the body


#### 8a. Pay by credit card (SYNCHRONOUS; confirmed immediately) 
```cmd
curl -X POST http://localhost:8000/orders/1/pay -H "Content-Type: application/json" -H "Authorization: Bearer %USER_TOKEN%" -d "{\"paymentMethod\":\"CREDIT_CARD\",\"amount\":120.00}"
```
- EXPECT 200: {"orderStatus":"CONFIRMED","message":"..."}


#### 8b. (Alternative) Pay by PIX (ASYNCHRONOUS; pending, then webhook confirms) 
- First make a second order to test the async path:
```cmd
curl -X POST http://localhost:8000/orders -H "Content-Type: application/json" -H "Authorization: Bearer %USER_TOKEN%" -d "{\"eventId\":1}"
```
```cmd
curl -X POST http://localhost:8000/orders/2/pay -H "Content-Type: application/json" -H "Authorization: Bearer %USER_TOKEN%" -d "{\"paymentMethod\":\"PIX\",\"amount\":120.00}"
```
- EXPECT 200: {"orderStatus":"PENDING","message":"PIX gerado; aguardando confirmação..."}
- ...wait ~5 seconds (webhook delay)...
```cmd
curl http://localhost:8000/orders/2 -H "Authorization: Bearer %USER_TOKEN%"
```
- EXPECT 200: {... "status":"CONFIRMED"}   <- the gateway called back


#### 9. Confirm the async notification was produced (RabbitMQ -> consumer) 
```cmd
curl http://localhost:8000/notifications -H "Authorization: Bearer %USER_TOKEN%"
```
- EXPECT 200: [ {"orderId":1,"userId":2,"sentAt":"..."}, ... ]


#### 10. Read a single order back 
```cmd
curl http://localhost:8000/orders/1 -H "Authorization: Bearer %USER_TOKEN%"
```
- EXPECT 200: {... "status":"CONFIRMED"}

If all ten steps pass, the entire critical path works: auth -> role-gated admin CRUD ->
reservation -> both payment paths -> async notification.

---

## 8. Endpoint-by-endpoint reference

Below: every endpoint, its method, who may call it, the exact send shape, and the
exact receive shape.

### 8.1 `POST /users`; register (PUBLIC)

```cmd
curl -X POST http://localhost:8000/users -H "Content-Type: application/json" -d "{\"email\":\"a@b.com\",\"name\":\"Ana\",\"password\":\"secret\"}"
```

- **Auth:** none (public).
- **Send:** `email` (unique), `name`, `password`.
- **Receive 201:** `{"email":...,"name":...,"role":"USER","id":N}`; note password is never returned, role is always `USER`.
- **Receive 404/400** if email already registered (handler maps it).

### 8.2 `POST /auth/login`; obtain a token (PUBLIC)

```cmd
curl -X POST http://localhost:8000/auth/login -H "Content-Type: application/json" -d "{\"email\":\"a@b.com\",\"password\":\"secret\"}"
```

- **Auth:** none (public).
- **Send:** `email`, `password`.
- **Receive 200:** `{"token":"<jwt>"}`.
- **Wrong credentials:** error object (invalid credentials).

### 8.3 `GET /users`; list users (AUTHENTICATED)

```cmd
curl http://localhost:8000/users -H "Authorization: Bearer %ADMIN_TOKEN%"
```

- **Auth:** any valid token.
- **Receive 200:** array of users.

### 8.4 `POST /events`; create event (ADMIN ONLY)

```cmd
curl -X POST http://localhost:8000/events -H "Content-Type: application/json" -H "Authorization: Bearer %ADMIN_TOKEN%" -d "{\"name\":\"Show\",\"eventDate\":\"2026-12-01T20:00:00\",\"price\":100.00,\"availableQuantity\":5}"
```

- **Auth:** ADMIN. USER -> `403`.
- **Send:** `name`, `eventDate` (ISO-8601 `yyyy-MM-ddTHH:mm:ss`), `price` (decimal), `availableQuantity` (int).
- **Receive 201:** the created event with its `id`.

### 8.5 `GET /events`; list events (AUTHENTICATED)

```cmd
curl http://localhost:8000/events -H "Authorization: Bearer %USER_TOKEN%"
```

- **Auth:** any valid token.
- **Receive 200:** array of events.

### 8.6 `GET /events/{id}`; one event (AUTHENTICATED)

```cmd
curl http://localhost:8000/events/1 -H "Authorization: Bearer %USER_TOKEN%"
```

- **Receive 200:** the event, or `404` if no such id.

### 8.7 `PATCH /events/{id}/quantity`; change stock (ADMIN ONLY)

```cmd
curl -X PATCH http://localhost:8000/events/1/quantity -H "Content-Type: application/json" -H "Authorization: Bearer %ADMIN_TOKEN%" -d "{\"quantity\":10}"
```

- **Auth:** ADMIN. USER -> `403`.
- **Send:** `quantity` (the new absolute value).
- **Receive 200:** updated event.

### 8.8 `PATCH /events/{id}/price`; change price (ADMIN ONLY)

```cmd
curl -X PATCH http://localhost:8000/events/1/price -H "Content-Type: application/json" -H "Authorization: Bearer %ADMIN_TOKEN%" -d "{\"price\":120.00}"
```

- **Auth:** ADMIN. USER -> `403`.
- **Send:** `price` (new decimal).
- **Receive 200:** updated event.

### 8.9 `POST /events/{id}/reserve`; atomic decrement (INTERNAL-ish)

This is what order-service calls internally; you can hit it directly to test the
atomic decrement in isolation.

```cmd
curl -X POST http://localhost:8000/events/1/reserve -H "Authorization: Bearer %USER_TOKEN%"
```

- **Receive 200** if a unit was decremented; **409** if sold out (or event missing).

### 8.10 `POST /orders`; create order / reserve (AUTHENTICATED, USER flow)

```cmd
curl -X POST http://localhost:8000/orders -H "Content-Type: application/json" -H "Authorization: Bearer %USER_TOKEN%" -d "{\"eventId\":1}"
```

- **Auth:** any valid token; the buyer is the **token's** user id.
- **Send:** `eventId` only. **Do NOT send `userId`**; it is derived from `X-User-Id`.
- **Receive 201:** `{"userId":<from token>,"eventId":...,"status":"PENDING","expiresAt":...,"id":N}`.
- **Receive 409:** if the event is sold out.

### 8.11 `POST /orders/{id}/pay`; pay (AUTHENTICATED)

```cmd
curl -X POST http://localhost:8000/orders/1/pay -H "Content-Type: application/json" -H "Authorization: Bearer %USER_TOKEN%" -d "{\"paymentMethod\":\"CREDIT_CARD\",\"amount\":120.00}"
```

- **Send:** `paymentMethod`; either {`CREDIT_CARD`, `PIX`, `BOLETO`}, `amount`.
- **CREDIT_CARD ->** synchronous: `{"orderStatus":"CONFIRMED",...}` or declined/error.
- **PIX / BOLETO ->** asynchronous: `{"orderStatus":"PENDING","message":"... aguardando ..."}`; webhook confirms/expires ~5 s later.
- **Already paid ->** `{"orderStatus":"CONFIRMED","message":"Pagamento já confirmado."}` (idempotent no-op).

### 8.12 `GET /orders/{id}`; read an order (AUTHENTICATED)

```cmd
curl http://localhost:8000/orders/1 -H "Authorization: Bearer %USER_TOKEN%"
```

- **Receive 200:** the order with current `status`.
- **No token ->** `401`.

### 8.13 `GET /orders`; list orders (AUTHENTICATED)

```cmd
curl http://localhost:8000/orders -H "Authorization: Bearer %USER_TOKEN%"
```

### 8.14 `GET /notifications`; list "sent" emails (AUTHENTICATED)

```cmd
curl http://localhost:8000/notifications -H "Authorization: Bearer %USER_TOKEN%"
```

- **Receive 200:** array of `{orderId,userId,sentAt}`; one per confirmed order.

### 8.15 `GET /whoami`; which instance answered (AUTHENTICATED)

```cmd
curl http://localhost:8000/whoami -H "Authorization: Bearer %USER_TOKEN%"
```

- **Receive 200:** `{"instance":"event-service-1"}` or `event-service-2` alternates.

### 8.16 Payment-mock admin (DIRECT; port 8090, NO token, NOT via gateway)


- read current knobs
```cmd
curl http://localhost:8090/admin/mode
```
- set knobs (any subset of fields)
```cmd
curl -X POST http://localhost:8090/admin/mode -H "Content-Type: application/json" -d "{\"failureRate\":1.0,\"latencyMs\":100,\"declineRate\":0.0,\"webhookDelayMs\":5000,\"confirmRate\":1.0}"
```

Knobs: `failureRate` (0–1, fraction returning 503), `latencyMs` (added delay),
`declineRate` (0–1, fraction of card charges declined), `webhookDelayMs` (PIX/boleto
callback delay), `confirmRate` (0–1, fraction of PIX/boleto that CONFIRM vs EXPIRE).

---

## 9. Authorization / profile failure tests

These prove the admin/user boundary. The **failure** is the success here.

#### 9.1 No token at all -> 401 Unauthorized
```cmd
curl -i http://localhost:8000/orders/1
```
- EXPECT: HTTP/1.1 401 Unauthorized  (empty body)

#### 9.2 Garbage / tampered token -> 401
```cmd
curl -i http://localhost:8000/events -H "Authorization: Bearer not.a.real.token"
```
- EXPECT: HTTP/1.1 401 Unauthorized

#### 9.3 Missing "Bearer " prefix -> 401
```cmd
curl -i http://localhost:8000/events -H "Authorization: %USER_TOKEN%"
```
- EXPECT: HTTP/1.1 401 Unauthorized

#### 9.4 USER attempts to CREATE an event -> 403 Forbidden
```cmd
curl -i -X POST http://localhost:8000/events -H "Content-Type: application/json" -H "Authorization: Bearer %USER_TOKEN%" -d "{\"name\":\"Hack\",\"eventDate\":\"2026-12-01T20:00:00\",\"price\":50.00,\"availableQuantity\":5}"
```
- EXPECT: HTTP/1.1 403 Forbidden  (empty body; rejected at the gateway)

#### 9.5 USER attempts to CHANGE quantity -> 403
```cmd
curl -i -X PATCH http://localhost:8000/events/1/quantity -H "Content-Type: application/json" -H "Authorization: Bearer %USER_TOKEN%" -d "{\"quantity\":999}"
```
- EXPECT: HTTP/1.1 403 Forbidden

#### 9.6 USER attempts to CHANGE price -> 403
```cmd
curl -i -X PATCH http://localhost:8000/events/1/price -H "Content-Type: application/json" -H "Authorization: Bearer %USER_TOKEN%" -d "{\"price\":1.00}"
```
- EXPECT: HTTP/1.1 403 Forbidden

#### 9.7 ADMIN can do the user action too (buying is open to any authenticated account)
```cmd
curl -X POST http://localhost:8000/orders -H "Content-Type: application/json" -H "Authorization: Bearer %ADMIN_TOKEN%" -d "{\"eventId\":1}"
```
- EXPECT: 201; admins are authenticated accounts; the gate only restricts admin-ONLY ops

#### 9.8 Expired token -> 401 (wait >1h, or just observe after expiry)
```cmd
curl -i http://localhost:8000/orders -H "Authorization: Bearer %USER_TOKEN%"
```
- EXPECT after expiry: HTTP/1.1 401 Unauthorized -> log in again

#### 9.9 Self-registration cannot escalate to ADMIN (role is ignored if sent)
```cmd
curl -X POST http://localhost:8000/users -H "Content-Type: application/json" -d "{\"email\":\"sneaky@x.com\",\"name\":\"X\",\"password\":\"p\",\"role\":\"ADMIN\"}"
```
- EXPECT 201 with role STILL "USER"; the server forces USER and ignores the field

| Scenario                                 | Expected status   |
| ---------------------------------------- | ----------------- |
| Valid admin token + admin op             | `200` / `201` |
| Valid user token + admin op              | `403`           |
| Valid user token + user op               | `200` / `201` |
| No token + protected op                  | `401`           |
| Bad/expired/tampered token               | `401`           |
| Public op (register/login) without token | `200` / `201` |

---

## 10. Payment-method behavior matrix

The three methods are genuinely different code paths. Reset the mock to defaults
before this block:

```cmd
curl -X POST http://localhost:8090/admin/mode -H "Content-Type: application/json" -d "{\"failureRate\":0.0,\"latencyMs\":100,\"declineRate\":0.0,\"webhookDelayMs\":5000,\"confirmRate\":1.0}"
```

| Method          | Sync/Async   | Immediate response          | Final state               | How it confirms             |
| --------------- | ------------ | --------------------------- | ------------------------- | --------------------------- |
| `CREDIT_CARD` | Synchronous  | `CONFIRMED` (or declined) | `CONFIRMED`/`FAILED`  | Inline in the `/pay` call |
| `PIX`         | Asynchronous | `PENDING`                 | `CONFIRMED`/`EXPIRED` | Gateway webhook ~5 s later  |
| `BOLETO`      | Asynchronous | `PENDING`                 | `CONFIRMED`/`EXPIRED` | Gateway webhook ~5 s later  |

#### 10.1 CREDIT_CARD; confirmed inline
```cmd
curl -X POST http://localhost:8000/orders -H "Content-Type: application/json" -H "Authorization: Bearer %USER_TOKEN%" -d "{\"eventId\":1}"
```
```cmd
curl -X POST http://localhost:8000/orders/<id>/pay -H "Content-Type: application/json" -H "Authorization: Bearer %USER_TOKEN%" -d "{\"paymentMethod\":\"CREDIT_CARD\",\"amount\":120.00}"
```
- EXPECT immediately: {"orderStatus":"CONFIRMED",...}

#### 10.2 PIX; pending then confirmed
```cmd
curl -X POST http://localhost:8000/orders -H "Content-Type: application/json" -H "Authorization: Bearer %USER_TOKEN%" -d "{\"eventId\":1}"
```
```cmd
curl -X POST http://localhost:8000/orders/<id>/pay -H "Content-Type: application/json" -H "Authorization: Bearer %USER_TOKEN%" -d "{\"paymentMethod\":\"PIX\",\"amount\":120.00}"
```
- EXPECT now: {"orderStatus":"PENDING",...}
- wait 5s, then:
```cmd
curl http://localhost:8000/orders/<id> -H "Authorization: Bearer %USER_TOKEN%"
```
- EXPECT: status CONFIRMED

#### 10.3 BOLETO; identical async path
```cmd
curl -X POST http://localhost:8000/orders -H "Content-Type: application/json" -H "Authorization: Bearer %USER_TOKEN%" -d "{\"eventId\":1}"
```
```cmd
curl -X POST http://localhost:8000/orders/<id>/pay -H "Content-Type: application/json" -H "Authorization: Bearer %USER_TOKEN%" -d "{\"paymentMethod\":\"BOLETO\",\"amount\":120.00}"
```
- PENDING now, CONFIRMED after webhook

#### 10.4 PIX that EXPIRES (user never pays); set confirmRate to 0
```cmd
curl -X POST http://localhost:8090/admin/mode -H "Content-Type: application/json" -d "{\"confirmRate\":0.0}"
```
```cmd
curl -X POST http://localhost:8000/orders -H "Content-Type: application/json" -H "Authorization: Bearer %USER_TOKEN%" -d "{\"eventId\":1}"
```
```cmd
curl -X POST http://localhost:8000/orders/<id>/pay -H "Content-Type: application/json" -H "Authorization: Bearer %USER_TOKEN%" -d "{\"paymentMethod\":\"PIX\",\"amount\":120.00}"
```
- wait 5s
```cmd
curl http://localhost:8000/orders/<id> -H "Authorization: Bearer %USER_TOKEN%"
```
- EXPECT: status EXPIRED   <- webhook returned EXPIRED
- restore:
```cmd
curl -X POST http://localhost:8090/admin/mode -H "Content-Type: application/json" -d "{\"confirmRate\":1.0}"
```

#### 10.5 CREDIT_CARD that is DECLINED (business decline, NOT retried)
```cmd
curl -X POST http://localhost:8090/admin/mode -H "Content-Type: application/json" -d "{\"declineRate\":1.0}"
```
```cmd
curl -X POST http://localhost:8000/orders -H "Content-Type: application/json" -H "Authorization: Bearer %USER_TOKEN%" -d "{\"eventId\":1}"
```
```cmd
curl -i -X POST http://localhost:8000/orders/<id>/pay -H "Content-Type: application/json" -H "Authorization: Bearer %USER_TOKEN%" -d "{\"paymentMethod\":\"CREDIT_CARD\",\"amount\":120.00}"
```
- EXPECT: an error/declined response; the order goes FAILED; it is NOT retried (declines are non-retryable)
- restore:
```cmd
curl -X POST http://localhost:8090/admin/mode -H "Content-Type: application/json" -d "{\"declineRate\":0.0}"
```

> **Watch it happen:** in a second terminal run
> `docker compose logs -f payment-gateway-mock` while you fire these. You'll see
> `Card approved`, `PIX charge ... PENDING ... webhook in 5000ms`, and
> `Webhook delivered for order N -> CONFIRMED` / `-> EXPIRED`.

---

## 11. Resilience: circuit breaker & retry tests

The circuit breaker protects the order-service -> gateway call. Configuration recap:
sliding window 5, opens at 60% failure, stays open 10 s, half-open allows 2 trial calls.

#### 11.0 Watch the order-service logs in another terminal
```cmd
docker compose logs -f order-service
```

#### 11.1 Force 100% technical failure (503) at the gateway
```cmd
curl -X POST http://localhost:8090/admin/mode -H "Content-Type: application/json" -d "{\"failureRate\":1.0}"
```

#### 11.2 Fire several card payments; each retries 3x, then the breaker OPENS
- (make fresh orders first; reuse the loop or run a few by hand)
```cmd
curl -X POST http://localhost:8000/orders -H "Content-Type: application/json" -H "Authorization: Bearer %USER_TOKEN%" -d "{\"eventId\":1}"
```
```cmd
curl -i -X POST http://localhost:8000/orders/<id>/pay -H "Content-Type: application/json" -H "Authorization: Bearer %USER_TOKEN%" -d "{\"paymentMethod\":\"CREDIT_CARD\",\"amount\":120.00}"
```
- In the logs: 3 retry attempts per call (exponential backoff), then after the window
- fills with failures the breaker opens and calls FAIL FAST (no gateway hit) via fallback.
- Response: error indicating the payment service is unavailable.

#### 11.3 Recover; set failure back to 0
```cmd
curl -X POST http://localhost:8090/admin/mode -H "Content-Type: application/json" -d "{\"failureRate\":0.0}"
```
- After ~10s the breaker goes HALF-OPEN, allows 2 trial calls; on success it CLOSES.
- Fire one more payment and watch it succeed in the logs.
```cmd
curl -X POST http://localhost:8000/orders -H "Content-Type: application/json" -H "Authorization: Bearer %USER_TOKEN%" -d "{\"eventId\":1}"
```
```cmd
curl -X POST http://localhost:8000/orders/<id>/pay -H "Content-Type: application/json" -H "Authorization: Bearer %USER_TOKEN%" -d "{\"paymentMethod\":\"CREDIT_CARD\",\"amount\":120.00}"
```

#### 11.4 Timeout-driven trip; make the gateway slower than the 3s TimeLimiter
```cmd
curl -X POST http://localhost:8090/admin/mode -H "Content-Type: application/json" -d "{\"latencyMs\":8000}"
```
```cmd
curl -X POST http://localhost:8000/orders -H "Content-Type: application/json" -H "Authorization: Bearer %USER_TOKEN%" -d "{\"eventId\":1}"
```
```cmd
curl -i -X POST http://localhost:8000/orders/<id>/pay -H "Content-Type: application/json" -H "Authorization: Bearer %USER_TOKEN%" -d "{\"paymentMethod\":\"CREDIT_CARD\",\"amount\":120.00}"
```
- The call exceeds the time limit -> counts as a failure -> contributes to opening the breaker.
- restore:
```cmd
curl -X POST http://localhost:8090/admin/mode -H "Content-Type: application/json" -d "{\"latencyMs\":100}"
```

| Knob                | Effect on resilience | Expected breaker behavior                                                               |
| ------------------- | -------------------- | --------------------------------------------------------------------------------------- |
| `failureRate:1.0` | every call 503       | retries exhaust -> breaker opens -> fast-fail                                           |
| `failureRate:0.0` | healthy              | breaker half-opens then closes                                                          |
| `latencyMs:8000`  | exceeds 3 s limit    | timeouts count as failures -> contribute to opening                                     |
| `declineRate:1.0` | business decline     | **NOT** retried, breaker **not** tripped; declines are ignored exceptions |

> **The key distinction to call out:** a `503`/timeout is a *technical* failure and
> IS retried + counts toward the breaker; a *declined card* is a *business* outcome
> and is NOT retried and does NOT trip the breaker. Test 10.5 vs 11.2 proves both.

---

## 12. Concurrency / oversell tests

Proves the atomic DB decrement: you can never sell more tickets than exist, even
across the two load-balanced event-service instances sharing one database.

#### 12.1 Create an event with exactly 1 ticket
```cmd
curl -X POST http://localhost:8000/events -H "Content-Type: application/json" -H "Authorization: Bearer %ADMIN_TOKEN%" -d "{\"name\":\"OneSeat\",\"eventDate\":\"2026-12-01T20:00:00\",\"price\":50.00,\"availableQuantity\":1}"
```
- note its id (say 2)

#### 12.2 Reserve it twice in a row; second must 409
```cmd
curl -i -X POST http://localhost:8000/orders -H "Content-Type: application/json" -H "Authorization: Bearer %USER_TOKEN%" -d "{\"eventId\":2}"
```
- EXPECT 201 (ticket taken)
```cmd
curl -i -X POST http://localhost:8000/orders -H "Content-Type: application/json" -H "Authorization: Bearer %USER_TOKEN%" -d "{\"eventId\":2}"
```
- EXPECT 409 Conflict: {"error":"No tickets available for event: 2"}

#### 12.3 Concurrent burst (PowerShell); fire 10 reservations at a 5-seat event,
`exactly 5 should succeed, 5 should 409. Create a 5-seat event first (id=3).`

PowerShell concurrency burst:

```powershell
$token = "PASTE_USER_TOKEN"
1..10 | ForEach-Object -Parallel {
  curl.exe -s -o NUL -w "%{http_code}`n" -X POST http://localhost:8000/orders `
    -H "Content-Type: application/json" `
    -H "Authorization: Bearer $using:token" `
    -d '{"eventId":3}'
} -ThrottleLimit 10
# EXPECT: five "201" and five "409" lines; never six 201s. The DB's atomic
# UPDATE ... WHERE available_quantity > 0 guarantees no oversell.
```

Linux/macOS equivalent:

```bash
TOKEN="PASTE_USER_TOKEN"
for i in $(seq 1 10); do
  curl -s -o /dev/null -w "%{http_code}\n" -X POST http://localhost:8000/orders \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $TOKEN" \
    -d '{"eventId":3}' &
done
wait
# EXPECT: five 201, five 409
```

> **Why this is safe across instances:** both `event-service-1` and `event-service-2`
> issue the same atomic `UPDATE events SET available_quantity = available_quantity - 1 WHERE id = ? AND available_quantity > 0`. The database, the single source of truth
> serializes them. An in-memory counter could not coordinate across two JVMs; the DB can.

---

## 13. Idempotency tests

Three layers are idempotent: the gateway (by idempotency key), the order-payment
(by order status), and the notification consumer (by orderId primary key).

#### 13.1 Pay the same order twice; second is a no-op, not a double charge
```cmd
curl -X POST http://localhost:8000/orders -H "Content-Type: application/json" -H "Authorization: Bearer %USER_TOKEN%" -d "{\"eventId\":1}"
```
```cmd
curl -X POST http://localhost:8000/orders/<id>/pay -H "Content-Type: application/json" -H "Authorization: Bearer %USER_TOKEN%" -d "{\"paymentMethod\":\"CREDIT_CARD\",\"amount\":120.00}"
```
- EXPECT: {"orderStatus":"CONFIRMED",...}
```cmd
curl -X POST http://localhost:8000/orders/<id>/pay -H "Content-Type: application/json" -H "Authorization: Bearer %USER_TOKEN%" -d "{\"paymentMethod\":\"CREDIT_CARD\",\"amount\":120.00}"
```
- EXPECT: {"orderStatus":"CONFIRMED","message":"Pagamento já confirmado."}  <- guarded no-op

#### 13.2 The kill-test (mid-flight kill + retry -> one charge, one ticket, one email)
- Slow the gateway so you can interrupt the call:
```cmd
curl -X POST http://localhost:8090/admin/mode -H "Content-Type: application/json" -d "{\"latencyMs\":8000}"
```
```cmd
curl -X POST http://localhost:8000/orders -H "Content-Type: application/json" -H "Authorization: Bearer %USER_TOKEN%" -d "{\"eventId\":1}"
```
- Fire the payment, then press Ctrl+C after ~2 seconds (client abandons):
```cmd
curl -X POST http://localhost:8000/orders/<id>/pay -H "Content-Type: application/json" -H "Authorization: Bearer %USER_TOKEN%" -d "{\"paymentMethod\":\"CREDIT_CARD\",\"amount\":120.00}"
```
```cmd
^C
```
- Reset latency and RETRY the exact same payment (same order -> same idempotency key):
```cmd
curl -X POST http://localhost:8090/admin/mode -H "Content-Type: application/json" -d "{\"latencyMs\":100}"
```
```cmd
curl -X POST http://localhost:8000/orders/<id>/pay -H "Content-Type: application/json" -H "Authorization: Bearer %USER_TOKEN%" -d "{\"paymentMethod\":\"CREDIT_CARD\",\"amount\":120.00}"
```
- Now verify exactly one of everything:
```cmd
curl http://localhost:8000/orders/<id> -H "Authorization: Bearer %USER_TOKEN%"
```
- status CONFIRMED (once)
```cmd
curl http://localhost:8000/notifications -H "Authorization: Bearer %USER_TOKEN%"
```
- one email row for this order
```cmd
curl http://localhost:8000/events -H "Authorization: Bearer %USER_TOKEN%"
```
- quantity decremented exactly once

In `docker compose logs payment-gateway-mock` you'll see `Idempotent hit for key order-<id>` on the retry; the gateway returned the original charge instead of
charging again.

#### 13.3 Duplicate notification (consumer idempotency)
`Publish the same order.confirmed event twice via the RabbitMQ UI (see §14)`\
`or pay twice. EXPECT: /notifications still shows ONE row for that orderId`\
`and the consumer logs "Email already sent ... skipping duplicate".`

---

## 14. Async / RabbitMQ / DLQ tests

#### 14.1 Confirm an order and watch the event flow to the consumer
```cmd
curl -X POST http://localhost:8000/orders -H "Content-Type: application/json" -H "Authorization: Bearer %USER_TOKEN%" -d "{\"eventId\":1}"
```
```cmd
curl -X POST http://localhost:8000/orders/<id>/pay -H "Content-Type: application/json" -H "Authorization: Bearer %USER_TOKEN%" -d "{\"paymentMethod\":\"CREDIT_CARD\",\"amount\":120.00}"
```
```cmd
curl http://localhost:8000/notifications -H "Authorization: Bearer %USER_TOKEN%"
```
- EXPECT: a row appears for this order (produced asynchronously by the consumer)

**RabbitMQ management UI**; `http://localhost:15672` (tickets / tickets):

- **Exchanges** tab -> `tickets.exchange` (topic) and `tickets.dlx` (dead-letter).
- **Queues** tab -> `order.confirmed.queue` (live) and `order.confirmed.dlq` (poison).
- Watch the message-rate graph spike when you confirm an order.

**Manually publish a good message** (UI -> Exchanges -> `tickets.exchange` -> Publish):

- Routing key: `order.confirmed`
- Payload: `{"orderId":1,"userId":2,"eventId":1}`
- Result: the consumer processes it (idempotency may skip if already sent).

**Poison-message / DLQ test** (UI -> publish to `tickets.exchange`):

- Routing key: `order.confirmed`
- Payload: `not valid json` (or `{"garbage":true}`)
- Result: the consumer fails to deserialize, the message is rejected, RabbitMQ routes
  it to `tickets.dlx` -> it lands in `order.confirmed.dlq`. Open the **Queues -> DLQ**
  and use **Get messages** to see the quarantined poison message. The live queue keeps
  flowing; one bad message can't block the pipeline.

---

## 15. Load-balancing tests

#### 15.1 Hit /whoami repeatedly; instances alternate (Nginx round-robin)
```cmd
curl http://localhost:8000/whoami -H "Authorization: Bearer %USER_TOKEN%"
```
```cmd
curl http://localhost:8000/whoami -H "Authorization: Bearer %USER_TOKEN%"
curl http://localhost:8000/whoami -H "Authorization: Bearer %USER_TOKEN%"
curl http://localhost:8000/whoami -H "Authorization: Bearer %USER_TOKEN%"
```
- EXPECT: {"instance":"event-service-1"} and {"instance":"event-service-2"} alternating

#### 15.2 Prove it survives an instance dying
```cmd
docker compose stop event-service-2
```
```cmd
curl http://localhost:8000/whoami -H "Authorization: Bearer %USER_TOKEN%"
```
```cmd
curl http://localhost:8000/whoami -H "Authorization: Bearer %USER_TOKEN%"
```
- EXPECT: every response now says event-service-1; the system stays up
```cmd
docker compose start event-service-2
```
- after it's healthy, alternation resumes

---

## 16. Observability checks

#### 16.1 Prometheus is scraping every service
`- Open in a browser:`\
`- http://localhost:9090/targets`\
`- EXPECT: event-service (x2), order-service, notification-service all "UP"`

#### 16.2 Query the custom business metric in Prometheus
`- http://localhost:9090  ->  query box  ->  tickets_sold_total  ->  Execute`\
`- EXPECT: a value that increments each time an order is CONFIRMED`

PromQL queries to try in Prometheus or Grafana:

| Goal               | Query                                                                       |
| ------------------ | --------------------------------------------------------------------------- |
| Throughput (req/s) | `rate(http_server_requests_seconds_count[1m])`                            |
| Latency p95        | `histogram_quantile(0.95, rate(http_server_requests_seconds_bucket[1m]))` |
| Error rate (5xx/s) | `rate(http_server_requests_seconds_count{status=~"5.."}[1m])`             |
| Tickets sold       | `tickets_sold_total`                                                      |

#### 16.3 Confirm logs are structured JSON
```cmd
docker compose logs --tail 20 order-service
```
- EXPECT: each line is a JSON object with "@timestamp","level","message", and a
- "correlationId" field (per-request trace id)

#### 16.4 Grafana
`http://localhost:3000  (admin / admin)`\
`Connections -> Data sources -> Prometheus should be pre-provisioned and "working".`\
`Build panels with the PromQL above, or view your saved dashboard.`

---

## 17. Full error-code catalogue

Every non-2xx you can intentionally produce, and how:

| Status                        | Meaning                      | How to trigger it                                           |
| ----------------------------- | ---------------------------- | ----------------------------------------------------------- |
| `200 OK`                    | success (read / sync action) | any valid GET, a confirmed card payment                     |
| `201 Created`               | resource created             | register user, create event, create order                   |
| `202 Accepted`              | accepted, processing async   | (internally, the mock's PIX/boleto path)                    |
| `400 Bad Request`           | malformed body               | send broken JSON, e.g.`-d "{bad"`                         |
| `401 Unauthorized`          | no / invalid / expired token | omit `Authorization`, or use a tampered/expired token     |
| `403 Forbidden`             | authenticated but wrong role | USER calls `POST /events` or `PATCH /events/**`         |
| `404 Not Found`             | no such resource             | `GET /events/9999`, pay a non-existent order id           |
| `409 Conflict`              | business conflict            | reserve a sold-out event; double-reserve a 1-seat event     |
| `500 Internal Server Error` | unhandled server error       | (should not occur in normal tests; indicates a bug)         |
| `503 Service Unavailable`   | dependency failed            | set mock `failureRate:1.0`, observe gateway-call failures |

#### 17.1 400; malformed JSON body
```cmd
curl -i -X POST http://localhost:8000/orders -H "Content-Type: application/json" -H "Authorization: Bearer %USER_TOKEN%" -d "{\"eventId\":}"
```
- EXPECT: 400 Bad Request

#### 17.2 404; pay an order that doesn't exist
```cmd
curl -i -X POST http://localhost:8000/orders/99999/pay -H "Content-Type: application/json" -H "Authorization: Bearer %USER_TOKEN%" -d "{\"paymentMethod\":\"CREDIT_CARD\",\"amount\":1.00}"
```
- EXPECT: 404 (order not found)   [known smell: when the breaker is involved this can
- surface as a generic 'service unavailable'; see the report's limitations section]

#### 17.3 409; reserve a sold-out event (after exhausting stock)
```cmd
curl -i -X POST http://localhost:8000/orders -H "Content-Type: application/json" -H "Authorization: Bearer %USER_TOKEN%" -d "{\"eventId\":2}"
```
- EXPECT: 409 Conflict

---

## 18. Reset & teardown

- Reset payment mock to clean defaults (no fault injection)
```cmd
curl -X POST http://localhost:8090/admin/mode -H "Content-Type: application/json" -d "{\"failureRate\":0.0,\"latencyMs\":100,\"declineRate\":0.0,\"webhookDelayMs\":5000,\"confirmRate\":1.0}"
```
- Stop everything, keep data
```cmd
docker compose stop
```
- Stop and remove containers + network, KEEP volumes (DB persists)
```cmd
docker compose down
```
- Full wipe; fresh database, IDs restart at 1, admin re-seeded
```cmd
docker compose down -v
```
- Rebuild from scratch
```cmd
docker compose up --build -d
```

---

## Quick reference card

```
LOGIN     POST /auth/login            {email,password}            -> {token}
REGISTER  POST /users                 {email,name,password}       -> user (role USER)
EVENT+    POST /events           [A]  {name,eventDate,price,availableQuantity}
EVENTqty  PATCH /events/{id}/quantity [A]  {quantity}
EVENTprc  PATCH /events/{id}/price    [A]  {price}
EVENTS    GET  /events                                            -> [events]
ORDER+    POST /orders                {eventId}      (userId from token)  -> order PENDING
PAY       POST /orders/{id}/pay       {paymentMethod,amount}
ORDER     GET  /orders/{id}                                       -> order
NOTIFY    GET  /notifications                                     -> [emails]
WHOAMI    GET  /whoami                                            -> {instance}
KNOBS     POST http://localhost:8090/admin/mode   (direct, no token)

[A] = ADMIN only. All others except register/login require a Bearer token.
```

