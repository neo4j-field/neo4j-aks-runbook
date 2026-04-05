# Neo4j on AKS — Production Runbooks

Step-by-step runbooks for deploying and operating **Neo4j Enterprise** clusters
and **Neo4j Ops Manager (NOM)** on **Azure Kubernetes Service (AKS)**.

These guides are built from real production deployments. They cover not just
the happy path, but the edge cases, gotchas, and operational decisions that
matter when running Neo4j at scale in the cloud.

---

## Runbooks

| Runbook | Description |
|---------|-------------|
| [Neo4j Cluster Install](Neo4j_Cluster_Install.md) | Deploy a multi-node Neo4j Enterprise cluster on AKS with TLS, persistent storage, and Azure Load Balancer |
| [NOM Server Install](NOM_Server_Install.md) | Deploy Neo4j Ops Manager server with HTTPS + gRPC services, Let's Encrypt TLS, and a dedicated backing database |
| [NOM Agent Install](NOM_Agent_Install.md) | Deploy the NOM monitoring agent as a sidecar inside each Neo4j pod — including multi-cluster setups |

---

## Why These Runbooks

**Production-hardened.** Each guide covers real failure modes encountered in
production — stale routing tables, Raft leader election behaviour, TLS cert
mismatches, Helm timing issues, and more. You won't find these details in the
standard product docs.

**Multi-cluster aware.** The NOM agent runbook explicitly handles the common
case where your NOM server and monitored Neo4j clusters run on different AKS
clusters — covering secret placement, cross-cluster DNS resolution, and TLS
trust across cluster boundaries.

**Security-first.** Passwords and secrets are always stored as Kubernetes
secrets, never hardcoded in Helm values files. The guides explain *why*, not
just *what* — so your team understands the reasoning behind each decision.

**Azure-native.** Built specifically for AKS — covers Azure StorageClass
configuration, Azure Internal Load Balancer annotations, AKS credential
management with `az aks get-credentials`, and Azure DNS record setup for
multi-service NOM deployments.

**Upgrade and renewal paths included.** Every runbook ends with maintenance
procedures: rolling Neo4j version upgrades, TLS certificate renewal (90-day
Let's Encrypt cycle), and NOM upgrades — so the runbooks stay useful beyond
the initial install.

**Fully parameterised.** All environment-specific values use `<PLACEHOLDER>`
syntax with inline comments and a Reference Values table at the end of each
guide. Adapt to your environment without hunting through the document.

---

## Who This Is For

- **Platform / infrastructure engineers** standing up Neo4j on AKS for the
  first time or migrating from VMs
- **Neo4j field engineers and partners** working with customers on AKS deployments
- **Architects** evaluating Neo4j's operational model on Kubernetes before
  committing to a production topology

---

## Prerequisites

- Azure subscription with AKS cluster(s) provisioned
- `az`, `kubectl`, `helm`, `openssl` installed locally
- Neo4j Enterprise license
- A wildcard or multi-SAN TLS certificate for your domain

---

## Versioning

| Component | Version |
|-----------|---------|
| Neo4j | 2026.x Enterprise |
| Neo4j Ops Manager | 1.15.x |
| Platform | Azure Kubernetes Service |

---

## Contributing

Issues and pull requests are welcome. If you hit a scenario not covered here —
an error, a configuration edge case, or a step that needed clarification —
please open an issue or submit a PR.
