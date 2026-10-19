# ArgoCD – GitOps for the HOA (society) application

This repo is the **single source of truth** for what runs in the `society`
namespace. Argo CD (running inside each cluster) watches this repo and keeps the
cluster in sync with it.

```
dev/
  root-app.yaml          # "App of Apps" parent – applied once by hand
  bootstrap/             # one-time setup files (Argo CD values, AppProject)
  apps/                  # one Argo CD Application per component
  manifests/
    common/              # ConfigMaps, Ingress, (Sealed) Secret
    backend/  frontend/  # app Deployments + Services (image tag set by CI)
    kafka/    zookeeper/ # infra Deployments + Services (manual sync)
prod/                    # (Milestone 2)
```

| Component | Argo CD app | Sync |
|---|---|---|
| ConfigMaps, Ingress, Secret | society-common | auto (after adoption) |
| Backend | society-backend | auto (after adoption) |
| Frontend | society-frontend | auto (after adoption) |
| Kafka | society-kafka | **manual** |
| Zookeeper | society-zookeeper | **manual** |

Postgres runs **outside** the cluster (`192.168.1.22:5433`) and is not managed here.

---

## Dev cluster setup – step by step

All commands run on the control-plane node `kcontroller` unless stated otherwise.

### Step 0 – Back up the current state
```bash
mkdir -p ~/k8s-backup-$(date +%F) && cd ~/k8s-backup-$(date +%F)
for r in deploy svc cm secret ingress; do kubectl -n society get $r -o yaml > society-$r.yaml; done
```
Keep this folder. To restore anything: `kubectl apply -f society-<type>.yaml`.

### Step 1 – Install Argo CD (Helm)
```bash
git clone git@github.com:contactgsmllc/ArgoCD.git ~/ArgoCD && cd ~/ArgoCD   # or copy the files
helm repo add argo https://argoproj.github.io/argo-helm && helm repo update
helm install argocd argo/argo-cd -n argocd --create-namespace -f dev/bootstrap/argocd-values.yaml
kubectl -n argocd get pods -w      # wait until all are Running (Ctrl+C to exit)
```
Get the admin password:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
```
Open the UI (from your laptop, through Tailscale):
```bash
# on kcontroller
kubectl -n argocd port-forward svc/argocd-server 8080:80
# on your laptop (second terminal)
ssh -L 8080:localhost:8080 <user>@<kcontroller-tailscale-ip>
# browse to http://localhost:8080  – user: admin
```
Change the admin password after the first login (User Info → Update Password).

### Step 2 – Give Argo CD read access to this repo (deploy key)
```bash
ssh-keygen -t ed25519 -N "" -C "argocd-dev" -f ~/argocd-dev-deploy
cat ~/argocd-dev-deploy.pub
```
GitHub → this repo → Settings → Deploy keys → **Add deploy key** → paste the
public key, name `argocd-dev`, **leave "Allow write access" unticked**.
```bash
kubectl -n argocd create secret generic repo-argocd-gitops \
  --from-literal=type=git \
  --from-literal=url=git@github.com:contactgsmllc/ArgoCD.git \
  --from-file=sshPrivateKey=$HOME/argocd-dev-deploy
kubectl -n argocd label secret repo-argocd-gitops argocd.argoproj.io/secret-type=repository
rm ~/argocd-dev-deploy ~/argocd-dev-deploy.pub
```
UI → Settings → Repositories should show the repo as **Successful**.

### Step 3 – Create the project and the root app
```bash
kubectl apply -f dev/bootstrap/appproject.yaml
kubectl apply -f dev/root-app.yaml
```
The UI now shows `society-dev-root` plus 5 child apps. The child apps are on
**manual sync**, so **nothing in the cluster has changed yet**.

### Step 4 – Check the diff before syncing (most important step)
For each app: UI → open the app → **App Diff**.
- Expected: no differences (or only harmless defaults).
- If a real difference appears: **fix the YAML in Git**, never the cluster.

When an app's diff is clean, click **Sync** → **Synchronize**. Pods should
**not** restart (same image, same spec). Verify:
```bash
kubectl -n society get pods      # AGE should not reset
```

### Step 5 – Turn on auto-sync for backend, frontend and common
In `dev/apps/society-{backend,frontend,common}.yaml` uncomment the
`automated:` block, commit and push. Kafka and Zookeeper stay manual.

### Step 6 – Sealed Secrets (secrets stored encrypted in Git)
See *Sealed Secrets* below.

### Step 7 – CI pipeline
The `dev` branch workflow in `HOA-Management-SAAS` builds the images and then
updates `newTag` in `dev/manifests/{backend,frontend}/kustomization.yaml`.
Argo CD sees the commit and rolls out the new version. See *How to deploy*.

---

## Sealed Secrets

Install the controller (once per cluster):
```bash
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets && helm repo update
helm install sealed-secrets sealed-secrets/sealed-secrets -n kube-system \
  --set fullnameOverride=sealed-secrets-controller
```
Install the `kubeseal` CLI on kcontroller (arm64):
```bash
KV=0.32.2   # check https://github.com/bitnami-labs/sealed-secrets/releases for latest
curl -sSL https://github.com/bitnami-labs/sealed-secrets/releases/download/v$KV/kubeseal-$KV-linux-arm64.tar.gz | tar xz kubeseal
sudo install -m 755 kubeseal /usr/local/bin/kubeseal && rm kubeseal
```
**Back up the sealing key** (without it, sealed secrets cannot be decrypted if
the cluster is rebuilt). Store it in a password manager – **never in Git**:
```bash
kubectl -n kube-system get secret -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml > sealed-secrets-key-BACKUP.yaml
```
Seal the existing backend secret. The command below copies **only** the name,
type and data – annotations are dropped on purpose because they can contain the
secret values in plain text:
```bash
kubectl -n society get secret society-backend-secret -o json \
| python3 -c 'import json,sys; s=json.load(sys.stdin); print(json.dumps({"apiVersion":"v1","kind":"Secret","type":s["type"],"metadata":{"name":s["metadata"]["name"],"namespace":"society"},"data":s["data"]}))' \
| kubeseal --controller-name sealed-secrets-controller --controller-namespace kube-system -o yaml \
> dev/manifests/common/society-backend-sealedsecret.yaml
```
Let the controller take over the existing Secret, then add the file to Git:
```bash
kubectl -n society annotate secret society-backend-secret sealedsecrets.bitnami.com/managed="true"
# uncomment "- society-backend-sealedsecret.yaml" in dev/manifests/common/kustomization.yaml
git add dev/manifests/common && git commit -m "dev: add sealed backend secret" && git push
```
**To change a secret value later:** create the Secret YAML locally, pipe it
through `kubeseal` exactly as above, commit the new sealed file. Never commit
the plain Secret.

---

## How to deploy (day to day)
1. Merge/push code to the `dev` branch of `HOA-Management-SAAS`.
2. GitHub Actions builds `dearsrini/society-backend:<commit-sha>` (and frontend),
   then commits the new tag to this repo.
3. Argo CD syncs automatically (within ~3 min). Watch it in the UI.

## How to roll back
- **UI:** app → History and Rollback → pick a previous version → Rollback, **or**
- **Git:** revert the "deploy …" commit in this repo and push.

## Troubleshooting
| Symptom | Check |
|---|---|
| App "OutOfSync" | App Diff – someone changed the cluster by hand; Sync to restore Git state |
| App "Degraded" | `kubectl -n society get pods` / `kubectl -n society logs <pod>` |
| Repo "Failed" in Settings | Deploy key missing or removed in GitHub |
| New image not deployed | GitHub Actions run failed? Tag updated in `kustomization.yaml`? |
