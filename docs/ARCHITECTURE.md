# Architecture — Electronic Voucher System High Availability

High-level reference for evaluating **electronic voucher distribution system uptime**, failover, and peak-load design for EVD and eTopup. Educational only; vendor designs vary.

## Logical Components

```
┌──────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│ Load balancers   │────►│ Stateless API    │────►│ Idempotency     │
│ (redundant pair) │     │ nodes (N+1)      │     │ store           │
└──────────────────┘     └────────┬─────────┘     └─────────────────┘
                                  │
                                  ▼
┌──────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│ Operator         │◄────│ Transaction      │────►│ Ledger + PIN DB │
│ connectors (CB)  │     │ service          │     │ primary/standby │
└──────────────────┘     └────────┬─────────┘     └─────────────────┘
                                  ▼
                         ┌──────────────────┐
                         │ Async queue:     │
                         │ SMS, reports,    │
                         │ reconciliation   │
                         └──────────────────┘
```

## Failover Patterns

| Pattern | When it fits | Watch out for |
|---------|--------------|---------------|
| **Active-passive database** | Strong consistency for ledger and PIN state | Promotion time, split-brain protection |
| **Active-active API tier** | Stateless request handling | Shared idempotency store must also be HA |
| **Per-operator connector pools** | Isolating slow or failing operators | Pool sizing during peaks |
| **Secondary site / region** | Site-level outages | Replication lag, DNS or routing switchover time |

## Suggested Transaction State Fields

| Field | Meaning |
|-------|---------|
| `request_id` | Client-supplied unique ID used for idempotency |
| `reseller_id` | Reseller or POS that sent the request |
| `product` | PIN voucher, pinless top-up, bundle, or gift voucher |
| `state` | `received` / `reserved` / `sent_to_operator` / `success` / `failed` / `unknown` |
| `operator_ref` | Operator's transaction reference, used for status queries |
| `funds_state` | `held` / `debited` / `released` |

## Signals to Monitor

- Success rate and latency per operator and per product  
- Count of transactions in `unknown` state and their age  
- Database replication lag  
- Idempotency hits (rising retries often signal a downstream problem)  
- PIN stock level per denomination against expected peak demand  

## Failure Modes to Design For

- Duplicate voucher sale after a client retry during failover  
- Reseller debited, voucher never delivered, no automatic reconciliation  
- One slow operator exhausting shared threads and stopping all sales  
- Standby database promoted with replication lag on PIN state  
- Reporting queries degrading the primary database at peak time  

## Related Reading

Live product reading: [electronic voucher management system](https://evdsystem.com/electronic-voucher-management-system-evd-telecom/), [EVMS migration playbook](https://evdsystem.com/evd-software/electronic-voucher-management-system-migration/).
