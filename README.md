# Electronic Voucher System High Availability

An educational guide to **electronic voucher distribution system uptime**: how EVD and eTopup platforms are designed for **failover**, **peak-load** periods, and safe recovery without losing or duplicating vouchers. Written for telecom operators, distributors, and payment teams evaluating platforms in the [EVD System](https://evdsystem.com/) and [MoboGage](https://evdsystem.com/about-mobogage/) family.

---

## What Does High Availability Mean for an EVD System?

An **electronic voucher distribution (EVD) system** sells airtime PINs, pinless top-ups (eTopup), data bundles, and gift vouchers through resellers, POS terminals, apps, and APIs. Every minute of downtime is a minute in which retailers cannot sell, customers cannot recharge, and resellers lose trust.

**High availability (HA)** is the design discipline of keeping that service running through component failures, maintenance, and traffic spikes. For voucher platforms, HA has a second, equally important goal: **correctness under failure**. A system that stays up but issues the same PIN twice, or debits a reseller without delivering a voucher, has failed in a more expensive way than one that briefly went down.

### Typical pressure points

- **Peak periods** — salary days, month start, public holidays, and promotions drive bursts of top-ups  
- **Operator dependencies** — recharge and voucher APIs at the telecom side can slow down or fail  
- **Reseller retries** — POS devices and partner APIs retry on timeout, multiplying load  
- **Maintenance windows** — upgrades and database changes must not stop sales  

---

## Architecture Overview: Redundant Transaction Path

```
POS / app / reseller API / USSD
        │
        ▼
Load balancers (redundant)
        │
        ▼
Stateless API nodes (N+1, auto-scaled)
        │
        ├──► Idempotency store (request ID → result)
        │
        ▼
Transaction service ──► Ledger + voucher inventory DB
        │                  (primary + synchronous standby, plus backups)
        │
        ▼
Operator connectors (per operator, with timeouts and circuit breakers)
        │
        ▼
Async queue ──► Notifications, reporting, reconciliation
```

### Core layers

| Layer | Responsibility |
|-------|----------------|
| **Edge and load balancing** | Spread traffic, health-check nodes, remove failed ones |
| **Stateless API tier** | Scale horizontally; no session state lost if a node dies |
| **Idempotency store** | Return the same result for a repeated request ID instead of selling twice |
| **Ledger and inventory database** | Single source of truth for balances and PIN state, replicated |
| **Operator connectors** | Isolated per operator so one slow operator does not block others |
| **Queues** | Move non-critical work (SMS, reports) off the sale path |
| **Monitoring and alerting** | Success rate, latency, operator health, stock levels |

See [docs/ARCHITECTURE.md](./docs/ARCHITECTURE.md) for failover patterns and failure modes.

---

## How HA Works in an EVD Platform, Step by Step

### 1. Remove single points of failure

Every tier on the sale path (load balancer, API, database, connectors) needs at least one standby or peer. Document which components are active-active and which are active-passive.

### 2. Make every sale idempotent

Each request carries a unique ID from the POS or partner. If the same ID arrives again (because of a retry), the platform returns the original result. This is the single most important protection against duplicate vouchers during failures.

### 3. Protect PIN inventory with atomic state changes

Reserving a PIN, debiting the reseller, and marking the PIN sold must happen in one transaction. After a failover, the standby must hold exactly the same state.

### 4. Isolate operator dependencies

Use per-operator timeouts, connection pools, and circuit breakers. When an operator's pinless top-up API is slow, fail those requests fast with a clear code while other operators and PIN sales continue.

### 5. Handle "unknown" outcomes explicitly

If a top-up request times out, the result is unknown, not failed. Hold the reseller's funds, query the operator for status, and reconcile before refunding or retrying.

### 6. Plan capacity for peaks, then test it

Estimate peak transactions per second from your own historical data, add headroom, and load-test the full path, including operator simulators, before known peak dates.

### 7. Practise failover

Run planned failover drills. A standby that has never been promoted is a hope, not a plan.

---

## Key Concepts

- **RTO (Recovery Time Objective)** — how long the service may be down after a failure  
- **RPO (Recovery Point Objective)** — how much committed data may be lost; for ledgers and PIN state this should be as close to zero as possible  
- **Active-active vs active-passive** — both sites serve traffic, or one waits on standby  
- **Synchronous vs asynchronous replication** — trade-off between zero data loss and write latency  
- **Circuit breaker** — stops calling a failing dependency for a while so it can recover  
- **Graceful degradation** — keep core sales running while non-essential features pause  

---

## Implementation Considerations

- **Define SLOs from the business side** — agree target availability and latency with operators and distributors, then design to them  
- **Separate read load** — reporting and dashboards should not run heavy queries on the primary transaction database  
- **Backpressure** — queue or reject excess load cleanly instead of letting every component slow down  
- **Zero-downtime releases** — rolling or blue-green deployments, with backward-compatible database changes  
- **Backups are not HA** — they protect against data loss, not downtime; test restores anyway  
- **Reconciliation after incidents** — compare platform, operator, and reseller records after any failover  

Platforms such as EVD System / MoboGage cover electronic voucher management, eTopup, reseller POS, and APIs, so availability planning spans the whole chain from operator connectivity to the retailer's device.

---

## FAQ

**What is high availability in an electronic voucher distribution system?**  
Designing the EVD platform so sales continue through component failures, maintenance, and traffic peaks, while never duplicating or losing vouchers.

**How do EVD platforms avoid selling the same PIN twice during a failover?**  
By using unique request IDs (idempotency), atomic PIN state changes, and replication that keeps the standby database identical to the primary for committed transactions.

**What should happen when an operator's top-up API times out?**  
The transaction should be marked unknown, funds held, and the status queried with the operator before any refund or retry.

**How do you prepare an eTopup platform for peak load?**  
Use your own historical peaks to size capacity with headroom, scale stateless tiers horizontally, protect the database, and load-test the full path before known peak dates.

**How does this relate to MoboGage / EVD System?**  
MoboGage's EVD System is an electronic voucher management and distribution platform. See the [electronic voucher management system for telecom](https://evdsystem.com/electronic-voucher-management-system-evd-telecom/) page and the [EVMS migration playbook](https://evdsystem.com/evd-software/electronic-voucher-management-system-migration/).

---

## Further Reading / Related Industry Resources

- [Electronic voucher management system for telecom](https://evdsystem.com/electronic-voucher-management-system-evd-telecom/) — EVMS product context  
- [Migrating to a new electronic voucher management system](https://evdsystem.com/evd-software/electronic-voucher-management-system-migration/) — operator cutover playbook  
- Related repo: [electronic-voucher-distribution-system](https://github.com/iwtglobal/electronic-voucher-distribution-system) — EVD system fundamentals  

---

## License

Documentation in this repository is provided under the MIT License. See [LICENSE](./LICENSE).

*Educational material only. Availability targets and architectures depend on each deployment. MoboGage and EVD System refer to offerings on evdsystem.com.*
