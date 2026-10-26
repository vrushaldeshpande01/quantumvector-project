# Microservices Architecture - Deep Dive Analysis
## Cloud-Native E-commerce Platform on AWS EKS

---

## Table of Contents
1. [Architecture Overview](#architecture-overview)
2. [Microservices Breakdown](#microservices-breakdown)
3. [Inter-Service Communication](#inter-service-communication)
4. [Language Choices & Justification](#language-choices--justification)
5. [Technology Stack Summary](#technology-stack-summary)

---

## Architecture Overview

This is a **polyglot microservices architecture** for a cloud-native e-commerce platform (like Google Cloud's Online Boutique demo). The system handles:
- **User Authentication & Authorization**
- **Product Catalog Management**
- **Shopping Cart Operations**
- **Checkout & Order Processing**
- **Payment Processing**
- **Email Notifications**
- **Currency Conversion**
- **Product Recommendations**
- **Shipping Calculations**
- **AI-powered Shopping Assistant**

### Communication Protocols
- **gRPC** for synchronous service-to-service communication (high performance, efficient)
- **REST/HTTP** for user-facing API endpoints
- **Protocol Buffers** for data serialization (language-agnostic)

---

## Microservices Breakdown

### 1. **Auth Service** (Python + Flask)
**Purpose:**
- User registration and login
- JWT token generation and verification
- Credential validation
- Session management

**Code Responsibilities:**
```python
- register()       → Create new user, hash password, generate JWT token
- login()          → Validate credentials, issue JWT token
- verify()         → Validate JWT token, return user info
- init_db()        → Initialize PostgreSQL user table
- get_secret()     → Fetch credentials from AWS Secrets Manager
```

**Key Features:**
- Uses **bcrypt** for password hashing (industry-standard)
- **PostgreSQL** for persistent user data
- **AWS Secrets Manager** for secure credential storage
- **JWT (HS256)** for stateless authentication
- 48-hour token expiration

**Why Python?**
- Rapid development for authentication logic
- Rich ecosystem (Flask, PyJWT, bcrypt, psycopg2)
- Good for I/O-bound operations (database queries)
- Easy to integrate with AWS services (boto3)

**Dependencies:**
- No direct service calls to other microservices
- **Called by:** Frontend, all services needing token validation

---

### 2. **Product Catalog Service** (Go)
**Purpose:**
- Manage product inventory
- Serve product details and pricing
- Handle dynamic catalog reloading (with intentional bug for demo)
- Product search functionality

**Key Features:**
- Loads `products.json` file into memory
- Responds to gRPC requests for product details
- Demonstrates performance issues with repeated JSON parsing
- Can be triggered to reload catalog (USR1 signal) or stop (USR2 signal)

**Why Go?**
- **High performance & low latency** - critical for catalog queries
- Efficient memory management and concurrency
- Fast JSON parsing
- Minimal resource footprint in containers
- Excellent for read-heavy workloads

**Dependencies:**
- gRPC server
- Internal product data (JSON)
- **Called by:** Checkout, Cart, Recommendation services

---

### 3. **Cart Service** (C#/.NET)
**Purpose:**
- Manage shopping cart per user
- Add/remove items from cart
- Persist cart state
- Validate cart items

**Key Features:**
- Uses **Redis** for fast cart caching
- **Google Cloud Spanner** or **PostgreSQL** for persistent storage
- Implements gRPC for service-to-service calls
- Health check endpoints

**Code Structure:**
```csharp
- Cart model with item management
- Redis caching for performance
- Spanner/PostgreSQL adapter for persistence
- gRPC service implementation
```

**Why C#?**
- **Enterprise-grade ecosystem** (.NET/Core)
- Excellent performance for stateful operations
- Rich data structures and OOP capabilities
- Strong typing (reduces bugs)
- Good async/await for concurrent requests
- Mature cloud integration (Azure, AWS support)

**Dependencies:**
- Redis (cache)
- PostgreSQL/Spanner (persistence)
- gRPC protocol
- **Called by:** Checkout, Frontend
- **Calls:** Product Catalog (to validate items)

---

### 4. **Product Catalog Service** (Go) - Alternative Name
Actually, this is **Checkout Service** - see below

---

### 4. **Checkout Service** (Go)
**Purpose:**
- Orchestrate the checkout process
- Calculate totals
- Coordinate with other services
- Build and process orders

**Key Features:**
- **Central orchestrator** - calls 6 other services
- Traces across services using OpenTelemetry
- Calculates shipping costs
- Applies currency conversion
- Initiates payment processing
- Sends confirmation emails

**Service Calls:**
```
CheckoutService calls:
├── ProductCatalogService (get product details)
├── CartService (get cart items)
├── CurrencyService (convert prices)
├── ShippingService (calculate shipping)
├── PaymentService (charge customer)
└── EmailService (send confirmation)
```

**Why Go?**
- **Orchestration & concurrency** - coordinates 6 async calls
- goroutines for parallel requests
- Fast execution for business logic
- Built-in context handling for distributed tracing
- Excellent error handling with typed errors

**Dependencies:**
- gRPC connections to 6 other services
- Money package for currency calculations
- OpenTelemetry for distributed tracing
- **Called by:** Frontend (initial checkout request)

---

### 5. **Payment Service** (Node.js/JavaScript)
**Purpose:**
- Process credit card charges
- Handle payment validation
- Simulate payment gateway integration
- Return payment confirmation

**Code Responsibilities:**
```javascript
ChargeServiceHandler(request) {
  - Validate card details
  - Attempt charge
  - Return success/failure response
}
```

**Key Features:**
- Simple charge processing logic
- Integrates with payment gateway simulation
- gRPC server exposing payment API
- Health check endpoints

**Why Node.js?**
- **Lightweight & fast** for I/O operations
- Perfect for simple request-response logic
- Easy async handling (promises, async/await)
- Small resource footprint
- Fast startup time
- Event-driven architecture matches payment operations

**Dependencies:**
- gRPC for communication
- Payment gateway SDK/integration
- **Called by:** Checkout Service

---

### 6. **Currency Service** (Node.js/JavaScript)
**Purpose:**
- Convert prices between currencies
- Return exchange rates
- Handle multi-currency shopping carts

**Key Features:**
- Loads currency conversion rates
- Responds to gRPC requests for conversions
- Includes profiling support
- OpenTelemetry tracing

**Why Node.js?**
- Lightweight data transformation service
- Simple request-response pattern
- Easy JSON handling
- Low latency for currency conversions
- Matches simplicity of operation (lookup + math)

**Dependencies:**
- gRPC server
- Currency data (hardcoded or loaded from file)
- **Called by:** Checkout Service

---

### 7. **Shipping Service** (Go)
**Purpose:**
- Calculate shipping costs
- Validate shipping addresses
- Return quote/estimate
- Handle different shipping methods

**Code Responsibilities:**
```go
GetQuote(address, items) {
  - Validate address
  - Calculate weight/dimensions
  - Apply shipping rates
  - Return quote with cost
}
```

**Key Features:**
- Quote calculation logic
- Address validation
- Multiple shipping method support
- gRPC service

**Why Go?**
- Fast calculations and data processing
- Efficient for CPU-bound operations (math)
- Low latency critical for user experience
- Resource-efficient in containers

**Dependencies:**
- gRPC server
- Internal shipping rate tables
- **Called by:** Checkout Service

---

### 8. **Email Service** (Python)
**Purpose:**
- Send order confirmation emails
- Generate HTML email templates
- Integrate with Gmail/SMTP
- Email logging and tracking

**Code Responsibilities:**
```python
SendOrderConfirmation(email, order) {
  - Load email template (Jinja2)
  - Render template with order data
  - Send via Gmail SMTP
  - Log email sent
}
```

**Key Features:**
- **Jinja2 templating** for HTML emails
- Gmail SMTP integration
- gRPC service interface
- Health checks
- Environment variable configuration
- Structured logging

**Why Python?**
- **String manipulation & templating** - Jinja2 is perfect
- Easy SMTP integration
- Rich standard library (email, smtplib)
- Good for data transformation (order → email)
- Readable code for business logic

**Dependencies:**
- Jinja2 (HTML templating)
- smtplib (Gmail integration)
- gRPC server
- **Called by:** Checkout Service

---

### 9. **Recommendation Service** (Python)
**Purpose:**
- Suggest products based on user history
- Machine learning inference
- Personalization engine
- Analytics

**Code Responsibilities:**
```python
GetRecommendations(user_id, context) {
  - Query user history
  - Run ML model
  - Return top N recommendations
  - Track recommendations
}
```

**Key Features:**
- gRPC service interface
- Possibly ML model inference (TensorFlow/PyTorch)
- User behavior analytics
- Caching recommendations

**Why Python?**
- **Machine learning & data science** - Python dominates this space
- NumPy, Pandas, Scikit-learn ecosystem
- TensorFlow/PyTorch support
- Easy data processing
- Industry standard for ML

**Dependencies:**
- gRPC server
- ML libraries (likely TensorFlow)
- Database for user history
- **Called by:** Frontend (for product suggestions)

---

### 10. **Shopping Assistant Service** (Python + Flask)
**Purpose:**
- AI-powered shopping assistance
- Computer vision for room/style detection
- RAG (Retrieval-Augmented Generation) for product recommendations
- Personalized product suggestions based on image

**Code Responsibilities:**
```python
talkToOpenAI(user_message, room_image) {
  Step 1:
    - Use GPT-4o Vision to analyze room image
    - Extract room style/description
  
  Step 2:
    - Vector search in product database
    - Find products matching room style + user request
    - Use LangChain + PGVector for similarity search
  
  Step 3:
    - Augment prompt with relevant products
    - Call GPT-4o to generate recommendations
    - Return product IDs + explanation
}
```

**Key Features:**
- **Multimodal AI** - vision + language
- **RAG pipeline** - retrieves relevant context
- **Vector embeddings** - semantic search
- **LangChain integration** - LLM orchestration
- **PostgreSQL Vector** - stores product embeddings
- OpenAI API integration

**Why Python?**
- **AI/ML-first language**
- LangChain, LLaMA Index, Pydantic ecosystems
- Easy API integration (OpenAI)
- Flask for lightweight REST API
- Perfect for prompt engineering & data transformation

**Dependencies:**
- OpenAI API (GPT-4o Vision + gpt-4o)
- LangChain for orchestration
- PGVector extension in PostgreSQL
- Flask for REST API
- **Called by:** Frontend (user sends image + message)

---

### 11. **Ad Service** (Java/Gradle)
**Purpose:**
- Serve contextual advertisements
- Product-based ad selection
- Banner/display ad logic
- Ad targeting

**Code Structure:**
```
- Gradle build system
- gRPC service implementation
- Ad matching logic
- Database integration
```

**Why Java?**
- **Enterprise reliability** and stability
- JVM performance (JIT compilation)
- Rich ecosystem for data processing
- Good for CPU-intensive ad matching
- Mature monitoring/observability tools
- Thread-safe operations for concurrent requests

**Dependencies:**
- gRPC (protocol buffers)
- Ad database/catalog
- **Called by:** Frontend (to fetch ads)

---

### 12. **Load Generator** (Python)
**Purpose:**
- Generate artificial load for testing
- Simulate real user traffic
- Performance testing
- Load testing scenarios

**Key Features:**
- Configurable request patterns
- User simulation
- Metric collection
- Latency simulation

**Why Python?**
- Easy to write complex load patterns
- Rich HTTP libraries (requests, httpx)
- Good for scripting and automation
- Can simulate user workflows

---

## Inter-Service Communication

### Communication Map

```
┌─────────────────────────────────────────────────────────────┐
│                        FRONTEND                               │
└──────────────┬──────────────────────────────────────────────┘
               │
        ┌──────┴──────────────┬───────────────────────┐
        │                     │                       │
        ↓                     ↓                       ↓
    ┌────────────┐    ┌────────────────┐     ┌──────────────┐
    │ AUTH       │    │ PRODUCT        │     │ RECOMMENDATION │
    │ SERVICE    │    │ CATALOG        │     │ SERVICE        │
    │ (Python)   │    │ (Go)           │     │ (Python)       │
    └────────────┘    └────────────────┘     └──────────────┘
        │                     │                       │
        │                     └───────┬───────────────┘
        │                             │
        └─────────────────┬───────────┘
                          │
                          ↓
                    ┌──────────────┐
                    │ CHECKOUT     │ (Orchestrator - Go)
                    │ SERVICE      │
                    └──────┬───────┘
                           │
        ┌──────────┬────────┼───────────┬─────────────┐
        │          │        │           │             │
        ↓          ↓        ↓           ↓             ↓
    ┌────────┐ ┌────────┐ ┌────────┐ ┌──────────┐ ┌────────┐
    │PAYMENT │ │SHIPPING│ │CURRENCY│ │CART      │ │EMAIL   │
    │SERVICE │ │SERVICE │ │SERVICE │ │SERVICE   │ │SERVICE │
    │(Node)  │ │(Go)    │ │(Node)  │ │(C#)      │ │(Python)│
    └────────┘ └────────┘ └────────┘ └──────────┘ └────────┘

Additional:
├── Shopping Assistant (Python) - Direct frontend communication
├── Ad Service (Java) - Frontend for ad serving
└── Load Generator (Python) - Testing/load generation
```

### Detailed Service Dependencies

#### Authentication Flow
```
1. Frontend sends credentials to AUTH SERVICE
2. AUTH SERVICE:
   - Validates against PostgreSQL
   - Returns JWT token
3. Frontend includes JWT in all subsequent requests
4. Other services can verify token with AUTH SERVICE
```

#### Checkout Flow (Most Complex)
```
Frontend sends checkout request
    ↓
CHECKOUT SERVICE receives request
    ├─→ CART SERVICE: Get cart items
    ├─→ PRODUCT CATALOG: Get product details & prices
    ├─→ CURRENCY SERVICE: Convert prices to user's currency
    ├─→ SHIPPING SERVICE: Calculate shipping cost
    ├─→ PAYMENT SERVICE: Charge credit card
    ├─→ EMAIL SERVICE: Send confirmation
    └─→ Returns final order confirmation
```

#### Shopping Assistant Flow
```
Frontend sends (image, message)
    ↓
SHOPPING ASSISTANT SERVICE
    ├─→ Call GPT-4o Vision API with image
    ├─→ Get room style/description
    ├─→ Vector search in PostgreSQL with embeddings
    ├─→ Retrieve relevant products
    ├─→ Call GPT-4o with products + message
    ├─→ Generate recommendations
    └─→ Return text + product IDs
```

### Communication Patterns

**Synchronous (gRPC):**
- Product Catalog ← Checkout
- Cart Service ← Checkout
- Currency Service ← Checkout
- Shipping Service ← Checkout
- Payment Service ← Checkout
- Email Service ← Checkout

**Asynchronous Options:**
- Could use message queue (SQS/RabbitMQ) for Email Service
- Could cache product catalogs to avoid per-request calls

---

## Language Choices & Justification

### Summary Table

| Service | Language | Why? | Key Trait |
|---------|----------|------|-----------|
| Auth | Python | Easy credential/JWT logic, boto3 AWS integration | I/O-bound |
| Product Catalog | Go | High performance, fast JSON, low latency | Read-heavy |
| Cart | C# | Enterprise features, strong typing, stateful | Data-intensive |
| Checkout | Go | Orchestration, goroutines for concurrent calls | Orchestrator |
| Payment | Node.js | Simple async logic, lightweight | I/O-bound |
| Currency | Node.js | Simple data transformation, fast | Lightweight |
| Shipping | Go | Fast calculations, low latency | CPU-bound |
| Email | Python | Jinja2 templating, string manipulation | Text processing |
| Recommendation | Python | ML/data science ecosystem | AI/ML |
| Shopping Assistant | Python | GPT integration, LLM orchestration | AI-first |
| Ad Service | Java | Enterprise reliability, JVM optimization | Ad matching |

### Design Principles Behind Language Selection

#### 1. **Python** - Best For:
- **Natural language processing** (Shopping Assistant, Email templates)
- **Machine learning** (Recommendation Service)
- **Rapid development** (Auth - business logic, not performance-critical)
- **AWS integration** (boto3 library)
- **Reasons chosen:** Developer velocity, rich ML/LLM ecosystems, AWS SDK

#### 2. **Go** - Best For:
- **High concurrency** (goroutines for 10k+ concurrent users)
- **Fast execution** (compiled, no GC overhead vs Python)
- **Service orchestration** (Checkout calls 6 services concurrently)
- **Read-heavy workloads** (Product Catalog)
- **Reasons chosen:** Performance, concurrency model, small container footprint

#### 3. **Node.js** - Best For:
- **Lightweight I/O services** (Payment, Currency - request-response only)
- **Async/event-driven** nature matches HTTP services
- **Minimal resource usage** (important in Kubernetes)
- **Reasons chosen:** Speed of development, small memory footprint, event loop

#### 4. **C#/.NET** - Best For:
- **Stateful operations** (Cart with Redis caching + persistence)
- **Enterprise features** (strong typing, mature libraries)
- **Complex data structures** (shopping cart logic)
- **Reasons chosen:** Type safety, OOP maturity, cloud-native support

#### 5. **Java** - Best For:
- **Ad matching algorithms** (CPU-intensive)
- **Enterprise reliability** (proven in production)
- **Concurrent request handling** (thread pools)
- **Reasons chosen:** JVM optimization, threading model, monitoring tools

### Why NOT Monolith?

- **Language choice flexibility**: Use best language per problem
- **Independent scaling**: Heavy services (Checkout) scale separately
- **Technology diversity**: Leverage ecosystem strengths
- **Team autonomy**: Python team ≠ Go team ≠ Node team
- **Fault isolation**: One service failing ≠ entire system down

### Why NOT ALL Same Language?

If all Python:
- ❌ Slow for high-concurrency (Checkout orchestrating 6 calls)
- ❌ Can't use Go's goroutines or Java's JVM optimizations
- ❌ Slower for compute-heavy (Ad matching)

If all Go:
- ❌ Overkill for simple services (Payment, Currency)
- ❌ Missing Python ML/LLM ecosystems
- ❌ Harder template/string manipulation for Email

---

## Technology Stack Summary

### Runtime Environments
- **Python 3.x** - Flask, gRPC, boto3, Jinja2, LangChain, OpenAI
- **Go 1.x** - gRPC, structured logging, OpenTelemetry
- **Node.js 18+** - gRPC, Pino logger, OpenTelemetry
- **C# .NET** - ASP.NET Core, gRPC, Redis, Entity Framework
- **Java** - Spring Boot, Gradle, gRPC

### Data Storage
- **PostgreSQL** - User data (Auth), potentially all persistent data
- **Redis** - Cart caching (high-speed reads/writes)
- **Google Cloud Spanner** - Optional distributed SQL (Cart alternative)
- **PGVector** - Vector embeddings for RAG (Shopping Assistant)

### Communication
- **gRPC** - Service-to-service (typed, fast, efficient)
- **REST/HTTP** - Frontend-to-backend
- **Protocol Buffers** - Data serialization (language-agnostic)

### Observability
- **OpenTelemetry** - Distributed tracing across services
- **Prometheus** - Metrics collection
- **Structured logging** - JSON logs for aggregation
- **CloudWatch** - AWS log aggregation and monitoring

### Security
- **AWS Secrets Manager** - Credential management
- **JWT (HS256)** - Stateless authentication
- **bcrypt** - Password hashing
- **SSL/TLS** - Encrypted communication
- **gRPC credentials** - Secure service-to-service

### Cloud & Deployment
- **AWS EKS** - Kubernetes cluster
- **Docker** - Container images (one per service)
- **Helm** - Kubernetes package management
- **Jenkins + ArgoCD** - CI/CD pipeline (GitOps)

### ML/AI Stack
- **OpenAI API** - GPT-4o, GPT-4o Vision, embeddings
- **LangChain** - LLM orchestration
- **LLaMA/embeddings** - Vector representations

### Testing & Load
- **Load Generator** - Artificial traffic generation
- **Proto testing** - gRPC service testing

---

## Key Architectural Patterns

### 1. **Service Discovery**
- gRPC automatically discovers services via Kubernetes DNS
- Service names resolve to cluster IPs

### 2. **Circuit Breaker Pattern**
- If Payment Service unavailable, Checkout should fail gracefully
- Could implement retry logic + timeout

### 3. **Distributed Tracing**
- Every request traced across all services
- Helpful for debugging checkout failures

### 4. **Health Checks**
- Each service exposes `/health` endpoint
- Kubernetes uses for liveness/readiness probes

### 5. **Graceful Degradation**
- If Recommendation Service down, Frontend still works
- If Ad Service down, checkout still proceeds

---

## Conclusion

This polyglot microservices architecture demonstrates:

1. **Right tool for the job** - Each language chosen for specific strengths
2. **Scalability** - Services scale independently based on load
3. **Resilience** - Failures isolated to specific services
4. **Observability** - Distributed tracing and logging across all services
5. **Cloud-native** - Designed for Kubernetes on AWS
6. **Realistic complexity** - Shows real inter-service dependencies

The choice of languages reflects production decisions:
- **Go** for high-performance orchestration
- **Python** for rapid development + ML capabilities
- **Node.js** for lightweight I/O services
- **C#** for stateful, enterprise-grade services
- **Java** for reliable, tested ad matching

This mirrors real-world systems at scale (e.g., Uber, Netflix, Amazon) where polyglot architectures are standard.
