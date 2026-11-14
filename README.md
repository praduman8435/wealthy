# Secure service-to-service communication with SPIFFE/SPIRE

**Assignment**
1. Install Kubernetes locally (kind) and run 3 services.
2. Use SPIFFE/SPIRE to secure the communication between them.
3. Run one service (**shipping**) **outside the cluster**, managed by the same SPIRE setup, and let **payments** call it.
4. Enable **route-level checks**.

**What's used** (all public, nothing custom-built):

| Piece | What |
|---|---|
| Kubernetes | `kind`, a local cluster running inside Docker |
| SPIRE | **Official SPIFFE Helm chart**, downloaded into `charts/`, with my settings in `values.yaml` |
| Apps | Public image **podinfo** (`ghcr.io/stefanprodan/podinfo`), a lightweight web app with a UI. Each service has its own colour. |
| mTLS sidecar | Public image **ghostunnel** (`ghostunnel/ghostunnel`), a TLS proxy that gets its certificate from SPIRE |
| Route-level checks | Public image **Envoy** (`envoyproxy/envoy`), which gets certificates from SPIRE (SDS) and checks URL paths (RBAC) |
| Outside service | **Docker Compose** with its own SPIRE agent that joins the same SPIRE server |

## Architecture

```
 ┌──────────────────────── kind cluster (Kubernetes) ────────────────────────┐
 │ namespace: spire     SPIRE Server (CA)  +  SPIRE Agent  +  CSI driver     │
 │                            ▲                                              │
 │ namespace: demo            │ certificates                                 │
 │                                                                           │
 │  frontend ──mTLS──► orders ──mTLS──► payments ──mTLS──┐                   │
 │     │              (Envoy:                            │                   │
 │     │               route checks)                     │                   │
 │     └──────── ✗ BLOCKED ─────────► payments           │                   │
 └───────────────────────────────────────────────────────┼───────────────────┘
          ▲ NodePort 30081                               │
          │ (agent joins with a join token)              ▼
 ┌────────┴──────── OUTSIDE the cluster (Docker Compose) ──────────────────┐
 │  SPIRE Agent (same SPIRE server)     shipping ◄── ghostunnel (mTLS)     │
 └─────────────────────────────────────────────────────────────────────────┘
```

| Service | Where | Identity (SPIFFE ID) | Who may call it |
|---|---|---|---|
| frontend | k8s | `spiffe://wealthy.local/ns/demo/sa/frontend` | (entry point) |
| orders | k8s | `spiffe://wealthy.local/ns/demo/sa/orders` | frontend only, **only on `/`, `/api/*`, `/echo`** |
| payments | k8s | `spiffe://wealthy.local/ns/demo/sa/payments` | orders only |
| shipping | **outside k8s** (Docker) | `spiffe://wealthy.local/vm/shipping` | payments only |

## Files

```
wealthy/
├── README.md                       this guide
├── values.yaml                     my settings for the SPIRE Helm chart
├── services.yaml                   frontend, orders (with Envoy), payments
├── charts/                         official SPIRE Helm charts (downloaded with "helm pull")
└── shipping/                       the service OUTSIDE the cluster
    ├── docker-compose.yaml         3 containers: SPIRE agent + shipping app + ghostunnel
    └── expose-spire-server.yaml    opens the SPIRE server (port 30081) for the outside agent
```

---

# Step-by-step guide (macOS)

## Step 0: Install the tools (one time)

```bash
brew install kind kubectl helm colima docker docker-compose
colima start --cpu 4 --memory 6
docker ps
```
`colima start` runs Docker (Docker Desktop also works). `docker ps` should run without error.

## Step 1: Create the Kubernetes cluster

```bash
cd /Users/user/wealthy
kind create cluster --name spire-demo
kubectl config use-context kind-spire-demo
kubectl get nodes
```
The node `spire-demo-control-plane` should be `Ready`.

> ⚠️ Your laptop also has EKS contexts. Check `kubectl config current-context` shows `kind-spire-demo` before continuing.

## Step 2: Install SPIRE with the official Helm chart

**2a. The charts** are already in `charts/`. This is how they were downloaded:
```bash
helm repo add spiffe https://spiffe.github.io/helm-charts-hardened/
helm repo update
helm pull spiffe/spire-crds --version 0.6.1  --untar --untardir charts
helm pull spiffe/spire      --version 0.30.4 --untar --untardir charts
```

**2b. My settings are in `values.yaml`:**

| Setting | Meaning |
|---|---|
| `trustDomain: wealthy.local` | Every identity starts with `spiffe://wealthy.local/` |
| `clusterName: demo-cluster` | Name the in-cluster agent uses to prove which cluster it's in |
| `spiffe-oidc-discovery-provider.enabled: false` | Turns off an optional extra we don't need |
| `nodeAttestor.joinToken.enabled: true` | Lets an agent **outside Kubernetes** join with a one-time token (for shipping) |
| `clusterStaticEntries.shipping` | Declares shipping's identity: *a container labelled `app=shipping`, on the agent `vm/shipping-host`, gets `spiffe://wealthy.local/vm/shipping`* |

**2c. Install:**
```bash
helm install spire-crds ./charts/spire-crds -n spire --create-namespace
helm install spire ./charts/spire -n spire -f values.yaml
```

Wait until everything is Running, then press `Ctrl+C`:
```bash
kubectl -n spire get pods -w
```
```
NAME                            READY   STATUS    RESTARTS   AGE
spire-agent-lmg2q               1/1     Running   0          2m
spire-server-0                  2/2     Running   0          2m
spire-spiffe-csi-driver-cjfj9   2/2     Running   0          2m
```

| Pod | What it does |
|---|---|
| `spire-server-0` | **spire-server**: the CA that signs certificates. **controller-manager**: registers pods automatically (ClusterSPIFFEID) and creates the shipping entry (ClusterStaticEntry). |
| `spire-agent-*` | One per node. Pods ask it "who am I?", and it checks the pod and returns its certificate. |
| `spire-spiffe-csi-driver-*` | Mounts the agent's socket into pods as a volume (`csi.spiffe.io`) |

## Step 3: Deploy the 3 services in Kubernetes

Apply the file, then wait until all pods are Running and press `Ctrl+C`:
```bash
kubectl apply -f services.yaml
kubectl -n demo get pods -w
```
```
NAME                        READY   STATUS    RESTARTS   AGE
frontend-79cf9f744f-dtmpz   3/3     Running   0          40s
orders-5b4995856f-9jw7r     3/3     Running   0          40s
payments-64cd679cd5-dqb44   3/3     Running   0          40s
```

What's in each pod:

| Pod | Containers |
|---|---|
| frontend | `app` (podinfo), `to-orders` (localhost:9000 → orders), `to-payments` (localhost:9001 → payments, **should fail**) |
| orders | `app`, **`mtls-in` = Envoy** (mTLS + **route-level checks**), `to-payments` (localhost:9000 → payments) |
| payments | `app`, `mtls-in` = ghostunnel (allows **orders**), `to-shipping` (localhost:9000 → **shipping outside the cluster**) |

SPIRE registers them by itself:
```bash
kubectl exec -n spire spire-server-0 -c spire-server -- spire-server entry show
```

## Step 4: Run shipping OUTSIDE the cluster

### The idea, in simple words

Inside Kubernetes every node runs a **SPIRE agent**. Pods ask it "who am I?" and it gives them a certificate.

Shipping runs **outside Kubernetes** (in plain Docker, like a separate server), so there is no SPIRE agent there. **So we start one ourselves** with Docker Compose. It connects to the **same SPIRE server** in Kubernetes.

Think of it like this:
- **SPIRE server** = the passport office (in Kubernetes)
- **SPIRE agent** = a branch office on the shipping machine
- **Join token** = a one-time password, so the server knows this branch is genuine
- **Docker label `app=shipping`** = how the branch recognises the shipping container and gives it the passport `spiffe://wealthy.local/vm/shipping`

`docker-compose.yaml` has just 3 containers:

| Container | What it does |
|---|---|
| `spire-agent` | Joins the SPIRE server with the join token and gives certificates on this machine |
| `shipping` | The app (same podinfo image as the others, orange colour) |
| `shipping-mtls` | ghostunnel in front of the app: mTLS on port 8443, **only payments may call** |

The rule "a container labelled `app=shipping` gets `spiffe://wealthy.local/vm/shipping`" is already in `values.yaml` (`clusterStaticEntries.shipping`), so it's managed by the same Helm setup.

### Commands

**4a. Open the SPIRE server for the outside agent** (port 30081):
```bash
kubectl apply -f shipping/expose-spire-server.yaml
```

**4b. Create a one-time join token:**
```bash
kubectl exec -n spire spire-server-0 -c spire-server -- spire-server token generate -spiffeID spiffe://wealthy.local/vm/shipping-host
```
```
Token: cc1c1aa6-6f4e-4fe7-be0b-ed4697866c71
```

**4c. Start shipping with that token.** Replace `<TOKEN>`:
```bash
cd shipping
JOIN_TOKEN=<TOKEN> docker compose up -d
docker compose ps
cd ..
```
All 3 containers should be `Up`.

**4d. Check that it joined:**
```bash
kubectl exec -n spire spire-server-0 -c spire-server -- spire-server agent list
```
You should see 2 agents: `k8s_psat` (the Kubernetes node) and `join_token` (the shipping machine).

## Step 5: Route-level checks (how they're enabled)

ghostunnel only checks **who** is calling. For **which URL paths** a caller may use, orders uses **Envoy** as its sidecar. The rule is in `services.yaml` (ConfigMap `orders-envoy`):

```yaml
- name: envoy.filters.http.rbac
  typed_config:
    rules:
      action: ALLOW
      policies:
        frontend-allowed-routes:
          principals:
            - authenticated:
                principal_name: { exact: "spiffe://wealthy.local/ns/demo/sa/frontend" }
          permissions:
            - url_path: { path: { exact: "/" } }
            - url_path: { path: { prefix: "/api/" } }
            - url_path: { path: { exact: "/echo" } }
```

Envoy works in 3 steps:
1. It gets orders' certificate and the trust bundle from the SPIRE agent over **SDS**.
2. It requires a client certificate (**mTLS**).
3. The **RBAC filter** checks **who** is calling (SPIFFE ID **=** frontend) **and which route** (`/`, `/api/*` or `/echo`). Anything else, such as `/env` (which shows environment variables) or `/headers`, gets **`403 RBAC: access denied`**.

The same idea in a service mesh is an Istio `AuthorizationPolicy` with `principals` + `paths` + `methods`.

## Step 6: Test it

| # | Test | Command | Expected |
|---|---|---|---|
| 1 | frontend → orders ✅ | `kubectl -n demo exec deploy/frontend -c to-orders -- curl -s localhost:9000/api/info` | `"hostname": "orders-..."` |
| 2 | **Route check**: frontend → orders `/env` ❌ | `kubectl -n demo exec deploy/frontend -c to-orders -- curl -s localhost:9000/env` | `RBAC: access denied` (HTTP 403) |
| 3 | No certificate → orders ❌ | `kubectl -n demo exec deploy/frontend -c to-orders -- curl -sk https://orders:8443/` | `exit code 55` (TLS rejected) |
| 4 | orders → payments ✅ | `kubectl -n demo exec deploy/orders -c to-payments -- curl -s localhost:9000/api/info` | `"hostname": "payments-..."` |
| 5 | frontend → payments ❌ | `kubectl -n demo exec deploy/frontend -c to-payments -- curl -s localhost:9001` | `exit code 52`. payments log: `unauthorized: invalid principal` |
| 6 | **payments → shipping (outside k8s)** ✅ | `kubectl -n demo exec deploy/payments -c to-shipping -- curl -s localhost:9000/api/info` | `"hostname": "shipping-outside-k8s"` |
| 7 | No certificate → shipping ❌ | `kubectl -n demo exec deploy/frontend -c to-orders -- curl -sk https://shipping:8443/` | `exit code 56`. shipping log: `client didn't provide a certificate` |
| 8 | **Full chain** frontend → orders → payments → shipping ✅ | `kubectl -n demo exec deploy/frontend -c to-orders -- curl -s -X POST localhost:9898/echo -d 'hello'` | `hello` nested 3 levels (one per hop) |

Logs that show **why** a call was blocked:
```bash
kubectl -n demo logs deploy/payments -c mtls-in | grep unauthorized
docker compose -f shipping/docker-compose.yaml logs shipping-mtls | grep handshake
```

**What this proves**
1. All traffic between services is **encrypted** and **both sides prove who they are** (mTLS).
2. **Access is decided by identity**, not by IP address.
3. **Route-level**: even an allowed caller can only use the allowed URL paths.
4. **The same SPIRE server** gives identities **inside and outside** Kubernetes.
5. **No secrets** are stored anywhere. Certificates are short-lived and auto-rotated.

## Step 7: See it in your browser

You need **3 terminal tabs**, all left running (click **+** in the terminal panel).

**Tab 1** forwards frontend's app and frontend's tunnels:
```bash
kubectl -n demo port-forward deploy/frontend 8080:9898 9000:9000 9001:9001
```

**Tab 2** forwards orders' tunnel to payments:
```bash
kubectl -n demo port-forward deploy/orders 9002:9000
```

**Tab 3** forwards payments' tunnel to shipping:
```bash
kubectl -n demo port-forward deploy/payments 9003:9000
```

| URL | Path | What you see |
|---|---|---|
| http://localhost:8080 | frontend | 🔵 Blue **FRONTEND** page |
| http://localhost:9000 | frontend → orders | 🟢 Green **ORDERS** page ✅ |
| http://localhost:9000/env | frontend → orders `/env` | ❌ `RBAC: access denied` (route-level check) |
| http://localhost:9002 | orders → payments | 🟣 Purple **PAYMENTS** page ✅ |
| http://localhost:9003 | payments → shipping (outside k8s) | 🟠 Orange **SHIPPING** page, "Served by shipping-outside-k8s" ✅ |
| http://localhost:9001 | frontend → payments | ❌ Error page (payments rejects frontend) |

## Step 8: Clean up

```bash
docker compose -f shipping/docker-compose.yaml down
kind delete cluster --name spire-demo
```

---

## Troubleshooting

| Problem | Fix |
|---|---|
| Commands go to the wrong cluster | `kubectl config use-context kind-spire-demo` |
| SPIRE pods `Pending` / slow | Give Docker more resources: `colima stop && colima start --cpu 4 --memory 6` |
| A test fails right after deploy | Wait about 10 seconds for the first certificates |
| Shipping agent logs `join token does not exist or has already been used` | A join token works **once**. Run `docker compose -f shipping/docker-compose.yaml down`, create a new token (4b), and start again (4c). |
| Test 6 fails | Check `docker compose -f shipping/docker-compose.yaml ps` (all 3 Up) and `... logs shipping-mtls` |
