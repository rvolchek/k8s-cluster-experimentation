# Bootstrap

Manual, one-time steps to bring up a fresh cluster. After this, Argo CD manages
everything else from Git. Run all commands from a machine whose kubeconfig points
at the target cluster (currently the MacBook, via the Tailscale operator proxy).

Prereqs already in place on the current cluster:
- Tailscale Kubernetes operator installed (gives kubectl access + the `tailscale`
  IngressClass used to expose the Argo UI).
- MagicDNS + HTTPS Certificates enabled on the tailnet (required for the Argo UI
  Ingress to get a Let's Encrypt cert).

## 1. Install Argo CD

```bash
helm repo add argo https://argoproj.github.io/argo-helm && helm repo update
helm upgrade --install argocd argo/argo-cd \
  -n argocd --create-namespace \
  -f bootstrap/argocd/values.yaml \
  --wait
```

## 2. Expose the Argo CD UI over Tailscale

```bash
kubectl apply -f bootstrap/argocd/ingress.yaml
```

UI URL: https://argocd.<your-tailnet>.ts.net  (first load may lag while the
Let's Encrypt cert is provisioned).

Get the initial admin password (username `admin`):

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

## 3. Apply the app-of-apps root

Requires this repo to be pushed to GitHub first (Argo pulls from the public repo).

```bash
kubectl apply -f clusters/docker-desktop/root.yaml
```

Then in the Argo UI, Sync the `root` app to register the platform Applications,
and Sync each platform app (e.g. `kuberay-operator`) to deploy it. Sync is manual
by design so you can review diffs first.
