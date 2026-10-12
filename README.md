# hello-gitops-infra

Deploy config for [hello-gitops-app](https://github.com/ok2ju/hello-gitops-app). Git is the source of truth: ArgoCD makes the cluster match this repo.

- `helm/hello/` holds the Helm chart. `values.yaml` is shared, and `values-<env>.yaml` holds per-environment overrides, including the image tag that CI writes.
- `argocd/` holds one ArgoCD Application per environment. Each one deploys the chart to its own namespace.
- `root.yaml` is the App-of-Apps root. It makes ArgoCD watch `argocd/`, so adding or deleting a file there adds or removes an environment on push. Apply it once by hand. This mirrors Sygnum's `helm/argocd-projects`, which uses a Helm chart instead of a plain folder.

Do not edit `image.tag` by hand. CI owns it.

## Local cluster

Prerequisites: Docker running, and `brew install kind kubectl`.

```sh
kind create cluster --name gitops

kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd rollout status deploy/argocd-server

kubectl apply -f root.yaml   # the only manual apply; ArgoCD creates everything in argocd/
```

ArgoCD UI at https://localhost:8081, user `admin`:

```sh
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
kubectl -n argocd port-forward svc/argocd-server 8081:443
```

View the app:

```sh
kubectl -n hello-dev  port-forward svc/hello-dev  8080:80   # http://localhost:8080
kubectl -n hello-prod port-forward svc/hello-prod 8090:80   # http://localhost:8090
```

ArgoCD checks the repo about every 3 minutes. Click **Refresh** in the UI to sync sooner.

## Rollback

Run `git revert` on the CI commit in this repo, and ArgoCD syncs the old tag back. The Rollback button in the ArgoCD UI is disabled while automated sync is on, because Git would override it on the next sync.

## Teardown

```sh
kind delete cluster --name gitops
```
