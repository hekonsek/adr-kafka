# Separate KRaft controller and broker node pools

## Context

Kafka deployments using KRaft separate the cluster metadata quorum from the broker data plane. KRaft controller nodes own the metadata quorum, while broker nodes handle client traffic, topic partitions, and replication.

When creating a Kafka cluster, it can be tempting to combine the controller and broker roles on the same nodes. This reduces the number of node pools and may look attractive, especially in lower environments where infrastructure cost is under pressure.

However, controller and broker nodes have different scaling characteristics. Broker capacity often needs to grow or shrink with traffic, partition count, retention, storage, and throughput requirements. Controller quorum sizing is more constrained and should be changed deliberately because it affects the metadata quorum. Combining both roles makes broker scaling unnecessarily coupled to the controller quorum and makes the cluster harder to operate predictably.

## Decision

When creating Kafka clusters with KRaft, we prefer to deploy KRaft controller nodes and broker nodes separately.

For Strimzi-based deployments, this means using separate node pools for controller nodes and broker nodes. Controller node pools should run the controller role. Broker node pools should run the broker role. We should avoid using combined controller and broker nodes as the default topology.

This rule applies to lower environments as well as production-like and production environments. Lower environments should still replicate the two-pool topology with separate controller and broker nodes, even if each pool uses smaller resource requests, smaller storage, fewer broker replicas, or reduced retention. We should not trade away topology fidelity just to save cost by combining controller and broker roles.

## Consequences

Pros:
* Broker capacity can be scaled independently from the KRaft controller quorum.
* Controller quorum changes remain explicit, deliberate, and rare.
* Lower environments exercise the same operational model as production-like environments.
* Cluster behavior is easier to reason about during upgrades, node replacement, and capacity changes.
* Failure domains and observability can distinguish metadata quorum issues from broker capacity issues.

Cons:
* Requires more Kubernetes nodes or pods than a combined-role topology.
* May increase baseline cost in development, test, and staging environments.
* Requires teams to maintain separate resource sizing for controller and broker pools.
* Small environments still need enough capacity planning to run both pools reliably.

## Alternatives considered

- **Combined controller and broker nodes**. Reduces the number of nodes and can lower baseline cost, but couples broker elasticity to controller quorum sizing and makes routine broker scaling more complex.
- **Combined roles only in lower environments**. Saves cost outside production, but creates an environment topology that does not match production-like operation. This weakens testing for upgrades, scaling, failure recovery, and observability.
- **Single shared pool with mixed roles**. Keeps a simpler pool structure, but still mixes different operational concerns and makes it harder to apply role-specific sizing, scheduling, and failure analysis.
