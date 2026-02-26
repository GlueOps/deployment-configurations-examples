# YAML Patterns — Traefik Example Skeletons

Copy-paste these skeletons when creating new Traefik examples. Replace `<placeholder>` values with your specifics.

---

## Pattern A: IngressRoute (CRD) via customResources

**Use for:** Most Traefik features — middleware, path routing, TLS, TCP, multi-route. This is the preferred pattern because `customResources` entries are processed through Helm's `tpl` function, so all template expressions work.

### `base/base-values.yaml`

```yaml
image:
  registry: docker.io
  repository: traefik/whoami
  tag: latest
  port: 80
```

### `envs/prod/values.yaml` — Basic (no middleware)

```yaml
deployment:
  enabled: true
  replicas: 1

service:
  enabled: true

customResources:
  - |
    apiVersion: traefik.io/v1alpha1
    kind: IngressRoute
    metadata:
      name: <app-name>
      annotations:
        kubernetes.io/ingress.class: public-traefik
        external-dns.alpha.kubernetes.io/target: public-v2.{{ .Values.captain_domain }}
    spec:
      entryPoints:
        - web
        - websecure
      routes:
        - match: Host(`{{ include "app.name" . }}.apps.{{ .Values.captain_domain }}`)
          kind: Rule
          services:
            - name: '{{ include "app.name" . }}'
              namespace: '{{ include "app.namespace" . }}'
              port: 80
```

### `envs/prod/values.yaml` — With Middleware

When adding middleware, define the Middleware resource as a separate `customResources` entry **before** the IngressRoute, then reference it in the IngressRoute's `middlewares` list:

```yaml
deployment:
  enabled: true
  replicas: 1

service:
  enabled: true

customResources:
  - |
    apiVersion: traefik.io/v1alpha1
    kind: Middleware
    metadata:
      name: <middleware-name>
    spec:
      <middleware-type>:
        <middleware-config>
  - |
    apiVersion: traefik.io/v1alpha1
    kind: IngressRoute
    metadata:
      name: <app-name>
      annotations:
        kubernetes.io/ingress.class: public-traefik
        external-dns.alpha.kubernetes.io/target: public-v2.{{ .Values.captain_domain }}
    spec:
      entryPoints:
        - web
        - websecure
      routes:
        - match: Host(`{{ include "app.name" . }}.apps.{{ .Values.captain_domain }}`)
          kind: Rule
          middlewares:
            - name: <middleware-name>
          services:
            - name: '{{ include "app.name" . }}'
              namespace: '{{ include "app.namespace" . }}'
              port: 80
```

### Notes for Pattern A

- Each `- |` entry is a YAML block scalar — indent content by 4 spaces from the `-` (or 2 spaces as shown above under the `|`)
- Middleware `metadata.name` does NOT need template expressions — use a static descriptive name
- IngressRoute `metadata.name` should be a static name matching the app
- The `external-dns.alpha.kubernetes.io/target` annotation goes on the **IngressRoute**, not on the Middleware
- A single `customResources` entry can contain multiple `---`-separated documents, but separate entries are cleaner

---

## Pattern B: Standard Kubernetes Ingress

**Use for:** Simple hostname routing where Traefik CRD features aren't needed. Note that Ingress `annotations` values are **NOT** processed through `tpl`, but `hostname` values **ARE**.

### `base/base-values.yaml`

```yaml
image:
  registry: docker.io
  repository: traefik/whoami
  tag: latest
  port: 80
```

### `envs/prod/values.yaml` — Basic

```yaml
deployment:
  enabled: true
  replicas: 1

service:
  enabled: true

ingress:
  enabled: true
  ingressClassName: public-traefik
  entries:
    - name: public
      hosts:
        - hostname: '{{ include "app.name" . }}.apps.{{ .Values.captain_domain }}'
```

### `envs/prod/values.yaml` — With Traefik Middleware via Annotation

⚠️ **Limitation:** The middleware annotation value must hardcode the namespace prefix (e.g., `nonprod-`) because annotations are not tpl-rendered. Users deploying to a different namespace must update this manually.

```yaml
deployment:
  enabled: true
  replicas: 1

service:
  enabled: true

ingress:
  enabled: true
  ingressClassName: public-traefik
  annotations:
    traefik.ingress.kubernetes.io/router.middlewares: <namespace>-<middleware-name>@kubernetescrd
  entries:
    - name: public
      hosts:
        - hostname: '{{ include "app.name" . }}.apps.{{ .Values.captain_domain }}'

customResources:
  - |
    apiVersion: traefik.io/v1alpha1
    kind: Middleware
    metadata:
      name: <middleware-name>
    spec:
      <middleware-type>:
        <middleware-config>
```

### Notes for Pattern B

- `hostname` must be wrapped in **single quotes** so the `{{ }}` expressions are treated as strings by YAML
- `ingressClassName: public-traefik` — always use this, never set it via annotation
- The `entries[].name` field (e.g., `public`) is an arbitrary label for the Helm chart template
- For multiple hosts, add more `hosts` entries or more `entries` items

---

## Pattern C: Service-Only (No Routing)

**Use for:** Backing services that are routed to by another app's IngressRoute (e.g., canary v1/v2 backends).

### `base/base-values.yaml`

```yaml
image:
  registry: docker.io
  repository: traefik/whoami
  tag: latest
  port: 80
```

### `envs/prod/values.yaml`

```yaml
deployment:
  enabled: true
  replicas: 1

service:
  enabled: true
```

Optionally add env variables to distinguish the service:

```yaml
deployment:
  enabled: true
  replicas: 1
  envVariables:
    - name: WHOAMI_NAME
      value: "<identifier>"

service:
  enabled: true
```

### Notes for Pattern C

- The Service will be named `{{ include "app.name" . }}` (e.g., `traefik-canary-v1-prod`)
- When referencing this service from another app's IngressRoute, you must hardcode the full service name since it's a cross-app reference (e.g., `traefik-canary-v1-prod`)
