# Reference — Template Expressions, Conventions & Gotchas

## Template Expressions

These Helm template expressions are available inside `customResources`, `customResourcesMap`, and `ingress.entries[].hosts[].hostname` values (the Helm chart processes them through `tpl`):

| Expression | Resolves To | Example Value |
|---|---|---|
| `{{ .Values.captain_domain }}` | Cluster domain (injected by platform) | `nonprod.jupiter.onglueops.rocks` |
| `{{ include "app.name" . }}` | ArgoCD app name = `<app-folder>-<env-folder>` | `traefik-basic-prod` |
| `{{ include "app.namespace" . }}` | Target namespace | `nonprod` |

### Where expressions work

| Location | `tpl`-rendered? | Example |
|---|---|---|
| `customResources` entries | ✅ Yes | `{{ .Values.captain_domain }}` works |
| `customResourcesMap` entries | ✅ Yes | Same as above |
| `ingress.entries[].hosts[].hostname` | ✅ Yes | `'{{ include "app.name" . }}.apps.{{ .Values.captain_domain }}'` |
| `ingress.annotations` values | ❌ No | Must hardcode namespace prefix |
| All other Helm values | ❌ No | Standard YAML values only |

---

## Traefik Conventions

### Required IngressRoute Annotations

Every IngressRoute must include these two annotations in `metadata.annotations`:

```yaml
kubernetes.io/ingress.class: public-traefik
external-dns.alpha.kubernetes.io/target: public-v2.{{ .Values.captain_domain }}
```

- `kubernetes.io/ingress.class` tells Traefik to claim this route
- `external-dns.alpha.kubernetes.io/target` tells external-dns to create a DNS record pointing to the load balancer

### Hostname Pattern

All apps are accessible at:

```
https://<app.name>.apps.<captain_domain>
```

Template form: `` Host(`{{ include "app.name" . }}.apps.{{ .Values.captain_domain }}`) ``

The platform manages a wildcard DNS record `*.apps.<captain_domain>` via external-dns.

### IngressClass

Always use `public-traefik`:

- IngressRoute: set via `kubernetes.io/ingress.class` annotation
- Standard Ingress: set via `ingressClassName: public-traefik` field (not as an annotation)

### EntryPoints

Standard IngressRoutes use both:

```yaml
entryPoints:
  - web        # HTTP (port 80)
  - websecure  # HTTPS (port 443)
```

TCP IngressRoutes (`IngressRouteTCP`) typically use only `websecure`.

---

## Naming Conventions

| Thing | Convention | Example |
|---|---|---|
| App directory | `traefik-<feature>` (kebab-case) | `traefik-ratelimit` |
| Standard Ingress app | `traefik-ingress-<feature>` | `traefik-ingress-sticky` |
| Canary app | `traefik-canary-<variant>` | `traefik-canary-v1` |
| IngressRoute `metadata.name` | Static, matches app directory name | `traefik-basic` |
| Middleware `metadata.name` | Static, descriptive of function | `traefik-headers-security` |
| Service reference in IngressRoute | `{{ include "app.name" . }}` | Resolves to `traefik-basic-prod` |

---

## Gotchas

### 1. Ingress annotations are NOT tpl-rendered

The `app` Helm chart iterates `ingress.annotations` with `range` and quotes the values. It does **not** call `tpl`. This means `{{ }}` expressions inside annotation values are rendered as literal strings, not evaluated.

**Impact:** When referencing a Traefik Middleware from a standard Ingress annotation, you must hardcode the namespace:

```yaml
# ❌ Won't work — rendered literally
traefik.ingress.kubernetes.io/router.middlewares: {{ include "app.namespace" . }}-my-middleware@kubernetescrd

# ✅ Works — hardcoded namespace
traefik.ingress.kubernetes.io/router.middlewares: nonprod-my-middleware@kubernetescrd
```

**Recommendation:** Prefer IngressRoute CRDs (Pattern A) over standard Ingress when middleware is involved.

### 2. customResources don't inherit metadata

Resources defined in `customResources` do **not** automatically get namespace, labels, or annotations from the Helm chart. You must specify `metadata.name` and `metadata.annotations` yourself. The resource is created in the release namespace by default (you don't need to set `metadata.namespace` unless deploying to a different one).

### 3. Service name equals app.name

The Helm chart names the Service `{{ include "app.name" . }}`. Your IngressRoute `services[].name` must match:

```yaml
services:
  - name: {{ include "app.name" . }}    # matches the auto-created Service
    namespace: {{ include "app.namespace" . }}
    port: 80                             # must match image.port
```

### 4. Service port must match image.port

The Service listens on `image.port` (or `service.port` if explicitly set). The IngressRoute `services[].port` must match this value. All current examples use port `80`.

### 5. Middleware namespace reference format

When referencing middleware from Ingress annotations, the format is:

```
<namespace>-<middleware-name>@kubernetescrd
```

Note the `-` between namespace and middleware name, and `@kubernetescrd` suffix.

### 6. Cross-app service references must be hardcoded

When one app routes to another app's Service (e.g., canary routing), the service name must be hardcoded because `{{ include "app.name" . }}` resolves to the **current** app's name, not the target app's:

```yaml
# In traefik-canary's IngressRoute:
services:
  - name: traefik-canary-v1-prod    # hardcoded: <target-app-folder>-<env>
    namespace: {{ include "app.namespace" . }}
    port: 80
```

### 7. deployment.enabled and service.enabled default to false

Both must be **explicitly set to `true`** — they default to `false` in the Helm chart. Forgetting either will result in no pods or no service being created.

### 8. Single customResources entry can contain multiple documents

A single `- |` block can contain `---`-separated YAML documents. The Helm chart splits on `---` and creates each as a separate resource. However, for clarity, prefer separate list entries.
