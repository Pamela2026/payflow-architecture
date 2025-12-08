# PayFlow Architecture Documentation

**Repository:** payflow-architecture  
**Purpose:** Detailed documentation and analysis of a production-ready fintech microservices platform.  

This repository provides a comprehensive breakdown of the PayFlow system, including:

- Architecture overview
- Microservice responsibilities
- Ports, environment variables, and dependencies
- Full “Send Money” request flow
- Sequence diagram
- Databases, cache, and message queue

---

## 1. System Architecture

```mermaid
flowchart LR
    F[Frontend] --> G[API Gateway<br>Port: 3000]
    G --> A[Auth Service<br>Port: 3004]
    G --> W[Wallet Service<br>Port: 3001]
    G --> T[Transaction Service<br>Port: 3002]
    T -->|Publish event| R[RabbitMQ]
    R --> N[Notification Service<br>Port: 3003]
    W --> T
    DB[(PostgreSQL)]
    REDIS[(Redis)]
    G --> DB
    A --> DB
    W --> DB
    T --> DB
    A --> REDIS
    W --> REDIS
    T --> REDIS
Explanation:
The Frontend communicates only with the API Gateway. All services are decoupled and communicate via HTTP or RabbitMQ events. Databases (PostgreSQL) store persistent data, and Redis is used for caching and token management.

2. Services Overview
Service	Purpose	Port	Env Vars	Dependencies
API Gateway	Handles authentication, routing, validation, metrics	3000	AUTH_SERVICE_URL, WALLET_SERVICE_URL, TRANSACTION_SERVICE_URL, NOTIFICATION_SERVICE_URL, CORS_ORIGIN	Auth, Wallet, Transaction, Notification, Redis
Auth Service	User registration, login, logout, token refresh	3004	DB_HOST, DB_PORT, DB_USER, DB_PASSWORD, DB_NAME, JWT_SECRET, REDIS_URL	PostgreSQL, Redis
Wallet Service	Manage wallets, balances, credit/debit	3001	DB_HOST, DB_PORT, DB_USER, DB_PASSWORD, DB_NAME, REDIS_URL	PostgreSQL, Redis
Transaction Service	Handle money transfers, create transactions	3002	DB_HOST, DB_PORT, DB_USER, DB_PASSWORD, DB_NAME, WALLET_SERVICE_URL, AUTH_SERVICE_URL, RABBITMQ_URL, REDIS_URL	Wallet Service, RabbitMQ, PostgreSQL
Notification Service	Send email/SMS notifications	3003	RABBITMQ_URL, EMAIL_HOST, EMAIL_USER, EMAIL_PASSWORD, TWILIO_SID, TWILIO_TOKEN	RabbitMQ, Email/SMS provider
Frontend	React SPA UI	3005 (dev)	REACT_APP_API_URL	API Gateway

3. Send Money Flow
Scenario: User A sends $50 to User B

mermaid
Copy code
sequenceDiagram
    participant F as Frontend
    participant G as API Gateway
    participant A as Auth Service
    participant T as Transaction Service
    participant W as Wallet Service
    participant R as RabbitMQ
    participant N as Notification Service

    F->>G: POST /api/transactions
    G->>A: Verify JWT
    G->>T: Forward Send Money request
    T->>W: Debit Sender
    W-->>T: Confirm Debit
    T->>W: Credit Receiver
    W-->>T: Confirm Credit
    T->>T: Record Transaction
    T->>R: Publish Event
    R->>N: Consume Event
    N->>Email/SMS: Send Notifications
    T-->>G: Return Success
    G-->>F: Response (Transaction Completed)
Notes:

API Gateway handles authentication, validation, rate-limiting, and metrics.

Transaction Service ensures atomicity with Wallet Service.

Notifications are handled asynchronously via RabbitMQ.

Redis is used for caching and token blacklisting.

4. Databases & Cache
Component	Purpose
PostgreSQL	Users, wallets, transactions, audit logs
Redis	Token blacklist, caching, idempotency
RabbitMQ	Asynchronous messaging for notifications

5. Shared Utilities
Module	Purpose
circuit-breaker.js	Prevent cascading failures
idempotency.js	Ensure request uniqueness
logger.js	Centralized logging
metrics.js	Prometheus metrics collection
retry.js	Retry failed calls
tracing.js	Distributed tracing for requests

6. Environment Variables Summary
Service	Key Env Vars
API Gateway	AUTH_SERVICE_URL, WALLET_SERVICE_URL, TRANSACTION_SERVICE_URL, NOTIFICATION_SERVICE_URL, CORS_ORIGIN
Auth Service	DB_HOST, DB_PORT, DB_USER, DB_PASSWORD, DB_NAME, JWT_SECRET, REDIS_URL
Wallet Service	DB_HOST, DB_PORT, DB_USER, DB_PASSWORD, DB_NAME, REDIS_URL
Transaction Service	DB_HOST, DB_PORT, DB_USER, DB_PASSWORD, DB_NAME, WALLET_SERVICE_URL, AUTH_SERVICE_URL, RABBITMQ_URL, REDIS_URL
Notification Service	RABBITMQ_URL, EMAIL_HOST, EMAIL_USER, EMAIL_PASSWORD, TWILIO_SID, TWILIO_TOKEN

