# System-design principles

A checklist for the architect interview. Each topic lists the **trade-off questions** that settle it and the **signals**: requirements that point toward one option. Ask a topic's questions only when the requirements make it live; a topic no requirement touches is skipped, not filled in.

Every recommendation names the requirement it rests on and cites a source below. Domain language and module design are out of scope here: call `domain-modeling` for terms and bounded contexts, `codebase-design` for module interfaces and seams.

## Sources

| Code | Book |
|---|---|
| DDIA | Kleppmann, *Designing Data-Intensive Applications*, 1st ed. |
| DBI | Petrov, *Database Internals* |
| FSA | Richards & Ford, *Fundamentals of Software Architecture*, 1st ed. |
| SAHP | Ford, Richards, Sadalage & Dehghani, *Software Architecture: The Hard Parts* |
| RI | Nygard, *Release It!*, 2nd ed. |
| SRE | Beyer et al., *Site Reliability Engineering* |
| SDI1 / SDI2 | Xu, *System Design Interview* Vol 1; Xu & Lam, Vol 2 |

`DDIA 5` means DDIA chapter 5.

## Topics by interview phase

| Phase | Topics |
|---|---|
| Non-functional requirements | Quality attributes; SLOs |
| Capacity estimates | Capacity estimation |
| Data model and access patterns | Data models |
| Storage, replication, partitioning | Storage engines; Replication; Partitioning; Transactions and consistency |
| Communication | Streams and logs; Service boundaries and data ownership |
| Failure modes | Stability patterns; Overload and load shedding |
| Stack | Architecture styles |

---

## Quality attributes

- Which three or four attributes matter most (scalability, availability, latency, elasticity, deployability, data consistency, cost)? Ranking forces trade-offs; "all of them" decides nothing. [FSA 4, FSA 5]
- Is each one stated as a measurable target (p99 latency, uptime %, recovery time), not an adjective? [FSA 6, DDIA 1]
- Do attributes differ between parts of the system? Different needs in different parts are the first sign of more than one deployable unit (architecture quantum). [FSA 7, SAHP 2]

Signals: a single uniform set of attributes → one quantum is enough. One part needs elasticity or far higher availability than the rest → separate quantum. [FSA 7]

## SLOs

- What do users notice when the system is bad: latency, errors, freshness, correctness? Those are the indicators to measure. [SRE 4]
- What target, over what window, and how much error budget does that leave? 100% is never the target. [SRE 3, SRE 4]
- Which percentile matters? Averages hide tail latency; fan-out multiplies it. [DDIA 1, SRE 4]

Signals: external customers or a contract → explicit SLOs with budget. Internal tool → looser targets buy simplicity. [SRE 3]

## Capacity estimation

- Daily active users, actions per user per day → average and peak requests per second (assume peak ≈ 2–3× average unless known). [SDI1 2]
- Read/write ratio, object size, retention → storage per year and bandwidth. [SDI1 2]
- Does the result fit on one machine with headroom? If yes, scale-out questions in later phases are premature. [SDI1 1, SDI1 2, DDIA 1]

Signals: fits one node → single primary database, vertical scaling first. Writes beyond one node or dataset beyond one disk → partitioning becomes live. [SDI1 1, DDIA 6]

## Data models

- Is the data mostly a tree of self-contained records (document), many-to-many relations (relational), or highly connected traversals (graph)? [DDIA 2]
- Which queries run on the hot path, and what shape do they need? Model for access patterns, not for the entities alone. [DDIA 2, DDIA 3]
- Is the schema enforced on write or interpreted on read, and how will records evolve across versions running side by side? [DDIA 2, DDIA 4]

Signals: joins and integrity constraints → relational. Records read whole with few cross-references → document. Multi-hop relationship queries → graph. Rolling deploys or mobile clients on old versions → compatible encoding with forward and backward compatibility. [DDIA 2, DDIA 4]

## Storage engines

- Is the workload write-heavy (append, ingest) or read-heavy with point and range lookups? [DDIA 3, DBI 1]
- Transactional (OLTP) or analytical scans over many rows (OLAP)? [DDIA 3]

Signals: high write throughput, tolerance for compaction cost → log-structured (LSM). Read-heavy, predictable latency, range scans → B-tree. Aggregations over large history → column store or separate warehouse. [DDIA 3, DBI 2, DBI 7]

## Replication

- Why replicate: availability, read scaling, or latency across regions? The reason picks the topology. [DDIA 5]
- Can a reader tolerate stale data? If not, which reads need read-your-writes or monotonic reads? [DDIA 5]
- Synchronous or asynchronous followers: how much data loss on failover is acceptable (RPO), and how fast must failover be (RTO)? [DDIA 5, DBI 11]

Signals: one region, read-heavy → single leader with read replicas. Writes in several regions or offline clients → multi-leader, with an explicit conflict-resolution rule. High availability with tunable consistency → leaderless quorums. [DDIA 5, DBI 11, DBI 12]

## Partitioning

- What key spreads load evenly, and which queries then cross partitions? [DDIA 6, SDI1 5]
- Are there hot keys (a celebrity, a big tenant) that defeat even hashing? [DDIA 6]
- Do secondary-index queries need to be local (cheap writes, scatter reads) or global (cheap reads, costly writes)? [DDIA 6]

Signals: point lookups by ID → hash partitioning, consistent hashing if nodes change often. Range queries by time or key → range partitioning, with a plan for hot recent ranges. [DDIA 6, SDI1 5, SDI1 6]

## Transactions and consistency

- Which invariants must never break (balance ≥ 0, a seat sold once)? Those need serializable isolation or an equivalent guard. [DDIA 7]
- Which anomalies would weaker isolation allow here (lost updates, write skew), and are they acceptable? [DDIA 7]
- Does any operation need linearizability (uniqueness, leader election, locks)? That needs consensus, and it costs latency and availability. [DDIA 9, DBI 14, SRE 23]
- Does a transaction span more than one service or store? Prefer redesigning the boundary; otherwise choose a saga with compensations over distributed two-phase commit. [DDIA 9, SAHP 12, DBI 13]

Signals: money, inventory, bookings → strong guarantees on the core write path. Feeds, counters, analytics → eventual consistency. [DDIA 7, SDI2 11, SDI2 7]

## Streams and logs

- Request/response, or does the caller not need an immediate answer? Async decouples availability but adds lag and debugging cost. [FSA 14, DDIA 11]
- Delivery guarantee needed: at-most-once, at-least-once, or effectively-once (at-least-once plus idempotent consumers)? [DDIA 11, SDI2 4]
- Must events be replayable or ordered per key? That points to a durable log, not a transient queue. [DDIA 11, SDI2 4]
- Realtime to clients: polling, long polling, server-sent events, or WebSockets? [SDI1 12]

Signals: several consumers derive their own views of the same changes → event log, change data capture. One-off background jobs → work queue. Bidirectional low-latency client traffic → WebSockets. [DDIA 11, DDIA 12, SDI1 12]

## Service boundaries and data ownership

- What forces split the system (deployability, scalability, fault isolation, team ownership), and what forces keep it together (shared transactions, workflow, shared data)? [SAHP 4, SAHP 7]
- Who owns each piece of data? One writer per table; others read through its interface or a copy. [SAHP 6, SAHP 9]
- How do services read data they don't own: call, replicated copy, cache, or shared store? [SAHP 10]
- Workflow across services: orchestrated or choreographed? [SAHP 11]
- How strict are contracts between services? [SAHP 13]

Signals: two parts always change and deploy together → keep them in one service. Strong consistency needed across them → same service or same database. Different scale or availability → split. [SAHP 7, SAHP 9]

## Architecture styles

- Given the ranked quality attributes and the number of quanta, which style fits: modular monolith, layered, service-based, event-driven, microservices? [FSA 18]
- How large is the team, and how independently must parts deploy? Microservices pay off only with independent deployability needs and the operational maturity to run them. [FSA 17, FSA 18]

Signals: one quantum, small team → modular monolith or service-based. Many quanta with distinct attributes → microservices or event-driven. High elasticity on bursty load → event-driven or space-based. [FSA 13, FSA 14, FSA 15, FSA 18]

## Stability patterns

- What does each integration point do when the remote side is slow, not down? Every outbound call needs a timeout. [RI 4, RI 5]
- Should repeated failures trip a circuit breaker so callers fail fast? [RI 5]
- Can one bad dependency exhaust shared resources (threads, connections)? Separate pools (bulkheads). [RI 5]
- Are retries bounded, with backoff and jitter, and are retried operations idempotent? Unbounded retries turn a blip into a cascade. [RI 5, SRE 22]
- Which features can degrade gracefully instead of failing the page? [RI 5, SRE 22]

Signals: any synchronous call to a third party or another service → timeout plus circuit breaker. A shared dependency serving critical and non-critical traffic → bulkhead. [RI 5]

## Overload and load shedding

- What happens when demand exceeds capacity: queue, reject, or degrade? Reject early and cheaply. [SRE 21, RI 5]
- Which requests are critical and keep service under overload? [SRE 21]
- Does load have a limit per client (rate limiting)? Which algorithm fits the burst pattern? [SDI1 4]
- Can one failure trigger retries and overload elsewhere (cascading failure)? [SRE 22]

Signals: public API or untrusted clients → rate limiting per client. Bursty traffic with a backlog that can wait → queue with backpressure. User-facing traffic that can't wait → shed by criticality. [SRE 21, SDI1 4, RI 5]
