# Deployment files

The generated repository contains specific files for integration with Kubernetes. All these files are located in the /`deploy` folder.


## Deploying ${{ values.name }}

This guide explains how to deploy the **${{ values.name }}** capability
to a Kubernetes cluster using ArgoCD and the Naftiko Skipper operator.

---

## Prerequisites

- A Kubernetes cluster with the Naftiko [Skipper Operator](https://shipyard.naftiko.io/fleet/1.0.0-alpha3/skipper/) installed and managing fleet-wide orchestration, compliance, and FinOps policies.
- [ArgoCD](https://argo-cd.readthedocs.io/en/stable/operator-manual/installation/) installed and connected to your cluster
- A GitHub Personal Access Token (PAT) with `Contents: Read-only` permission on this repo

---

## Folder structure

deploy/
├── configmap.yaml                                 ← ikanos spec wrapped for Skipper
├── ${{ values.name }}-import-{namespace}.yaml     ← one per consumes entry (if any)
└── ${{ values.name }}.yaml                        ← Capability CR (specRef pattern)

Skipper reads the Capability CR, resolves the ConfigMaps, and reconciles:
- a **Deployment** running the ikanos engine
- a **Service** exposing all declared ports
- an **Ingress** if any expose carries the `public` tag

---

## Step 1 — Register the repo in ArgoCD

If your repo is **private**, create a credentials secret first:

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: ${{ values.name }}-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
type: Opaque
stringData:
  type: git
  url: https://github.com/YOUR_GITHUB_USERNAME/${{ values.repoName }}
  username: <YOUR_GITHUB_USERNAME>
  password: <YOUR_GITHUB_PAT>
EOF
```
> Generate a PAT at GitHub → Settings → Developer settings →
> Personal access tokens → Fine-grained tokens → Contents: Read-only

---

## Step 2 — Apply the ArgoCD Application

```bash
kubectl apply -f - <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cap-${{ values.name }}
  namespace: argocd
  labels:
    app.kubernetes.io/part-of: naftiko
    naftiko.io/capability: ${{ values.name }}
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR_GITHUB_USERNAME/${{ values.repoName }}
    targetRevision: HEAD
    path: deploy
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - ApplyOutOfSyncOnly=true
      - CreateNamespace=false
EOF
```
ArgoCD will automatically sync the `deploy/` folder and apply all manifests.

---

## Step 3 — Verify the deployment

```bash
# Check ArgoCD sync status
kubectl get application cap-${{ values.name }} -n argocd

# Check capability phase
kubectl get capability ${{ values.name }} -n default

# Wait for the pod to be ready
kubectl wait pod \
  -l naftiko.io/capability=${{ values.name }} \
  --for=condition=Ready --timeout=60s -n default

# Get the endpoint
kubectl get capability ${{ values.name }} -n default \
  -o jsonpath='{.status.endpoint}'
```

---

## Step 4 — Test the endpoint

```bash
kubectl port-forward svc/${{ values.name }} 3001:3001 -n default &
```

Then:
- **REST** — open Bruno and use `http://localhost:3001/` as base URL
- **MCP** — open MCP Inspector, configure `http://localhost:3001/` with Streamable HTTP

---

## Troubleshooting

### Capability phase is `Failed`

```bash
kubectl describe capability ${{ values.name }} -n default
```

### Force a reconciliation

```bash
kubectl annotate capability ${{ values.name }} \
  reconcile-at=$(date +%s) --overwrite -n default
```

### Check Skipper operator logs

```bash
kubectl logs deployment/naftiko-skipper -n naftiko-system --tail=50
```
