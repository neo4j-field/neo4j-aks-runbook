# NOM Server Installation Runbook
## Deploy Neo4j Ops Manager on AKS

**NOM Version:** 1.15.x
**Neo4j Version:** 2026.x Enterprise (backing store)
**Platform:** Azure Kubernetes Service (AKS)

---

## Overview

Neo4j Ops Manager (NOM) is a monitoring and management platform for Neo4j
clusters. The NOM server is deployed as a Kubernetes Deployment in its own
namespace. It stores its own operational data in a dedicated Neo4j database
and exposes two services:

- **HTTPS (port 443)** — web UI and REST API
- **gRPC (port 9090)** — agent communication channel

NOM does not require an ingress controller. Both services are exposed via Azure
LoadBalancer directly.

**Prerequisites before starting:**
- Neo4j cluster is running and all nodes are healthy (see `Neo4j_Cluster_Install.md`)
- You have `kubectl` access to the AKS cluster
- You have a TLS certificate and private key for your NOM hostname
  (a wildcard cert works well — e.g. `*.example.com`)
- Required tools: `az`, `kubectl`, `helm`, `openssl`

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

## Step 2 — Add the NOM Helm Repository

The NOM Helm chart is hosted separately from the Neo4j chart.

```bash
helm repo add nom https://dist.neo4j.org/ops-manager/helm-charts/
helm repo update

# Confirm the chart is available
helm search repo nom/neo4j-ops-manager-server
```

---

## Step 3 — Prepare Neo4j to Host NOM Data

NOM stores its own data (dashboards, alerts, agent registrations) in Neo4j.
This step creates a dedicated database and user for that purpose.

**Why a dedicated database?**
NOM 1.15.x uses a schema migration library that cannot parse `INTEGER | FLOAT`
property type constraints present in the default `neo4j` database in Neo4j 2026.x.
A fresh database (`nomdata`) has no such constraints, so migrations run cleanly.

Connect to any healthy Neo4j node:

```bash
kubectl exec <NEO4J_POD_NAME> \
  -n <NEO4J_NAMESPACE> \
  --context <YOUR_KUBE_CONTEXT> \
  -c neo4j -- \
  cypher-shell -a bolt://localhost:7687 \
  -u neo4j -p '<NEO4J_PASSWORD>'

# Example:
# kubectl exec gs-neo4j-1-0 -n neo4j --context my-aks-cluster -c neo4j -- \
#   cypher-shell -a bolt://localhost:7687 -u neo4j -p 'mypassword'
```

> Always use `-a bolt://localhost:7687` explicitly. The default `neo4j://` routing
> URI triggers TLS negotiation on localhost, which fails if a wildcard cert is
> configured (localhost does not match `*.example.com`).

Run the following Cypher commands:

```cypher
-- Dedicated NOM storage database.
-- 2 primaries ensures high availability within the cluster.
CREATE DATABASE nomdata IF NOT EXISTS TOPOLOGY 2 PRIMARIES 0 SECONDARIES;

-- Dedicated NOM user — do not reuse the neo4j admin account
CREATE USER nom_user IF NOT EXISTS
  SET PASSWORD '<NOM_DB_PASSWORD>'
  CHANGE NOT REQUIRED;

-- NOM requires admin role to manage its own schema
GRANT ROLE admin TO nom_user;

-- Route nom_user to nomdata by default so NOM migrations
-- run against the clean database, not the default neo4j database
ALTER USER nom_user SET HOME DATABASE nomdata;
```

Verify everything is online before continuing:

```cypher
SHOW DATABASES YIELD name, currentStatus, role
WHERE currentStatus <> 'online';
-- Expected: 0 rows (all databases online)
```

---

## Step 4 — Prepare the TLS Certificate for NOM

NOM requires its TLS certificate in PKCS12 format. The certificate must cover:
- The public hostname users will access (e.g. `nom.example.com`)
- The gRPC hostname agents will connect to (e.g. `nom-grpc.example.com`)

Using a wildcard cert (e.g. `*.example.com`) covers both with a single certificate
and avoids browser trust warnings.

```bash
# Convert PEM cert + key to PKCS12
openssl pkcs12 -export \
  -in public.crt \
  -inkey private.key \
  -out nom.pfx \
  -passout pass:<PKCS12_PASSWORD>    # e.g. pass:changeit

# Base64-encode — the output goes into nom.yaml in the next step
base64 -i nom.pfx | tr -d '\n'
```

Copy the base64 output to your clipboard — you will paste it into `nom.yaml`.

> **Why PKCS12?** The NOM Helm chart expects the certificate as an inline
> base64-encoded PKCS12 blob in the values file. There is no option to mount
> a Kubernetes secret for the NOM server cert directly.

---

## Step 5 — Create the NOM Namespace

NOM runs in its own namespace, separate from the Neo4j pods.

```bash
kubectl create namespace <NOM_NAMESPACE> --context <YOUR_KUBE_CONTEXT>
# Example: kubectl create namespace nom --context my-aks-cluster
```

---

## Step 6 — Configure nom.yaml

Create a `nom.yaml` Helm values file. Replace all `<PLACEHOLDER>` values.

```yaml
config:
  logLevel: "info"
  maxHeapSize: "8g"    # Set to ~75% of the memory limit below
  jwtTTL: "2h"         # How long user sessions stay active

secrets:
  # Bolt URI for NOM's Neo4j backing store.
  # Use the Kubernetes service DNS name — never hardcode the LoadBalancer IP,
  # which can change if the service is recreated or modified.
  #
  # Format: neo4j://<SERVICE_NAME>.<NAMESPACE>.svc.cluster.local:7687
  # The service name is: <HELM_RELEASE_NAME>-lb-neo4j
  #
  # Example: neo4j://gs-neo4j-lb-neo4j.neo4j.svc.cluster.local:7687
  storageUri: "neo4j://<NEO4J_LB_SERVICE_NAME>.<NEO4J_NAMESPACE>.svc.cluster.local:7687"

  storageUsername: "nom_user"
  storagePassword: "<NOM_DB_PASSWORD>"   # Must match what was set in Step 3

  # Random secret used to sign JWT tokens — generate with:
  # openssl rand -base64 48 | tr -d '\n'
  jwtSecret: "<RANDOM_64_CHAR_SECRET>"

  # PKCS12 cert from Step 4 (base64-encoded)
  tlsPkcs12CertFileContent: "<BASE64_PKCS12_FROM_STEP_4>"
  tlsPassword: "<PKCS12_PASSWORD>"       # Must match -passout in Step 4

service:
  http:
    port: 443     # NOM UI and REST API — exposed via Azure LoadBalancer
  grpc:
    port: 9090    # Agent communication — exposed via a separate Azure LoadBalancer

ingress:
  enabled: false  # Not needed — services are exposed directly via LoadBalancer

resources:
  limits:
    cpu: "2"
    memory: "8G"
  requests:
    cpu: "0.2"
    memory: "4G"

securityContext:
  runAsNonRoot: true
  runAsUser: 7474    # neo4j user
  runAsGroup: 7474
  fsGroup: 7474
```

> **How to find `<NEO4J_LB_SERVICE_NAME>`:**
> ```bash
> kubectl get svc -n <NEO4J_NAMESPACE> --context <YOUR_KUBE_CONTEXT>
> # Look for a service of type LoadBalancer with "lb-neo4j" in the name
> ```

---

## Step 7 — Install NOM

```bash
helm install nom-server nom/neo4j-ops-manager-server \
  --version <NOM_VERSION> \
  -f nom.yaml \
  -n <NOM_NAMESPACE> \
  --kube-context <YOUR_KUBE_CONTEXT>

# Example:
# helm install nom-server nom/neo4j-ops-manager-server \
#   --version 1.15.0 -f nom.yaml -n nom --kube-context my-aks-cluster
```

Wait for the pod to be ready:

```bash
kubectl rollout status deployment/nom-server \
  -n <NOM_NAMESPACE> --context <YOUR_KUBE_CONTEXT>
```

> **If Kubernetes services are missing after install** (this can happen on the
> first install into a new namespace), run a `helm upgrade` to reconcile:
> ```bash
> helm upgrade nom-server nom/neo4j-ops-manager-server \
>   --version <NOM_VERSION> -f nom.yaml \
>   -n <NOM_NAMESPACE> --kube-context <YOUR_KUBE_CONTEXT>
> ```

---

## Step 8 — Get the NOM Service IPs

After install, Azure provisions LoadBalancer IPs for both NOM services. Retrieve them:

```bash
kubectl get svc -n <NOM_NAMESPACE> --context <YOUR_KUBE_CONTEXT>
```

You will see two LoadBalancer services:

| Service | Port | Purpose |
|---------|------|---------|
| `nom-server-http` | 443 | NOM UI + REST API |
| `nom-server-grpc` | 9090 | Agent gRPC |

Note the `EXTERNAL-IP` for each. These are the IPs you add as DNS records.

---

## Step 9 — Add DNS Records

At your domain registrar, add two A records pointing to the IPs from Step 8.

| Hostname | Type | Value | Purpose |
|----------|------|-------|---------|
| `nom.<YOUR_DOMAIN>` | A | `<NOM_HTTP_EXTERNAL_IP>` | NOM UI and REST API |
| `nom-grpc.<YOUR_DOMAIN>` | A | `<NOM_GRPC_EXTERNAL_IP>` | Agent gRPC connections |

> **Why two separate DNS names?**
> Azure provisions separate IPs for the HTTP and gRPC services. NOM agents
> validate TLS against the hostname in `CONFIG_SERVER_ADDRESS` — using a
> subdomain of your wildcard cert (e.g. `nom-grpc.example.com`) ensures the
> cert is valid for that connection without requiring a separate certificate.
>
> **Hairpin NAT:** AKS pods cannot reach their own cluster's external
> LoadBalancer IP from inside the pod. The gRPC service has a *different*
> external IP than the Neo4j pods' LoadBalancer, so hairpin NAT is not an
> issue here. Agents connect to `nom-grpc.<YOUR_DOMAIN>` which resolves
> to the NOM gRPC IP, not their own pod's IP.

---

## Step 10 — Validate

### NOM pod is running

```bash
kubectl get pods -n <NOM_NAMESPACE> --context <YOUR_KUBE_CONTEXT>
# Expected: nom-server-<id> shows 1/1 Running
```

### NOM server started successfully

```bash
kubectl logs -n <NOM_NAMESPACE> --context <YOUR_KUBE_CONTEXT> \
  deployment/nom-server | grep "Successfully started"
# Expected: INFO  ... Successfully started Server Application
```

### NOM UI accessible with trusted cert

Open `https://nom.<YOUR_DOMAIN>` in a browser.
- If using a Let's Encrypt or other public CA cert: no warning, green padlock
- Log in: `admin` / `passw0rd` (default first login) — change the password immediately

### NOM can read from Neo4j

In the NOM UI, navigate to **Home**. If NOM connected to its backing store
successfully, the dashboard loads without errors. If you see a database
connection error, check:

```bash
kubectl logs -n <NOM_NAMESPACE> --context <YOUR_KUBE_CONTEXT> \
  deployment/nom-server | grep -i "error\|exception\|failed" | tail -20
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| NOM pod crashes on startup: `No enum constant PropertyType.INTEGER_\|_FLOAT` | NOM migration library cannot parse Neo4j 2026.x constraints on the default `neo4j` database | Create `nomdata` DB and set `nom_user` home database (Step 3) |
| NOM pod crashes: `Failed to obtain initial connection` | `storageUri` points to wrong host/port, or `nom_user` credentials are wrong | Verify `storageUri` uses k8s DNS name; verify password matches Step 3 |
| NOM pod crashes: `storageUri` resolves to unreachable IP | `storageUri` was hardcoded to a LoadBalancer IP that changed | Replace IP with k8s service DNS name in `nom.yaml` then `helm upgrade` |
| Browser cert warning for NOM UI | Certificate does not cover `nom.<YOUR_DOMAIN>` | Use a wildcard or multi-SAN cert that includes the public NOM hostname |
| `helm install` succeeds but services are missing | Helm timing issue on first install to new namespace | Run `helm upgrade` with the same arguments |
| NOM UI loads but shows database error | `nom_user` home database not set, NOM using wrong database | Verify `ALTER USER nom_user SET HOME DATABASE nomdata` was run (Step 3) |

---

## Upgrading NOM

To upgrade to a newer NOM version:

```bash
helm repo update

helm upgrade nom-server nom/neo4j-ops-manager-server \
  --version <NEW_NOM_VERSION> \
  -f nom.yaml \
  -n <NOM_NAMESPACE> \
  --kube-context <YOUR_KUBE_CONTEXT>
```

NOM runs as a single pod — there is no rolling upgrade. Expect a brief
outage during the pod restart. Agents will reconnect automatically once
NOM is back up.

---

## TLS Certificate Renewal

NOM's certificate must be renewed before it expires. For Let's Encrypt certs
this is every 90 days.

```bash
# 1. Convert renewed cert to PKCS12
openssl pkcs12 -export \
  -in public.crt \
  -inkey private.key \
  -out nom.pfx \
  -passout pass:<PKCS12_PASSWORD>

# 2. Update nom.yaml with new base64 value
base64 -i nom.pfx | tr -d '\n'
# Paste output into nom.yaml -> secrets.tlsPkcs12CertFileContent

# 3. Apply
helm upgrade nom-server nom/neo4j-ops-manager-server \
  --version <NOM_VERSION> -f nom.yaml \
  -n <NOM_NAMESPACE> --kube-context <YOUR_KUBE_CONTEXT>
```

---

## Reference Values (neo4jfield.org Example)

| Setting | Example Value |
|---------|--------------|
| NOM namespace | `nom` |
| NOM Helm release name | `nom-server` |
| NOM UI hostname | `nom.neo4jfield.org` |
| NOM gRPC hostname | `nom-grpc.neo4jfield.org` |
| NOM HTTP external IP | `52.226.247.39` |
| NOM gRPC external IP | `4.157.103.245` |
| `storageUri` | `neo4j://gs-neo4j-lb-neo4j.neo4j.svc.cluster.local:7687` |
| Neo4j namespace | `neo4j` |
| NOM DB name | `nomdata` |
| NOM DB user | `nom_user` |
| AKS context | `gs-k8s-neo4j-clstr` |
