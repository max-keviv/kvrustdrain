Architecture Design
-------------------

KVRustDrain's architecture is engineered to deliver a robust, scalable solution for real-time data management, inspired by Juspay's KV Framework used in NammaYatri. It integrates Redis for in-memory key-value storage with PostgreSQL for persistent relational storage, orchestrated via Rust's asynchronous runtime (tokio). Below is a detailed breakdown of its design.

### Core Components

1.  **Redis Client (redis-rs)**
    -   **Role**: Primary store for real-time data operations.

    -   **Details**: Uses the redis crate to connect to Redis, an in-memory key-value store optimized for speed (sub-millisecond latency).

    -   **Data Structure**: Stores objects as hashes (e.g., booking:1 with fields id, driver_id, status) and queues updates in a Redis list (booking_queue) or Streams (planned enhancement).

    -   **Purpose**: Handles high-frequency reads and writes, ensuring immediate data availability.

3.  **PostgreSQL Client (diesel)**
    -   **Role**: Persistent storage for long-term data retention and complex queries.

    -   **Details**: Leverages diesel, a type-safe ORM and query builder, to interact with PostgreSQL. Defines schemas (e.g., bookings table) and ensures data integrity.

    -   **Purpose**: Receives batched updates from Redis, providing durability and support for relational queries unavailable in Redis.

5.  **Repository Pattern**
    -   **Role**: Unified abstraction for data operations.

    -   **Details**: Implements a generic Repository<T> trait with methods like create and get_by_id. For example, BookingRepository writes to Redis and reads from it, abstracting the dual-storage complexity.

    -   **Purpose**: Simplifies developer interaction, hiding the Redis-to-PostgreSQL sync logic.

7.  **Background Drainer (tokio)**
    -   **Role**: Asynchronous sync mechanism between Redis and PostgreSQL.

    -   **Details**: A tokio::spawn task polls the Redis queue (e.g., every 100ms, configurable via Config::drain_interval_ms) and inserts data into PostgreSQL using Diesel.

    -   **Purpose**: Ensures eventual consistency by offloading persistence to a background process, reducing latency for real-time operations.

### Detailed Data Flow

The architecture follows a staged pipeline to balance speed and reliability:

```
[Application]
    ↓
[Repository API] → [Redis (Immediate Write)]
                   ↓
             [Redis Queue] → [Background Drainer]
                                  ↓
                            [PostgreSQL (Persistent Store)]
```

1.  **Write Path**:

    -   Application calls repo.create(booking).

    -   Data is serialized and written to Redis as a hash (e.g., booking:1 with fields).

    -   The same data is pushed to a Redis queue (e.g., booking_queue via RPUSH).

    -   **Latency**: Sub-millisecond due to Redis's in-memory nature.

3.  **Read Path**:

    -   Application calls repo.get_by_id(1).

    -   Fetches directly from Redis (e.g., HGETALL booking:1), returning the data if present.

    -   **Latency**: Near-instant, bypassing PostgreSQL for speed.

5.  **Drain Path**:

    -   Drainer task polls the queue (e.g., LPOP booking_queue).

    -   Deserializes each entry and inserts it into PostgreSQL (e.g., INSERT INTO bookings via Diesel).

    -   Handles conflicts (e.g., ON CONFLICT DO NOTHING) to avoid duplicates.

    -   **Latency**: Asynchronous, tunable via polling interval, typically 100-500ms delay.

### Visual Representation

```
+-------------------+       +-------------------+       +--------------------+
| Application       |       | Redis             |       | PostgreSQL         |
| (e.g., Ride       | →CRUD→| - Keys: booking:1 | →Queue→| - Table: bookings  |
|  Hailing Service)|       | - Queue:          |       | - Persistent Data  |
|                   |       |   booking_queue   |       |                    |
+-------------------+       +-------------------+       +--------------------+
                              ↑         ↓
                              |    [Drainer Task]
                              |    - Polls Queue
                              |    - Syncs to DB
                              +------------------+
```

### Design Decisions

-   **Redis as Primary Store**: Chosen for its in-memory performance (up to 350,000 ops/s, ~0.44ms latency per NammaYatri's metrics). Writes hit Redis first to prioritize speed over immediate persistence.

-   **PostgreSQL for Persistence**: Ensures data durability and supports relational queries (e.g., joining bookings with drivers), which Redis can't natively handle.

-   **Asynchronous Draining**: Decouples real-time operations from database writes, reducing load on PostgreSQL and improving cost efficiency (Redis is cheaper to scale for high throughput).

-   **Type Safety**: Uses Diesel's compile-time query validation and Rust's ownership model to prevent runtime errors, mirroring Haskell's safety in euler-hs.

-   **Queue Mechanism**: Starts with Redis lists for simplicity; future versions will adopt Redis Streams for reliability and scalability (e.g., consumer groups, fault tolerance).

### Technical Considerations

-   **Scalability**: Redis supports horizontal scaling via clustering; the drainer can be parallelized across multiple instances for higher throughput.

-   **Fault Tolerance**: Queued data in Redis persists until drained, allowing retries if PostgreSQL fails temporarily.

-   **Performance Trade-offs**: Real-time reads are instant via Redis, but writes to PostgreSQL lag slightly (configurable), favoring speed over immediate consistency.

-   **Extensibility**: The repository pattern allows adding new models (e.g., Driver, Ride) without altering the core logic.

### Example Metrics (Inspired by NammaYatri)

-   **Redis Latency**: ~0.44ms (p50), ~2.45ms (p99).

-   **Write Throughput**: Up to 350,000 ops/s on a single Redis instance.

-   **Drain Delay**: 100ms default, adjustable based on workload.