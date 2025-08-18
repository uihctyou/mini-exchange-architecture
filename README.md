
# Mini Exchange - System Architecture Documentation

![Version](https://img.shields.io/badge/version-v1.0.0-red)
![Status](https://img.shields.io/badge/status-active-brightgreen)
![License](https://img.shields.io/badge/License-MIT-green?logo=opensourceinitiative)

A minimal cryptocurrency exchange backend and frontend architecture for learning, interview preparation, and portfolio demonstration.  
This project is designed to showcase **microservices architecture**, **full-stack development**, and **trading system fundamentals**.



## 1. Overview

The system simulates a simplified cryptocurrency spot exchange, including:

- User authentication & account management
- Order placement (limit & market orders)
- Order matching engine
- Trade settlement & clearing
- Real-time market data streaming
- Web frontend for trading and monitoring



## 2. High-Level Architecture

```plain text

[ Admin Portal (Web/Mobile) ]                     [ User Portal (Web/Mobile) ]       
(User Mgmt, Risk Ctrl, Market Surveillance)       (Auth, Order, Order Book, Trades, Assets)                
              |                                  /                |
              |                                 /                 |
              |                                /                  |
              |                               /                   |
           REST/JWT                       REST/JWT            WebSocket
              |                              |                    |
         ┌────▼────────┐                ┌────▼────────┐     ┌─────▼───────┐
         │ API Gateway │                │ API Gateway │     │ WS Gateway  │ ← (Market Data / Order Book / Trades)
         └────┬────────┘                └────┬────────┘     └────┬────────┘
              |                              |                   |  Pub/Sub (Kafka / Redis Stream)
   ┌──────────▼──────────────────────────────▼───────────────────▼───────────────────────────────────────┐
   │                                    Service Layer (Spring Boot)                                      │
   │                Auth Service | Account Service | Risk Service | Order Service                        │
   │                Matching Engine | Clearing Service | Market Data Service                             │
   └───────────────────┬─────────────┬───────────┬──────────┬─────────┬───────────────┬──────────────────┘
                       │             │           │          │         │               │
               ┌───────▼───┐     ┌───▼────┐      │     ┌────▼────┐    │   ┌───────────▼──────────┐
               │ PostgreSQL│     │ Redis  │      │     │ Matching│    │   │ Outbox (TX Log)      │ ← Event Publish
               │  (ACID)   │     │ Cache/ │      │     │ Engine  │    │   │ (Integration Events) │
               │           │     │ Lock   │      │     └────┬────┘    │   └───────────┬──────────┘
               └───────────┘     └────────┘      │          │         │               │
                                                 │   Internal Events  │         Monitoring / Logging
                                                 │                    │  
                                             ┌───▼───────┐       ┌────▼────────────────┐
                                             │ Trades DB │       │ Prometheus / Graf   │
                                             │ (append)  │       │ ELK / OpenTelemetry │
                                             └───────────┘       └─────────────────────┘
```



### **Frontend** (Next.js)
   
* **User Portal Web / Mobile UI**

  * Order placement interface
  * Live order book & trade feed (WebSocket)
  * Balance & transaction history
  * Notifications / Alerts for order updates
 
* **Admin Portal Web / Mobile UI**
  * User & market management dashboard
  * System monitoring & configuration management
  * Analytics and reporting interfaces


### **Backend Microservices** (Java + Spring Boot / Spring Cloud)
   
| Service                             | Responsibility                                                                  |
| ----------------------------------- | ------------------------------------------------------------------------------- |
| **Auth Service**                    | JWT authentication, user registration, optional OAuth2 / MFA support            |
| **Account Service**                 | Manage balances, transaction history, respond to Clearing Service events        |
| **Risk / Validation Service**       | Validate orders, check user balances, enforce limits and trading rules          |
| **Order Service**                   | Receive orders, persist order state, forward to Matching Engine                 |
| **Matching Engine**                 | Match buy/sell orders based on price-time priority, emit Trade Events           |
| **Clearing Service**                | Settle trades, update Account balances reliably (Outbox/Event Sourcing pattern) |
| **Market Data Service**             | Aggregate and stream order book, trade, and K-line data via WebSocket           |
| **Notification Service (optional)** | Push notifications (Email/SMS/WebSocket) for order and trade events             |
| **Admin Service (optional)**        | Expose APIs for admin dashboard, monitoring, and configuration management       |
| **Monitoring & Logging Service**    | Collect metrics, logs, distributed tracing (Sleuth / Zipkin / ELK)              |

### **Infrastructure**
* **API Gateway**: Spring Cloud Gateway / Nginx
* **Service Discovery & Config**: Eureka / Consul + Spring Cloud Config
* **Message Queue**: Kafka / RabbitMQ (event-driven communication)
* **Database**: PostgreSQL (relational, account and order data)
* **Cache**: Redis (order book, live market data)
* **Containerization & Orchestration**: Docker Compose / Kubernetes
* **Metrics & Monitoring**: Prometheus + Grafana, distributed tracing & logging



## 3. Data Flow

### **Order Execution Flow**
```mermaid
sequenceDiagram
    participant User
    participant Frontend
    participant API Gateway
    participant Order Service
    participant Risk Service
    participant Matching Engine
    participant Clearing Service
    participant Account Service

    User->>Frontend: Place order
    Frontend->>API Gateway: Send order request
    API Gateway->>Risk Service: Validate order
    Risk Service-->>API Gateway: Validation result
    API Gateway->>Order Service: Forward order
    Order Service-->>Frontend: Order received confirmation
    Order Service->>Matching Engine: Submit order
    Matching Engine->>Matching Engine: Match with existing orders
    Matching Engine-->>Order Service: Update order status (partial/full fill)
    Matching Engine-->>Clearing Service: Emit Trade Event
    Clearing Service->>Account Service: Update balances
    Account Service-->>Frontend: Notify updated balances

````



## 4. Technology Stack

![Java](https://img.shields.io/badge/Java-17-blue)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.2.0-brightgreen?logo=springboot)
![Spring Cloud](https://img.shields.io/badge/Spring%20Cloud-2023.0-lightgrey?logo=spring)
![Next.js](https://img.shields.io/badge/Next.js-14-black?logo=nextdotjs)
![TypeScript](https://img.shields.io/badge/TypeScript-5-blue?logo=typescript)
![Kafka](https://img.shields.io/badge/Kafka-Event%20Streaming-231f20?logo=apachekafka)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-DB-blue?logo=postgresql)
![Redis](https://img.shields.io/badge/Redis-Cache-red?logo=redis)
![Docker](https://img.shields.io/badge/Docker-Enabled-2496ed?logo=docker)
![Kubernetes](https://img.shields.io/badge/Kubernetes-Orchestration-326ce5?logo=kubernetes)
![Swagger](https://img.shields.io/badge/API%20Docs-Swagger-85ea2d?logo=swagger)

| Layer             | Technology / Tools                                                 | Notes / Purpose                                                                          |
| ----------------- | ------------------------------------------------------------------ | ---------------------------------------------------------------------------------------- |
| **Frontend**      | Next.js, TypeScript, WebSocket, Tailwind / Ant Design              | UI for users & admins, live market data updates                                          |
| **Backend**       | Java 17, Spring Boot, Spring Cloud, REST / gRPC                    | Core microservices, inter-service communication, API endpoints                           |
| **Messaging**     | Kafka / RabbitMQ                                                   | Event-driven communication between microservices (Order → Matching → Clearing → Account) |
| **Database**      | PostgreSQL                                                         | Relational DB for accounts, orders, transactions; ACID compliance                        |
| **Cache**         | Redis                                                              | Low-latency storage for order book, live market data, and temporary state                |
| **Deployment**    | Docker Compose / Kubernetes + CI/CD (GitHub Actions / Jenkins)     | Containerization, orchestration, and automated deployment                                |
| **Monitoring**    | Prometheus + Grafana, Sleuth / Zipkin                              | Metrics, observability, distributed tracing, performance monitoring                      |
| **Documentation** | Markdown, Mermaid diagrams, OpenAPI / Swagger                      | Architecture diagrams, sequence diagrams, API documentation                              |



## 5. Repository Structure

```
mini-exchange-architecture/
│
├── diagrams/               # System diagrams (architecture, sequence, ERD)
├── docs/                   # Additional documentation
│   ├── frontend/           # Frontend docs
│   └── backend/            # Backend docs
├── README.md               # This file
└── LICENSE
```

> Using Kubernetes Deployments to ensure zero-downtime rolling updates and auto-healing for all microservices.



## 6. How to Use This Repo

1. **Study the architecture** before diving into the backend/frontend repos.
2. **Check the diagrams** to understand the system flow.
3. **Refer to API specs** for integration between services.
4. **Use the deployment configs** to run the system locally or in the cloud.



## 7. Related Repositories

* **Backend Microservices** → [mini-exchange-backend](https://github.com/uihctyou/mini-exchange-backend)
* **Frontend** → [mini-exchange-frontend](https://github.com/uihctyou/mini-exchange-frontend)



## 8. License

![License](https://img.shields.io/badge/License-MIT-green?logo=opensourceinitiative)

MIT License

