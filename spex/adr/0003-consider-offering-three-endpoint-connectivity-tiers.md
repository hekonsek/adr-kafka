# Consider offering three endpoint connectivity tiers

## Context

Kafka clients need network access not only to a bootstrap address, but also to the broker addresses returned by Kafka metadata discovery. This makes endpoint design more important than a single hostname or load balancer. DNS, certificate subject alternative names, firewall rules, load balancer configuration, and advertised listeners must work for every broker endpoint exposed to a client.

Strimzi fits a tiered endpoint model because Kafka clusters can define multiple listeners. [Strimzi supports](https://strimzi.io/docs/operators/latest/configuring.html) internal listeners for Kubernetes-local access and external listener types such as `loadbalancer`, where listener configuration can define bootstrap and per-broker settings. This matches Kafka client behavior after bootstrap metadata discovery.

Different client deployments have different connectivity needs. Workloads inside the same Kubernetes cluster should not be forced through public endpoints. Workloads in other clusters, VPCs, or corporate networks usually need private routability. Partners, SaaS systems, and unmanaged networks may need public access, but that access has a larger security and governance surface.

## Decision

We will consider Kafka endpoint access as a product offering with three ordered connectivity tiers:

| Tier | Name | Audience | Endpoint type | Positioning |
| --- | --- | --- | --- | --- |
| 1 | In-cluster endpoint | Workloads in the same Kubernetes cluster | `*.svc.cluster.local` or Strimzi internal listener | Best latency, simplest operation, lowest exposure, and lowest infrastructure cost |
| 2 | Private endpoint | Workloads in other clusters, VPCs, or corporate networks | Internal cloud load balancers, private DNS, VPC peering, VPN, or equivalent private routing | Recommended for enterprise integration, MirrorMaker 2, and private applications |
| 3 | Public endpoint | Partners, SaaS platforms, and unmanaged networks | Public load balancers and public DNS | Maximum reach, strongest governance required |

The endpoint selection rule is:

> Use the most private endpoint that satisfies the client's deployment topology. Public access is supported, but not preferred for internal workloads.

Endpoint choice should be made per client or application, not globally per Kafka cluster. The same Kafka cluster may expose multiple listeners, but each application should be onboarded to one preferred access mode.

For MirrorMaker 2, private endpoints should be the default recommendation when the source and target clusters are not in the same Kubernetes cluster. Replication traffic benefits from private routability, stable bandwidth, lower exposure, and avoiding public internet paths.

Public endpoints must be treated as a distinct product tier rather than just the same Kafka service exposed publicly. Public access must require TLS, strong client authentication such as SASL, OAuth, or mTLS, authorization with ACLs, quotas, client identity policy, rate limits where supported by the platform, audit logs, documented onboarding, and a clear operational ownership model.

For LoadBalancer-based listeners, cost and quota planning must account for the Kafka exposure pattern. A typical exposed broker topology requires a bootstrap load balancer plus one load balancer per broker, so a cluster with `N` brokers can require `N + 1` load balancers for that listener.

## Consequences

Pros:
* Gives teams a clear default ordering for endpoint selection.
* Keeps internal workloads on the lowest-exposure and lowest-cost path.
* Provides a private-network recommendation for enterprise integrations and MirrorMaker 2.
* Allows public access when necessary without normalizing it for internal traffic.
* Makes per-broker routability, DNS, certificates, firewall rules, and advertised listeners explicit design requirements.
* Helps cloud cost and load balancer quota planning by making the `N brokers + 1 bootstrap` pattern visible.

Cons:
* Requires teams to maintain more than one listener and onboarding path when multiple tiers are enabled.
* Increases documentation and support requirements for client onboarding.
* Public access needs stricter governance, monitoring, abuse protection, and incident response readiness.
* Private endpoint tiers depend on cloud networking, DNS, routing, and firewall coordination outside Kafka itself.
* LoadBalancer-based tiers can increase infrastructure cost and cloud quota consumption as broker counts grow.

## Alternatives considered

- **Single Kubernetes-internal endpoint only**. Simplest and cheapest, but does not support clients outside the Kafka cluster's Kubernetes network.
- **Single private endpoint only**. Works for many enterprise integrations, but can add unnecessary network distance for same-cluster workloads and does not cover partners or SaaS systems without private connectivity.
- **Single public endpoint for all clients**. Maximizes reach, but exposes internal workloads to avoidable public-network governance, cost, and security concerns.
- **Cluster-level endpoint choice**. Simpler to document, but too coarse. Kafka clusters can expose multiple listeners, and endpoint selection should follow each application's deployment topology.
- **Ad hoc endpoint selection per project**. Flexible, but leads to inconsistent listener naming, onboarding, firewall rules, DNS patterns, certificate handling, and cost planning.
