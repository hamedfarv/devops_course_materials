# Helm Multi-Environment Demo — Step-by-Step

A complete walkthrough of deploying the same chart to **dev**, **staging**, and **prod** on a local kind cluster.
Covers: custom values, upgrade, rollback, ingress, HPA, nodeSelector, and port-forward.

---

## Prerequisites

- kind cluster running (`kind-cluster.yaml` in this repo)
- `helm`, `kubectl`, `curl` installed
- Working directory: root of this repo (`devops_course/`)

---

## Cluster Overview

```
kind cluster: my-cluster
├── my-cluster-control-plane   (port 80 → host 80, ingress-ready=true)
├── my-cluster-worker          (labeled env=prod  ← we add this)
└── my-cluster-worker2
```

Port mappings relevant to this demo:

| Host port | Node port | Used for |
|---|---|---|
| `localhost:30081` | 30081 on control-plane | dev NodePort |
| `localhost:8082` | port-forward | staging ClusterIP |
| `localhost:8083` | port-forward to ingress-nginx | prod Ingress |

---

## Step 1 — Verify the cluster is running

```bash
kubectl get nodes
```

Expected output:
```
NAME                       STATUS   ROLES           AGE   VERSION
my-cluster-control-plane   Ready    control-plane   ...   v1.32.2
my-cluster-worker          Ready    <none>          ...   v1.32.2
my-cluster-worker2         Ready    <none>          ...   v1.32.2
```

---

## Step 2 — Install the nginx Ingress Controller

The prod environment uses Ingress. kind ships without one, so we install it.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

Wait for it to be ready:

```bash
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
```

Verify:
```bash
kubectl get pods -n ingress-nginx
```

Expected:
```
NAME                                        READY   STATUS    RESTARTS
ingress-nginx-controller-xxxx               1/1     Running   0
```

> **Why?** The ingress controller is the component that watches `Ingress` resources
> and routes HTTP traffic to the correct backend service.

---

## Step 3 — Label a node for prod

The prod values file has `nodeSelector: { env: prod }`, which pins pods to nodes
with that label. We label one worker node:

```bash
kubectl label node my-cluster-worker env=prod
```

Verify the label was applied:
```bash
kubectl get nodes --show-labels | grep env=prod
```

> **Why?** In production you typically have dedicated nodes (larger instances,
> specific hardware, security policies). The nodeSelector ensures prod pods only
> run on those nodes.

---

## Step 4 — Preview what each environment will deploy

Before touching the cluster, render templates locally to see what Helm will generate:

```bash
# Dev
helm template myapp-dev ./test-helm-template -f helm/envs/values-dev.yaml

# Staging
helm template myapp-staging ./test-helm-template -f helm/envs/values-staging.yaml

# Prod
helm template myapp-prod ./test-helm-template -f helm/envs/values-prod.yaml
```

To diff staging vs prod:
```bash
diff \
  <(helm template myapp ./test-helm-template -f helm/envs/values-staging.yaml) \
  <(helm template myapp ./test-helm-template -f helm/envs/values-prod.yaml)
```

> **Why `helm template`?** It renders YAML locally without connecting to the cluster.
> Use this to review changes before applying them. It's your first line of defence
> against configuration mistakes.

---

## Step 5 — Deploy DEV

```bash
helm upgrade --install myapp-dev ./test-helm-template \
  -f helm/envs/values-dev.yaml \
  -n dev --create-namespace
```

**What this does:**
- `upgrade --install` — installs if not present, upgrades if already deployed (idempotent)
- `-f helm/envs/values-dev.yaml` — applies dev overrides on top of `values.yaml`
- `-n dev --create-namespace` — deploys into the `dev` namespace, creating it if needed

Verify:
```bash
kubectl get pods,svc -n dev
```

Expected:
```
NAME                                              READY   STATUS    RESTARTS
pod/myapp-dev-test-helm-template-xxxx             1/1     Running   0

NAME                                 TYPE       CLUSTER-IP   PORT(S)
service/myapp-dev-test-helm-template NodePort   10.96.x.x    80:30081/TCP
```

**Test access** (NodePort is mapped directly to host):
```bash
curl http://localhost:30081
# Should return nginx welcome page HTML
```

Or open `http://localhost:30081` in a browser.

---

## Step 6 — Deploy STAGING

```bash
helm upgrade --install myapp-staging ./test-helm-template \
  -f helm/envs/values-staging.yaml \
  -n staging --create-namespace
```

Verify:
```bash
kubectl get pods,svc -n staging
```

Expected: **2 pods** running (staging has `replicaCount: 2`), service type `ClusterIP`.

**Test access via port-forward** (ClusterIP is only reachable inside the cluster):
```bash
kubectl port-forward svc/myapp-staging-test-helm-template 8082:80 -n staging
```

In a separate terminal (or background the above with `&`):
```bash
curl http://localhost:8082
```

> **Why ClusterIP in staging?** It forces traffic through a proper Ingress in a
> real cluster. For local testing, `port-forward` tunnels directly to the service.

---

## Step 7 — Deploy PROD

```bash
helm upgrade --install myapp-prod ./test-helm-template \
  -f helm/envs/values-prod.yaml \
  -n prod --create-namespace
```

Verify:
```bash
kubectl get pods,svc,ingress,hpa -n prod
```

Expected:
```
NAME                                           READY   STATUS
pod/myapp-prod-test-helm-template-xxxx         1/1     Running
pod/myapp-prod-test-helm-template-xxxx         1/1     Running
pod/myapp-prod-test-helm-template-xxxx         1/1     Running

NAME                                 TYPE        PORT(S)
service/myapp-prod-test-helm-template ClusterIP  80/TCP

NAME                                            CLASS   HOSTS
ingress/myapp-prod-test-helm-template           nginx   myapp-prod.local

NAME                                        MINPODS   MAXPODS   REPLICAS
hpa/myapp-prod-test-helm-template           3         6         3
```

Check that all 3 prod pods landed on the labeled node:
```bash
kubectl get pods -n prod -o wide
# All pods should show NODE = my-cluster-worker
```

**Test access via Ingress** (port-forward to nginx ingress controller):
```bash
kubectl port-forward svc/ingress-nginx-controller 8083:80 -n ingress-nginx
```

```bash
curl -H "Host: myapp-prod.local" http://localhost:8083
```

Or add to `/etc/hosts` and open in browser:
```bash
echo "127.0.0.1 myapp-prod.local" | sudo tee -a /etc/hosts
curl http://myapp-prod.local:8083
```

---

## Step 8 — View all running Helm releases

```bash
helm list -A
```

Expected:
```
NAME          NAMESPACE  REVISION  STATUS    CHART
my-wordpress  default    1         deployed  wordpress-31.0.2
myapp-dev     dev        1         deployed  test-helm-template-0.1.0
myapp-staging staging    1         deployed  test-helm-template-0.1.0
myapp-prod    prod       1         deployed  test-helm-template-0.1.0
```

---

## Step 9 — Inspect what values are active

```bash
# Only your overrides (what you passed via -f)
helm get values myapp-dev -n dev

# Full merged result: your overrides + all chart defaults
helm get values myapp-dev -n dev --all

# See the actual Kubernetes manifests currently deployed
helm get manifest myapp-dev -n dev
```

---

## Step 10 — Upgrade: deploy a new image tag

Scenario: the dev team cut a new image. Update dev to use it.

```bash
helm upgrade myapp-dev ./test-helm-template \
  -f helm/envs/values-dev.yaml \
  --set image.tag=1.26-alpine \
  -n dev
```

`--set` overrides a single value inline. It takes the **highest precedence** —
it wins over anything in the `-f` file.

Verify the pod restarted with the new image:
```bash
kubectl get pods -n dev
kubectl describe pod -n dev -l app.kubernetes.io/instance=myapp-dev | grep Image:
```

---

## Step 11 — View release history

```bash
helm history myapp-dev -n dev
```

Output:
```
REVISION  UPDATED    STATUS      CHART                    DESCRIPTION
1         12:48:47   superseded  test-helm-template-0.1.0  Install complete
2         12:50:05   deployed    test-helm-template-0.1.0  Upgrade complete
```

> Every `helm upgrade` creates a new revision. The previous revision is kept as
> `superseded` and can be restored at any time.

---

## Step 12 — Rollback to a previous revision

The new image tag caused issues. Rollback to revision 1:

```bash
helm rollback myapp-dev 1 -n dev
```

Check history again:
```bash
helm history myapp-dev -n dev
```

Output:
```
REVISION  STATUS      DESCRIPTION
1         superseded  Install complete
2         superseded  Upgrade complete
3         deployed    Rollback to 1
```

> Rollback creates a **new revision** (3) that matches the config of revision 1.
> Nothing is lost — you can still re-upgrade to revision 2 if needed.

---

## Step 13 — Understand the difference between environments

| Feature | Dev | Staging | Prod |
|---|---|---|---|
| `replicaCount` | 1 | 2 | — (managed by HPA) |
| `image.tag` | `1.25-alpine` | `1.25-alpine` | `1.25.3` (exact) |
| `service.type` | `NodePort` | `ClusterIP` | `ClusterIP` |
| `ingress.enabled` | false | false | true |
| `autoscaling.enabled` | false | false | true (3–6 pods) |
| `nodeSelector` | none | none | `env: prod` |
| `affinity` | none | none | pod anti-affinity |
| Access method | `localhost:30081` | `port-forward` | Ingress host header |

---

## Step 14 — Cleanup

```bash
helm uninstall myapp-dev -n dev
helm uninstall myapp-staging -n staging
helm uninstall myapp-prod -n prod

kubectl delete namespace dev staging prod
kubectl delete -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
kubectl label node my-cluster-worker env-   # remove the label
```

Or destroy the entire cluster:
```bash
kind delete cluster --name my-cluster
```

---

## Quick Command Cheatsheet

```bash
# Render templates locally (no cluster)
helm template <release> <chart> -f <values>

# Install or upgrade (idempotent)
helm upgrade --install <release> <chart> -f <values> -n <ns> --create-namespace

# Check what's running
helm list -A
helm status <release> -n <ns>

# Inspect values
helm get values <release> -n <ns>           # your overrides only
helm get values <release> -n <ns> --all     # full merged values

# Revision history
helm history <release> -n <ns>

# Rollback
helm rollback <release> <revision> -n <ns>

# Uninstall
helm uninstall <release> -n <ns>

# Lint before deploying
helm lint <chart>

# Validate against cluster
helm install <release> <chart> --dry-run
```
