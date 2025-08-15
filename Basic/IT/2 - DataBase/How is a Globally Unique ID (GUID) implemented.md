
A globally unique ID is an identifier designed to be unique across distributed systems, multiple nodes, or different business scenarios. 

## Why we need it

### Avoid duplication in distributed systems

In modern app, systems are rarely single-machine -- they are distributed across multiple servers, databases, or even geographic regions (e.g., a social media app with servers in Asia and Europe). Without a global uniqueness guarantee:

- Two users signing up simultaneously on different servers might get the same ID.
- Data from different regions merging into a central database could overwrite each other due to duplicate IDs.

A globally unique ID ensures that even across distributed nodes, every record (user, order, post) has an ID that never collides.

### Enable Data Integration and Migration

Businesses often need to combine data from multiple sources:

- Merging two databases after a company acquisition.
- Syncing data between a main app and a third-party tool (e.g., an e-commerce platform integrating with a logistics system).

If each system uses its own local ID system (e.g., both use `1`, `2`, `3`), merging them would create conflicts. A globally unique ID acts as a universal "identifier" that works across all systems, making integration seamless.

### Simplify Tracking and Referencing

Unique IDs serve as a universal "handle" to reference entities across a system:

- In an e-commerce app, an order ID must be unique to track payments, shipments, and returns without confusion.
- In a healthcare system, a patient ID must never repeat to avoid mixing medical records.

Without uniqueness, referencing an entity (e.g., "update order #5") could accidentally target the wrong record, leading to errors or data corruption.

### Support Decentralized Systems

Many modern systems are decentralized, meaning no single authority controls ID generation (e.g., peer-to-peer networks, blockchain, or microservices where each service generates its own IDs).

In such cases, a central ID generator (like a single database) would become a bottleneck or single point of failure. Globally unique IDs generated independently (e.g., via UUID or Snowflake) let each component generate IDs locally without coordination, ensuring scalability and resilience.

### Real world Example

Imageine a food delivery app:

- Users place orders via a mobile app (server A).
- Restaurants confirm orders via a web portal (server B).
- Delivery riders update statuses via a separate API (server C).

If server A, B, and C each generated IDs starting from `1`, an order from server A (ID `5`) and a restaurant from server B (ID `5`) would clash when their data is stored in the central database. 

A globally unique ID (e.g., ord_6f8e7d9c-...) ensures all systems can reference the same order without confusion.

---

Its implementation depends on business requirements (e.g. ID length, readability, generation efficiency, orderliness) and common solutions include the following:

## Database-Based Implementations

### Auto-Increment Primary Key

**Principle**: Leverage the auto-increment feature of databases (e.g., MySQL, PostgreSQL) to generate unique IDs automatically when inserting data.

**Implementation**: Set the primary key as `AUTO_INCREMENT` (MySQL) or `SERIAL` (PostgreSQL). The database internally maintains a counter that increments by 1 for each new insertion.

**Pros and Cons**:

- Pros: Simple to implement; IDs are ordered and occupy less storage.

- Cons:

	- Single point of failure: Relies on a single database, so a database outage halts ID generation.
	- Poor scalability: In sharded databases, auto-increment IDs across tables may conflict unless preconfigured with unique step sizes and starting values.

### Database Cluster + Segment Allocation

**Principle**: A dedicated "ID generator" database (clustered to avoid single points of failure) pre-generates ID segments (e.g., 1-1000, 1001-2000) and distributes them to business databases or application nodes. Nodes request new segments once their current segment is exhausted.

**Implementation**:

- Maintain an `id_generator` table storing the current segment range for each business (e.g., `biz_type`, `current_max_id`, `step`).
- Application nodes request a segment (e.g., 1000 IDs) on startup, cache it locally, and incrementally use IDs. They request a new segment when the current one is depleted.

**Pros and Cons**:

- Pros: Reduces database access frequency and supports distributed scenarios.

- Cons: Segment length requires careful tuning (too short → frequent requests; too long → potential ID waste); the database may still become a bottleneck.

## UUID/GUID

## Snowflake Algorithm

## Other Solutions















