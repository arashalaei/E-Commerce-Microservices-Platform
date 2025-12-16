# 🛍️ E-Commerce Microservices Platform

A complete microservices learning project showcasing modern backend patterns and practices.

## 📋 Project Overview

**Domain**: Simple E-Commerce Platform with Order Management

**Services**:
1. **User Service** - User management and authentication
2. **Product Service** - Product catalog
3. **Order Service** - Order orchestration (Saga coordinator)
4. **Payment Service** - Payment processing simulation
5. **Inventory Service** - Stock management
6. **Notification Service** - Email/SMS notifications

## 🎯 What This Demonstrates

### Architecture Patterns
- ✅ Hexagonal Architecture (Ports & Adapters)
- ✅ Domain-Driven Design (DDD)
- ✅ Event-Driven Architecture
- ✅ Saga Pattern (Orchestration)
- ✅ CQRS (Command Query Responsibility Segregation)
- ✅ Event Sourcing (for Order Service)
- ✅ Circuit Breaker Pattern
- ✅ API Gateway Pattern

### Technologies
- ✅ Go (Golang)
- ✅ NATS (JetStream for events)
- ✅ gRPC (inter-service communication)
- ✅ PostgreSQL (transactional data)
- ✅ Redis (caching + rate limiting)
- ✅ OpenTelemetry (tracing + metrics)
- ✅ Prometheus + Grafana (monitoring)
- ✅ ELK Stack (logging)
- ✅ Docker + Docker Compose

## 📁 Project Structure

```
ecommerce-platform/
├── services/
│   ├── user-service/             # Each service has FULL hexagonal architecture
│   ├── product-service/          # Each service has FULL hexagonal architecture
│   ├── order-service/            # Saga orchestrator + FULL hexagonal architecture
│   ├── payment-service/          # Each service has FULL hexagonal architecture
│   ├── inventory-service/        # Each service has FULL hexagonal architecture
│   └── notification-service/     # Each service has FULL hexagonal architecture
│
├── api-gateway/                 # API Gateway (HTTP → gRPC)
│   ├── cmd/
│   ├── internal/
│   │   ├── handler/
│   │   ├── middleware/
│   │   ├── proxy/
│   │   └── rate_limiter/
│   └── go.mod
│
├── pkg/                         # Shared libraries
│   ├── events/                  # Event contracts
│   │   ├── user_events.go
│   │   ├── order_events.go
│   │   ├── payment_events.go
│   │   └── inventory_events.go
│   ├── nats/
│   │   ├── subjects.go
│   │   ├── publisher.go
│   │   └── subscriber.go
│   ├── grpc/
│   │   └── interceptors/
│   │       ├── auth.go
│   │       ├── logging.go
│   │       └── tracing.go
│   ├── observability/
│   │   ├── logger/
│   │   ├── tracer/
│   │   └── metrics/
│   ├── errors/
│   ├── jwt/
│   └── database/
│       ├── postgres.go
│       └── transaction.go
│
├── proto/                       # gRPC definitions
│   ├── user/
│   │   └── v1/
│   │       └── user.proto
│   ├── product/
│   ├── order/
│   ├── payment/
│   └── inventory/
│
├── deployments/
│   ├── docker-compose.yaml
│   └── kubernetes/
│       ├── services/
│       ├── monitoring/
│       └── infrastructure/
│
├── scripts/
│   ├── setup-infra.sh
│   ├── generate-proto.sh
│   └── seed-data.sh
│
├── docs/
│   ├── architecture/
│   │   ├── system-design.md
│   │   ├── saga-pattern.md
│   │   └── event-flow.md
│   ├── api/
│   │   └── openapi.yaml
│   └── runbook.md
│
├── Makefile
└── README.md
```

## 🔄 Complete Order Flow (Saga Pattern)

### Happy Path:
```
1. User → API Gateway → Order Service: CreateOrder(items, userId)
2. Order Service: Create Order (PENDING)
3. Order Service → Inventory Service (gRPC): ReserveStock
4. Inventory Service → NATS: Publish StockReserved event
5. Order Service → Payment Service (gRPC): ProcessPayment
6. Payment Service → NATS: Publish PaymentProcessed event
7. Order Service: Update Order (CONFIRMED)
8. Order Service → NATS: Publish OrderConfirmed event
9. Notification Service: Listen OrderConfirmed → Send Email

Cache: Product details, User info
Tracing: Full request traced via OpenTelemetry
```

### Failure Path (Saga Compensation):
```
1. Order Service → Inventory Service: ReserveStock ✅
2. Order Service → Payment Service: ProcessPayment ❌ (FAILED)
3. Order Service starts compensation:
   → Inventory Service: ReleaseStock (compensate)
   → Update Order (CANCELLED)
   → NATS: Publish OrderCancelled event
4. Notification Service: Send cancellation email
```

## 🏗️ Service Details

### 1️⃣ User Service
**Responsibility**: Authentication & User Management
**Tech Stack**:
- PostgreSQL (user data)
- Redis (session cache)
- JWT tokens
- gRPC server

**Features**:
- Register/Login (JWT)
- User profile CRUD
- Role-based access control

**DDD Elements**:
- Aggregate: User
- Value Objects: Email, Password
- Domain Events: UserRegistered, UserUpdated

---

### 2️⃣ Product Service
**Responsibility**: Product Catalog
**Tech Stack**:
- PostgreSQL (product data)
- Redis (product cache with TTL)
- gRPC server

**Features**:
- Product CRUD
- Search/Filter products
- Category management

**DDD Elements**:
- Aggregate: Product
- Value Objects: Money, SKU
- Read Model: ProductListView (CQRS)

**Caching Strategy**:
- Cache-aside pattern
- TTL: 5 minutes
- Cache invalidation on update

---

### 3️⃣ Order Service ⭐ (Saga Orchestrator)
**Responsibility**: Order orchestration + Event Sourcing
**Tech Stack**:
- PostgreSQL (order state)
- NATS JetStream (event store)
- Redis (idempotency keys)
- gRPC server

**Features**:
- Create order (orchestrate saga)
- Saga compensation on failure
- Event sourcing for order history
- Idempotent API (prevent duplicate orders)

**DDD Elements**:
- Aggregate: Order
- Entities: OrderItem
- Value Objects: OrderStatus, Money
- Domain Service: SagaOrchestrator
- Domain Events: OrderCreated, OrderConfirmed, OrderCancelled

**Saga Implementation**:
```go
type SagaStep struct {
    Execute    func(ctx context.Context) error
    Compensate func(ctx context.Context) error
}

type OrderSaga struct {
    steps []SagaStep
}

func (s *OrderSaga) Execute(ctx context.Context) error {
    executed := []SagaStep{}
    
    for _, step := range s.steps {
        if err := step.Execute(ctx); err != nil {
            // Compensation in reverse order
            for i := len(executed) - 1; i >= 0; i-- {
                executed[i].Compensate(ctx)
            }
            return err
        }
        executed = append(executed, step)
    }
    return nil
}
```

---

### 4️⃣ Payment Service
**Responsibility**: Payment processing (simulated)
**Tech Stack**:
- PostgreSQL (payment records)
- gRPC server
- Circuit Breaker (resilience)

**Features**:
- Process payment (simulate 3rd party)
- Refund payment (compensation)
- Payment history

**DDD Elements**:
- Aggregate: Payment
- Domain Events: PaymentProcessed, PaymentFailed, PaymentRefunded

**Circuit Breaker**:
- Prevents cascading failures
- Fallback: Return "payment service unavailable"

---

### 5️⃣ Inventory Service
**Responsibility**: Stock management
**Tech Stack**:
- PostgreSQL (inventory data)
- Redis (stock cache)
- gRPC server
- Optimistic locking (prevent overselling)

**Features**:
- Reserve stock (saga step)
- Release stock (compensation)
- Stock queries

**DDD Elements**:
- Aggregate: Inventory
- Domain Events: StockReserved, StockReleased

**Concurrency Handling**:
```go
// Optimistic locking with version field
UPDATE inventory 
SET quantity = quantity - ?, version = version + 1
WHERE product_id = ? AND version = ? AND quantity >= ?
```

---

### 6️⃣ Notification Service
**Responsibility**: Send notifications
**Tech Stack**:
- NATS subscriber (event-driven)
- No database (stateless)
- Email/SMS simulation

**Features**:
- Listen to: OrderConfirmed, OrderCancelled, PaymentFailed
- Send emails (simulated)

---

### 7️⃣ API Gateway
**Responsibility**: Single entry point
**Tech Stack**:
- HTTP REST API
- gRPC clients to services
- Redis (rate limiting)
- JWT validation

**Features**:
- HTTP → gRPC translation
- Authentication middleware
- Rate limiting (100 req/min per user)
- Request tracing

---

## 🔧 Infrastructure Components

### NATS JetStream
```yaml
Streams:
  - ORDERS: events.order.*
  - PAYMENTS: events.payment.*
  - INVENTORY: events.inventory.*
  - USERS: events.user.*

Consumers:
  - notification-service: All events
  - order-service: payment.*, inventory.*
```

### Redis
```
Use Cases:
1. Caching (product catalog, user sessions)
2. Rate limiting (API Gateway)
3. Idempotency keys (Order Service)
4. Distributed locks (Inventory Service)
```

### PostgreSQL
```
Databases:
- user_db
- product_db
- order_db
- payment_db
- inventory_db
```

### Observability Stack
```
- OpenTelemetry: Distributed tracing
- Prometheus: Metrics collection
- Grafana: Dashboards
- ELK: Logs aggregation
```

---

## 📊 Key Metrics to Track

```
Business Metrics:
- Orders per second
- Order success rate
- Average order value
- Payment success rate

Technical Metrics:
- Request latency (p50, p95, p99)
- Error rate per service
- Cache hit ratio
- Database connection pool
- NATS message throughput
- Circuit breaker status
```

---

## 🧪 Testing Strategy

```
Unit Tests (70%):
- Domain logic
- Use cases
- Value objects

Integration Tests (20%):
- Repository tests (Testcontainers)
- NATS event flow
- gRPC clients

E2E Tests (10%):
- Complete order flow
- Saga compensation
- API Gateway → Services
```

---

## 🚀 Getting Started

### Prerequisites
```bash
- Go 1.21+
- Docker & Docker Compose
- Make
- Protocol Buffers compiler
```

### Quick Start
```bash
# 1. Clone repository
git clone https://github.com/yourusername/ecommerce-platform

# 2. Start infrastructure
make infra-up

# 3. Generate gRPC code
make proto-gen

# 4. Run all services
make services-up

# 5. Seed data
make seed-data

# 6. Open Grafana
open http://localhost:3000
```

### Test the System
```bash
# Create user
curl -X POST http://localhost:8080/api/v1/users/register \
  -d '{"email":"test@example.com","password":"123456"}'

# Login
curl -X POST http://localhost:8080/api/v1/users/login \
  -d '{"email":"test@example.com","password":"123456"}'

# Create order (trigger Saga)
curl -X POST http://localhost:8080/api/v1/orders \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"items":[{"product_id":1,"quantity":2}]}'

# Check traces in Jaeger
open http://localhost:16686
```

---

## 📚 Learning Path

### Phase 1: Foundation (Week 1-2)
- [ ] Setup monorepo structure
- [ ] Implement User Service (basic CRUD)
- [ ] Setup PostgreSQL + migrations
- [ ] Setup NATS
- [ ] Basic logging

### Phase 2: Core Services (Week 3-4)
- [ ] Product Service with caching
- [ ] Inventory Service with optimistic locking
- [ ] Payment Service with circuit breaker
- [ ] gRPC communication between services

### Phase 3: Advanced Patterns (Week 5-6)
- [ ] Order Service with Saga pattern
- [ ] Event Sourcing for orders
- [ ] CQRS for product queries
- [ ] Notification Service (event-driven)

### Phase 4: Production Ready (Week 7-8)
- [ ] API Gateway with rate limiting
- [ ] OpenTelemetry integration
- [ ] Prometheus + Grafana dashboards
- [ ] Error handling + retries
- [ ] Integration tests
- [ ] Documentation

---

## 🎓 What You'll Learn

### Architecture
- Microservices decomposition
- Service boundaries (DDD)
- Event-driven communication
- Saga pattern for distributed transactions
- CQRS and Event Sourcing

### Go Best Practices
- Clean architecture
- Interface-driven design
- Error handling
- Context usage
- Goroutines and channels

### Distributed Systems
- CAP theorem in practice
- Eventual consistency
- Idempotency
- Distributed tracing
- Circuit breakers
- Rate limiting

### DevOps
- Docker containerization
- Multi-service orchestration
- Health checks
- Graceful shutdown
- Monitoring and alerting

---

## 📝 Portfolio Highlights

When showcasing this project:

1. **README with Architecture Diagram**
2. **Live Demo** (deploy to cloud)
3. **Grafana Dashboard Screenshot**
4. **Distributed Trace Example**
5. **Code Walkthrough Video**
6. **Blog Post**: "Implementing Saga Pattern in Go"

---

## 🔗 Resources

- [Domain-Driven Design](https://martinfowler.com/tags/domain%20driven%20design.html)
- [Microservices Patterns](https://microservices.io/patterns/)
- [NATS Documentation](https://docs.nats.io/)
- [OpenTelemetry Go](https://opentelemetry.io/docs/instrumentation/go/)

---

## 📈 Next Steps (Optional)

- Add API versioning
- Implement GraphQL gateway
- Add Kafka for high-throughput events
- Kubernetes deployment
- Service mesh (Istio/Linkerd)
- Multi-tenancy
- Feature flags
