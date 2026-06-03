# System Design

## Taffic Management

How requests enter and flow through the system

#### Load Balancing

* Distribute traffic across multiple instances
* Prevent hotspots and improve availability

#### Caching

* Store frequently accessed data closer to consumers
* Reduce latency and database load


#### Rate Limiting

* Restrict request volume over time
* Protect against abuse and resource exhaustion

---

## Data Management

How is data stored, queried, and replicated

#### Database

* Model data for efficient access patterns
* Consider indexes, normalisation, partitioning

#### Replication

* Maintain copies of data across nodes
* Improve availability and read scalability

#### Paritioning / Sharding

* Split data across databases
* Handle larger datasets and traffic volumes

#### Consistency

* Ensure data remains correct across systems
* Balance consistency against availability and performance

#### Messaging / Queues

* Decouple services asynchronously
* Smooth traffic spikes and improve resilience

---

## Reliability

What happens when things fail

#### Availability

* Keep the system operational despite failures
* Use redundancy and failover mechanisms

#### Fault Tolerance

* Continue operating when components fail (retries, replication, circuit breakers)
* Categorise, recover from, or surface failures appropriately

#### Resilience

* Recover quickly from outages and transient failures
* Prevent cascading failures

#### Disaster Recovery

* Restore services after major failures
* Backups, recovery procedures, DR testing

---

## Security

Who can access what

#### Authentication

* Verify user identity (passwords, OAuth, API keys)

#### Authorisation

* Control what authenticated users can do (roles, permissions, policies)

#### Encryption

* Protect data in transit and at rest

#### Secrets

* Securely store credentials and API keys
* Avoid hard-coding secrets

---

## Operations

How do we observe and operate it

#### Monitoring & Alerting

* Detect issues automatically
* Notify operators before customers notice

#### Observability

* Understand system behaviour in production
* Logs, metrics, traces, dashboards
* Record events for debugging and auditing
* Include correlation / request IDs

#### Tracing

* Track requests across services
* Identify slow or failing dependencies

#### Deployments

* Safely release new versions
* Blue-green, canary, rolling deployments

--

## Scalability & Performance

Can it handle future growth and is it fast enough

#### Capacity Planning

* Estimate future resource needs
* Prevent unexpected bottlenecks
* Scale vertically (bigger machines), or horizontally (more machines)
* Identify bottlenecks early

#### Concurrency

* Handle multiple operations simultaneously
* Prevent race conditions and deadlocks

---

## Integration & Evolution

Can it change without breaking consumers

#### API Design

* Create predictable, versioned interfaces

#### Backward Compatibility

* Allow old clients to continue working
* Minimise breaking changes

---
