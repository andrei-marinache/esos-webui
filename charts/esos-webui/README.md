# esos-webui Helm chart

Deploys the ESOS WebUI (Nuxt/Nitro, port 3000) for ArgoCD/GitOps.

## What it creates
- **Deployment** — 1 replica, `Recreate` strategy (SQLite is single-writer), non-root UID/GID 1000.
- **ConfigMap** — non-secret env (SSH host/user, DB path, SCST paths, poll interval, `extraEnv`).
- **Secret** — optional; only when `secrets.create=true`.
- **PVC** — one RWO claim mounted at both `/app/data` and `/opt/esos-webui/binaries` via subPath.
- **Service** (ClusterIP) + **Ingress** (optional TLS).

## Image
There is no upstream published image — build and push your own:
```
docker build -t ghcr.io/andrei-marinache/esos-webui:1.0.2 .
docker push ghcr.io/andrei-marinache/esos-webui:1.0.2
```
For a private GHCR package set `imagePullSecrets`.

## Secrets — two modes

**1. Plaintext (chart makes the Secret):**
```yaml
secrets:
  create: true
  encryptionKey: <hex>     # NUXT_ENCRYPTION_KEY
  jwtSecret: <hex>         # NUXT_JWT_SECRET
  sshPrivateKey: <base64>  # NUXT_SSH_PRIVATE_KEY (base64 -w0 of the key)  — or sshPassword
```

**2. External Secrets Operator (recommended):**
```yaml
secrets:
  create: false
  existingSecret: esos-webui-secrets
```
The referenced Secret must have keys named exactly as the env vars:
`NUXT_ENCRYPTION_KEY`, `NUXT_JWT_SECRET`, `NUXT_SSH_PRIVATE_KEY` (or `NUXT_SSH_PASSWORD`),
and optionally `NUXT_OIDC_STATE_PEPPER`, `NUXT_GITHUB_TOKEN`.

## LDAP
No LDAP env config here. LDAP providers are set in the admin UI and stored **encrypted in the
DB** (protected by `NUXT_ENCRYPTION_KEY`). So LDAP just needs the persistent PVC + encryption key —
both already provided. Only optional tuning knobs (search limits, event retention) go in `extraEnv`.

## Storage
`persistence.storageClassName` pins a class (empty = cluster default). Because it's RWO + SQLite,
never scale past 1 replica. Set `persistence.existingClaim` to reuse a claim.

## ArgoCD Application (example)
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: esos-webui
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/andrei-marinache/esos-webui.git
    targetRevision: main
    path: charts/esos-webui
    helm:
      valueFiles: [values.yaml]   # or an env-specific values file
  destination:
    server: https://kubernetes.default.svc
    namespace: esos-webui
  syncPolicy:
    automated: { prune: true, selfHeal: true }
    syncOptions: [CreateNamespace=true]
```
