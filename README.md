<div align="center">  
  <br><br>

  <h1 style="font-family: 'Segoe UI', sans-serif; font-weight: 300; letter-spacing: 6px; text-transform: uppercase; border-bottom: none;">WANDERLUST</h1>
  <h3 style="font-family: 'Segoe UI', sans-serif; font-weight: 400; letter-spacing: 2px; text-transform: uppercase; color: #666;">Autonomous Distributed Cloud Ecosystem</h3>

  <p>
    <img src="https://img.shields.io/badge/Core_Architecture-Event_Driven_Microservices-000?style=for-the-badge&logo=codepen&logoColor=white" alt="Architecture"/>
    <img src="https://img.shields.io/badge/Primary_Runtime-Golang_1.21-00ADD8?style=for-the-badge&logo=go&logoColor=white" alt="Go"/>
    <img src="https://img.shields.io/badge/Graph_Service-Node.js_LTS-339933?style=for-the-badge&logo=nodedotjs&logoColor=white" alt="Node"/>
    <img src="https://img.shields.io/badge/Containerization-Docker_Mesh-2496ED?style=for-the-badge&logo=docker&logoColor=white" alt="Docker"/>
    <img src="https://img.shields.io/badge/Data_Fabric-Neo4j_%7C_Mongo_%7C_Postgres-008CC1?style=for-the-badge&logo=neo4j&logoColor=white" alt="Polyglot"/>
    <img src="https://img.shields.io/badge/Observability-LGTM_Stack-F46800?style=for-the-badge&logo=grafana&logoColor=white" alt="Observability"/>
  </p>

  <p align="center" style="max-width: 900px; margin: auto; padding-top: 20px; font-size: 1.1em; line-height: 1.6;">
    <strong>Wanderlust is a production-grade, polyglot microservices ecosystem engineered to revolutionize the digital tourism domain.</strong> 
    <br>
    Built to demonstrate extreme scalability and fault tolerance, it orchestrates a mesh of 15+ containerized services using <strong>SAGA distributed transactions</strong>, <strong>Graph Theory algorithms</strong>, and a full-spectrum <strong>Telemetry Pipeline</strong>. It serves as a reference implementation for handling distributed state in a cloud-native environment.
  </p>

  <br>
  <a href="#architectural-vision"><strong>Explore Architecture »</strong></a>
  <span style="padding: 0 10px">|</span>
  <a href="#engineering-deep-dive"><strong>Engineering Deep Dive »</strong></a>
  <span style="padding: 0 10px">|</span>
  <a href="#infrastructure-as-code"><strong>Infrastructure Strategy »</strong></a>
</div>

<br>

---

## 1. Architectural Vision and Paradigm

The transition from monolithic architectures to distributed systems introduces complexity in data consistency and network latency. **Wanderlust** was engineered to solve these specific challenges through a **Polyglot Microservices Architecture**.

Unlike traditional applications that rely on a single database ("Golden Hammer"), Wanderlust adopts a **Database-per-Service** pattern, selecting the optimal persistence layer for each specific domain problem. The system is decoupled via asynchronous event streams, ensuring that a failure in the "Blog" subsystem does not impact the critical "Financial" transaction path.

### Core Engineering Pillars
*   **Vertical Decoupling:** Services are isolated by domain boundaries (DDD), communicating strictly via well-defined gRPC contracts and Event Buses.
*   **Polyglot Persistence Strategy:**
    *   **Neo4j (Graph):** For O(1) adjacency traversals in social recommendation algorithms.
    *   **MongoDB (Document):** For high-write throughput of unstructured telemetry and rich-text content.
    *   **PostgreSQL (Relational):** For ACID-compliant financial ledgers and transaction integrity.
*   **Event-Driven Consistency:** Leveraging **RabbitMQ** to ensure eventual consistency across disparate data stores without tight coupling.

---

## 2. Engineering Deep Dive

### 2.1 The SAGA Orchestration Pattern
One of the most complex challenges in distributed systems is maintaining data integrity across boundaries without global locks (2PC). Wanderlust implements an **Orchestration-based SAGA pattern** to handle the "Tour Purchase" lifecycle.

*   **The Challenge:** A purchase requires atomic updates across three isolated databases: reserving inventory in MongoDB (Tour Service), charging a wallet in PostgreSQL (Purchase Service), and unlocking permissions in Neo4j (Stakeholder Service).
*   **The Solution:** The `Stakeholder Service` acts as the SAGA Orchestrator (State Machine).
    1.  **Transaction Start:** The Orchestrator emits a command to the Purchase Service.
    2.  **Local ACID:** Purchase Service executes a local transaction in PostgreSQL and returns `Success` or `Failure`.
    3.  **Forward Recovery:** Upon success, the Orchestrator triggers the Tour Service reservation via RabbitMQ.
    4.  **Compensating Transaction:** If the Tour Service returns `CapacityExceeded`, the Orchestrator triggers a **Compensation Event** (`RefundWallet`), forcing the Purchase Service to rollback the financial change. This guarantees that the system always converges to a consistent state.

### 2.2 Graph-Powered Social Engine (Node.js & Neo4j)
While the core backend is high-performance **Go**, the `Following Service` leverages **Node.js** for its non-blocking I/O capabilities, paired with **Neo4j**.
*   **Recommendation Algorithms:** We moved beyond simple SQL joins to implement **Graph Theory** algorithms. The system identifies "Triadic Closures" (Friend-of-a-Friend) to generate highly relevant user recommendations.
*   **Performance:** Graph traversals allow for constant-time performance for relationship queries, regardless of the total dataset size, enabling the platform to scale to millions of edges.

### 2.3 High-Performance RPC Mesh
To minimize internal latency, synchronous service-to-service communication utilizes **gRPC** with Protocol Buffers (`proto3`).
*   **Efficiency:** Protobuf serialization results in binary payloads that are **7-10x smaller** than equivalent JSON, drastically reducing network overhead in the cluster.
*   **Contract Strictness:** `.proto` files serve as the single source of truth for API contracts, enforcing type safety across the Go and Node.js microservices.

---

## 3. Infrastructure and Container Orchestration

The entire infrastructure is codified (`IaC`) in a sophisticated `docker-compose` definition, orchestrating the lifecycle of over 15 interdependent containers.

### 3.1 Self-Healing Infrastructure
The system implements rigorous **Healthcheck Dependencies** to prevent "Race Conditions" during startup (e.g., a service starting before its database is ready).

```yaml
  stakeholder-service:
    depends_on:
      rabbitmq: 
        condition: service_healthy
      stakeholders-db:
        condition: service_healthy # Neo4j
      saga-db:
        condition: service_healthy # Postgres
```

### 3.2 Secure Networking & Volume Management
*   **Isolation:** Backend services operate on a private bridge network (`backend`), totally inaccessible from the public internet. Only the **API Gateway** exposes port `8000`.
*   **Data Durability:** We utilize named Docker volumes (e.g., `stakeholder_images_data`, `blog_db_data`) to decouple data lifecycle from container lifecycle, ensuring persistence across deployments.
*   **Security:** File uploads are sanitized and stored in dedicated volume mounts (`/app/uploads`), isolated from the execution environment to prevent Path Traversal attacks.

---

## 4. Full-Stack Observability (The LGTM Stack)

A microservices architecture is opaque without granular telemetry. Wanderlust integrates a production-grade observability pipeline ("LGTM" - Loki, Grafana, Tempo/Jaeger, Mimir/Prometheus) directly into the container mesh.

### 4.1 Distributed Tracing (Jaeger)
We utilize **OpenTelemetry** instrumentation. Every request entering the API Gateway is tagged with a `CorrelationID`. This allows us to visualize the **Waterfall Trace** of a request as it hops from Gateway → RabbitMQ → Go Service → Node Service, enabling precise identification of latency bottlenecks.

### 4.2 Metrics & Alerting (Prometheus & Grafana)
*   **Infrastructure Metrics:** `Node-Exporter` scrapes underlying host stats (CPU Context Switches, Disk I/O).
*   **Runtime Metrics:** **Prometheus** scrapes custom instrumentation from the Go runtime (Goroutines, GC Pause Duration) and Node.js event loop lag.
*   **Visualization:** Custom **Grafana** dashboards provide a "Single Pane of Glass" view into the cluster's health.

### 4.3 Centralized Logging (Loki & Promtail)
Scattered logs are a nightmare in distributed systems. We deploy **Promtail** as a sidecar agent to scrape Docker logs and ship them to **Loki**.
*   **LogQL:** This enables powerful querying (e.g., `Rate of Errors over 5m`) directly correlated with time-series metrics.

---

## 5. Domain Decomposition (Service Map)

The domain is split into Bounded Contexts, each managed by a dedicated microservice.

| Service | Tech Stack | Storage | Responsibility |
| :--- | :--- | :--- | :--- |
| **API Gateway** | **Go (Gin)** | - | **Edge Service.** Handles SSL termination, JWT verification, and Request Routing. |
| **Stakeholders** | **Go** | **Neo4j** | **Identity Provider.** Manages Profiles and acts as the SAGA Orchestrator. |
| **Social Graph** | **Node.js** | **Neo4j** | **Graph Engine.** Handles "Follow" relationships and Recommendation Algorithms. |
| **Blog Service** | **Go** | **MongoDB** | **Content Engine.** Handles Markdown blogs, Comments, and Likes via Event Sourcing. |
| **Tour Service** | **Go** | **MongoDB** | **Geospatial Engine.** Manages Tour Drafts and GPS Checkpoints. |
| **Purchase** | **Go** | **PostgreSQL** | **Financial Engine.** Manages Wallets, Shopping Carts, and Purchase Tokens. |
| **Position Sim** | **Go** | Memory | **Telemetry Gen.** Simulates tourist movement for location-based testing. |

---

## 6. Installation & Deployment Strategy

**Prerequisites:** Docker Engine 24.0+ & Docker Compose v2.20+.

The project utilizes **Multi-Stage Docker Builds** to minimize attack surface and image size (distroless/alpine).

```bash
# 1. Clone the Ecosystem
git clone https://github.com/bteodora/wanderlust-distributed-core.git

# 2. Ignite the Mesh (Builds Go & Node binaries)
docker-compose up -d --build

# 3. Verify Healthchecks
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

**Access Points:**
*   **API Gateway:** `http://localhost:8000`
*   **Grafana Intelligence:** `http://localhost:3001` (admin/admin)
*   **Jaeger Tracing UI:** `http://localhost:16686`
*   **RabbitMQ Console:** `http://localhost:15672` (guest/guest)
