# MasterCard Java, Concurrency, and Database Interview Guide

Interview-ready answers with links to Java, Oracle, PostgreSQL, MySQL, GitHub, Jakarta Persistence, and Atlassian documentation.

> Questions 7 and 8 were duplicates, so they are combined below.

## Concurrency and collections

### 1. What is a race condition, and when does it occur?

A race condition occurs when the result depends on the timing or order of concurrent operations. It commonly happens when threads access shared mutable state, at least one thread writes, and the operations are not coordinated.

For example, two threads can both read a counter as `10`, add one, and both write back `11`. One increment is lost. The Java Language Specification describes a related issue called a *data race*: conflicting accesses that lack a happens-before relationship.

**Source:** [Java Language Specification — Threads and Locks](https://docs.oracle.com/javase/specs/jls/se21/html/jls-17.html#jls-17.4)

### 2. What is the solution to prevent race conditions?

First identify the shared state and the invariant that must remain true. Then protect the whole operation that maintains that invariant—not just individual reads or writes.

Depending on the problem, I might use:

- `synchronized` or a `Lock` for a critical section.
- An atomic type such as `AtomicInteger` for a single-value update.
- A concurrent collection such as `ConcurrentHashMap` for shared map operations.
- Immutability, thread confinement, or message passing to avoid shared mutation.

The right choice depends on whether the operation spans multiple values or steps. A thread-safe collection does not automatically make a multi-step business operation atomic.

**Source:** [Oracle Java Tutorial — Intrinsic Locks and Synchronization](https://docs.oracle.com/javase/tutorial/essential/concurrency/locksync.html)

### 3. What is the difference between `putIfAbsent` and `computeIfAbsent` in `ConcurrentHashMap`?

`putIfAbsent(key, value)` inserts a value you have already computed, but only if the key has no mapping. It returns the existing value if one was present, or `null` if the insertion happened.

`computeIfAbsent(key, mappingFunction)` checks the key and computes the value only if needed. The map applies the computation atomically for that key; if the function returns `null`, it does not add a mapping. The function should be short and should not try to update other mappings in the same map.

I’d use `putIfAbsent` when I already have the value; I’d use `computeIfAbsent` for lazy initialization.

**Source:** [Java API — `ConcurrentHashMap`](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/ConcurrentHashMap.html)

### 4. If two methods are `synchronized` but there is still a race condition, why?

Synchronization protects a particular monitor, not a method name. Two instance `synchronized` methods lock the same object only when called on the same instance. A `static synchronized` method locks the `Class` object, which is a different lock.

A race can remain if the methods run on different instances, use different lock objects, access shared state through an unsynchronized method, or coordinate only part of a multi-step operation. The key question is: **Do all accesses to this shared state use the same lock?**

**Source:** [Oracle Java Tutorial — Synchronized Methods](https://docs.oracle.com/javase/tutorial/essential/concurrency/syncmeth.html)

### 5. How would you investigate a race condition despite synchronized methods?

I’d identify the shared field and trace every read and write, including calls through other objects. Then I’d confirm which monitor each code path holds and whether the full invariant-changing operation is inside the critical section.

I’d check whether multiple application instances are involved: a Java monitor coordinates threads inside one JVM, not separate processes or servers. To reproduce the issue, I’d add targeted concurrency tests or logging with request and thread IDs. For a production hang or contention issue, I’d collect thread dumps or a Java Flight Recorder recording and look for blocked threads and monitor waits.

**Sources:** [Oracle Java Tutorial — Intrinsic Locks](https://docs.oracle.com/javase/tutorial/essential/concurrency/locksync.html) · [Oracle Java — Troubleshoot Performance with JFR](https://docs.oracle.com/en/java/javase/26/troubleshoot/troubleshoot-performance-issues-using-jfr.html)

### 6. In which scenario would you use a big `HashMap`?

I’d use a large `HashMap` when I need fast in-memory key-based lookup, the data fits comfortably in the heap, and the map is either thread-confined or safely published and not concurrently modified.

Examples include a request-local lookup table, a loaded reference-data set, or a single-threaded batch job’s index. I’d estimate the memory cost, choose a reasonable initial capacity to reduce resizing, and measure garbage-collection impact. If multiple threads need to mutate it, I’d use an appropriate concurrent design instead of sharing a plain `HashMap`.

### 7–8. Is deadlock possible with `HashMap`?

A plain `HashMap` does not acquire locks itself, so it does not independently cause a lock deadlock. Deadlock is possible in application code that surrounds map access with locks—for example, two threads acquire two external locks in opposite orders.

Separately, concurrent structural modification of a shared `HashMap` without synchronization is unsupported and can produce incorrect or unpredictable behavior. Java documentation says that concurrent access involving structural modification must be externally synchronized.

**Source:** [Java API — `HashMap`](https://docs.oracle.com/en/java/javase/24/docs/api/java.base/java/util/HashMap.html)

### 9. Does `ConcurrentHashMap` allow null values?

No. `ConcurrentHashMap` does not allow null keys or values; attempts to use them throw `NullPointerException`.

**Source:** [Java API — `ConcurrentHashMap`](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/ConcurrentHashMap.html)

### 10. Why doesn’t `ConcurrentHashMap` allow null keys or null values?

Because `get(key)` returning `null` can then mean unambiguously that the key has no mapping. If null values were allowed, a caller could not distinguish “no mapping” from “mapping exists with a null value” without a separate check—and that check could race with another thread’s update.

That clear absence signal also supports atomic operations such as `putIfAbsent` and `computeIfAbsent`.

**Source:** [Java API — `ConcurrentMap`](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/ConcurrentMap.html)

### 13. What issues can arise when a `HashMap` is shared across threads in production, even if it works in testing?

A plain `HashMap` is not thread-safe for concurrent updates. Depending on the access pattern, the application can see lost updates, inconsistent results, visibility problems, or `ConcurrentModificationException` while iterating. That exception is only a best-effort bug signal; it is not a concurrency-control mechanism.

The issue may appear only under production load because thread timing and contention differ from a small test. I’d replace the design with thread confinement, explicit synchronization, or a concurrent collection based on the required operation.

**Source:** [Java API — `HashMap`](https://docs.oracle.com/en/java/javase/24/docs/api/java.base/java/util/HashMap.html)

### 14. Why might requests time out with a shared `HashMap` even though the application does not crash?

The map alone does not prove the cause. I’d look for what the application does around it:

- A synchronized wrapper or external lock may serialize requests, creating a queue.
- Iteration or copying a large map may make a request slow.
- A concurrency bug may cause retries, exceptions, or excessive work.
- The map may be unrelated, while requests wait on a database, network call, or exhausted thread or connection pool.

I’d compare latency and error rates by endpoint, inspect thread states and pool metrics, and capture a thread dump or JFR recording during the slowdown. JFR can help distinguish CPU work, I/O waits, and monitor contention.

**Source:** [Oracle Java — Troubleshoot Performance with JFR](https://docs.oracle.com/en/java/javase/26/troubleshoot/troubleshoot-performance-issues-using-jfr.html)

### 16. In which scenarios do concurrency issues occur?

They can occur whenever concurrent work interacts with state or resources that are not designed or coordinated for it. Common cases include:

- Multiple threads updating the same in-memory object or collection.
- Read-modify-write operations, such as checking a balance and then decrementing it.
- Multiple requests updating the same database row.
- Shared caches, counters, files, or singleton state.
- Retries or duplicate requests racing with the original operation.
- Multiple application instances updating the same external resource.

I’d ask which invariant can be violated, which actors can update it, and where coordination must happen.

## Latency and database troubleshooting

### 11. How would you investigate an application that normally responds in 100 ms but now takes 4–5 seconds?

I’d first establish the scope and timing: which endpoints, when the change started, which instances or regions, and whether the extra time is in the application or a dependency. I’d compare current metrics with the 100 ms baseline, then break request time into application processing, database, network calls, queueing, and serialization.

Next I’d inspect CPU and memory, garbage-collection pauses, thread-pool and connection-pool saturation, dependency latency, errors, and recent changes. During the slowdown, I’d capture thread dumps or JFR data to see whether request threads are running, blocked on locks, or waiting on I/O. Then I’d test the leading hypothesis and verify the fix against the same latency metrics.

**Source:** [Oracle Java — Troubleshoot Performance with JFR](https://docs.oracle.com/en/java/javase/26/troubleshoot/troubleshoot-performance-issues-using-jfr.html)

### 12. How would you investigate a slow API, including the database and indexes?

I’d trace a slow request end to end and identify which database query accounts for the time. I’d check query frequency, parameters, rows returned, lock waits, connection-pool waits, and whether the query changed or its data volume grew.

Then I’d inspect the execution plan and compare estimated rows with actual rows. I’d check whether the query filters and joins align with useful indexes, whether it scans or sorts too much data, and whether database statistics are current. I’d avoid adding indexes blindly: indexes take space and add write and maintenance cost. After a targeted change, I’d compare the plan and latency under representative data and load.

**Sources:** [PostgreSQL — Using `EXPLAIN`](https://www.postgresql.org/docs/current/using-explain.html) · [MySQL — `EXPLAIN ANALYZE`](https://dev.mysql.com/doc/refman/8.4/en/explain.html) · [MySQL — Optimization and Indexes](https://dev.mysql.com/doc/refman/8.4/en/optimization-indexes.html)

### 15. Why might all requests time out with exceptions even though the server is running?

A running process does not mean it can serve requests. For example, all request threads could be waiting on a slow dependency, the database connection pool could be exhausted, or a shared lock could be blocking progress. A full queue or repeated dependency failures can also cause broad timeouts.

I’d check health checks and logs, request-thread states, queue depth, database and HTTP connection pools, dependency latency, and recent deployments. I’d find where requests are waiting before changing timeout values; increasing a timeout can mask the bottleneck and make resource saturation worse.

## GitHub and database topics

### 17. What branching strategies are available, and how do you configure branch protection?

Common strategies include:

- **Feature branch / GitHub Flow:** short-lived branches and pull requests into the default branch.
- **Trunk-based development:** frequent integration into a shared trunk, often using short-lived branches and feature flags.
- **Gitflow:** longer-lived `develop` and release branches, plus feature and hotfix branches.

I’d choose based on release cadence and deployment practices. On GitHub, I’d protect important branches with a branch protection rule or ruleset. Typical requirements include pull requests, a minimum number of approvals, required status checks, code-owner review, resolved conversations, and restrictions on direct pushes, force pushes, or deletion.

**Sources:** [GitHub — Branches](https://docs.github.com/en/pull-requests/reference/branches) · [GitHub — Protected Branches](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches) · [Atlassian — Gitflow](https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow) · [Atlassian — Trunk-based Development](https://www.atlassian.com/continuous-delivery/continuous-integration/trunk-based-development)

### 18. What databases have you worked with? How does PostgreSQL handle MVCC differently from MySQL?

Use only the databases you have actually worked with. A concise answer based on your stated PostgreSQL and MySQL background could be:

> “I’ve worked with PostgreSQL and MySQL. Both support MVCC, so a transaction can read a consistent version of data while concurrent transactions make changes. The implementation details differ: PostgreSQL keeps row versions in the table and reclaims obsolete versions through vacuuming. InnoDB uses undo records to reconstruct earlier row versions for consistent reads, then purges old versions when they are no longer needed.”

I’d add that both systems also use locks where needed, and their isolation levels and query plans affect observed behavior.

**Sources:** [PostgreSQL — Concurrency Control](https://www.postgresql.org/docs/current/mvcc.html) · [MySQL — InnoDB Multi-Versioning](https://dev.mysql.com/doc/refman/8.4/en/innodb-multi-versioning.html)

### 19. What are cascade types and orphan removal?

In JPA/Hibernate, cascade controls whether an operation on a parent entity—such as persist, merge, or remove—also propagates to associated child entities.

`orphanRemoval = true` means that when a child is removed from the parent’s relationship, the provider deletes that child entity when the persistence context is flushed. It fits a true parent-owned relationship where the child should not exist independently. I would not enable it casually for shared or independently managed entities.

**Source:** [Hibernate ORM User Guide — Orphan Removal](https://docs.hibernate.org/stable/orm/userguide/html_single/)

### 20. Have you used a locking mechanism, such as database-level locking?

Answer this from your actual experience. A useful structure is: **what conflict you needed to prevent, which mechanism you used, and the trade-off.**

For example, if accurate:

> “I’ve used optimistic locking when concurrent updates are uncommon: a version column detects a conflicting update, and the application can retry or return a conflict. For a workflow that must reserve or update a specific row before proceeding, I’d consider a short pessimistic row lock, such as `SELECT ... FOR UPDATE`. I keep the transaction short and handle lock timeouts or deadlocks.”

**Sources:** [Jakarta EE Tutorial — Persistence Locking](https://jakarta.ee/learn/docs/jakartaee-tutorial/current/persist/persistence-locking/persistence-locking.html) · [PostgreSQL — Explicit Locking](https://www.postgresql.org/docs/current/explicit-locking.html)

## Source list

- [Java Language Specification — Threads and Locks](https://docs.oracle.com/javase/specs/jls/se21/html/jls-17.html#jls-17.4)
- [Java API — `ConcurrentHashMap`](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/ConcurrentHashMap.html)
- [Java API — `HashMap`](https://docs.oracle.com/en/java/javase/24/docs/api/java.base/java/util/HashMap.html)
- [Oracle Java — Troubleshoot Performance with JFR](https://docs.oracle.com/en/java/javase/26/troubleshoot/troubleshoot-performance-issues-using-jfr.html)
- [PostgreSQL — Concurrency Control](https://www.postgresql.org/docs/current/mvcc.html)
- [PostgreSQL — Using `EXPLAIN`](https://www.postgresql.org/docs/current/using-explain.html)
- [MySQL — InnoDB Multi-Versioning](https://dev.mysql.com/doc/refman/8.4/en/innodb-multi-versioning.html)
- [MySQL — `EXPLAIN ANALYZE`](https://dev.mysql.com/doc/refman/8.4/en/explain.html)
- [GitHub — Protected Branches](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches)
- [Hibernate ORM User Guide](https://docs.hibernate.org/stable/orm/userguide/html_single/)
- [Jakarta EE Tutorial — Persistence Locking](https://jakarta.ee/learn/docs/jakartaee-tutorial/current/persist/persistence-locking/persistence-locking.html)
