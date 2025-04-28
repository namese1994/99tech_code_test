Provide your solution here:



---


# **Product Specification – Core Trading Platform (Crypto Spot Market)**

## Purpose  
This document defines the **Product Specification** for the **Crypto Spot Trading Platform Core System**, focusing on **order placement**, **real-time price and trade streaming**, **full historical data retrieval**, and **client web frontend**.  
It establishes the key objectives and requirements necessary for system design and implementation.

---

## Assumptions & Scope

- The platform targets **24/7 global crypto spot trading** — continuous operation including weekends and holidays.
- Authentication is assumed to be handled externally (OAuth2 tokens provided).
- This phase covers:
  - Order Management (placement, cancellation)
  - Matching Engine (atomic matching, trade generation)
  - Real-Time Price & Trade Streaming (WebSocket delivery)
  - Historical Price/Trade Data Retrieval (REST API)
  - Web App Frontend (desktop web interface for users)
- **Out of Scope**:  
  Wallets, fiat gateways, margin/futures trading, KYC onboarding, notifications.

---

##  **Key Characteristics of Crypto Trading**

- **24/7 Global Availability**  
  The platform must operate continuously without maintenance downtime.

- **Highly Volatile Traffic**  
  - Baseline: ~500 RPS.  
  - Burst: up to 2,500 RPS within minutes during market volatility.  
  - Frequent micro-bursts (30s–5min) triggered by market events (e.g., price spikes, financial news).  
  - Weekend traffic drops, mainly due to lower institutional activity, while retail trading remains active.

- **Real-Time Data Streaming**  
  Continuous updates across hundreds of assets require sub-20ms WebSocket latency and resilient event streaming.

- **High Precision and Consistency**  
  - Must maintain p99 latency < 100ms under load.  
  - Strict ACID compliance for orders, trades, and pricing to guarantee correctness.

- **Dynamic Asset Management**  
  Support on-the-fly addition of trading pairs without service interruption.

- **Scalable Historical Data Access**  
  Clients must retrieve historical price/trade data over arbitrary time ranges efficiently, while the system manages growing time-series data cost-effectively.

- **Responsive Frontend Delivery**  
  Deliver a fast and reliable Web Application, with real-time price feeds and historical data queries, optimized for desktop users.


---

## Functional Requirements (FR)

- **FR1:** Accept new and cancel order requests via REST API.
- **FR2:** Persist orders immediately with ACID guarantees (status = pending).
- **FR3:** Match orders atomically inside database transactions.
- **FR4:** Emit price updates and trade events to Kafka upon matching.
- **FR5:** Stream real-time **price updates** to clients via WebSocket over Redis.
- **FR6:** Stream real-time **trade events** to clients via WebSocket over Redis.
- **FR7:** Persist historical price and trade data in a structured database.
- **FR8:** Provide RESTful APIs for querying historical price and trade data by symbol and time range.
- **FR9:** Deliver a responsive **Web App Frontend** that:
  - Subscribes to Price and Trade WebSocket feeds.
  - Sends Order Placement/Cancellation via API.
  - Fetches historical data via REST API.
  - Supports multiple symbols (dynamic).

---

## Non-Functional Requirements (NFR)

- **NFR1:** Sustain 500 RPS baseline load, with burst up to 2,500 RPS within 5 minutes.
- **NFR2:** Ensure system matching p99 latency < 100ms; WebSocket stream latency p99 < 20ms.
- **NFR3:** Guarantee ≥99.99% availability with Multi-AZ setup.
- **NFR4:** Persist event logs in Kafka with 7-day retention.
- **NFR5:** Encrypt all traffic via TLS 1.3; OAuth2 for client authentication.
- **NFR6:** Allow dynamic addition of new trading pairs without downtime.
- **NFR7:** Deliver the Web Frontend as a low-latency, highly available static web app, suitable for scaling globally (CDN delivery).

---

## Interfaces

- **Order API**:  
  — place new order  
  — cancel order

- **Price WebSocket**:  
  — real-time price subscription

- **Trade WebSocket**:  
  — real-time trade subscription

- **History API**:  
  — real-time price subscription

- **Web Frontend App**:
  - Connects to WebSocket servers.
  - Calls REST APIs for order management and history queries.
  - Renders price board, order book, and trade tape dynamically.

---



# **Solution Architecture Decision Report**

## **Strategic Guiding Principles**

Before diving into specific components, I've structured my selections based on clear strategic considerations:

- **Managed Services First:** Prioritize AWS-managed services for reduced operational overhead, faster setup, and lower maintenance. Suite to **start up business**.
- **Right-Sized Scalability:** Select solutions that auto-scale with volatile crypto-market traffic spikes.
- **Cost-Aware Design:** Balance initial cost efficiency with long-term operational savings.
- **Future-Proofing:** Choose technologies that easily adapt to future features (e.g., risk engine, analytics tools, integrations).

---

## Services Diagram


##  Component-by-Component Breakdown

### 【1】Frontend Delivery

- **Amazon S3 (Static Website Hosting)**:  
  Hosts the Single Page Application (SPA) frontend. Static hosting on S3 ensures durability, scalability, and very low operational overhead.

- **Amazon CloudFront (CDN Distribution)**:  
  Distributes the frontend globally, caches static assets at edge locations, minimizing latency for end-users worldwide.

- **Amazon Route 53 (DNS Management)**:  
  Manages DNS resolution, health checks, and failover routing for frontend traffic.

> **Why necessary?**  
> Static SPA must be globally available with low latency to ensure a seamless user experience for real-time trading interfaces.

---

### 【2】Ingress and Routing

- **Application Load Balancer (ALB) → EKS Ingress Controller**:  
  Handles all inbound REST and WebSocket connections from users. ALB offers flexible Layer-7 routing, SSL termination, and integration with EKS.

> **Why necessary?**  
> Enables secure, scalable, and efficient routing to microservices, while supporting WebSocket upgrades for real-time streams.

---

### 【3】Event Streaming Layer – Service Description
Is a buffer layer, avoid blocking when interract with Historical DB and Cache DB

- **Amazon Kinesis Data Streams** serves as the central event bus directly before and after the Matching Engine.  
  - Every matched order produces two events: **`trade.executed`** (trade tape) and **`price.update`** (best bid/ask snapshot).  
  - Producers write to Kinesis with a **partition key = trading pair**, guaranteeing in-order delivery within each symbol.  

> **Why serverless?**  
> Traffic in crypto trading is highly volatile and unpredictable; serverless automatically scales with real-world usage peaks.

---

### 【4】Microservices Layer (Running on Amazon EKS)

- **Order-Entry Service**:  
  Receives order placement/cancellation requests from the frontend. Validates input, persists pending orders into the database, and publishes `orders.new` events to Kafka.

- **Matching Engine**:  
  Consumes new orders from Kafka. Executes atomic order matching with full ACID compliance inside Aurora PostgreSQL. Publishes trade and price update events.

- **Price WebSocket Server**:  
  Subscribes to Redis price channels and pushes real-time price updates to WebSocket clients.

- **Trade WebSocket Server**:  
  Subscribes to Redis trade channels and pushes real-time trade execution updates.

- **History-API Service**:  
  Exposes REST endpoints allowing clients to query historical price and trade data by symbol and time range.

- **Kafka Consumers (Price Consumer, Trade Consumer, History Writer)**:  
  - Price Consumer: Updates Redis price channels based on Kafka `price.update` events.
  - Trade Consumer: Updates Redis trade channels based on Kafka `trade.executed` events.
  - History Writer: Persists price and trade events into a partitioned Aurora database for historical queries.

> **Why decompose services like this?**  
> - Isolates concerns (order intake vs matching vs streaming vs history)  
> - Enables independent scaling based on workload patterns (e.g., scale WebSocket servers separately when user connections spike).  
> - Improves resilience and fault tolerance across services.

---

### 【5】Transactional Database

- **Amazon Aurora PostgreSQL**:  
  Main ACID-compliant database storing pending orders, executed trades, and order book state.  
  Ensures atomicity and strong consistency for critical trading operations.

> **Why necessary?**  
> Real-money transactions require strict ACID guarantees.  
> Aurora offers predictable latency, high availability (Multi-AZ), and future upgrade path to Global Database if needed.

---

### 【6】Historical Time-Series Storage

- **Amazon Aurora PostgreSQL (Partitioned Tables)**:  
  Stores historical price and trade data.  
  Data is partitioned by symbol and date to optimize query performance over large datasets.

> **Why initially stay with Aurora?**  
> - Minimizes system complexity during the MVP phase.  
> - Provides immediate JOIN capability across live and historical data if needed (when opening price dashboard).  
> - Enables cost-effective management until data grows large enough to justify separate time-series optimized storage.

---

### 【7】Real-Time Caching and Fan-Out Layer

- **Amazon ElastiCache Redis (Cluster-Mode)**:  
  Stores latest price and trade updates in Redis pub/sub channels.  
  Supports sub-millisecond retrieval for fan-out to thousands of WebSocket clients.

> **Why necessary?**  
> Redis acts as a highly efficient intermediary to offload the Kafka cluster and ensure real-time responsiveness for streaming to clients.

> **Why cluster mode?**  
> - Allows horizontal scaling via online resharding.  
> - Prevents bottlenecks during sudden trading bursts.


## **Detailed Component Selection & Strategic Rationale**

### 1. **Web Frontend Delivery**

**Requirement:**  
Serve a single-page application (SPA) globally, with minimal latency and cost-efficiency.

| Chosen Solution | Strategic Rationale | Alternative Solution & Comparison |
|-----------------|---------------------|------------------------------------|
| **S3 + CloudFront** | - Extremely low operational overhead, highly reliable. <br>- Excellent global latency due to CloudFront edge caching. <br>- Cost-effective pricing (pay-per-use). | **AWS Amplify Hosting:** <br>- Simplifies frontend developer workflows with built-in Git integration. <br>- Slightly higher costs due to added conveniences. <br>- Suitable if rapid frontend iterations and integrated CI/CD workflows become a priority. <br><br> **EKS + Nginx** <br>- serve static by Nginx <br>- Or route to hosting with dynamic website ran in pod if required  |

---

### 2. **Ingress & Load Balancing (REST & WebSocket)**

**Requirement:**  
Effectively route both REST API requests and WebSocket connections securely to backend microservices, providing flexible scalability under fluctuating traffic.

| Chosen Solution | Strategic Rationale | Alternative Solutions & Comparison |
|-----------------|---------------------|------------------------------------|
| **AWS Application Load Balancer (ALB) + EKS Ingress** | - Native Kubernetes integration simplifies deployment and management. <br>- Cost-effective for consistently high request volumes. <br>- Flexible HTTP/2 support enhances real-time interactions (e.g., WebSocket upgrades). | **API Gateway HTTP API:** <br>- Built-in JWT authentication and rate limiting simplifies security management at lower traffic volumes. <br>- Becomes significantly more expensive at high, sustained RPS. Ideal if steady-state traffic remains relatively low.<br><br>**Network Load Balancer + Global Accelerator:** <br>- Offers lower latency and stable IP addresses globally. <br>- Beneficial if latency becomes critical or multi-region load balancing is required in the future. |

---

### 3. **Event Streaming & Messaging Layer**

**Requirement**  
Deliver a reliable, high-throughput event backbone (after the Matching Engine) to broadcast `price.update`, `trade.executed`, and other domain events to internal consumers today—and to future services (risk, analytics, audit, partner feeds) tomorrow.

| **Chosen Solution** | **Strategic Rationale** | **Alternative Solutions & When to Favour Them** |
|---------------------|-------------------------|-------------------------------------------------|
| **Amazon Kinesis Data Streams** | • Fully **serverless**: no brokers to size, patch, or scale.<br>• Sub-second autoscaling on shard count—ideal for the **spiky, burst-heavy** traffic of a crypto exchange.<br>• Simple, predictable pricing: pay for PUT payloads and shard-hours; no per-GB egress fees when consumers are in-VPC.<br>• Exactly-once processing achievable with Kinesis Enhanced Fan-Out + a per-partition sequence number, which is **sufficient for price/trade streams** that are naturally append-only. | **MSK Serverless (Kafka API)**<br>—Choose when strict topic-level ordering across many partitions, rich Kafka ecosystem tooling, or binary protocol compatibility with external partners is essential.<br>—Higher base cost (cluster-hour + GB in/out) but supports large historical retention and consumer-managed offsets.<br><br>**MSK Provisioned (+ Tiered Storage)**<br>—Best when traffic becomes **steady >40 % of peak 24/7** and the team is ready to handle broker sizing and patching.<br>—Lowest long-term cost per GB at very high sustained throughput; tiered storage off-loads cold segments to S3. |

---

### 4. **Microservices Compute Layer**

**Requirement:**  
Run scalable microservices reliably and cost-effectively—including REST APIs, the real-time Matching Engine, event consumers, and WebSocket servers.

| Chosen Solution | Strategic Rationale | Alternative Solutions & Comparison |
|-----------------|---------------------|------------------------------------|
| **Amazon EKS + Karpenter with Graviton Instances** | - Kubernetes provides mature orchestration and ecosystem support, reducing complexity. <br>- Karpenter auto-scales nodes rapidly (within minutes) to handle sudden traffic spikes effectively. <br>- Graviton instances significantly reduce compute costs (20–30% savings). <br>- Given the continuous 24/7 runtime, server-based solutions offer better cost-performance efficiency than purely serverless options. | **ECS on Fargate:**<br>- Lower operational complexity, suitable if the Kubernetes ecosystem appears overly complex for your team.<br>- Slightly higher cost (~20%) when running workloads continuously.<br><br>**EC2 with Auto Scaling Group and without container**<br>- Higher performance than container solutions. Suitable for large traffic. <br>- Higher operational effort and higher complexity for setupping, configuring and updating <br>- |

---

### 5. **Transactional (ACID) Database Layer**

**Requirement:**  
Reliable, performant database ensuring ACID compliance for atomic order transactions, matching logic, and real-time consistency.

| Chosen Solution | Strategic Rationale | Alternative Solutions & Comparison |
|-----------------|---------------------|------------------------------------|
| **Amazon Aurora PostgreSQL (Provisioned)** | - Proven performance and predictable cost (with reserved instances). <br>- Easily supports future Global Database setup for disaster recovery or global expansion. <br>- Best choice given stable high-performance ACID transactions requirements. | **Aurora Serverless v2:**<br>- Automatically scales based on traffic; ideal if significant idle periods occur regularly.<br>- Less optimal if your DB consistently experiences high load.<br><br>**RDS PostgreSQL Multi-AZ (GP3 storage):**<br>- Lower initial costs for smaller DB loads; requires manual sharding later as system scales significantly.<br><br>**Aurora Global Database:**<br>- Superior cross-region replication. Ideal future choice once geographic redundancy and global DR become critical requirements. |

---

### 6. **Historical Time-Series Storage**

**Requirement:**  
Efficiently store and query a large and growing volume of historical price and trading data.  
Support real-time querying for recent data, and cost-effective archiving and retrieval for older data (e.g., user-facing charts, audits, reporting).

| Chosen Solution | Strategic Rationale | Alternative Solutions & Comparison |
|-----------------|---------------------|------------------------------------|
| **Partitioned Aurora PostgreSQL (shared cluster)** | - Immediate compatibility with the core relational model (orders, trades, assets).<br>- Simplifies initial deployment by sharing the same Aurora instance as transactional data.<br>- Good performance for moderate-sized time-series workloads when using table partitioning (by asset and time).<br>- Suitable when real-time historical data queries must support strong JOINs or relational constraints. | **Amazon Timestream:**<br>- Purpose-built serverless time-series database.<br>- Highly optimized for ingestion and query of large volumes of time-series data.<br>- Scales automatically without managing servers.<br>- Limited JOIN capabilities; best when historical queries are mostly single-table (e.g., charting price history per symbol).<br><br>**Amazon DocumentDB (MongoDB-compatible):**<br>- Flexible JSON-like schema, good for semi-structured tick/trade records.<br>- Suitable if the historical data schema is expected to evolve frequently.<br>- Higher operational cost at scale; not as optimized for pure time-series as Timestream.<br><br>**Aurora Serverless v2 (separate cluster):**<br>- Move historical tables to a dedicated Aurora Serverless v2 cluster.<br>- Provides autoscaling compute capacity (0.5–128 ACU).<br>- Ideal when the majority of history queries are sporadic but occasional bursty loads occur (e.g., during user charting spikes).<br><br>**S3 Data Lake (Parquet format) + AWS Glue ETL + Athena:**<br>- Offload older historical data (>90 days) to S3 for ultra-low storage cost.<br>- Query historical archives on demand with Athena (serverless SQL queries).<br>- Latency unsuitable for live client queries (<5s per query); more appropriate for compliance, audit, batch reporting, or BI aggregation.|



---

### 7. **Realtime Cache / PubSub Layer**

**Requirement:**  
Low-latency (<1ms) caching and fan-out messaging for streaming real-time price and trade updates.

| Chosen Solution | Strategic Rationale | Alternative Solutions & Comparison |
|-----------------|---------------------|------------------------------------|
| **ElastiCache Redis Cluster-Mode** | - Proven sub-millisecond latency performance, critical for real-time market data updates. <br>- Supports online resharding for horizontal scalability without downtime. | **MemoryDB for Redis:**<br>- Offers durability and fast recovery from failures, ensuring data persistence.<br>- Ideal if persistence of last cached prices across restarts is critical (e.g., regulatory requirements).<br>- Around 30% more expensive.<br><br>**Redis Global Datastore:**<br>- Enables immediate cross-region disaster recovery and reduced latency for global users.<br>- Suitable when DR and global redundancy become mandatory.<br><br>**Auto-Scaling ElastiCache Redis:**<br>- Automatically manages Redis cluster scaling based on load, suitable when traffic patterns become highly variable. |

---

### 8. **Observability & Monitoring Layer**

**Requirement:**  
Comprehensive monitoring, logging, and tracing integrated seamlessly with AWS.

| Chosen Solution | Strategic Rationale | Alternative Solutions & Comparison |
|-----------------|---------------------|------------------------------------|
| **CloudWatch + Managed Prometheus + Managed Grafana** | - Fully managed AWS-native solution requiring minimal operational effort. <br>- Cost-effective pay-as-you-go model aligned with cloud-native workloads. | **AWS OpenSearch:**<br>- Suitable when deep log analytics, detailed troubleshooting, and text-search capabilities become priorities.<br><br>**ClickHouse Cloud on AWS**<br>- lower cost for storing log and metrics. <br>- need more operation and setup effort |

---

### 9. **Security & Compliance**

**Requirement:**  
Strong perimeter security, DDoS protection, efficient secret management, and secure TLS endpoints.

| Chosen Solution | Strategic Rationale | Alternative Solutions & Comparison |
|-----------------|---------------------|------------------------------------|
| **AWS WAF + Shield Standard + ACM + Secrets Manager** | - Provides foundational, cost-effective security suitable for initial platform launch. | **Shield Advanced:**<br>- Recommended once platform grows, with higher DDoS risks and critical uptime SLA.<br><br>**AWS Private CA:**<br>- Useful if internal microservices require mutual TLS (mTLS).<br><br>**AWS IAM Identity Center (AWS SSO):**<br>- Simplifies multi-account management and third-party IdP integration for enterprise-grade IAM. |



---

# Plans for Scaling When the Product Grows Beyond Current Setup
### 1. **Event Streaming (Kinesis to Kafka Migration)**

- **Current setup**: Amazon Kinesis Data Streams (serverless)
- **Scaling trigger**:
  - Number of downstream consumers exceeds **20 services**
  - Sustained throughput approaches or exceeds **200 MB/s** aggregate data ingestion.


- **Planned scaling approach**:
  - **Step 1**: Set up Amazon MSK Serverless alongside Kinesis; route selected high-volume or complex event streams first.
  - **Step 2**: Gradually migrate consumers from Kinesis to MSK Serverless topics.
  - **Step 3**: If traffic stabilizes and remains consistently high (>50% of peak), migrate further to MSK Provisioned with Tiered Storage to lower long-term operational costs.

---

### 2. **Database Layer (Aurora PostgreSQL)**

- **Current setup**: Aurora PostgreSQL provisioned (single writer, multiple readers)
- **Scaling trigger**:
  - Write capacity (ACU utilization) consistently exceeds **80%** or experiences noticeable latency degradation (>50 ms p99 regularly).
  - Reader replicas > **10 nodes**, impacting replication latency.

- **Planned scaling approach**:
  - **Horizontal Sharding**: Split Aurora clusters by geographical market segments (US, EU, Asia).
  - **Aurora Global Database**: Set up active-active cross-region replication if latency-sensitive global expansion is required.


---

### 3. **Historical Data Storage**

- **Current setup**: Aurora PostgreSQL partitioned tables
- **Scaling trigger**:
  - Query latency regularly exceeding **50 ms** for common user-facing queries.
  - Data volume growth rate exceeds **10 TB per quarter**.

- **Planned scaling approach**:
  - **Hot data** (recent 30–90 days): migrate to Amazon Timestream (serverless, auto-tiered).
  - **Warm/Cold data** (>90 days): Archive in S3 Data Lake (Parquet) with AWS Glue ETL. Athena queries handle ad-hoc access.


---

### 4. **Microservices Compute (EKS Cluster)**

- **Current setup**: Amazon EKS with Karpenter 
- **Scaling trigger**:
  - Sustained EKS cluster size reaches > **500 pods**, causing scheduling latency and complexity.
  - Node autoscaling response no longer acceptable during burst (EKS node provision latency consistently >3 min).

- **Planned scaling approach**:
  - Split workloads into dedicated EKS clusters (e.g., Order Processing cluster, WebSocket cluster, Analytics & Batch cluster).
  - Adopt hybrid EKS + EC2 strategy: Split the traffic, one part for EKS with auto-scaling, the other for stable workloads on dedicated EC2 nodes.
  - Introduce separate node groups tuned for specialized workloads (Matching Engine high-CPU instances, Analytics high-memory instances).
  - Use large resource instance nodes to reduce the number of managed nodes

---

### 5. **Real-Time Cache & PubSub Layer (Redis)**

- **Current setup**: ElastiCache Redis Cluster-mode
- **Scaling trigger**:
  - Redis throughput regularly exceeds **1 million requests/second**.
  - Pub/Sub consumer latency exceeds **50 ms**, affecting WebSocket fan-out performance.

- **Planned scaling approach**:
  - Deploy Redis Clusters dedicated per data domain (e.g., separate Price cache cluster, Trade cache cluster, Order-book cluster).
  - Implement Redis Global Datastore if cross-region caching for low-latency global customers becomes mandatory.

---

### 6. **Frontend Delivery (S3 + CloudFront)**

- **Current setup**: Amazon S3 static hosting + CloudFront CDN
- **Scaling trigger**:
  - Significant frontend logic expansion (e.g., server-side rendering or personalization).
  - CDN invalidation and cache latency impacts end-user UX.

- **Planned scaling approach**:
  - Adopt hybrid CloudFront Functions + Lambda@Edge for dynamic, real-time customization at CDN edge.
  - Move towards EKS or App Runner storage if frontend logic complexity is high or with a dynamic website.

---

### 7. **Security, Observability, and Compliance**

- **Current setup**: AWS WAF + Shield Standard, ACM, Secrets Manager, CloudWatch/AMP/Grafana
- **Scaling trigger**:
  - Increased regulatory and compliance complexity (financial audits, SOC2, ISO 27001).
  - Significant spike in DDoS/security threats or targeted attacks.

- **Planned scaling approach**:
  - Upgrade to Shield Advanced with Dedicated Response Team support.
  - Switch to Clickhouse to save on log/metric storage costs.

---

## **Summary of Scaling Strategy**

| Component                | Current Choice        | Next-Step Scaling                             |
|--------------------------|-----------------------|-----------------------------------------------|
| Event Streaming          | Kinesis Data Streams  | ➡️ MSK Serverless / Provisioned Kafka         |
| Transactional DB         | Aurora PostgreSQL     | ➡️ Aurora Sharding / Global Database          |
| Historical Data          | Aurora Partitioned    | ➡️ Timestream + S3/Athena Data Lake           |
| Microservices Compute    | EKS + Karpenter       | ➡️ Multi-Cluster EKS, Hybrid EC2/EKS          |
| Cache & Pub/Sub          | Redis Cluster-mode    | ➡️ Multi-cluster Redis, Global Datastore      |
| Frontend Hosting         | S3 + CloudFront       | ➡️ Amplify, Lambda@Edge , App Runner , EKS    |
| Security & Observability | Shield Standard, WAF  | ➡️ Shield Advanced, Clickhouse                |

---
