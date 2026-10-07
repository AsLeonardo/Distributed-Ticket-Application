
# create event
curl -X POST http://localhost:8080/events -H "Content-Type: application/json" -d "{\"name\":\"Show A\",\"eventDate\":\"2026-09-01T20:00:00\",\"price\":150.00,\"availableQuantity\":2}"

# list events
curl http://localhost:8080/events

# reserve ticket
curl -X POST http://localhost:8080/events/1/reserve


###

# register a user
curl -X POST http://localhost:8081/users -H "Content-Type: application/json" -d "{\"email\":\"leo@test.com\",\"name\":\"Leonardo\"}"

# create an event with 1 ticket (on event-service)
curl -X POST http://localhost:8080/events -H "Content-Type: application/json" -d "{\"name\":\"Show B\",\"eventDate\":\"2026-09-15T21:00:00\",\"price\":200.00,\"availableQuantity\":1}"

# buy the ticket
curl -X POST http://localhost:8081/orders -H "Content-Type: application/json" -d "{\"userId\":1,\"eventId\":1}"

# try to buy again = 409
curl -X POST http://localhost:8081/orders -H "Content-Type: application/json" -d "{\"userId\":1,\"eventId\":1}"

###

# happy path
curl -X POST http://localhost:8081/orders/1/pay -H "Content-Type: application/json" -d "{\"paymentMethod\":\"CREDIT_CARD\",\"amount\":200.00}"

# trip the breaker = failure rate to 100%
curl -X POST http://localhost:8090/admin/mode -H "Content-Type: application/json" -d "{\"failureRate\":1.0}"

# recover
curl -X POST http://localhost:8090/admin/mode -H "Content-Type: application/json" -d "{\"failureRate\":0.0}"

###

# 1. create user, event, order fresh
curl -X POST http://localhost:8081/users -H "Content-Type: application/json" -d "{\"email\":\"leo@test.com\",\"name\":\"Leonardo\"}"
curl -X POST http://localhost:8080/events -H "Content-Type: application/json" -d "{\"name\":\"Show A\",\"eventDate\":\"2026-10-01T20:00:00\",\"price\":100.00,\"availableQuantity\":10}"
curl -X POST http://localhost:8081/orders -H "Content-Type: application/json" -d "{\"userId\":1,\"eventId\":1}"

# 2. verify happy path first
curl -X POST http://localhost:8081/orders/1/pay -H "Content-Type: application/json" -d "{\"paymentMethod\":\"CREDIT_CARD\",\"amount\":100.00}"
# → {"message":"Payment processed"}

# 3. create a second order for the breaker demo
curl -X POST http://localhost:8081/orders -H "Content-Type: application/json" -d "{\"userId\":1,\"eventId\":1}"

# 4. flip failure rate, pay order 2
curl -X POST http://localhost:8090/admin/mode -H "Content-Type: application/json" -d "{\"failureRate\":1.0}"
curl -X POST http://localhost:8081/orders/2/pay -H "Content-Type: application/json" -d "{\"paymentMethod\":\"CREDIT_CARD\",\"amount\":100.00}"
# → {"error":"Payment service unavailable. Try again later."}

# 5. recover, wait 10s, pay order 2 again
curl -X POST http://localhost:8090/admin/mode -H "Content-Type: application/json" -d "{\"failureRate\":0.0}"
# wait 10 seconds
curl -X POST http://localhost:8081/orders/2/pay -H "Content-Type: application/json" -d "{\"paymentMethod\":\"CREDIT_CARD\",\"amount\":100.00}"
# → {"message":"Payment processed"}

###

```
Set up inventory:
bashcurl -X POST http://localhost:8000/events -H "Content-Type: application/json" -d "{\"name\":\"All Methods\",\"eventDate\":\"2026-12-20T20:00:00\",\"price\":90.00,\"availableQuantity\":10}"
Credit card — synchronous, confirmed immediately:
bashcurl -X POST http://localhost:8000/orders -H "Content-Type: application/json" -d "{\"userId\":1,\"eventId\":1}"
curl -X POST http://localhost:8000/orders/1/pay -H "Content-Type: application/json" -d "{\"paymentMethod\":\"CREDIT_CARD\",\"amount\":90.00}"
curl http://localhost:8000/orders/1
# → CONFIRMED right away
PIX — asynchronous, PENDING then CONFIRMED via webhook:
bashcurl -X POST http://localhost:8000/orders -H "Content-Type: application/json" -d "{\"userId\":1,\"eventId\":1}"
curl -X POST http://localhost:8000/orders/2/pay -H "Content-Type: application/json" -d "{\"paymentMethod\":\"PIX\",\"amount\":90.00}"
curl http://localhost:8000/orders/2
# → PENDING immediately
# wait ~5 seconds (the webhook delay)
curl http://localhost:8000/orders/2
# → CONFIRMED — the gateway called back
Boleto — same async path:
bashcurl -X POST http://localhost:8000/orders -H "Content-Type: application/json" -d "{\"userId\":1,\"eventId\":1}"
curl -X POST http://localhost:8000/orders/3/pay -H "Content-Type: application/json" -d "{\"paymentMethod\":\"BOLETO\",\"amount\":90.00}"
# PENDING now, CONFIRMED after the webhook fires
PIX expiry demo (user never pays the PIX → it expires):
bashcurl -X POST http://localhost:8090/admin/mode -H "Content-Type: application/json" -d "{\"confirmRate\":0.0}"
curl -X POST http://localhost:8000/orders -H "Content-Type: application/json" -d "{\"userId\":1,\"eventId\":1}"
curl -X POST http://localhost:8000/orders/4/pay -H "Content-Type: application/json" -d "{\"paymentMethod\":\"PIX\",\"amount\":90.00}"
# wait ~5s
curl http://localhost:8000/orders/4
# → EXPIRED — the webhook came back EXPIRED
curl -X POST http://localhost:8090/admin/mode -H "Content-Type: application/json" -d "{\"confirmRate\":1.0}"
Watch docker compose logs -f payment-gateway-mock during the PIX calls — you'll see Webhook delivered for order X → CONFIRMED. That callback landing on your /webhooks/payment-callback is the second async integration, now real.
```


```
ticket-system
├─ docker-compose.yml
├─ event-service
│  ├─ .mvn
│  │  └─ wrapper
│  │     └─ maven-wrapper.properties
│  ├─ Dockerfile
│  ├─ HELP.md
│  ├─ mvnw
│  ├─ mvnw.cmd
│  ├─ pom.xml
│  ├─ src
│  │  ├─ main
│  │  │  ├─ java
│  │  │  │  └─ com
│  │  │  │     └─ tickets
│  │  │  │        └─ event_service
│  │  │  │           ├─ event
│  │  │  │           │  ├─ CreateEventRequest.java
│  │  │  │           │  ├─ Event.java
│  │  │  │           │  ├─ EventController.java
│  │  │  │           │  ├─ EventRepository.java
│  │  │  │           │  ├─ EventService.java
│  │  │  │           │  └─ InsufficientInventoryException.java
│  │  │  │           ├─ EventServiceApplication.java
│  │  │  │           ├─ GlobalExceptionHandler.java
│  │  │  │           └─ HealthController.java
│  │  │  └─ resources
│  │  │     ├─ application.properties
│  │  │     └─ application.yml
│  │  └─ test
│  │     └─ java
│  │        └─ com
│  │           └─ tickets
│  │              └─ event_service
│  │                 └─ EventServiceApplicationTests.java
│  └─ target
│     ├─ classes
│     │  ├─ application.properties
│     │  ├─ application.yml
│     │  └─ com
│     │     └─ tickets
│     │        └─ event_service
│     │           ├─ event
│     │           │  ├─ CreateEventRequest.class
│     │           │  ├─ Event.class
│     │           │  ├─ EventController.class
│     │           │  ├─ EventRepository.class
│     │           │  ├─ EventService.class
│     │           │  └─ InsufficientInventoryException.class
│     │           ├─ EventServiceApplication.class
│     │           ├─ GlobalExceptionHandler.class
│     │           └─ HealthController.class
│     ├─ generated-sources
│     │  └─ annotations
│     ├─ generated-test-sources
│     │  └─ test-annotations
│     └─ test-classes
│        └─ com
│           └─ tickets
│              └─ event_service
│                 └─ EventServiceApplicationTests.class
├─ notification-service
│  ├─ .mvn
│  │  └─ wrapper
│  │     └─ maven-wrapper.properties
│  ├─ Dockerfile
│  ├─ HELP.md
│  ├─ mvnw
│  ├─ mvnw.cmd
│  ├─ pom.xml
│  ├─ src
│  │  ├─ main
│  │  │  ├─ java
│  │  │  │  └─ com
│  │  │  │     └─ tickets
│  │  │  │        └─ notification_service
│  │  │  │           ├─ NotificationConsumer.java
│  │  │  │           ├─ NotificationController.java
│  │  │  │           ├─ NotificationServiceApplication.java
│  │  │  │           ├─ OrderConfirmedEvent.java
│  │  │  │           ├─ RabbitConfig.java
│  │  │  │           ├─ SentEmail.java
│  │  │  │           └─ SentEmailRepository.java
│  │  │  └─ resources
│  │  │     ├─ application.properties
│  │  │     ├─ application.yml
│  │  │     ├─ static
│  │  │     └─ templates
│  │  └─ test
│  │     └─ java
│  │        └─ com
│  │           └─ tickets
│  │              └─ notification_service
│  │                 └─ NotificationServiceApplicationTests.java
│  └─ target
│     ├─ classes
│     │  ├─ application.properties
│     │  ├─ application.yml
│     │  └─ com
│     │     └─ tickets
│     │        └─ notification_service
│     │           ├─ NotificationConsumer.class
│     │           ├─ NotificationController.class
│     │           ├─ NotificationServiceApplication.class
│     │           ├─ OrderConfirmedEvent.class
│     │           ├─ RabbitConfig.class
│     │           ├─ SentEmail.class
│     │           └─ SentEmailRepository.class
│     └─ test-classes
│        └─ com
│           └─ tickets
│              └─ notification_service
│                 └─ NotificationServiceApplicationTests.class
├─ order-service
│  ├─ .mvn
│  │  └─ wrapper
│  │     └─ maven-wrapper.properties
│  ├─ Dockerfile
│  ├─ HELP.md
│  ├─ mvnw
│  ├─ mvnw.cmd
│  ├─ pom.xml
│  ├─ src
│  │  ├─ main
│  │  │  ├─ java
│  │  │  │  └─ com
│  │  │  │     └─ tickets
│  │  │  │        └─ order_service
│  │  │  │           ├─ GlobalExceptionHandler.java
│  │  │  │           ├─ order
│  │  │  │           │  ├─ CreateOrderRequest.java
│  │  │  │           │  ├─ EventServiceClient.java
│  │  │  │           │  ├─ InsufficientInventoryException.java
│  │  │  │           │  ├─ Order.java
│  │  │  │           │  ├─ OrderConfirmedEvent.java
│  │  │  │           │  ├─ OrderController.java
│  │  │  │           │  ├─ OrderRepository.java
│  │  │  │           │  ├─ OrderService.java
│  │  │  │           │  ├─ PaymentDeclinedException.java
│  │  │  │           │  ├─ PaymentService.java
│  │  │  │           │  └─ RabbitConfig.java
│  │  │  │           ├─ OrderServiceApplication.java
│  │  │  │           └─ user
│  │  │  │              ├─ User.java
│  │  │  │              ├─ UserController.java
│  │  │  │              ├─ UserRepository.java
│  │  │  │              └─ UserService.java
│  │  │  └─ resources
│  │  │     ├─ application.properties
│  │  │     └─ application.yml
│  │  └─ test
│  │     └─ java
│  │        └─ com
│  │           └─ tickets
│  │              └─ order_service
│  │                 └─ OrderServiceApplicationTests.java
│  └─ target
│     ├─ classes
│     │  ├─ application.properties
│     │  ├─ application.yml
│     │  └─ com
│     │     └─ tickets
│     │        └─ order_service
│     │           ├─ GlobalExceptionHandler.class
│     │           ├─ order
│     │           │  ├─ CreateOrderRequest.class
│     │           │  ├─ EventServiceClient.class
│     │           │  ├─ InsufficientInventoryException.class
│     │           │  ├─ Order$Status.class
│     │           │  ├─ Order.class
│     │           │  ├─ OrderConfirmedEvent.class
│     │           │  ├─ OrderController.class
│     │           │  ├─ OrderRepository.class
│     │           │  ├─ OrderService.class
│     │           │  ├─ PaymentDeclinedException.class
│     │           │  ├─ PaymentService.class
│     │           │  └─ RabbitConfig.class
│     │           ├─ OrderServiceApplication.class
│     │           └─ user
│     │              ├─ User.class
│     │              ├─ UserController.class
│     │              ├─ UserRepository.class
│     │              └─ UserService.class
│     ├─ generated-sources
│     │  └─ annotations
│     ├─ generated-test-sources
│     │  └─ test-annotations
│     └─ test-classes
│        └─ com
│           └─ tickets
│              └─ order_service
│                 └─ OrderServiceApplicationTests.class
├─ payment-gateway-mock
│  ├─ .mvn
│  │  └─ wrapper
│  │     └─ maven-wrapper.properties
│  ├─ Dockerfile
│  ├─ HELP.md
│  ├─ mvnw
│  ├─ mvnw.cmd
│  ├─ pom.xml
│  ├─ src
│  │  ├─ main
│  │  │  ├─ java
│  │  │  │  └─ com
│  │  │  │     └─ tickets
│  │  │  │        └─ payment_gateway_mock
│  │  │  │           ├─ AdminController.java
│  │  │  │           ├─ ChargeController.java
│  │  │  │           ├─ ChargeRequest.java
│  │  │  │           ├─ ChargeResponse.java
│  │  │  │           ├─ GatewayConfig.java
│  │  │  │           └─ PaymentGatewayMockApplication.java
│  │  │  └─ resources
│  │  │     ├─ application.properties
│  │  │     └─ application.yml
│  │  └─ test
│  │     └─ java
│  │        └─ com
│  │           └─ tickets
│  │              └─ payment_gateway_mock
│  │                 └─ PaymentGatewayMockApplicationTests.java
│  └─ target
│     ├─ classes
│     │  ├─ application.properties
│     │  ├─ application.yml
│     │  └─ com
│     │     └─ tickets
│     │        └─ payment_gateway_mock
│     │           ├─ AdminController.class
│     │           ├─ ChargeController.class
│     │           ├─ ChargeRequest.class
│     │           ├─ ChargeResponse.class
│     │           ├─ GatewayConfig.class
│     │           └─ PaymentGatewayMockApplication.class
│     └─ test-classes
│        └─ com
│           └─ tickets
│              └─ payment_gateway_mock
│                 └─ PaymentGatewayMockApplicationTests.class
├─ prometheus
│  └─ prometheus.yml
├─ README.md
└─ test.md

```