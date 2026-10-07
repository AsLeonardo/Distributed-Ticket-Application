
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

## Start it
`docker compose up --build`

### 1. Prove load balancing:
`curl http://localhost:8000/whoami` \
`curl http://localhost:8000/whoami`

### 2. Register a user:
`curl -X POST http://localhost:8000/users -H "Content-Type: application/json" -d "{\"email\":\"leo@test.com\",\"name\":\"Leonardo\"}"`

### 3. Create an event:
`curl -X POST http://localhost:8000/events -H "Content-Type: application/json" -d "{\"name\":\"Show D\",\"eventDate\":\"2026-11-01T20:00:00\",\"price\":120.00,\"availableQuantity\":3}"`

### 4. List events (note the id, usage bellow):
`curl http://localhost:8000/events`

### 5. Create an order (reserve a ticket):
`curl -X POST http://localhost:8000/orders -H "Content-Type: application/json" -d "{\"userId\":1,\"eventId\":1}"`

### 6. Pay (confirms order + publishes order.confirmed):
`curl -X POST http://localhost:8000/orders/1/pay -H "Content-Type: application/json" -d "{\"paymentMethod\":\"CREDIT_CARD\",\"amount\":120.00}"`

### 7. Check the async notification arrived:
`curl http://localhost:8000/notifications`

### Circuit breaker demo (admin endpoint not routed through gateway, use 8090)
100% failure \
`curl -X POST http://localhost:8090/admin/mode -H "Content-Type: application/json" -d "{\"failureRate\":1.0}"`

watch retries then breaker opening in logs \
`curl -X POST http://localhost:8000/orders/1/pay -H "Content-Type: application/json" -d "{\"paymentMethod\":\"CREDIT_CARD\",\"amount\":120.00}"`

recover \
`curl -X POST http://localhost:8090/admin/mode -H "Content-Type: application/json" -d "{\"failureRate\":0.0}"`

### Oversell prevention demo (create event with quantity 1, reserve it twice, second must 409):
`curl -X POST http://localhost:8000/events -H "Content-Type: application/json" -d "{\"name\":\"Soldout\",\"eventDate\":\"2026-12-01T20:00:00\",\"price\":50.00,\"availableQuantity\":1}"`

`curl -X POST http://localhost:8000/orders -H "Content-Type: application/json" -d "{\"userId\":1,\"eventId\":1}"`

`curl -X POST http://localhost:8000/orders -H "Content-Type: application/json" -d "{\"userId\":1,\"eventId\":1}"`

### http://localhost:8000 -> API Gateway (all app traffic)
### http://localhost:9090/targets -> Prometheus (confirm all targets UP)
### http://localhost:3000 -> Grafana (admin/admin)
### http://localhost:15672 -> RabbitMQ management (tickets/tickets)

`docker compose ps`ㅤㅤㅤㅤㅤㅤㅤㅤㅤㅤㅤ ㅤ# status of all containers       \
`docker compose logs -f order-service`ㅤ ㅤㅤ# follow one service's logs  \
`docker compose logs -f payment-gateway-mock`                            \
`docker compose down`ㅤㅤㅤㅤㅤㅤㅤㅤㅤㅤㅤ # stop everything                  \
`docker compose down -v`ㅤㅤㅤㅤㅤㅤㅤ ㅤ ㅤ # stop + wipe volumes (fresh DB)

### Grafana -> Dashboards -> New -> Add visualization -> Select Prometheus -> Paste The Query -> Set a Title -> Apply

#### Throughput (requests per second):
`rate(http_server_requests_seconds_count[1m])`

#### Latency (95th percentile, seconds):
`histogram_quantile(0.95, rate(http_server_requests_seconds_bucket[1m]))`

#### Error rate (non-2xx responses per second):
`rate(http_server_requests_seconds_count{status=~"5.."}[1m])`

#### Custom business metric:
`tickets_sold_total`


#### scenario:
a client fires a payment, the network drops mid-flight (client never sees the response), the client retries with the same key; and the system ends with exactly one charge and one ticket.

set up fresh order
`curl -X POST http://localhost:8000/events -H "Content-Type: application/json" -d "{\"name\":\"Idem Test\",\"eventDate\":\"2026-12-20T20:00:00\",\"price\":90.00,\"availableQuantity\":5}"`
`curl -X POST http://localhost:8000/orders -H "Content-Type: application/json" -d "{\"userId\":1,\"eventId\":1}"`

make the gateway slow so you have time to kill the request
`curl -X POST http://localhost:8090/admin/mode -H "Content-Type: application/json" -d "{\"latencyMs\":8000}"`

fire the payment, then hit Ctrl+C after ~2 seconds; client abandons, but the charge may have landed
`curl -X POST http://localhost:8000/orders/1/pay -H "Content-Type: application/json" -d "{\"paymentMethod\":\"CREDIT_CARD\",\"amount\":90.00}"`
^C

Reset latency and retry the exact same payment (same order -> same idempotency key):
`curl -X POST http://localhost:8090/admin/mode -H "Content-Type: application/json" -d "{\"latencyMs\":100}"`
`curl -X POST http://localhost:8000/orders/1/pay -H "Content-Type: application/json" -d "{\"paymentMethod\":\"CREDIT_CARD\",\"amount\":90.00}"`

Now prove the end state; exactly one of everything:
order is CONFIRMED, not double-processed
`curl http://localhost:8000/orders/1`

exactly one email
`curl http://localhost:8000/notifications`

inventory decremented exactly once (started at 5, should be 4)
`curl http://localhost:8000/events`