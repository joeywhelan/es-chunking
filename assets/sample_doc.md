# Building Resilient Distributed Systems

Modern software systems rarely live on a single machine. As demand grows and availability expectations tighten, engineers distribute work across multiple nodes, data centers, and even continents. But distribution introduces an entirely new class of problems. This guide walks through the core principles, strategies, and failure modes that shape how resilient distributed systems are built in practice.

## Foundations of Distribution

### Why Distribute?

There are three primary reasons to distribute a system: capacity, availability, and latency. A single server has finite CPU, memory, and disk. When a workload exceeds those limits, the only options are to scale vertically (bigger hardware) or horizontally (more machines). Vertical scaling hits a ceiling quickly and tends to become non-linearly expensive. Horizontal scaling, by contrast, offers a theoretically unbounded capacity curve — at the cost of coordination complexity.

Availability is the second driver. A single machine is a single point of failure. Hardware fails, networks partition, and data centers lose power. Distributing a system across independent failure domains means that one node's outage does not have to become a system-wide outage. The goal is not to prevent failure — failure is inevitable — but to ensure that the system continues to function in a degraded mode when components fail.

Latency is the third consideration. Physics imposes a floor on how quickly a packet can travel between two points. If your users span multiple continents, serving them all from a single region adds tens or hundreds of milliseconds of round-trip time. Distributing data and computation closer to users reduces that latency.

### The Fallacies of Distributed Computing

In 1994, Peter Deutsch and others at Sun Microsystems articulated a set of assumptions that developers new to distributed systems tend to make — and that turn out to be false. These fallacies remain surprisingly relevant three decades later:

1. The network is reliable.
2. Latency is zero.
3. Bandwidth is infinite.
4. The network is secure.
5. Topology does not change.
6. There is one administrator.
7. Transport cost is zero.
8. The network is homogeneous.

Every design decision in a distributed system must account for the fact that messages can be lost, delayed, reordered, or duplicated. Network partitions are not edge cases — they are operational realities. Systems that treat the network as a reliable transport layer will behave unpredictably under stress, which is precisely when predictable behavior matters most.

## Consensus and Coordination

### The Role of Consensus

Many distributed problems reduce to getting a group of nodes to agree on something: which node is the leader, what the current configuration is, whether a transaction should commit. Consensus algorithms provide a formal framework for reaching agreement in the presence of failures.

The FLP impossibility result, published by Fischer, Lynch, and Paterson in 1985, proved that no deterministic consensus algorithm can guarantee progress in an asynchronous system where even a single node may crash. In practice, this means that every real-world consensus protocol relies on some combination of timeouts, randomization, or partial synchrony assumptions to make forward progress.

### Leader Election

Leader election is one of the most common applications of consensus. In a single-leader architecture, one node is designated as the primary and is responsible for accepting writes, coordinating replication, or making authoritative decisions. The remaining nodes act as followers or replicas.

The challenge is what happens when the leader fails. The system must detect the failure, elect a new leader, and ensure that the new leader has all the data it needs to continue where the old one left off. This process must avoid split-brain scenarios, where two nodes simultaneously believe they are the leader. Split-brain can cause data corruption, conflicting writes, and inconsistencies that are extremely difficult to resolve after the fact.

Protocols like Raft and Paxos solve this by requiring a quorum — a majority of nodes — to agree on leadership transitions. Raft in particular has gained widespread adoption because of its relative simplicity compared to Paxos. It decomposes consensus into three sub-problems: leader election, log replication, and safety, making it easier to implement and reason about.

### Distributed Locks and Fencing Tokens

Distributed locks allow a process to claim exclusive access to a shared resource across multiple nodes. A common implementation uses a coordination service like etcd or ZooKeeper: a process acquires a lock by writing a key with a TTL, and other processes check for that key before proceeding.

However, distributed locks are deceptively dangerous. A process might acquire a lock, experience a long garbage collection pause or network delay, and then resume execution believing it still holds the lock — after the TTL has expired and another process has taken over. This can lead to two processes simultaneously operating on the same resource.

Fencing tokens mitigate this problem. Each time a lock is granted, the coordination service issues a monotonically increasing token. The protected resource checks the token on every operation and rejects any request with a token lower than the highest it has already seen. This ensures that even if a stale lock holder resumes, its operations are rejected.

## Data Replication Strategies

### Single-Leader Replication

In single-leader replication, one node accepts all writes and propagates changes to followers. Reads can be served by any replica, though reading from a follower may return stale data if replication lag exists. This model is simple to reason about and avoids write conflicts entirely, since all writes flow through a single point.

The tradeoff is that the leader is a bottleneck for write throughput and a single point of failure for write availability. Failover — promoting a follower to leader — introduces a window of uncertainty during which writes may be lost or duplicated, depending on whether replication is synchronous or asynchronous.

### Multi-Leader Replication

Multi-leader replication allows multiple nodes to accept writes independently. This is useful for multi-datacenter deployments where each datacenter has its own leader, reducing write latency for local clients. It is also the model used by many collaborative editing tools, where each client acts as a local leader.

The fundamental problem with multi-leader replication is write conflicts. Two leaders may independently accept conflicting writes to the same record. Conflict resolution strategies include last-writer-wins (simple but lossy), custom merge functions (flexible but complex), and CRDTs (conflict-free replicated data types, which guarantee convergence without coordination but only support certain data structures).

### Leaderless Replication

Leaderless replication, popularized by Amazon's Dynamo paper and used in systems like Apache Cassandra, eliminates the concept of a designated leader entirely. Clients send writes to multiple replicas simultaneously and reads query multiple replicas, using quorum arithmetic to determine consistency. A write is considered successful if acknowledged by W out of N replicas; a read is considered consistent if it queries R replicas and R + W > N.

This model offers high availability and tolerates individual node failures gracefully. The tradeoff is weaker consistency guarantees and the need for anti-entropy mechanisms — like read repair and Merkle trees — to reconcile divergent replicas in the background.

## Handling Failure

### Failure Detection

Before a system can respond to a failure, it must detect one. In a distributed environment, the difference between "that node is down" and "the network between us is slow" is often indistinguishable. A node that does not respond to a heartbeat within a timeout might be crashed, overloaded, or simply on the other side of a network partition.

Most failure detectors use a combination of heartbeats and adaptive timeouts. Phi accrual failure detectors, used in systems like Akka and Cassandra, maintain a statistical model of historical heartbeat intervals and compute a suspicion level rather than a binary alive-or-dead determination. This approach adapts to varying network conditions and reduces false positives during transient slowdowns.

### Circuit Breakers and Bulkheads

Circuit breakers prevent a failing downstream dependency from cascading failure through the system. When calls to a service exceed a failure threshold, the circuit breaker trips and subsequent calls fail immediately without attempting the request. After a cooldown period, the breaker enters a half-open state and allows a limited number of test requests through. If those succeed, the circuit closes; if they fail, it trips again.

Bulkheads isolate components so that a failure in one does not consume resources needed by others. The term comes from shipbuilding: a hull divided into watertight compartments can survive a breach in one section. In software, bulkheads typically manifest as separate thread pools, connection pools, or process boundaries for different dependencies. If one dependency becomes slow and exhausts its allocated resources, the rest of the system continues to function normally.

### Retries and Idempotency

Retries are the most basic failure-handling mechanism, but naive retry logic causes more problems than it solves. If a service is overloaded and every client retries immediately, the retry storm amplifies the original load and delays recovery. Exponential backoff with jitter spreads retry attempts over time and reduces the likelihood of synchronized retry waves.

Idempotency is the complementary requirement. When a client retries a request, it may be because the original request failed — or because the response was lost while the request actually succeeded. If the operation is not idempotent, the retry may apply the effect twice: charging a customer twice, sending a message twice, or creating duplicate records. Designing operations to be idempotent from the start — typically by using client-generated unique request IDs that the server deduplicates — avoids an entire category of subtle, hard-to-diagnose bugs.

## Scaling Patterns

### Vertical vs. Horizontal Scaling

Vertical scaling means adding more resources to a single node: more CPU cores, more RAM, faster disks. It is simple, requires no architectural changes, and works well up to the limits of available hardware. Modern cloud instances offer machines with hundreds of cores and terabytes of memory, which is sufficient for many workloads.

Horizontal scaling means adding more nodes. It requires the application to be designed for distribution — stateless request handling, externalized session storage, and either replicated or partitioned data. The coordination overhead is real, but horizontal scaling offers two properties that vertical scaling cannot: fault tolerance across independent machines and a capacity ceiling that is limited by economics rather than physics.

### Partitioning and Sharding

When a dataset outgrows a single node, it must be split across multiple nodes. Partitioning (also called sharding) divides data so that each partition lives on a different node. The partition key determines which node owns a given record.

Range partitioning assigns contiguous key ranges to different nodes, making range scans efficient but risking hot spots if access patterns cluster around certain key ranges. Hash partitioning distributes keys uniformly across nodes, eliminating hot spots but making range queries expensive since adjacent keys land on different nodes. Consistent hashing, used in many distributed databases and caches, minimizes the number of keys that must be reassigned when nodes join or leave the cluster.

Rebalancing — moving partitions between nodes as the cluster grows or shrinks — must be handled carefully to avoid downtime or data loss. Most production systems rebalance incrementally, moving one partition at a time while maintaining read availability from the remaining replicas.

### Caching Strategies

Caching reduces load on backend systems by serving frequently accessed data from a faster store. In a distributed system, caching introduces its own set of challenges: cache invalidation, consistency between cache and source of truth, and thundering herd problems when a popular cache entry expires and many clients simultaneously hit the backend.

Write-through caching writes to the cache and backend simultaneously, ensuring consistency at the cost of write latency. Write-behind caching writes to the cache first and asynchronously flushes to the backend, improving write latency but risking data loss if the cache node fails before the flush. Read-through caching populates the cache on a miss, which is simple but means the first request after an eviction always hits the backend.

## Observability in Distributed Systems

### Distributed Tracing

In a monolithic application, a stack trace shows the full execution path of a request. In a distributed system, a single user request may traverse dozens of services, and a failure in one may manifest as a timeout in another. Distributed tracing assigns a unique trace ID to each incoming request and propagates it through every service call, creating a tree of spans that represents the complete journey of that request through the system.

Tracing answers questions that logs and metrics alone cannot: which service is the bottleneck for this specific slow request? Which downstream call failed and caused the upstream timeout? Is the latency concentrated in one service or spread across many? Standards like OpenTelemetry provide vendor-neutral instrumentation libraries and wire protocols, making it possible to trace across heterogeneous technology stacks without coupling to a specific tracing backend.

### Structured Logging

Traditional unstructured log lines — free-text strings written to stdout — become nearly useless at scale. Searching for a specific request across hundreds of service instances producing millions of log lines per hour requires structure. Structured logging emits log events as key-value pairs or JSON objects, with consistent fields like timestamp, service name, trace ID, log level, and request metadata.

Structured logs can be ingested into a centralized log store — such as Elasticsearch — and queried with the same precision as a database. Correlating logs with trace IDs connects the detailed per-service narrative to the high-level request flow captured by distributed tracing. The combination of structured logs and traces provides both the "what happened" and the "where it happened" views that operators need during incident response.

### Health Checks and SLOs

Health checks expose the status of a service to load balancers, orchestrators, and monitoring systems. A liveness check indicates whether the process is running and responsive. A readiness check indicates whether the service is ready to accept traffic — it may be alive but still warming caches, loading configuration, or waiting for a dependency. Kubernetes, for example, uses liveness probes to restart hung processes and readiness probes to remove unready pods from the service's endpoint list.

Service Level Objectives (SLOs) define the reliability targets that a service commits to: 99.9% of requests complete in under 200 milliseconds, or 99.95% of requests succeed without error. SLOs shift the operational conversation from reactive alerting to budget-based decision-making. If the error budget is healthy, the team can ship features and take risks. If the error budget is depleted, the team shifts focus to reliability work. This framework, popularized by Google's Site Reliability Engineering practice, provides a quantitative basis for balancing velocity and stability — a tension that every engineering organization faces but few resolve explicitly.
