# NOM Agent Installation Runbook
## Deploy the NOM Agent as a Sidecar on AKS

**Neo4j Version:** 2026.x Enterprise
**NOM Version:** 1.15.x
**Platform:** Azure Kubernetes Service (AKS)

---

## Overview

The NOM agent runs as a sidecar container inside each Neo4j pod. It connects to
the NOM server over gRPC, monitors the local Neo4j instance, and streams metrics,
logs, and health data back to NOM.

The agent binary is extracted from the Neo4j Docker image at pod startup via an
init container — no separate agent image is required.

> **Multi-cluster note:** The NOM server and the Neo4j cluster being monitored are
> often on **different AKS clusters**. All Kubernetes secrets (agent credentials,
> TLS cert) must be created on the **agent's cluster** (where Neo4j runs), not on
> the NOM server's cluster. The NOM server only needs to be reachable over the
> network from the agent pods.

**Prerequisites before starting:**
- Neo4j cluster is running and all nodes are healthy
- NOM server is installed and accessible (see `NOM_Server_Install.md`)
- You have admin access to the NOM UI
- The NOM gRPC hostname (e.g. `nom-grpc.example.com`) resolves from inside the
  agent cluster's pods — verified in Step 1 below
- The NOM server's public TLS certificate (public.crt) is available locally —
  you will create a Kubernetes secret from it in Step 3

---

## Step 1 — Connect to the AKS Cluster

Obtain credentials for the cluster so `kubectl` and `helm` commands work.

```bash
az aks get-credentials \
  --resource-group <YOUR_RESOURCE_GROUP> \
  --name <YOUR_AKS_CLUSTER_NAME> \
  --overwrite-existing

# Example:
# az aks get-credentials --resource-group my-rg --name my-aks-cluster --overwrite-existing

# Verify
kubectl get nodes --context <YOUR_KUBE_CONTEXT>
```

### Pre-flight: Verify DNS resolution from inside the cluster

The NOM gRPC hostname must resolve from within the agent pod's network — external
DNS propagation alone is not sufficient. Verify before proceeding:

```bash
# Spin up a temporary pod and test DNS from inside the cluster's network
kubectl run dns-test --image=alpine --restart=Never --rm -it \
  -n <NEO4J_NAMESPACE> \
  --context <YOUR_KUBE_CONTEXT> \
  -- nslookup <NOM_GRPC_HOSTNAME>

# Example:
# kubectl run dns-test --image=alpine --restart=Never --rm -it \
#   -n neo4j --context my-aks-cluster \
#   -- nslookup nom-grpc.example.com

# Expected: returns a valid A record IP (not NXDOMAIN)
```

> If you get `NXDOMAIN`, add the DNS A record for your NOM gRPC hostname at your
> registrar and wait for propagation before continuing. The agent will fail silently
> at startup if this hostname does not resolve.

---

## Step 2 — Register the Agent in NOM UI

Each Neo4j instance needs its own registered agent entry in NOM. The registration
generates an OAuth2 client ID and secret that the agent uses to authenticate with
the NOM server.

1. Open `https://<YOUR_NOM_HOSTNAME>` (e.g. `https://nom.example.com`)
2. Log in as admin
3. Navigate to **Agents** → **Add Agent**
4. Enter a name and description (e.g. `server1-agent`, `NOM agent for server1`)
5. Copy the generated **Client ID** and **Client Secret** — you will not be able
   to retrieve the secret again after closing the dialog

Repeat for each Neo4j node that will run an agent.

---

## Step 3 — Create Kubernetes Secrets on the Agent Cluster

All secrets must be created on the **agent's AKS cluster** (where Neo4j runs),
not on the NOM server's cluster.

### 3a — Agent credentials

The OAuth client secret from Step 2 must be stored as a Kubernetes secret — never
hardcode it in the Helm values file.

```bash
kubectl create secret generic nom-agent-<NODE_NAME> \
  --from-literal=client-secret='<CLIENT_SECRET_FROM_NOM_UI>' \
  -n <NEO4J_NAMESPACE> \
  --context <YOUR_KUBE_CONTEXT>

# Example (repeat for each node):
# kubectl create secret generic nom-agent-server1 \
#   --from-literal=client-secret='<SERVER1_SECRET>' \
#   -n neo4j --context my-aks-cluster
#
# kubectl create secret generic nom-agent-server2 \
#   --from-literal=client-secret='<SERVER2_SECRET>' \
#   -n neo4j --context my-aks-cluster
```

### 3b — NOM server TLS certificate

The agent needs the NOM server's public certificate to verify the gRPC TLS
connection. How you store it depends on whether Neo4j SSL is already configured
on this cluster:

**Option A — Neo4j SSL is already configured on this cluster**

If you already have a Kubernetes secret containing your TLS cert (e.g.
`neo4j-ssl-cert` with `public.crt` and `private.key`), no new secret is needed.
You will reference the existing secret name in Step 4.

**Option B — No Neo4j SSL on this cluster (e.g. a secondary/central cluster)**

Create a dedicated secret containing only the public cert. The agent only needs
the public cert to verify the NOM server — the private key is not required and
should not be copied to a cluster that does not own the cert.

```bash
# Copy public.crt from your NOM cert to the local machine, then:
kubectl create secret generic nom-tls-cert \
  --from-file=public.crt=./public.crt \
  -n <NEO4J_NAMESPACE> \
  --context <YOUR_KUBE_CONTEXT>

# Example:
# kubectl create secret generic nom-tls-cert \
#   --from-file=public.crt=./public.crt \
#   -n neo4j --context my-aks-cluster
```

> **Why public.crt only?** The private key is used by the NOM *server* to
> establish TLS. The agent (client) only needs the public cert to trust the
> server's certificate. Distributing the private key to every monitored cluster
> is unnecessary and a security risk.

---

## Step 4 — Add the Agent Sidecar to the Neo4j Helm Values

Add the following sections to your Neo4j Helm values file (e.g. `server1-e.yaml`).
Replace all `<PLACEHOLDER>` values — example values are shown in comments.

### 4a — Additional Volumes

Declare the volumes that the init container and sidecar need. Add these at the
top level of your values file.

Choose the appropriate ssl volume definition based on your cluster's SSL setup:

**Option A — Cluster with Neo4j SSL already configured** (secret has public.crt + private.key):

```yaml
additionalVolumes:
  - name: agent-bin
    emptyDir: {}
  - name: ssl-cert                         # references existing Neo4j SSL secret
    secret:
      secretName: <YOUR_NEO4J_SSL_SECRET>  # e.g. neo4j-ssl-cert
      defaultMode: 0640
```

**Option B — Cluster without Neo4j SSL** (public cert only, created in Step 3b):

```yaml
additionalVolumes:
  - name: agent-bin
    emptyDir: {}
  - name: nom-tls-cert                     # public cert only — no private key
    secret:
      secretName: nom-tls-cert
      defaultMode: 0640
```

### 4b — Additional Volume Mounts (Neo4j main container)

These mounts apply to the Neo4j main container for Bolt/HTTPS TLS.

**Option A — Cluster with Neo4j SSL** (mount both cert and key):

```yaml
additionalVolumeMounts:
  - name: ssl-cert
    mountPath: /ssl/public.crt
    subPath: public.crt
  - name: ssl-cert
    mountPath: /ssl/private.key
    subPath: private.key
```

**Option B — Cluster without Neo4j SSL** (no mounts needed for main container):

```yaml
additionalVolumeMounts: []
```

### 4c — Pod Spec: Init Container + Agent Sidecar

The init container extracts the agent binary from the Neo4j image. The sidecar
then runs it continuously. Add this under `podSpec:` in your values file:

```yaml
podSpec:
  initContainers:
    - name: agent-extractor
      # Use the same Neo4j image version as your running cluster.
      # The agent binary is bundled inside the image under products/.
      image: neo4j:<NEO4J_VERSION>-enterprise   # e.g. neo4j:2026.01.4-enterprise
      command:
        - "/bin/bash"
        - "-c"
        - >
          tar -xzf products/neo4j-ops-manager-agent-*-linux-amd64.tar.gz
          --strip-components 1 && mv bin/agent /agent/bin/agent
      volumeMounts:
        - name: agent-bin
          mountPath: /agent/bin

  containers:
    - name: nom-agent
      image: alpine:latest
      command:
        - "/agent/bin/agent"
        - "console"
      env:
        # Address of the NOM gRPC service.
        # Must be a hostname that matches your NOM server's TLS certificate.
        # For a wildcard cert (*.example.com), use a subdomain: nom-grpc.example.com
        - name: CONFIG_SERVER_ADDRESS
          value: "<NOM_GRPC_HOSTNAME>:9090"        # e.g. nom-grpc.example.com:9090

        # NOM OAuth2 token endpoint — used by the agent to authenticate on startup
        - name: CONFIG_TOKEN_URL
          value: "https://<NOM_HOSTNAME>/api/login/agent"  # e.g. https://nom.example.com/api/login/agent

        # Client ID from Step 2 (NOM UI) — unique per agent
        - name: CONFIG_TOKEN_CLIENT_ID
          value: "<CLIENT_ID_FROM_NOM_UI>"

        # Client secret from Step 3 — read from Kubernetes secret (never hardcoded)
        - name: CONFIG_TOKEN_CLIENT_SECRET
          valueFrom:
            secretKeyRef:
              name: nom-agent-<NODE_NAME>          # e.g. nom-agent-server1
              key: client-secret

        # Path to the cert file the agent uses to verify the NOM server's TLS cert.
        # Must be mounted below in volumeMounts.
        - name: CONFIG_TLS_TRUSTED_CERTS
          value: "/ssl/public.crt"

        # Agent identity — shown in NOM UI under Agents
        - name: CONFIG_AGENT_NAME
          value: "<NODE_NAME>-agent"               # e.g. server1-agent
        - name: CONFIG_AGENT_DESCRIPTION
          value: "NOM agent for <NODE_NAME>"

        - name: CONFIG_LOG_LEVEL
          value: "info"

        # Neo4j instance details — the agent connects here to collect metrics.
        # Use bolt://localhost (plain, no TLS) so localhost cert mismatch is avoided.
        - name: CONFIG_INSTANCE_1_NAME
          value: "<NODE_NAME>"                     # e.g. server1
        - name: CONFIG_INSTANCE_1_BOLT_URI
          value: "bolt://localhost:7687"
        - name: CONFIG_INSTANCE_1_BOLT_USERNAME
          value: "neo4j"
        - name: CONFIG_INSTANCE_1_BOLT_PASSWORD
          value: "<NEO4J_PASSWORD>"

        # Query log collection — port must match server-logs.xml config
        - name: CONFIG_INSTANCE_1_QUERY_LOG_PORT
          value: "9500"
        - name: CONFIG_INSTANCE_1_LOG_CONFIG_PATH
          value: "/var/lib/neo4j/conf/server-logs.xml"

      volumeMounts:
        - name: agent-bin
          mountPath: /agent/bin

        # SSL cert for verifying the NOM server certificate over gRPC.
        # Use the volume name that matches what you declared in 4a:
        #   Option A (Neo4j SSL cluster):  name: ssl-cert
        #   Option B (no Neo4j SSL):       name: nom-tls-cert
        - name: <SSL_VOLUME_NAME>          # ssl-cert (Option A) or nom-tls-cert (Option B)
          mountPath: /ssl/public.crt
          subPath: public.crt

        # Data volume — agent reads Neo4j data directory for disk metrics.
        # subPath: data is required because the Neo4j Helm chart stores data
        # under a data/ subdirectory within the PVC root.
        - name: data
          mountPath: /var/lib/neo4j/data
          subPath: data


> **Why `bolt://localhost:7687` (plain bolt) for the agent?**
> The agent connects to Neo4j on the loopback interface. If Bolt TLS is enabled
> with a wildcard cert (e.g. `*.example.com`), `localhost` does not match the cert,
> causing a TLS handshake failure. Setting `server.bolt.tls_level: "OPTIONAL"` in
> your Neo4j config allows plain bolt on loopback while still enforcing TLS for
> external clients.

---

## Step 5 — Deploy the Agent (Rolling, One Node at a Time)

For a cluster setup, always upgrade one node, verify cluster quorum, then proceed
to the next. Taking multiple nodes down simultaneously can cause quorum loss.

```bash
# Upgrade the first node
helm upgrade <RELEASE_NAME_NODE1> neo4j/neo4j \
  -f <NODE1_VALUES_FILE> \
  -n <NEO4J_NAMESPACE> \
  --kube-context <YOUR_KUBE_CONTEXT> \
  --timeout 10m

# Example:
# helm upgrade gs-neo4j-1 neo4j/neo4j -f server1-e.yaml -n neo4j \
#   --kube-context my-aks-cluster --timeout 10m
```

The StatefulSet has a 3600s termination grace period. To speed up pod restart
in a non-production environment:

```bash
kubectl delete pod <POD_NAME> \
  -n <NEO4J_NAMESPACE> \
  --context <YOUR_KUBE_CONTEXT> \
  --grace-period=0 --force

# Example: kubectl delete pod gs-neo4j-1-0 -n neo4j --context my-aks-cluster \
#   --grace-period=0 --force
```

Wait for the pod to show `2/2 Running` (main container + agent sidecar):

```bash
kubectl get pod <POD_NAME> -n <NEO4J_NAMESPACE> --context <YOUR_KUBE_CONTEXT> -w
```

**Before upgrading the next node, verify cluster quorum:**

```bash
kubectl exec <POD_NAME> -n <NEO4J_NAMESPACE> --context <YOUR_KUBE_CONTEXT> \
  -c neo4j -- \
  cypher-shell -a bolt://localhost:7687 \
  -u neo4j -p '<NEO4J_PASSWORD>' \
  'SHOW SERVERS YIELD name, address, state, health'
# All nodes must show state="Enabled", health="Available"
```

Repeat for each remaining node.

---

## Step 6 — Validate

### Agent is running

```bash
kubectl get pods -n <NEO4J_NAMESPACE> --context <YOUR_KUBE_CONTEXT>
# Each Neo4j pod should show 2/2 Running (main + sidecar)
```

### Agent logs show successful connection

```bash
kubectl logs <POD_NAME> -c nom-agent \
  -n <NEO4J_NAMESPACE> --context <YOUR_KUBE_CONTEXT> --tail=20

# Expected lines:
# INFO  | Version: 1.15.x
# INFO  | connected to NOM server address:<NOM_GRPC_HOSTNAME>:9090
# INFO  | sending ping
```

### NOM UI shows agents connected

Open `https://<YOUR_NOM_HOSTNAME>` → **Agents**.
Each registered agent should show status **Connected**.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| Pod shows `1/2` or sidecar in `CrashLoopBackOff` | Agent binary not extracted — init container failed | Check init container logs: `kubectl logs <pod> -c agent-extractor -n <ns>` |
| Agent log: `x509: certificate signed by unknown authority` | `CONFIG_TLS_TRUSTED_CERTS` points to wrong file or file not mounted | Verify the ssl volumeMount is present in the `nom-agent` container's `volumeMounts` and the volume name matches the secret created in Step 3b |
| Agent log: `x509: certificate is valid for ..., not <hostname>` | NOM server cert does not cover `CONFIG_SERVER_ADDRESS` hostname | Use a hostname that matches the NOM cert (e.g. subdomain of wildcard cert) |
| Agent log: `NXDOMAIN` or `no such host` for NOM gRPC hostname | DNS A record missing or not yet propagated | Add DNS A record for the NOM gRPC hostname and verify from inside the cluster (Step 1 pre-flight) |
| Agent log: `connection refused` on port 9090 | `CONFIG_SERVER_ADDRESS` resolves to an IP the pod cannot reach (hairpin NAT) | Use an external DNS name that resolves to a different IP than the pod's own LB |
| Agent log: `secret not found` on startup | Kubernetes secret created on wrong cluster | Secrets must exist on the agent's cluster — re-run Step 3 targeting the correct `--context` |
| Agent log: `server_id not found` | Agent cannot read Neo4j data directory | Verify data volumeMount has `subPath: data` |
| Agent connects but Neo4j instance shows offline in NOM | Bolt URI unreachable or wrong credentials | Verify `CONFIG_INSTANCE_1_BOLT_URI`, username, and password; check `server.bolt.tls_level` |
| Agent gRPC disconnects after ~2 hours | JWT token expired | Normal — agent auto-renews; if repeated, check NOM server logs for auth errors |

---

## Reference Values (neo4jfield.org Example)

| Setting | Example Value |
|---------|--------------|
| `CONFIG_SERVER_ADDRESS` | `nom-grpc.neo4jfield.org:9090` |
| `CONFIG_TOKEN_URL` | `https://nom.neo4jfield.org/api/login/agent` |
| `CONFIG_TLS_TRUSTED_CERTS` | `/ssl/public.crt` (LE wildcard cert `*.neo4jfield.org`) |
| Neo4j namespace | `neo4j` |
| NOM namespace | `nom` |

**East cluster** (`gs-k8s-neo4j-clstr`) — Neo4j SSL configured:

| Setting | Value |
|---------|-------|
| AKS context | `gs-k8s-neo4j-clstr` |
| SSL secret name | `neo4j-ssl-cert` (has `public.crt` + `private.key`) |
| Volume name in values file | `ssl-cert` |
| `additionalVolumeMounts` | mounts both `public.crt` and `private.key` |
| Agent secret names | `nom-agent-server1`, `nom-agent-server2` |

**Central cluster** (`gs-k8s-neo4j-clstr-c`) — No Neo4j SSL:

| Setting | Value |
|---------|-------|
| AKS context | `gs-k8s-neo4j-clstr-c` |
| SSL secret name | `nom-tls-cert` (`public.crt` only — no private key) |
| Volume name in values file | `nom-tls-cert` |
| `additionalVolumeMounts` | `[]` (empty — no SSL on Neo4j) |
| Agent secret names | `nom-agent-server1` (reused same registration) |
