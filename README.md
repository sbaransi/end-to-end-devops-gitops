# End-to-End DevOps GitOps Repository

This repository holds the Kubernetes configuration for the Flask app from
<https://github.com/sbaransi/end-to-end-devops-project>.

Argo CD watches the `main` branch and keeps the cluster the same as this repository. Nobody runs `kubectl apply` for the app: a change here is a deployment.

## Layout

```text
applicationsets/
  flask-applicationset.yaml   # creates one Argo CD Application per environment
end-to-end-devops-project/
  dev/   Chart.yaml  values.yaml  templates/
  qa/    Chart.yaml  values.yaml  templates/
  prd/   Chart.yaml  values.yaml  templates/
```

Each environment folder is a small Helm chart. The templates (ConfigMap, Deployment, Service, Ingress) are the same in all three folders. Only `values.yaml` is different.

## Environments

| | dev | qa | prd |
|---|---|---|---|
| Namespace | `dev` | `qa` | `prd` |
| Replicas | 5 | 2 | 3 |
| Workers (`GUNICORN_WORKERS` in the ConfigMap) | 2 | 2 | 4 |
| Ingress host | `dev.devops.local` | `qa.devops.local` | `prd.devops.local` |
| Image | `sammybaransi537/end-to-end-devops-project` | same | same |
| Tag | Set by Jenkins on every build | Changed by hand | Changed by hand |

Tags are Jenkins build numbers, so every environment points to one fixed image.

The app currently starts with `python app.py`, so the workers value is only stored in the ConfigMap.

## ApplicationSet

`applicationsets/flask-applicationset.yaml` uses a **Git directory generator**:

- It lists the folders under `end-to-end-devops-project/` on `main`: dev, qa and prd.
- For each folder it creates an Application named `<folder>-flask-app`, for example `dev-flask-app`.
- That Application deploys the folder's chart with its `values.yaml` into the namespace of the same name.
- Sync is automatic, with `prune` (delete what was removed from Git), `selfHeal` (undo manual changes in the cluster) and `CreateNamespace=true`.

Apply it once, after Argo CD is installed:

```bash
kubectl apply -f applicationsets/flask-applicationset.yaml
kubectl get applicationsets,applications -n argocd
```

Argo CD checks this repository about every 3 minutes.

## How Jenkins updates dev

After Jenkins pushes a new image to Docker Hub, its last stage:

1. clones this repository,
2. changes the `tag:` line in `end-to-end-devops-project/dev/values.yaml` to the build number,
3. commits `Update dev image tag to <build number>` as "Jenkins CI",
4. pushes to `main`.

Argo CD then updates the dev Deployment, and Kubernetes replaces the pods with a rolling update. qa and prd are not touched.

Jenkins pushes straight to `main` because that is the branch Argo CD watches. All manual changes go through a pull request.

## Manual changes and promotion to qa or prd

Jenkins adds commits to `main`, so bring `dev` up to date first:

```bash
git switch dev
git pull
git merge --ff-only origin/main
git push origin dev
git switch -c feature/promote-qa
```

Edit `end-to-end-devops-project/qa/values.yaml` and set `tag` to a build number that already works in dev. Then:

```bash
git add end-to-end-devops-project/qa/values.yaml
git commit -m "Promote build <N> to qa"
git push -u origin feature/promote-qa
git switch dev
git merge --no-ff feature/promote-qa
git push origin dev
```

Open a pull request from `dev` to `main` on GitHub and merge it. Argo CD deploys the change within a few minutes.

## Check the environments

```bash
kubectl get applications -n argocd
kubectl get deploy,pods,svc,endpoints,ingress -n dev
kubectl get deploy,pods,svc,endpoints,ingress -n qa
kubectl get deploy,pods,svc,endpoints,ingress -n prd
kubectl get deploy devops-flask-deploy -n dev -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
curl -H "Host: dev.devops.local" http://127.0.0.1:8080/health
```

Every environment should have a ConfigMap, a Deployment with all pods Ready, a Service with one endpoint per pod, and an Ingress with its own host. The Ingress controller is on host port 8080 (see the code repository README).

"Synced" and "Healthy" in Argo CD are not enough on their own. Always check the pods, endpoints and the app's response too.
