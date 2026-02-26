# AI Agent Instructions — deployment-configurations-examples

This repository contains ready-to-deploy Traefik ingress examples for the GlueOps Platform. Each example uses the [`app` Helm chart](https://github.com/GlueOps/project-template-helm-chart-app) and is auto-discovered by ArgoCD.

## How This Repo Works

ArgoCD uses a **git directory generator** that scans `apps/*/envs/*/` paths. For each match, it creates an ArgoCD Application that renders the `app` Helm chart with values merged in this order (last wins):

1. `common/common-values.yaml` — shared across all apps
2. `env-overlays/<env-group>/env-values.yaml` — shared across an environment group
3. `apps/<app>/base/base-values.yaml` — shared across all envs of one app
4. `apps/<app>/envs/<env>/values.yaml` — environment-specific config

The platform injects `captain_domain` (e.g., `<cluster>.<tenant>.onglueops.<tld>`) as an inline Helm value, available as `{{ .Values.captain_domain }}`.

## Adding a New Traefik Example — Checklist

Follow these steps exactly when creating a new example app:

### 1. Choose a name

- Use `traefik-<feature>` naming (kebab-case)
- For standard Ingress examples, use `traefik-ingress-<feature>`
- For canary examples, use `traefik-canary-<variant>`

### 2. Create the directory structure

```
apps/<app-name>/
├── base/
│   └── base-values.yaml
└── envs/
    └── prod/
        └── values.yaml
```

### 3. Write `base/base-values.yaml`

All Traefik examples use the same base image:

```yaml
image:
  registry: docker.io
  repository: traefik/whoami
  tag: latest
  port: 80
```

### 4. Write `envs/prod/values.yaml`

Pick the appropriate pattern from [.ai/patterns.md](.ai/patterns.md):

- **Pattern A** — IngressRoute CRD via `customResources` (preferred for most Traefik features)
- **Pattern B** — Standard Kubernetes Ingress via `ingress` config
- **Pattern C** — Service-only, no routing (for backing services like canary backends)

### 5. Update the README

Add the new app to the appropriate catalog table in `README.md`:

- IngressRoute examples → "Traefik IngressRoute (CRD) Examples" table
- Standard Ingress examples → "Standard Kubernetes Ingress Examples" table
- Canary examples → "Cookie-Based Canary Routing" table

Also add an entry in the "Directory Structure" tree.

### 6. Verify

- Confirm `deployment.enabled: true` and `service.enabled: true` are set (both default to `false`). Exception: routing-only apps (e.g., `traefik-canary`) that only define IngressRoutes pointing at other apps' services don't need these.
- Confirm IngressRoute has both required annotations (`kubernetes.io/ingress.class` and `external-dns.alpha.kubernetes.io/target`)
- Confirm hostname follows `{{ include "app.name" . }}.apps.{{ .Values.captain_domain }}` pattern
- Confirm service references use `'{{ include "app.name" . }}'` (name) and `'{{ include "app.namespace" . }}'` (namespace)
- Confirm YAML indentation under `customResources: - |` is correct (2-space indent inside the block scalar)

## Required Reading

Before creating or modifying any example app, you **must** read these files:

1. **Read `.ai/patterns.md` first** — contains the exact YAML skeletons to copy from. Pick the correct pattern (A, B, or C) before writing any YAML.
2. **Read `.ai/reference.md` second** — contains template expressions, naming conventions, and gotchas that cause broken deployments if ignored (e.g., annotations not being tpl-rendered, service name mismatches).
3. **Read `README.md`** — contains the app catalog tables you must update when adding a new example.
