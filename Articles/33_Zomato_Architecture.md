# System Design of Zomato

## 1. Overview
Zomato is a comprehensive food delivery and restaurant discovery platform. It allows users to search for restaurants, view menus, place orders, track deliveries, and make payments. 

The goal is to design a scalable, highly available system to handle millions of users and real-time order deliveries.

---

## 2. Architecture Diagram

```mermaid
flowchart TD
    subgraph Clients
        UApp[User App<br/>Android/iOS]
        WApp[Web App]
        RApp[Restaurant App]
    end

    AG[API Gateway]

    subgraph Backend Services
        US[User Service]
        RS[Restaurant Service]
        MS[Menu Service]
        OS[Order Service]
        PS[Payment Service]
        DS[Delivery Service]
        NS[Notification Service]
    end

    subgraph Data Stores
        UDB[(User DB)]
        RDB[(Restaurant DB)]
        ODB[(Order DB)]
        Redis[(Cache - Redis)]
        NoSQL[(NoSQL - MongoDB)]
    end

    MQ[[Message Queue<br/>Kafka/RabbitMQ]]
    ES[Search Service<br/>Elasticsearch]
    S3[Image Storage<br/>S3/CDN]
    ADP[Analytics Data Pipeline]

    subgraph External Services
        Maps[Google Maps]
        PG[Payment Gateway]
        SMS[SMS/Email]
        PN[Push Notifications]
    end

    Clients --> AG
    AG --> BackendServices
    US & RS & MS & OS & PS & DS & NS <--> DataStores
    OS & PS & DS --> MQ
    RS & MS --> ES
    US & RS & MS --> S3
    BackendServices --> ADP

    DS --> Maps
    PS --> PG
    NS --> SMS
    NS --> PN
```

*(Note: The diagram above illustrates the high-level flow and interactions between various microservices, data stores, and external components.)*

---

## 3. Components Explanation

### 1. Clients
Users, restaurants, and delivery partners use mobile (Android/iOS) and web apps to interact with the platform.

### 2. API Gateway
The **single entry point** for all incoming requests. It handles:
- Routing
- Rate limiting
- Authentication
- Request aggregation

### 3. Backend Services (Microservices)
- **User Service:** Handles user registration, login, profile management.
- **Restaurant Service:** Manages restaurant onboarding, details, ratings, and availability.
- **Menu Service:** Manages menu items, categories, and prices.
- **Order Service:** Handles order creation, status updates, and cancellations.
- **Payment Service:** Processes payments, refunds, wallets, and promotional offers.
- **Delivery Service:** Assigns delivery partners, tracks live location, and optimizes routes.
- **Notification Service:** Sends SMS, email, and push notifications to users.

### 4. Data Stores
- **Relational DB (MySQL/PostgreSQL):** Used for core transactional data (users, restaurants, orders) to maintain ACID properties.
- **Cache (Redis):** Stores session data, frequently accessed data (like restaurant menus), leaderboards, and active offers.
- **NoSQL DB (MongoDB/Cassandra):** Used for unstructured or high-volume data like reviews, logs, and activity feeds.

### 5. Message Queue
Systems like **Kafka** or **RabbitMQ** decouple services and handle asynchronous tasks such as order status updates, notifications, and payment processing events.

### 6. Search Service
**Elasticsearch** is used for the fast, fuzzy searching of restaurants, cuisines, and specific dishes.

### 7. Image/File Storage
Stores restaurant images, food pictures, and documents in object storage (like AWS S3) and delivers them globally via a **CDN** (Content Delivery Network).

### 8. Analytics Data Pipeline
Collects real-time events (orders, searches, clicks) and processes them for business insights, dashboards, and personalized recommendations.

### 9. External Services
- **Maps (Google Maps):** Used for location mapping, distance calculation, ETA calculation, and route optimization.
- **Payment Gateway:** Third-party integrations (Stripe, Razorpay, etc.) for cards, wallets, and Netbanking.
- **SMS/Email Service:** Third-party services (Twilio, SendGrid) for OTPs, order updates, and marketing offers.
- **Push Notification:** FCM/APNs for real-time notifications to users and delivery partners.

---

## 4. High-Level Order Flow

```mermaid
sequenceDiagram
    participant User
    participant OrderService
    participant PaymentService
    participant Restaurant
    participant DeliveryService

    User->>OrderService: 1. Places Order
    OrderService->>PaymentService: 2. Initiates Payment
    PaymentService-->>OrderService: 3. Payment Processed
    OrderService->>Restaurant: 4. Order Status Sent (Accepted)
    OrderService->>DeliveryService: 5. Assigns Delivery Partner
    DeliveryService-->>User: 6. Order Delivered
```

1. **User places order**
2. **Order Service creates order**
3. **Payment Service processes payment**
4. **Order status sent to Restaurant**
5. **Delivery Partner assigned**
6. **Order delivered to User**

---

## 5. Key Design Considerations

- **Scalability:** Microservices architecture, horizontal scaling, and robust load balancers.
- **High Availability:** Multi-zone deployment, database replication, and automatic failover.
- **Performance:** Extensive caching (Redis), CDN for static assets, database indexing, and async processing (Message Queues).
- **Reliability:** Message queues for guaranteed delivery, retry mechanisms, and frequent database backups.
- **Security:** Standardized authentication (OAuth2), strict authorization, and data encryption (at rest and in transit).
- **Monitoring:** Centralized logging, metric dashboards, and automated alerting.
