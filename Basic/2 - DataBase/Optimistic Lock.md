
## What is an Optimistic Lock?

An optimistic lock is a concurrency control mechanism used in database systems and software applications to handle concurrent access to shared resoures. Unlike **pessimistic locks**, which assume conflicts are likely and immediately block access to a resource (e.g., by locking a row in a database), optimistic locks assume conflicts are rare.

Instead of blocking access upfront, they allow multiple transactions or processes to proceed simultaneously and only check for conflicts when a transaction tries to commit or update the resource.

## What does an optimistic lock lock?

Strictly speaking, an optimistic lock does not lock the resource in the traditional sense (i.e. it does not block other operations from accessing or modifying the resource). Instead, it uses a versioning mechanism to track changes to the resource. Common implementation include:

- A version column in a database table: each time a row is updated, the version number increments. When a transaction tries to update the row, it checks if the current version matches the version it read initially. If they match, the update proceeds and increments the version; if not, a conflict is detected.
- A timestamp: instead of a version number, the resource includes a timestamp of its last modification. Transaction check if the timestamp is unchanged before committing.

In essence, it locks the consistency of the resource's state by ensuring that updates are only applied if the resource hasnot been altered by others since the transaction started.

## Scenarios for Using Optimistic Locks

Optimistic locks are ideal for situations where:

- Conflicks are infrequent: Minimizing the overhead of blocking (which pessimistic locks introduce) improves performance.
- Read operations are more common than writes: Reducing lock contention keeps read-heavy systems responsive.

Example of scenarios include:

1. **E-commerce inventory management**: When multiple users check out the same product, optimistic locks prevent overselling by verifying that the inventory count hasn’t changed since the user added the item to their cart.
2. **Content management systems (CMS)**: Multiple editors editing the same article. The system checks if the article’s version (or last modified timestamp) matches the version the editor loaded before saving their changes.
3. **Banking transactions (for non-critical updates)**: Updating a user’s profile (e.g., email or address) where concurrent edits are rare. The system ensures the profile wasn’t modified by another session before applying changes.
4. **Distributed systems**: Coordinating updates across microservices, where blocking with pessimistic locks could introduce latency or deadlocks.

In summary, optimistic locks prioritize throughput and reduce blocking in low-conflict environments, relying on version checks to enforce data consistency at commit time.