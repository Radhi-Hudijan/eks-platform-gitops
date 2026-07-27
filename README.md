# EKS Platform — GitOps (`eks-platform-gitops`)

The GitOps source of truth for the EKS platform. ArgoCD (installed by [`eks-platform-infra`](https://github.com/Radhi-Hudijan/eks-platform-infra)) watches this repo and continuously reconciles the cluster to match it. Uses the **App of Apps** pattern with an **AppProject** governance boundary and an **ApplicationSet** for multi-environment delivery.

---

## 🌳 How It Works (App of Apps)

```
kubectl apply -f bootstrap/root-app.yaml   ← the ONLY manual command
        │
        ▼
root-app (Application)  watches the argocd/ folder in Git
        │
        ├── argocd/projects/demo-project.yaml   → creates the AppProject (guardrails)
        └── argocd/applications/demo-app-appset.yaml → creates the ApplicationSet
                                                              │
                                                              ├── demo-app-dev  → namespace: dev
                                                              └── demo-app-prod → namespace: prod
                                                                      │
                                                                      ▼
                                            deploys the Helm chart in apps/demo-app/helm
                                            (nginx + Service + Ingress/ALB + HPA)
```

**Golden rule:** Git is the source of truth. Manual `kubectl` changes are reverted by ArgoCD (`selfHeal`). To change the cluster, change Git.

---

## 📁 Repo Structure

```
eks-platform-gitops/
├── bootstrap/
│   └── root-app.yaml              # 👈 apply once; manages everything else
│
├── argocd/                        # root-app watches this (recurse)
│   ├── projects/
│   │   └── demo-project.yaml       # AppProject — security/governance boundary
│   └── applications/
│       └── demo-app-appset.yaml    # ApplicationSet — generates one App per env
│
└── apps/
    └── demo-app/
        └── helm/                   # the actual Helm chart
            ├── Chart.yaml
            ├── values.yaml         # base values (common to all envs)
            ├── values-dev.yaml     # dev overrides (1 replica, no HPA)
            ├── values-prod.yaml    # prod overrides (3 replicas, HPA 3–6)
            └── templates/
                ├── deployment.yaml
                ├── service.yaml
                ├── ingress.yaml     # annotated for the AWS LB Controller (ALB)
                ├── hpa.yaml
                ├── serviceaccount.yaml
                └── _helpers.tpl
```

---

## 🧩 The Three ArgoCD Objects

| Object | File | Role |
|---|---|---|
| **root-app** (Application) | `bootstrap/root-app.yaml` | Manages all other apps from Git (App of Apps) |
| **demo-project** (AppProject) | `argocd/projects/` | Restricts allowed repos, namespaces, and resource kinds |
| **demo-app** (ApplicationSet) | `argocd/applications/` | Templates one Application per environment |

### AppProject guardrails (`demo-project`)
- **sourceRepos** — only this repo may be pulled from
- **destinations** — only `dev`, `prod`, `demo`, `argocd` namespaces allowed
- **clusterResourceWhitelist** — only `Namespace` allowed cluster-scoped
- If an app tries to deploy outside these, ArgoCD blocks it (by design).

### ApplicationSet (`demo-app`)
A **list generator** produces one Application per environment:
```yaml
generators:
  - list:
      elements:
        - env: dev
        - env: prod
```
Each generated app layers `values.yaml` + `values-{{env}}.yaml` and deploys to the `{{env}}` namespace. **Add an environment = add one line.**

---

## 📦 The Helm Chart (`apps/demo-app/helm`)

A minimal but production-shaped chart deploying nginx behind an ALB.

| Template | Creates |
|---|---|
| `deployment.yaml` | Pods (omits `replicas` when HPA is on — avoids fighting the autoscaler) |
| `service.yaml` | ClusterIP service |
| `ingress.yaml` | Ingress with `ingressClassName: alb` → provisions an ALB |
| `hpa.yaml` | CPU-based HorizontalPodAutoscaler |
| `serviceaccount.yaml` | SA (annotate for IRSA if the app needs AWS access) |

### Multi-environment values
| File | replicas | HPA | Resources |
|---|---|---|---|
| `values.yaml` (base) | 2 | on (2–5) | 100m / 128Mi |
| `values-dev.yaml` | 1 | off | 50m / 64Mi |
| `values-prod.yaml` | 3 | on (3–6) | (inherits base) |

Render locally to verify:
```bash
helm template demo apps/demo-app/helm -f apps/demo-app/helm/values.yaml -f apps/demo-app/helm/values-prod.yaml
```

---

## 🚀 Bootstrap

Prerequisite: the cluster + ArgoCD are up (see [`eks-platform-infra`](https://github.com/Radhi-Hudijan/eks-platform-infra)).

```bash
# One command brings up the AppProject, ApplicationSet, and all environments
kubectl apply -f bootstrap/root-app.yaml

# Watch it cascade
kubectl -n argocd get applications
# NAME            SYNC STATUS   HEALTH
# root-app        Synced        Healthy
# demo-app-dev    Synced        Healthy
# demo-app-prod   Synced        Healthy
```

Access the ArgoCD UI:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d ; echo
kubectl -n argocd port-forward svc/argocd-server 8080:443
# https://localhost:8080  (user: admin)
```

Get an app's ALB URL:
```bash
kubectl -n prod get ingress -o jsonpath='{.items[0].status.loadBalancer.ingress[0].hostname}' ; echo
```

---

## ➕ Add a New Environment

1. Create `apps/demo-app/helm/values-staging.yaml`
2. Add `namespace: staging` to `demo-project` destinations
3. Add `- env: staging` to the ApplicationSet generator
4. `git commit && git push` — ArgoCD auto-creates `demo-app-staging`. No `kubectl`.

---

## 🔁 Self-Heal (GitOps Guarantee)

ArgoCD reverts drift automatically:
```bash
# Delete a resource — ArgoCD recreates it from Git
kubectl -n prod delete service demo-app-prod
# ...within ~30s it's back

# Change a Git-managed field — ArgoCD reverts it
kubectl -n prod set image deployment/demo-app-prod demo-app=nginx:1.25
# ...reverts to the tag in Git
```

---

## 🧹 Teardown (top-down — order matters)

```bash
kubectl -n argocd delete application root-app       # 1. stop the App of Apps
kubectl -n argocd delete applicationset demo-app    # 2. stop child regeneration
kubectl -n argocd delete application --all          # 3. remove any leftovers
```
Deleting a generated child first is futile — the ApplicationSet recreates it. Always delete top-down.

---

## 📚 Related

- Infrastructure (Terraform): [`eks-platform-infra`](https://github.com/Radhi-Hudijan/eks-platform-infra)
