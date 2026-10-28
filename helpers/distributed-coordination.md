# Distributed Coordination

- This document discusses how to do distributed coordination in cases where data is stored per instance and instances are dyanmic
- Strategies for this:
  - Manual (external-entities)
  - Memberlist (application-level)

## Manual

- If we need coordination with external entities like multiple shared database, then we need to do this manually
  - the use case is where each database needs to handle a specific subset of requests
- We need to have a fixed DNS on a proxy which routes to the currently active database instance
  - this is so that instance failure/maintenance doesn't affect the DNS resolution
  - caveat is that each independent database instance needs to have a unique DNS and therefore, a unique proxy
- The app then maintains a list of all the entities via DNS names
  - then using a consistent hash ring with the same hash algorithm, all app instances resolve the requests to the appropriate database instance

## Memberlist

### Introduction

- Memberlist is a Go library that manages cluster membership and node failure detection based on a gossip-based protocol
- It is eventually consistent but converges to average state quickly (O(logN))
- It is fairly configurable for distributed discovery of system components balancing convergence speed with bandwidth usage
- For Java/Python systems requiring memberlist capabilities
  - Spring ecosystem has Spring Cloud Consul
  - Python ecosystem has tattle
- But if we need framework-agnostic setup
  - we would need to setup a Consul sidecar which exposes gRPC API endpoints for memberlist capabilities

### Internal workings

- a headless service with a `dnssrv+` prefix allows discovering all endpoints (pod instance + ports) matching the current pod selector
- when a pod dies, Kubernetes removes it from the service's DNS record and when a new pod is ready, Kubernetes adds it
- the rest of the steps are not kubernetes dependent and happens via memberlist
- the new pod reaches out to one of those pods and asks to JOIN, and that node sends back the current list of members
- the new pod merges that list of members to its own and then gossips to a random subset of members
- these members receive, update their list and then gossip to another random subset each
- each addition/removal is a queued operation with a counter
  - periodically, the queue is checked and gossiped about each item
  - where each gossip updates the counter for that item
  - the same operation is not attempted again if counter runs out thus avoiding endless propagation
- there is also a separate periodic full-state sync with one specific node
  - both nodes exchange their tables and merge it on thier end to finalize state

### Usage Methodology

- Once an instance discovers what nodes are available, it can create a consistent hash ring to route requests to specific instances or read from them later on
  - all of them use the same hash algorithm so that the hash rings are also eventually consistent
- One position in the hash ring can have a specified replication factor so that request can be sent to multiple instances designated for that position
  - more about consistent hash rings at https://medium.com/towards-data-engineering/sharding-and-resharding-ii-6b3da2c2259c

## Conclusions

- Handle app-level membership using Consul if apps that need this functionality are polyglot (different Frameworks)
- Handle app-level membership with Spring Cloud Consul if apps that need this functionality are all Spring
- Handle external-entity memberships (Cache/DB instances) manually with unique DNS names
- Queues don't need this consideration as partitions already guarantee a specific consumer for each partition
