# jotter-deploy

**What runs where.** This repo describes which version of [jotter](https://github.com/sarabesh/Jotter) runs in each environment. [Argo CD](https://argo-cd.readthedocs.io/) watches it and makes the cluster match. Nobody runs `kubectl apply` for jotter: **to deploy, change a file here through a PR.**

| Environment | Namespace | URL (local kind cluster) | Deploys |
|---|---|---|---|
| staging | `jotter-staging` | http://localhost:8081 | automatically after merge |
| prod | `jotter-prod` | http://localhost:8082 | after merge **and** clicking Sync in Argo CD |

## Layout
```
base/                      how jotter runs anywhere: Deployment, Service, PVC
overlays/staging/          base + image tag + namespace + port 30081
overlays/prod/             base + image tag + namespace + port 30082
argocd/                    the two Argo CD Applications (applied once by hand)
.github/workflows/         validate: kustomize build + kubeconform on every PR
```
See the final YAML for an environment: `kubectl kustomize overlays/staging`.

## Deploy a new version
1. jotter's release workflow publishes `ghcr.io/sarabesh/jotter:X.Y.Z` (signed).
2. PR here: `overlays/staging/kustomization.yaml` → `newTag: "X.Y.Z"`, title `deploy(staging): jotter X.Y.Z`.
3. `validate` passes → merge → Argo CD syncs staging (within ~3 min, or click **Refresh**).
4. Check http://localhost:8081/health shows `X.Y.Z`.
5. Same PR for `overlays/prod/`, merge, then **Sync** `jotter-prod` in the Argo CD UI.

**Roll back:** revert the PR. Argo CD puts the previous version back.

## One-time setup
Needs the local kind cluster `jotter` with Argo CD (see `coding-devops-agent/platform/local/`).
```bash
kubectl apply -f argocd/          # register the two Applications with Argo CD
```
Argo CD UI: http://localhost:8080 (user `admin`; password: `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}"`, base64-decoded).

## Rules
- Only image tags that jotter's `release.yml` published. Later (CD-2) Kyverno refuses images not signed by `sarabesh/Jotter/.github/workflows/release.yml@refs/heads/main`.
- jotter keeps SQLite on a 1 GiB volume: **1 replica only**, `Recreate` upgrades (a few seconds of downtime).
- Data lives as long as the cluster: `kind delete cluster` deletes the ideas.
