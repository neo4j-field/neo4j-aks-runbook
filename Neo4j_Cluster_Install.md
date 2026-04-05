# Neo4j Cluster Installation Runbook
## Deploy a Neo4j Enterprise Cluster on AKS

**Neo4j Version:** 2026.x Enterprise
**Platform:** Azure Kubernetes Service (AKS)
**Topology:** Causal Cluster — 3-node Raft consensus (PRIMARY mode)

---

## Overview

This runbook deploys a Neo4j Enterprise cluster across multiple nodes using the
official Neo4j Helm chart. Each node is installed as a separate Helm release, all
sharing the same cluster name and discovery endpoint list.

**Cluster topology used here:**
Two nodes run inside AKS (managed by this runbook). A third node runs in a
separate region or environment and joins via the same endpoint list. The Raft
consensus algorithm requires a majority (2 of 3) of nodes to be available at
all times.

```
AKS Cluster (e.g. East region)
├── server1  — Neo4j PRIMARY (internal IP: e.g. 10.224.0.5)
└── server2  — Neo4j PRIMARY (internal IP: e.g. 10.224.0.10)

External (e.g. Central region)
└── server3  — Neo4j PRIMARY (IP: e.g. 172.16.0.8)
```

> **Quorum rule:** At least 2 of 3 nodes must be up and reachable at all times.
> Never restart or upgrade more than one node at a time.

**Prerequisites before starting:**
- AKS cluster is provisioned and sized (recommended: at least 4 vCPU per Neo4j node)
- You have a TLS certificate and private key covering your Neo4j hostnames
  (a wildcard cert such as `*.example.com` works well)
- Required tools: `az`, `kubectl`, `helm`, `openssl`
- Required files: `public.crt` (full cert chain), `private.key`

---

## Step 1 — Connect to the AKS Cluster

```bash
az aks get-credentials \
  --resource-group <YOUR_RESOURCE_GROUP> \
  --name <YOUR_AKS_CLUSTER_NAME> \
  --overwrite-existing

# Example:
# az aks get-credentials --resource-group my-rg --name my-aks-cluster

kubectl get nodes --context <YOUR_KUBE_CONTEXT>
```

---

## Step 2 — Add the Neo4j Helm Repository

```bash
helm repo add neo4j https://helm.neo4j.com/neo4j
helm repo update

# Confirm the chart is available
helm search repo neo4j/neo4j
```

---

## Step 3 — Create the Namespace

All Neo4j nodes in the same AKS cluster share one namespace.

```bash
kubectl create namespace <NEO4J_NAMESPACE> --context <YOUR_KUBE_CONTEXT>
# Example: kubectl create namespace neo4j --context my-aks-cluster
```

---

## Step 4 — Create the Storage Class

Neo4j pods use a PersistentVolumeClaim for data storage. The storage class must
exist before the Helm chart is installed — pods will stay in `Pending` if it is
missing.

Create a file `neo4j-sc-data.yaml`:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: neo4j-data-premium
provisioner: disk.csi.azure.com
parameters:
  skuName: Premium_LRS    # Premium SSD — recommended for production Neo4j workloads
reclaimPolicy: Retain     # Retain data when PVC is deleted — prevents accidental data loss
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

Apply it:

```bash
kubectl apply -f neo4j-sc-data.yaml --context <YOUR_KUBE_CONTEXT>

# Verify
kubectl get storageclass neo4j-data-premium --context <YOUR_KUBE_CONTEXT>
```

> `reclaimPolicy: Retain` means deleting the Helm release or PVC does **not**
> automatically delete the underlying Azure disk. This is intentional for
> production — manual cleanup is required but data is never silently lost.

---

## Step 5 — Create the Neo4j Auth Secret

The initial admin password is stored as a Kubernetes secret, not in the values
file. This keeps the password out of source control.

```bash
kubectl create secret generic neo4j-auth \
  --from-literal=NEO4J_AUTH=neo4j/<NEO4J_PASSWORD> \
  -n <NEO4J_NAMESPACE> \
  --context <YOUR_KUBE_CONTEXT>

# Example:
# kubectl create secret generic neo4j-auth \
#   --from-literal=NEO4J_AUTH=neo4j/mypassword \
#   -n neo4j --context my-aks-cluster
```

The secret name (`neo4j-auth`) must match `neo4j.passwordFromSecret` in your
values file.

---

## Step 6 — Create the SSL Secret

Store the TLS certificate and private key as a Kubernetes secret so they can be
mounted into the Neo4j pods. Both the certificate file (full chain) and the
private key are required.

```bash
kubectl create secret generic neo4j-ssl-cert \
  --from-file=public.crt=public.crt \
  --from-file=private.key=private.key \
  -n <NEO4J_NAMESPACE> \
  --context <YOUR_KUBE_CONTEXT>
```

> The secret is mounted at `/ssl/` inside the Neo4j container. The `defaultMode:
> 0640` in the values file ensures it is readable by the neo4j user (uid 7474)
> but not world-readable.

---

## Step 7 — Plan Internal IP Addresses

Each node needs a fixed internal IP for cluster discovery and Raft communication.
On Azure, assign fixed IPs by requesting them via the internal LoadBalancer
annotation. Choose IPs within your AKS subnet range.

| Node | Internal IP | Purpose |
|------|------------|---------|
| server1 | e.g. `10.224.0.5` | Cluster discovery + Raft on this IP |
| server2 | e.g. `10.224.0.10` | Cluster discovery + Raft on this IP |
| server3 (external) | e.g. `172.16.0.8` | Reachable from AKS nodes |

All three IPs go into `dbms.cluster.endpoints` in every node's config.

> Internal IPs are preferred over Azure Private DNS for cluster communication —
> simpler to configure, no DNS resolution latency, and no dependency on the
> Azure private zone service for Raft heartbeats.

---

## Step 8 — Configure Each Node's Helm Values

Create one values file per node. The structure is the same for each node;
only the per-node values (advertised address, internal IP, release name) differ.

### server1-e.yaml (annotated)

```yaml
neo4j:
  name: "<CLUSTER_NAME>"          # Shared across all nodes in the cluster
                                  # Example: gs-neo4j
  edition: "enterprise"
  acceptLicenseAgreement: "yes"
  passwordFromSecret: "neo4j-auth"  # Must match the secret name from Step 5
  minimumClusterSize: 3
  mode: "core"
  resources:
    # Set CPU high enough to avoid thread pool exhaustion.
    # Neo4j's thread pool = max(10, processors * 2).
    # With cpu: "1" (the chart default), enabling Prometheus metrics exhausts
    # the 10-thread pool. Use at least cpu: "3" on a 4-vCPU node.
    cpu: "3"        # Leave 1 vCPU for the NOM agent sidecar, OS, and k8s overhead
    memory: "10Gi"  # Adjust based on node size; leave headroom for sidecar

config:
  # Tag used for routing and server placement policies
  initial.server.tags: "<REGION_TAG>"           # e.g. "east"

  # Cluster discovery — LIST resolver with all node IPs and discovery port (6000)
  dbms.cluster.discovery.resolver_type: "LIST"
  dbms.cluster.endpoints: "<IP1>:6000,<IP2>:6000,<IP3>:6000"
  # Example: "10.224.0.5:6000,10.224.0.10:6000,172.16.0.8:6000"

  # How many primaries must be present before the cluster forms on first boot.
  # Set equal to the total number of primary nodes.
  dbms.cluster.minimum_initial_system_primaries_count: "3"
  initial.dbms.default_primaries_count: "3"
  initial.server.mode_constraint: "PRIMARY"

  # Listen on all interfaces inside the pod
  server.default_listen_address: "0.0.0.0"

  # Advertised to Bolt clients in the routing table.
  # Must use a DNS name that matches your TLS certificate so external clients
  # can validate TLS. Do NOT use the internal IP here.
  server.default_advertised_address: "<NODE_PUBLIC_DNS>"  # e.g. server1.example.com

  # Raft and cluster discovery use internal IPs — SSL is NOT applied to
  # cluster communication, so internal IPs are safe and more reliable here.
  server.cluster.raft.advertised_address: "<NODE_INTERNAL_IP>:7000"  # e.g. 10.224.0.5:7000
  server.cluster.advertised_address: "<NODE_INTERNAL_IP>:6000"       # e.g. 10.224.0.5:6000

  # Prometheus metrics endpoint — required for NOM agent monitoring
  server.metrics.prometheus.enabled: "true"

  # HTTPS — serves Neo4j Browser on port 7473
  server.https.enabled: "true"
  dbms.ssl.policy.https.enabled: "true"
  dbms.ssl.policy.https.base_directory: "/ssl"
  dbms.ssl.policy.https.certificate: "public.crt"
  dbms.ssl.policy.https.private_key: "private.key"
  dbms.ssl.policy.https.client_auth: "NONE"

  # Bolt TLS — set to OPTIONAL, not REQUIRED.
  # OPTIONAL allows plain bolt://localhost connections (used by the NOM agent
  # sidecar and internal tooling) while still offering TLS to external clients.
  # REQUIRED would break the NOM agent because localhost does not match a
  # wildcard cert like *.example.com.
  dbms.ssl.policy.bolt.enabled: "true"
  dbms.ssl.policy.bolt.base_directory: "/ssl"
  dbms.ssl.policy.bolt.certificate: "public.crt"
  dbms.ssl.policy.bolt.private_key: "private.key"
  dbms.ssl.policy.bolt.client_auth: "NONE"
  server.bolt.tls_level: "OPTIONAL"

jvm:
  additionalJvmArguments:
    # Required for Neo4j 2026.x on JDK 17+ to allow native memory access
    - "--enable-native-access=ALL-UNNAMED"

volumes:
  data:
    mode: dynamic
    dynamic:
      storageClassName: neo4j-data-premium   # Must match the StorageClass from Step 4
      accessModes:
        - ReadWriteOnce
      requests:
        storage: 200Gi    # Adjust based on your expected data volume

services:
  # Public LoadBalancer — exposes Neo4j Browser (7474/7473) and Bolt (7687)
  # to external clients. Azure assigns a public IP.
  neo4j:
    enabled: true
    spec:
      type: LoadBalancer

  # Internal LoadBalancer — used for cluster discovery and Raft communication.
  # Fixed internal IP ensures stable cluster topology even if the pod restarts.
  internals:
    annotations:
      service.beta.kubernetes.io/azure-load-balancer-internal: "true"
      service.beta.kubernetes.io/azure-load-balancer-internal-subnet: "<AKS_SUBNET_NAME>"  # e.g. aks-subnet
    spec:
      type: LoadBalancer
      loadBalancerIP: "<NODE_INTERNAL_IP>"   # Must match server.cluster.advertised_address above

# Required when Neo4j nodes span multiple Kubernetes clusters or external hosts.
# Tells the Helm chart not to assume all cluster members are in the same k8s cluster.
multiCluster: true

# SSL cert volume — mounted at /ssl/ for HTTPS and Bolt TLS
additionalVolumes:
  - name: ssl-cert
    secret:
      secretName: neo4j-ssl-cert
      defaultMode: 0640    # Readable by neo4j user (uid 7474) via fsGroup

additionalVolumeMounts:
  - name: ssl-cert
    mountPath: /ssl/public.crt
    subPath: public.crt
  - name: ssl-cert
    mountPath: /ssl/private.key
    subPath: private.key
```

### server2-e.yaml

Identical to `server1-e.yaml` with these values changed:

| Setting | server2 value |
|---------|--------------|
| `server.default_advertised_address` | `server2.<YOUR_DOMAIN>` |
| `server.cluster.raft.advertised_address` | `<SERVER2_INTERNAL_IP>:7000` |
| `server.cluster.advertised_address` | `<SERVER2_INTERNAL_IP>:6000` |
| `services.internals.spec.loadBalancerIP` | `<SERVER2_INTERNAL_IP>` |

> **server3 (external node):** Install using the same config values as above,
> adapted for that environment. Set `dbms.cluster.endpoints` identically on all
> three nodes. The `initial.server.tags` can differ (e.g. `"central"`) for
> routing policy purposes.

---

## Step 9 — Install the Nodes

Install both AKS nodes. They will wait for the third node (server3) to join
before the cluster fully forms — this is expected.

```bash
helm install <RELEASE_NAME_NODE1> neo4j/neo4j \
  -f server1-e.yaml \
  -n <NEO4J_NAMESPACE> \
  --kube-context <YOUR_KUBE_CONTEXT> \
  --timeout 10m

helm install <RELEASE_NAME_NODE2> neo4j/neo4j \
  -f server2-e.yaml \
  -n <NEO4J_NAMESPACE> \
  --kube-context <YOUR_KUBE_CONTEXT> \
  --timeout 10m

# Example:
# helm install gs-neo4j-1 neo4j/neo4j -f server1-e.yaml -n neo4j \
#   --kube-context my-aks-cluster --timeout 10m
# helm install gs-neo4j-2 neo4j/neo4j -f server2-e.yaml -n neo4j \
#   --kube-context my-aks-cluster --timeout 10m
```

Watch pod status:

```bash
kubectl get pods -n <NEO4J_NAMESPACE> --context <YOUR_KUBE_CONTEXT> -w
# Wait for pods to reach Running state (1/1, or 2/2 if NOM agent sidecar is included)
```

> Pods may show `0/1` for a few minutes while the PVC is provisioned and
> the cluster waits for quorum. This is normal. Once all 3 nodes (including
> server3) are up, the cluster forms and pods become Ready.

---

## Step 10 — Add DNS Records

After install, get the public IP assigned to each node's LoadBalancer service.
If you want to expose the cluster behind a single external hostname (recommended),
point it to one of the node IPs or use an Azure Load Balancer in front of them.

```bash
kubectl get svc -n <NEO4J_NAMESPACE> --context <YOUR_KUBE_CONTEXT>
# Look for services of type LoadBalancer with an EXTERNAL-IP assigned
```

Minimum DNS record needed for external access:

| Hostname | Type | Value | Purpose |
|----------|------|-------|---------|
| `neo4j.<YOUR_DOMAIN>` | A | `<PUBLIC_LB_IP>` | Neo4j Browser + Bolt |

> Individual node DNS names (e.g. `server1.example.com`, `server2.example.com`)
> are used by the `server.default_advertised_address` config so the Bolt routing
> table returns hostnames that match the TLS certificate. These do not need to
> be publicly resolvable unless external clients connect to individual nodes
> directly.

---

## Step 11 — Validate

### All pods are running

```bash
kubectl get pods -n <NEO4J_NAMESPACE> --context <YOUR_KUBE_CONTEXT>
# Expected: all Neo4j pods in Running state
```

### Cluster quorum is healthy

```bash
kubectl exec <POD_NAME> \
  -n <NEO4J_NAMESPACE> --context <YOUR_KUBE_CONTEXT> \
  -c neo4j -- \
  cypher-shell -a bolt://localhost:7687 \
  -u neo4j -p '<NEO4J_PASSWORD>' \
  'SHOW SERVERS YIELD name, address, state, health'
# Expected: all 3 servers — state="Enabled", health="Available"
```

> Always use `-a bolt://localhost:7687` explicitly when running cypher-shell
> inside the pod. The default `neo4j://` routing URI triggers TLS negotiation
> on localhost, which mismatches a wildcard cert and causes connection failures.

### All databases are online

```bash
kubectl exec <POD_NAME> \
  -n <NEO4J_NAMESPACE> --context <YOUR_KUBE_CONTEXT> \
  -c neo4j -- \
  cypher-shell -a bolt://localhost:7687 \
  -u neo4j -p '<NEO4J_PASSWORD>' \
  'SHOW DATABASES YIELD name, currentStatus, role'
# Expected: neo4j and system — all "online" on all 3 nodes
```

### HTTPS and Bolt TLS are running

```bash
kubectl logs <POD_NAME> -c neo4j \
  -n <NEO4J_NAMESPACE> --context <YOUR_KUBE_CONTEXT> \
  | grep -E "HTTPS enabled|Bolt enabled"
# Expected:
# INFO  HTTPS enabled on 0.0.0.0:7473.
# INFO  Bolt enabled on 0.0.0.0:7687.
```

### Browser accessible via HTTPS

```bash
curl -Ik https://neo4j.<YOUR_DOMAIN>:7473
# Expected: HTTP/1.1 200 — green padlock in browser
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| Pods stuck in `Pending` | StorageClass missing or PVC not bound | Apply `neo4j-sc-data.yaml` (Step 4); check `kubectl get pvc -n <ns>` |
| Pods stuck in `0/1 Running` — cluster not forming | One or more nodes unreachable — quorum not met | Verify all 3 nodes are up and `dbms.cluster.endpoints` is correct on all of them |
| Internal LB has wrong IP or keeps changing | `loadBalancerIP` not set in values | Set `services.internals.spec.loadBalancerIP` to the desired fixed internal IP |
| cypher-shell fails inside pod with TLS error | Default `neo4j://` scheme triggers TLS on localhost, mismatching the cert | Always use `cypher-shell -a bolt://localhost:7687` |
| Thread pool exhaustion (`51N38`) with Prometheus enabled | `cpu: "1"` (chart default) gives only 10 threads; Prometheus scraping exhausts them | Set `neo4j.resources.cpu: "3"` or higher in the values file |
| External clients see TLS hostname mismatch | `server.default_advertised_address` set to an IP or a name not covered by the cert | Use a DNS hostname that matches the TLS cert (e.g. a subdomain of your wildcard) |
| Helm upgrade did not update the internal LB service | Service has `helm.sh/resource-policy: keep` annotation | Manually patch the service with `kubectl annotate` or `kubectl edit svc` |

---

## Rolling Upgrade / Config Change

For any config change or Neo4j version upgrade, always update one node at a
time and verify cluster health before touching the next.

```bash
# Upgrade node 1
helm upgrade <RELEASE_NAME_NODE1> neo4j/neo4j \
  -f server1-e.yaml \
  -n <NEO4J_NAMESPACE> \
  --kube-context <YOUR_KUBE_CONTEXT> \
  --timeout 10m

# The StatefulSet has a 3600s graceful shutdown by default.
# In non-production, force-delete the pod to speed up the restart:
kubectl delete pod <POD_NAME_NODE1> \
  -n <NEO4J_NAMESPACE> --context <YOUR_KUBE_CONTEXT> \
  --grace-period=0 --force

# Wait for the pod to be Running again, then verify all 3 nodes are healthy:
kubectl exec <POD_NAME_NODE1> \
  -n <NEO4J_NAMESPACE> --context <YOUR_KUBE_CONTEXT> -c neo4j -- \
  cypher-shell -a bolt://localhost:7687 -u neo4j -p '<NEO4J_PASSWORD>' \
  'SHOW SERVERS YIELD name, state, health'
# All 3 must be Enabled / Available before proceeding

# Only then upgrade node 2
helm upgrade <RELEASE_NAME_NODE2> neo4j/neo4j \
  -f server2-e.yaml \
  -n <NEO4J_NAMESPACE> \
  --kube-context <YOUR_KUBE_CONTEXT> \
  --timeout 10m
```

---

## TLS Certificate Renewal

Update the Kubernetes secret with the renewed certificate and roll the pods
one at a time:

```bash
# Replace the secret in-place (--dry-run + apply avoids re-create/delete)
kubectl create secret generic neo4j-ssl-cert \
  --from-file=public.crt=public.crt \
  --from-file=private.key=private.key \
  -n <NEO4J_NAMESPACE> --context <YOUR_KUBE_CONTEXT> \
  --dry-run=client -o yaml | kubectl apply -f - --context <YOUR_KUBE_CONTEXT>

# Roll pods one at a time (see Rolling Upgrade above)
# Pods read the secret at startup — a rolling restart picks up the new cert
```

---

## Reference Values (neo4jfield.org Example)

| Setting | Example Value |
|---------|--------------|
| Cluster name (`neo4j.name`) | `gs-neo4j` |
| Neo4j namespace | `neo4j` |
| AKS context | `gs-k8s-neo4j-clstr` |
| AKS subnet name | `aks-subnet` |
| Auth secret name | `neo4j-auth` |
| SSL secret name | `neo4j-ssl-cert` |
| server1 release name | `gs-neo4j-1` |
| server2 release name | `gs-neo4j-2` |
| server1 internal IP | `10.224.0.5` |
| server2 internal IP | `10.224.0.10` |
| server3 external IP | `172.16.0.8` |
| server1 advertised address | `server1.neo4jfield.org` |
| server2 advertised address | `server2.neo4jfield.org` |
| Public Neo4j hostname | `neo4j.neo4jfield.org` → `4.255.28.111` |
| StorageClass | `neo4j-data-premium` |
| Data volume size | `200Gi` |
| Node size | `Standard_D4ds_v5` (4 vCPU, 16GB RAM) |
