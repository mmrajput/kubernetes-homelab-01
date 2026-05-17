# Platform Services

All platform services are deployed and managed declaratively via ArgoCD GitOps. No `kubectl apply` for platform resources — commit to Git and ArgoCD reconciles.

## Service Inventory

| Service | Chart | Version | Namespace | URL |
|---------|-------|---------|-----------|-----|
| ArgoCD | argo-cd | 9.3.0 | argocd | argocd.mmrajputhomelab.org |
| cert-manager | cert-manager | v1.16.2 | cert-manager | — |
| ingress-nginx | ingress-nginx | 4.11.3 | ingress-nginx | — |
| Vault | vault | 0.32.0 | vault | vault.mmrajputhomelab.org |
| External Secrets | external-secrets | 2.1.0 | external-secrets | — |
| Keycloak | keycloakx | 7.1.9 | keycloak | keycloak.mmrajputhomelab.org |
| CloudNativePG | cloudnative-pg | 0.27.1 | cnpg-system | — |
| Longhorn | longhorn | 1.7.2 | longhorn-system | longhorn.mmrajputhomelab.org |
| MinIO | minio | 5.4.0 | minio | minio-console.mmrajputhomelab.org |
| Velero | velero | 12.0.0 | velero | — |
| kube-prometheus-stack | kube-prometheus-stack | 65.8.1 | monitoring | prometheus.mmrajputhomelab.org |
| Grafana | grafana | 10.5.15 | monitoring | grafana.mmrajputhomelab.org |
| Loki | loki | 6.20.0 | monitoring | — |
| Promtail | promtail | 6.16.6 | monitoring | — |
| ARC systems | gha-runner-scale-set-controller | 0.14.0 | arc-systems | — |
| ARC runners | gha-runner-scale-set | 0.14.0 | arc-runners | — |
| Falco | falco | 4.11.0 | falco | — |

Full inventory with all endpoints: [`docs/reference/platform-inventory.md`](../docs/reference/platform-inventory.md)

## Directory Structure

```
platform/
├── README.md                          # This file
├── argocd/
│   ├── README.md
│   ├── root-app.yaml                  # App-of-Apps root (bootstrap once)
│   └── apps/                          # ArgoCD Application + AppSet manifests
│       ├── networking/
│       ├── security/
│       ├── data/
│       ├── observability/
│       ├── ci-cd/
│       └── workloads/
├── networking/
│   ├── cert-manager/
│   ├── nginx-ingress/
│   └── network-policies/             # Default-deny + per-namespace rules
├── security/
│   ├── vault/
│   ├── external-secrets/             # ClusterSecretStore + ExternalSecrets
│   ├── keycloak/
│   └── falco/                        # Runtime threat detection (modern_ebpf + Falcosidekick → Loki)
├── data/
│   ├── cnpg/clusters/                # CNPG PostgreSQL cluster definitions
│   ├── longhorn/
│   ├── minio/
│   ├── velero/
│   └── rclone/
├── observability/
│   ├── prometheus/
│   ├── grafana/
│   ├── loki/                         # Loki + Promtail values
│   └── alertmanager/
└── ci-cd/
    ├── arc-systems/
    └── arc-runners/
```

## Access Pattern

All services are exposed via Cloudflare Tunnel → nginx-ingress (NodePort 30080/30443) → service. TLS is terminated by cert-manager (wildcard cert, Cloudflare DNS-01). There are no open inbound ports on the home network.

```
Browser
  ↓
Cloudflare Tunnel (mmrajputhomelab.org)
  ↓
nginx-ingress controller (NodePort 30443)
  ↓
Ingress resource (host matching)
  ↓
ClusterIP service → Pod
```

## Bootstrap (One-Time)

These steps are performed once per cluster lifetime. Everything after this flows through ArgoCD.

```bash
# 1. Apply namespace manifests with PSS labels (ArgoCD bug workaround — see cluster-rebuild runbook)
kubectl apply -f bootstrap/namespaces/

# 2. Install ArgoCD from upstream raw manifest
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v3.2.3/manifests/install.yaml

# 3. Apply the root App-of-Apps — this bootstraps the entire platform
kubectl apply -f platform/argocd/root-app.yaml

# 4. Apply secrets that cannot come from ESO (bootstrapping dependency)
#    cert-manager: Cloudflare API token
kubectl create secret generic cloudflare-api-token \
  --from-literal=api-token=<TOKEN> -n cert-manager
#    cloudflared: Tunnel token
kubectl create secret generic cloudflare-tunnel-token \
  --from-literal=token=<TUNNEL_TOKEN> -n cloudflare
```

Full step-by-step procedure: [`docs/runbooks/cluster-rebuild.md`](../docs/runbooks/cluster-rebuild.md)

## ArgoCD Conventions

- All apps: `namespace: argocd`, multi-source (Helm chart + `$values` Git ref), `CreateNamespace=false`, `selfHeal: true`
- Sync interval: 120s
- `ServerSideApply=true` for CRD-heavy operators (CNPG, ESO, ARC)
- Hard refresh: `kubectl annotate application <name> -n argocd argocd.argoproj.io/refresh=hard --overwrite`
- Never `kubectl apply` platform resources while selfHeal is active — disable it first

## Adding a New Platform Service

```bash
# 1. Add Helm values file
mkdir -p platform/<layer>/<service>
vim platform/<layer>/<service>/values.yaml

# 2. Add ArgoCD Application manifest
vim platform/argocd/apps/<layer>/<service>-app.yaml

# 3. Commit and push (feature branch)
git add platform/<layer>/<service>/ platform/argocd/apps/<layer>/
git commit -m "feat(<layer>): add <service>"
git push
# ArgoCD discovers and deploys automatically
```

## Related Documentation

- [ArgoCD README](argocd/README.md)
- [Platform Inventory](../docs/reference/platform-inventory.md)
- [Cluster Rebuild Runbook](../docs/runbooks/cluster-rebuild.md)
- [ADR-006: GitOps Tool](../docs/adr/ADR-006-gitops-tool.md)
- [ADR-008: App-of-Apps Pattern](../docs/adr/ADR-008-app-of-apps-pattern.md)

---

**Last Updated:** May 2026
