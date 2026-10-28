# Distributed Coordination

- This document discusses how to do distributed coordination in cases where data is stored per instance and instances are dyanmic
- Strategies for this:
  - Manual
  - Memberlist

## Manual

- [TODO]

## Memberlist

- [REWORD-BELOW]

```
You don't — and that's the key insight. You don't need a specific long-term member. You need a stable DNS name that always resolves to whatever members are currently alive.

In Mimir on Kubernetes, this is a headless Service:

memberlist:
  join_members:
    - "dnssrv+mimir-gossip-ring.mimir.svc.cluster.local:7946"

The dnssrv+ prefix tells memberlist to do a DNS SRV lookup on that name. A headless Kubernetes Service has no single IP — it returns a DNS record for every pod currently matching the selector
Pod mimir-abc123 dies → Kubernetes removes it from the Service's DNS records.
Pod mimir-xyz789 is scheduled → Kubernetes adds it.
The DNS name mimir-gossip-ring.mimir.svc.cluster.local always resolves to the set of currently-running pods. 

So when a new pod starts and does Join, it:
Resolves the DNS name → gets a list of current pod IPs (none of which need to be "long-term").
Tries them until one answers.
Gets the full table via TCP sync.
```
