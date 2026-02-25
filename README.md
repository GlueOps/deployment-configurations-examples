# deployment-configurations-examples

Example [deployment-configurations](https://github.com/GlueOps/deployment-configurations) for the GlueOps Platform. This repository contains **19 ready-to-deploy Traefik ingress examples** that demonstrate routing patterns using the [`app` Helm chart](https://github.com/GlueOps/project-template-helm-chart-app).

All examples use **Helm template expressions** instead of hardcoded domains, so they work on **any** GlueOps captain cluster without modification.

---

## Quick Start

### Option A: Use this entire repo

1. Fork or copy this repository into your GitHub organization
2. Point your captain cluster's `platform.yaml` at your fork (update the `deployment_configurations` repo reference)
3. ArgoCD will auto-discover every app under `apps/*/envs/*/` and deploy them
4. Access your apps at `https://<app-name>-<env>.apps.<captain_domain>`

### Option B: Reference individual apps

Copy specific app folders from `apps/` into your own `deployment-configurations` repo:

```bash
# Example: copy just the basic IngressRoute example
cp -r apps/traefik-basic /path/to/your/deployment-configurations/apps/
```

Then push to your repo — ArgoCD picks it up automatically.

---

## How It Works

The GlueOps Platform uses an **ArgoCD ApplicationSet** with a [git directory generator](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Generators-Git/) that:

1. Scans `apps/*/envs/*/` directories in this repo
2. Creates an ArgoCD Application for each discovered path (e.g., `apps/traefik-basic/envs/prod/` → app named `traefik-basic-prod`)
3. Renders each app using the [`app` Helm chart](https://github.com/GlueOps/project-template-helm-chart-app) with values merged in this order:
   - `common/common-values.yaml` — shared across all apps
   - `env-overlays/<env-group>/env-values.yaml` — shared across an environment group
   - `apps/<app>/base/base-values.yaml` — shared across all envs of one app
   - `apps/<app>/envs/<env>/values.yaml` — environment-specific config
4. Injects `captain_domain` (e.g., `nonprod.jupiter.onglueops.rocks`) as an inline Helm value, making it available as `{{ .Values.captain_domain }}`

### Template Expressions

These Helm template expressions are available inside `customResources`, `customResourcesMap`, and `ingress.entries[].hosts[].hostname` values (the chart processes them through [`tpl`](https://helm.sh/docs/howto/charts_tips_and_tricks/#using-the-tpl-function)):

| Expression | Resolves To | Example |
|-----------|-------------|---------|
| `{{ .Values.captain_domain }}` | Your cluster's domain | `nonprod.jupiter.onglueops.rocks` |
| `{{ include "app.name" . }}` | ArgoCD app name (`<app-folder>-<env-folder>`) | `traefik-basic-prod` |
| `{{ include "app.namespace" . }}` | Target namespace | `nonprod` |

---

## App Catalog

### Traefik IngressRoute (CRD) Examples

These use `customResources` to create Traefik-native `IngressRoute` CRDs with full middleware support.

| App | Description |
|-----|-------------|
| [`traefik-basic`](apps/traefik-basic/) | Simplest IngressRoute — hostname to service |
| [`traefik-paths`](apps/traefik-paths/) | Path-based routing with `PathPrefix` matchers and catch-all |
| [`traefik-multi-route`](apps/traefik-multi-route/) | Multiple IngressRoutes with different subdomains for one app |
| [`traefik-headers`](apps/traefik-headers/) | Security headers middleware (CORS, HSTS, X-Frame-Options) |
| [`traefik-ip-allow`](apps/traefik-ip-allow/) | IP allow list middleware — restrict access by CIDR |
| [`traefik-ratelimit`](apps/traefik-ratelimit/) | Rate limiting middleware (requests/second with burst) |
| [`traefik-basicauth`](apps/traefik-basicauth/) | HTTP Basic Auth middleware with Secret-backed credentials |
| [`traefik-tls`](apps/traefik-tls/) | HTTPS redirect middleware + TLS version enforcement |
| [`traefik-tcp`](apps/traefik-tcp/) | `IngressRouteTCP` for raw TCP/TLS passthrough routing |
| [`traefik-tcp-postgres`](apps/traefik-tcp-postgres/) | `IngressRouteTCP` for PostgreSQL with TLS termination + ALPN |

### Standard Kubernetes Ingress Examples

These use the Helm chart's built-in `ingress` configuration to create standard `networking.k8s.io/v1` Ingress resources.

| App | Description |
|-----|-------------|
| [`traefik-ingress`](apps/traefik-ingress/) | Basic hostname routing via standard Ingress |
| [`traefik-ingress-sticky`](apps/traefik-ingress-sticky/) | Sticky sessions with cookie-based affinity |
| [`traefik-ingress-paths`](apps/traefik-ingress-paths/) | Path-based routing using Ingress `paths` |
| [`traefik-ingress-multi-host`](apps/traefik-ingress-multi-host/) | Multiple hostnames on one Ingress |
| [`traefik-ingress-middleware`](apps/traefik-ingress-middleware/) | Standard Ingress + Traefik Middleware via annotations ⚠️ |
| [`traefik-ingress-tls`](apps/traefik-ingress-tls/) | HTTPS redirect via Ingress annotation + Middleware ⚠️ |

### Cookie-Based Canary Routing

These three apps work together to demonstrate canary deployments:

| App | Description |
|-----|-------------|
| [`traefik-canary-v1`](apps/traefik-canary-v1/) | Stable version — Deployment + Service (no ingress) |
| [`traefik-canary-v2`](apps/traefik-canary-v2/) | Canary version — Deployment + Service (no ingress) |
| [`traefik-canary`](apps/traefik-canary/) | Routing config — IngressRoute with cookie-based split |

Default traffic goes to v1. Set cookie `canary=v2` to route to v2.

---

## Directory Structure

```
deployment-configurations-examples/
├── README.md
├── common/
│   └── common-values.yaml              # Shared across ALL apps (empty placeholder)
├── env-overlays/
│   ├── nonprod/
│   │   └── env-values.yaml             # Shared across nonprod environments (empty placeholder)
│   └── prod/
│       └── env-values.yaml             # Shared across prod environments (empty placeholder)
└── apps/
    ├── traefik-basic/                   # Basic IngressRoute
    │   ├── base/
    │   │   └── base-values.yaml        # Image config (shared across envs)
    │   └── envs/
    │       └── prod/
    │           └── values.yaml         # Deployment + IngressRoute config
    ├── traefik-paths/                   # Path-based routing
    ├── traefik-multi-route/            # Multiple subdomains
    ├── traefik-headers/                # Security headers middleware
    ├── traefik-ip-allow/               # IP allow list middleware
    ├── traefik-ratelimit/              # Rate limiting middleware
    ├── traefik-basicauth/              # Basic auth middleware
    ├── traefik-tls/                    # HTTPS redirect + TLS options
    ├── traefik-tcp/                    # TCP passthrough routing
    ├── traefik-tcp-postgres/           # TCP + PostgreSQL routing
    ├── traefik-ingress/                # Standard Ingress (basic)
    ├── traefik-ingress-sticky/         # Standard Ingress (sticky sessions)
    ├── traefik-ingress-paths/          # Standard Ingress (path routing)
    ├── traefik-ingress-multi-host/     # Standard Ingress (multi-host)
    ├── traefik-ingress-middleware/     # Standard Ingress + Middleware ⚠️
    ├── traefik-ingress-tls/            # Standard Ingress + HTTPS redirect ⚠️
    ├── traefik-canary/                 # Canary routing config
    ├── traefik-canary-v1/              # Canary stable version
    └── traefik-canary-v2/              # Canary new version
```

---

## ⚠️ Important: Standard Ingress Middleware Annotation Limitation

Two apps — `traefik-ingress-middleware` and `traefik-ingress-tls` — reference Traefik Middleware resources from standard Ingress annotations:

```yaml
ingress:
  annotations:
    traefik.ingress.kubernetes.io/router.middlewares: nonprod-traefik-ingress-mw-headers@kubernetescrd
```

**The `nonprod` prefix in the annotation value is hardcoded** because Ingress annotations are **not** processed through Helm's `tpl` function. The format is `<namespace>-<middleware-name>@kubernetescrd`.

**If your apps deploy to a namespace other than `nonprod`**, you must manually update the namespace prefix in these annotation values.

> **Recommendation:** For full portability, prefer **IngressRoute** (CRD) examples over standard Ingress when middleware is involved. IngressRoute resources are defined inside `customResources`, which _are_ processed through `tpl`, so all template expressions work.

---

## Prerequisites

- **GlueOps Platform** with a configured captain cluster
- **Helm chart:** [`app`](https://github.com/GlueOps/project-template-helm-chart-app) v0.10.0+
- **Traefik:** IngressClass `public-traefik` available on the cluster
- **DNS:** Wildcard record `*.apps.<captain_domain>` resolving to the Traefik load balancer (managed automatically by the platform via external-dns)

## Creating Your Own `deployment-configurations` Repository

If you're starting fresh (not using this examples repo), create from the official template:

👉 **[Create from template](https://github.com/new?template_name=deployment-configurations&template_owner=GlueOps)**

Then copy individual examples from this repo as needed.

## Resources

- [GlueOps Platform Documentation](https://docs.glueops.dev)
- [`app` Helm Chart](https://github.com/GlueOps/project-template-helm-chart-app)
- [Traefik Documentation](https://doc.traefik.io/traefik/)
- [ArgoCD ApplicationSet — Git Directory Generator](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Generators-Git/)
