# Prefer using Strimzi for non-managed Kafka deployments

## Context

We are building systems that use Apache Kafka and need a standard recommendation for deployments where Kafka is not provided as a managed service.

Running Kafka ourselves means we must handle broker lifecycle, rolling upgrades, storage, certificates, authentication, authorization, topic and user management, observability, and failure recovery. Without a standard operational model, teams tend to create project-specific Helm values, raw Kubernetes manifests, or shell scripts that are difficult to review, upgrade, and operate consistently.

## Decision

For non-managed Kafka deployments on Kubernetes, we prefer to use [Strimzi](https://strimzi.io/) as the default way to deploy and operate Kafka.

Managed Kafka services may still be preferred when the project explicitly chooses a managed platform. This ADR applies when the project has decided to run Kafka itself instead of relying on a managed Kafka offering.

Strimzi should be used to manage Kafka through Kubernetes-native custom resources, including Kafka clusters, Kafka topics, Kafka users, Kafka Connect, and Kafka MirrorMaker 2 where applicable. We should stay close to upstream Strimzi conventions and examples unless a project has a clear operational reason to diverge.

## Consequences

Pros:
* Consistent deployment model for non-managed Kafka environments.
* Kafka lifecycle management is delegated to a mature Kubernetes operator.
* Rolling upgrades, listeners, TLS, authentication, authorization, topic management, and user management can be expressed declaratively.
* Easier onboarding for teams already operating Kubernetes workloads.
* Better fit for GitOps workflows than hand-written scripts or ad hoc broker management.

Cons:
* Requires Kubernetes and operator operational knowledge.
* Couples non-managed Kafka deployments to Strimzi custom resources and release cadence.
* Still requires disciplined capacity planning, monitoring, storage management, and disaster recovery design.
* Adds another platform component that must be upgraded, observed, and secured.

## Alternatives considered

- **Managed Kafka services**. Usually the best option when available and acceptable, because the provider owns much of the operational burden. This ADR is specifically for cases where a managed Kafka service is not used.
- **Raw Kubernetes manifests or StatefulSets**. Gives maximum control, but pushes broker lifecycle, upgrades, certificates, and configuration drift management back onto each project.
- **Generic Helm charts**. Can simplify initial installation, but usually provide less Kafka-specific operational automation than Strimzi.
- **Manual virtual machine deployments**. Useful in some legacy environments, but harder to standardize, automate, and review than Kubernetes-native operator-managed deployments.
