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
1. jotter's release workflow publishes `ghcr.io/sarabesh/jotter:X.Y.Z` (signed), then its `deploy-staging` job **commits** `overlays/staging/kustomization.yaml` → `newTag: "X.Y.Z"` straight to `main` here (`deploy(staging): jotter X.Y.Z`, pushed with this repo's deploy key).
2. Argo CD syncs staging (within ~3 min, or click **Refresh**). Check http://localhost:8081/health shows `X.Y.Z`.
3. **Promote by hand:** PR changing `overlays/prod/kustomization.yaml` to the same tag, title `deploy(prod): jotter X.Y.Z` → `validate` passes → merge → **Sync** `jotter-prod` in the Argo CD UI.

The staging bump can also be done by hand the same way (PR to `overlays/staging/`), e.g. to roll back.

**Roll back:** revert the PR. Argo CD puts the previous version back.

## One-time setup
Needs the local kind cluster `jotter` with Argo CD (see `coding-devops-agent/platform/local/`).
```bash
kubectl apply -f argocd/          # register the two Applications with Argo CD
```
Argo CD UI: http://localhost:8080 (user `admin`; password: `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}"`, base64-decoded).

### Deploy key for jotter's release workflow (one-time)
```bash
ssh-keygen -t ed25519 -N "" -C "jotter release -> jotter-deploy" -f release-push-key
```
- `release-push-key.pub` (public) → **this repo** → Settings → Deploy keys → Add, tick **Allow write access**.
- `release-push-key` (private) → **Jotter repo** → Settings → Secrets and variables → Actions → New secret `JOTTER_DEPLOY_KEY`.
- Delete both local files afterwards. If this repo gets a ruleset on `main`, add **Deploy keys** to its bypass list so the staging commit isn't blocked.

## Rules
- Only image tags that jotter's `release.yml` published. Later (CD-2) Kyverno refuses images not signed by `sarabesh/Jotter/.github/workflows/release.yml@refs/heads/main`.
- jotter keeps SQLite on a 1 GiB volume: **1 replica only**, `Recreate` upgrades (a few seconds of downtime).
- Data lives as long as the cluster: `kind delete cluster` deletes the ideas.
