# 🚖 How Uber Finds Nearby Drivers: System Design Architecture

![Build Status](https://github.com/yourusername/uber-backend-architecture-breakdown)
![System Design](https://newsletter.systemdesign.one/p/how-does-uber-find-nearby-drivers)
![Tech Stack]()

> A detailed architectural breakdown of Uber's real-time marketplace, focusing on high-throughput geospatial matching and low-latency dispatching.

## 📖 Table of Contents
- Executive Summary
- High-Level Architecture
- The Challenge: Geospatial Indexing
- The Dispatch Flow (Step-by-Step)
- Data Storage & Partitioning
- References

---

## Executive Summary

The core engineering challenge of Uber is **matching supply (drivers) with demand (riders) in real-time**. This requires a system capable of handling millions of GPS updates per second while maintaining sub-second latency for read queries.

**Key constraints:**
* **Real-time:** Updates must be reflected instantly.
* **Accuracy:** Drivers must be truly "nearby" (routing distance vs. physical distance).
* **Scalability:** Must handle massive spikes (e.g., New Year's Eve).

---

## High-Level Architecture

Uber operates on a microservices architecture. Below is the interaction between the Rider, Driver, and the Backend services.

```mermaid
graph TD
    %% Actors
    Rider(("Rider App"))
    Driver(("Driver App"))

    %% Load Balancer
    LB["Load Balancer"]

    %% Gateway
    API["API Gateway"]

    %% Services
    subgraph "Core Services"
        LocationSvc["Location Service"]
        DispatchSvc["Dispatch Service (DISCO)"]
        MapSvc["Maps / ETA Service"]
        UserProfile["User Profile Service"]
    end

    %% Data Stores
    subgraph "Data Layer"
        Redis[("Redis (Pub/Sub)")]
        Cassandra[("Cassandra (History)")]
        Kafka{{"Apache Kafka"}}
    end

    %% Flows
    Rider -->|"Request Ride"| LB
    Driver -->|"Update GPS"| LB
    LB --> API
    
    API --> LocationSvc
    LocationSvc --> Kafka
    Kafka --> Redis
    Kafka --> Cassandra
    
    API --> DispatchSvc
    DispatchSvc -->|"Query Nearby"| Redis
    DispatchSvc -->|"Calculate Cost"| MapSvc
```

---

## The Challenge: Geospatial Indexing

Standard SQL databases cannot handle "Find k-nearest neighbors" queries fast enough at Uber's scale. Uber solves this using **Geospatial Indexing**.

### The Solution: H3 (Hexagonal Hierarchical Spatial Index)

Uber partitions the world into hexagons. Why Hexagons?

| Feature | Squares (Google S2) ⬜ | Hexagons (Uber H3) 🛑 |
| :--- | :--- | :--- |
| **Neighbors** | 8 (Edges + Vertices) | **6 (Edges only)** |
| **Distance** | Center-to-neighbor distance varies | **Center-to-neighbor is equidistant** |
| **Distortion** | Higher distortion near poles | **Low distortion** |
| **Traversal** | Complex pathfinding | **Smooth approximations of circles** |

```mermaid
graph TD
    subgraph "Why H3 (Hexagons)?"
        direction TB
        City["Level 5: City Scale"] -->|Divides into| District["Level 7: Neighborhood"]
        District -->|Divides into| Street["Level 9: Street Block"]
        
        style City fill:#f9f,stroke:#333,stroke-width:2px
        style District fill:#bbf,stroke:#333,stroke-width:2px
        style Street fill:#dfd,stroke:#333,stroke-width:2px
    end
    
    subgraph "Advantage"
        A[Driver Location] -->|Mapped to| B(Hexagon ID)
        B -->|Query| C{Neighbors}
        C -->|Result| D[6 Equidistant Cells]
    end
```

> **Implementation:** Every driver's GPS location is mapped to a unique H3 Hexagon ID. The system queries the driver's hexagon and the 6 immediate neighbors to find candidates.

---

## The Dispatch Flow (Step-by-Step)

How a ride request is processed:

```mermaid
sequenceDiagram
    autonumber
    actor Rider
    participant App as Mobile App
    participant LB as Load Balancer
    participant DISCO as Dispatch Svc
    participant GEO as Redis (Geo)
    participant ETA as Routing Svc
    actor Driver

    Rider->>App: Clicks "Request Ride"
    App->>LB: POST /api/trips (Lat, Long)
    LB->>DISCO: Forward Request
    
    rect rgb(240, 248, 255)
    note right of DISCO: 1. Find Candidates
    DISCO->>DISCO: Convert Lat/Long -> H3 ID
    DISCO->>GEO: GEORADIUS (H3_ID, 2km)
    GEO-->>DISCO: Return [Driver A, Driver B, Driver C]
    end

    rect rgb(255, 240, 240)
    note right of DISCO: 2. Filter & Rank
    loop For each Driver
        DISCO->>ETA: GetETA(Origin, DriverLoc)
        ETA-->>DISCO: 4 mins
    end
    DISCO->>DISCO: Sort by ETA ASC
    end
    
    DISCO->>Driver: Send Job Offer (WebSocket)
    Driver-->>DISCO: Accept Job
    DISCO-->>App: Driver Found!
```

---

## Data Storage & Partitioning

To handle the load, the data layer is split into "Hot" (Real-time) and "Cold" (Historical) storage.

```mermaid
flowchart LR
    %% Nodes
    Driver((Driver))
    API[Ingestion API]
    Kafka{Kafka Stream}
    Redis[(Redis Pub/Sub)]
    Cass[(Cassandra)]
    
    %% Styles
    classDef storage fill:#eee,stroke:#333,stroke-width:2px;
    class Redis,Cass storage;

    %% Connections
    Driver -->|"GPS Ping Every 4s"| API
    API -->|Publish| Kafka
    
    subgraph "Hot Path (Real-time)"
    Kafka -->|Update| Redis
    Redis -->|Read| Matcher[Matching Service]
    end
    
    subgraph "Cold Path (History)"
    Kafka -->|Archive| Cass
    Cass -->|Analyze| Analytics[Data Science]
    end
```

### 1. Hot Storage (Redis)
* **Purpose:** Tracks current driver locations.
* **Data Structure:** `Key: DriverID | Value: {Lat, Long, Status, H3_ID}`
* **TTL (Time To Live):** Keys expire after short intervals to ensure data freshness.

### 2. Cold Storage (Cassandra)
* **Purpose:** Trip history, analytics, and audit logs.
* **Why Cassandra?** Optimized for massive write throughput (location pings every 4 seconds from millions of drivers).

### 3. Message Queue (Kafka)
* **Purpose:** Acts as a buffer between the high-volume driver API and the database. Ensures no location data is lost during traffic spikes.

---

## References

* System Design One Newsletter
* Uber Engineering: H3 Geo-Index
* Uber Engineering: Ringpop
