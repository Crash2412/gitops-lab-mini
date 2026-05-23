# GitOps Lab Mini

A small GitOps lab for running a Kubernetes development stack with Argo CD.

This repository uses the app-of-apps pattern: one root Argo CD `Application` points to the development cluster folder, and that folder defines the apps Argo CD should keep in sync.

Repository: `https://github.com/Crash2412/gitops-lab-mini.git`

## What It Deploys

| Component | Path | Namespace | Purpose |
| --- | --- | --- | --- |
| root-app | `clusters/development/root-app.yaml` | `argocd` | Bootstraps all development apps |
| demo-nginx | `apps/demo-nginx` | `default` | Simple nginx demo app exposed as `demo.local` |
| ingress-nginx | `charts/ingress-nginx` | `ingress-nginx` | Ingress controller with metrics enabled |
| cert-manager | `charts/cert-manager` | `cert-manager` | Certificate management with CRDs installed |
| metrics-server | `charts/metrics-server` | `kube-system` | Kubernetes resource metrics |
| prometheus | `charts/prometheus` | `monitoring` | kube-prometheus-stack with persistent Prometheus storage |
| loki | `charts/loki` | `logging` | Single-binary Loki with filesystem persistence |
| promtail | `charts/promtail` | `logging` | Sends cluster logs to Loki |

## Repository Layout

```text
apps/
  demo-nginx/          # Raw Kubernetes manifests for the demo app
charts/
  cert-manager/        # Helm wrapper chart
  ingress-nginx/       # Helm wrapper chart
  loki/                # Helm wrapper chart
  metrics-server/      # Helm wrapper chart
  prometheus/          # Helm wrapper chart
  promtail/            # Helm wrapper chart
clusters/
  development/         # Argo CD Applications for the development cluster
```

Each wrapper chart contains a `Chart.yaml`, `Chart.lock`, a vendored chart archive under `charts/`, and environment-specific values in `values/development.yaml`.

## Prerequisites

Install these tools on Windows 11 or Ubuntu:

- Git
- kubectl
- Helm
- A local Kubernetes cluster, such as Docker Desktop Kubernetes, kind, Minikube, or k3d
- Argo CD installed in the cluster

The manifests assume the standard in-cluster Kubernetes API address:

```text
https://kubernetes.default.svc
```

On Windows, PowerShell commands are shown with the `powershell` code fence. On Ubuntu/Linux, use the `bash` commands.

## Bootstrap

Clone the repository:

```powershell
git clone https://github.com/Crash2412/gitops-lab-mini.git
cd gitops-lab-mini
```

```bash
git clone https://github.com/Crash2412/gitops-lab-mini.git
cd gitops-lab-mini
```

Create the Argo CD namespace if it does not already exist:

```powershell
kubectl create namespace argocd
```

```bash
kubectl create namespace argocd
```

Install Argo CD if it is not already installed:

```powershell
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait for Argo CD to become ready:

```powershell
kubectl wait --for=condition=available --timeout=300s deployment/argocd-server -n argocd
```

```bash
kubectl wait --for=condition=available --timeout=300s deployment/argocd-server -n argocd
```

Apply the root application:

```powershell
kubectl apply -f clusters/development/root-app.yaml
```

```bash
kubectl apply -f clusters/development/root-app.yaml
```

Argo CD will then discover and sync the applications defined in `clusters/development`.

## Access Argo CD

Port-forward the Argo CD API server:

```powershell
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open:

```text
https://localhost:8080
```

Get the initial admin password:

```powershell
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | ForEach-Object { [Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($_)) }
```

```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d
echo
```

Username:

```text
admin
```

## Access The Demo App

The demo app uses this host:

```text
demo.local
```

The ingress controller service is configured as `NodePort`. Get its ports:

```powershell
kubectl get svc -n ingress-nginx
```

```bash
kubectl get svc -n ingress-nginx
```

For a local Windows setup, add a hosts entry if needed:

```text
127.0.0.1 demo.local
```

The Windows hosts file is usually:

```text
C:\Windows\System32\drivers\etc\hosts
```

For Ubuntu/Linux, add the same hosts entry to `/etc/hosts`:

```bash
sudo sh -c 'echo "127.0.0.1 demo.local" >> /etc/hosts'
```

Then open `https://demo.local:<node-port>` or use the access method required by your local Kubernetes provider.

The demo certificate is self-signed, so the browser may show a certificate warning.

## Useful Checks

Check Argo CD applications:

```powershell
kubectl get applications -n argocd
```

```bash
kubectl get applications -n argocd
```

Check deployed workloads:

```powershell
kubectl get pods -A
```

```bash
kubectl get pods -A
```

Check the demo app:

```powershell
kubectl get deploy,svc,ingress,certificate -n default
```

```bash
kubectl get deploy,svc,ingress,certificate -n default
```

Check observability namespaces:

```powershell
kubectl get pods -n monitoring
kubectl get pods -n logging
```

```bash
kubectl get pods -n monitoring
kubectl get pods -n logging
```

Validate the Helm wrapper charts locally:

```powershell
helm lint charts/cert-manager
helm lint charts/ingress-nginx
helm lint charts/loki
helm lint charts/metrics-server
helm lint charts/prometheus
helm lint charts/promtail
```

```bash
helm lint charts/cert-manager
helm lint charts/ingress-nginx
helm lint charts/loki
helm lint charts/metrics-server
helm lint charts/prometheus
helm lint charts/promtail
```

## Updating The Git Repository URL

Argo CD syncs from the `repoURL` values in `clusters/development/*.yaml`.

If the GitHub repository changes, update every `repoURL` in that folder and update the local remote:

```powershell
git remote set-url origin https://github.com/<owner>/<repo>.git
```

```bash
git remote set-url origin https://github.com/<owner>/<repo>.git
```

Then commit and push the manifest changes.

## Notes

- This repository is configured for the `main` branch.
- The lab is intended for a local development cluster, not production.
- Loki and Prometheus request persistent storage with the `standard` storage class. If your cluster uses a different default storage class, update the relevant values files under `charts/loki/values/` and `charts/prometheus/values/`.
- `demo.local`, `demo-nginx`, and the Kubernetes resource names are generic lab names.
