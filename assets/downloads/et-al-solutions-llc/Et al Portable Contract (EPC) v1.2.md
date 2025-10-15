# Et al Portable Contract (EPC) v1.2

**Tagline:** *One contract. Any runtime. Zero collisions.*
**Release date:** 2025-10-13

---

## Statement of Principle

With EPC guidelines and guardrails in place, teams only need to build a runtime adapter for any implementation pattern to ensure the high quality, productivity, and competitiveness that are the hallmarks of an Et al Solution.

---

## Core Tenets

1. **Identity & Namespacing** — Every externally visible name (hosts, namespaces, services, buckets, queues) includes a unique `STACK` identifier (e.g., `acme-dev`). No collisions.
2. **12-Factor Config** — All per-deploy settings are strictly separated from code and provided via environment variables at runtime. Apps read config only from env. No code changes between environments.
3. **Secrets Discipline** — Dev may use `--env-file`. Prod never does. Production secrets are injected from the platform (Secret Manager / Key Vault / etc.).
4. **Single Front Door** — A shared entrypoint (LB/Ingress/Proxy) terminates traffic. Backends do not bind host ports directly. Routing is by hostname derived from `STACK`.
5. **Safety Rails** — Production deployment requires an explicit ARM toggle. All deploys are idempotent and fail fast on policy violations.
6. **Observability by Convention** — Health endpoints and log/metric labels include `STACK`.
7. **Parity Without Manual Edits** — No hand-editing files between customers or environments. Scripts/templates consume env and render artifacts automatically.
8. **Deterministic Images (new)** — All containers use explicit image tags and (optionally) digests, e.g., `docker.io/library/traefik:v2.10[@sha256:…]`, to avoid drift and enable reliable discovery.
9. **Portable Filesystem (new)** — Dev repos and bind-mounted paths live on the native runtime filesystem (e.g., WSL ext4 under `~/code`), not bridged mounts like `/mnt/c`. Adapters must be OS-agnostic.

---

## Non-Negotiables (SOP Snippets)

* No hard-coded hostnames. Hosts are `${STACK}.${LOCAL_DOMAIN}` (dev) and `${STACK}.${BASE_DOMAIN}` (prod).
* No `--env-file` in production anywhere (scripts, CI, manifests).
* ARM toggle required (e.g., `ops/ALLOW_PROD_DEPLOY.on`) before any prod deploy.
* Idempotent infra: rerunning init/build/deploy must be safe.
* Everything namespaced by `STACK`.
* **Single front door per host**: exactly one Traefik/Ingress binds `:80/:443`; app stacks join a shared external network (e.g., `proxy`).
* **Label-based discovery only**: operational scripts identify containers via Compose/K8s labels (e.g., `com.docker.compose.project`, `…service`) — never by `ancestor=` image heuristics.
* **No bridged-drive bind mounts** (WSL): forbid `/mnt/c/...` in Compose; use relative paths within `~/code/<repo>`.
* **Namespaced volumes**: rely on Compose project scoping (`<project>_<volume>`) or explicitly prefix volume names with `${STACK}`/project (e.g., `${COMPOSE_PROJECT_NAME}_le_data`).
* **Explicit project identity**: set a stable Compose project name (`COMPOSE_PROJECT_NAME` or `-p`) so containers, networks, and volumes are predictably namespaced.

---

## Adapter Pattern (How We Apply EPC)

For each runtime (Docker Compose, Kubernetes, Cloud Run/App Runner/ACA/DO, Functions, etc.), we provide a small adapter that:

* wires the edge (host → service) using `${STACK}` hosts,
* maps standard env names (e.g., `WORDPRESS_DB_HOST`, `WORDPRESS_DB_USER`, …),
* references platform secrets for prod,
* preserves EPC non-negotiables.

---

## Boilerplate (verbatim for SOW/README)

> **Et al Portable Contract (EPC).** This project adheres to EPC: identity via STACK namespacing; 12-factor configuration; platform-managed secrets in production; a single ingress with host-based routing; explicit ARM gating for production deploys; deterministic container images; portable filesystem rules for dev; and idempotent, zero-edit automation across environments. To target a new runtime, we supply a minimal adapter without changing application code.

---

## How We Keep It Zero-Touch & Repeatable

* **No hard-coded hosts anywhere.** Hosts are computed:

  * **Dev (Compose):** Use a project-root `.env` with **concrete** values to avoid Compose interpolation pitfalls. Example →
    `STACK=acctdemo`, `LOCAL_DOMAIN=localtest.me`, `APP_HOST=acctdemo.localtest.me`, `COMPOSE_PROJECT_NAME=${STACK}`.
  * **Prod:** `APP_HOST=${STACK}.${BASE_DOMAIN}` is computed via CI/platform env (never stored in a file).
* **Dev env files:**

  * **Root** `.env` **(Compose interpolation source)**: concrete values for `STACK`, `LOCAL_DOMAIN`, `APP_HOST`, `COMPOSE_PROJECT_NAME`, and any **non-secret** dev DB values.
  * Optional `env/${STACK}.env` for scripts/tooling; keep it consistent with `.env`.
* **Scripts load env & apply templates** (envsubst, Helm/Kustomize vars, Terraform variables). In dev, prefer root `.env` for Compose interpolation; use `--env-file` only for container-level env where appropriate.
* **ARM toggle enforced** in any prod script regardless of platform.
* **Standard env names** across all runtimes (`WORDPRESS_DB_*`, `WP_*`, `APP_HOST`, `STACK`) so app code never changes.
* **Deterministic images**: adapters specify fully qualified image refs with explicit tags (and optional digests).
* **Portable filesystem**: dev repos live under native FS (e.g., `~/code` on WSL); Compose mounts are relative (`./…`) inside the repo; **no** `/mnt/c/...` mounts.

---

## Current Runtime Adapters (Same Contract, Different Runtime)

### 1) Kubernetes (+ managed DB)

**Collision-proofing:**

* Namespace per stack: `kubectl create ns ${STACK}`
* Ingress host: `${STACK}.${LOCAL_DOMAIN}` (dev) / `${STACK}.${BASE_DOMAIN}` (prod)

**Example (templated with envsubst):**

```yaml
# k8s/deploy.yaml.tpl
apiVersion: apps/v1
kind: Deployment
metadata: { name: app, namespace: ${STACK}, labels: { app: ${STACK}-app } }
spec:
  replicas: 1
  selector: { matchLabels: { app: ${STACK}-app } }
  template:
    metadata: { labels: { app: ${STACK}-app } }
    spec:
      containers:
      - name: app
        image: ${IMAGE}
        env:
        - { name: WORDPRESS_DB_HOST, value: "${DB_HOST}" }
        - { name: WORDPRESS_DB_USER, value: "wp_user" }
        - { name: WORDPRESS_DB_NAME, value: "wordpress" }
        - { name: WORDPRESS_DB_PASSWORD, valueFrom: { secretKeyRef: { name: ${STACK}-secrets, key: DB_PASS } } }
        ports: [{ containerPort: 8080 }]
---
apiVersion: v1
kind: Service
metadata: { name: app-svc, namespace: ${STACK} }
spec:
  selector: { app: ${STACK}-app }
  ports: [{ port: 80, targetPort: 8080 }]
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata: { name: app-ing, namespace: ${STACK}, annotations: { kubernetes.io/ingress.class: traefik } }
spec:
  rules:
  - host: ${APP_HOST}
    http:
      paths:
      - path: /
        pathType: Prefix
        backend: { service: { name: app-svc, port: { number: 80 } } }
```

**Render & apply (no file edits):**

```bash
export STACK=summitdemo APP_HOST=${STACK}.local.test IMAGE=ghcr.io/org/app:dev DB_HOST=mysql.default.svc:3306
envsubst < k8s/deploy.yaml.tpl | kubectl apply -f -
```

**Secrets:** `kubectl -n ${STACK} create secret generic ${STACK}-secrets --from-literal=DB_PASS=...`

---

### 2) Serverless Containers (Cloud Run / App Runner / ACA / DO App Platform)

* **Edge:** platform domain mapping; host = `${STACK}.${BASE_DOMAIN}`
* **DB:** managed service (Cloud SQL / RDS / Azure MySQL / DO Managed MySQL)
* **Env/secrets:** set via platform CLI; **never** `--env-file` in prod

**Deploy skeleton (cloud-agnostic):**

```bash
STACK=${STACK:?} IMAGE=${IMAGE:?} APP_HOST=${STACK}.${BASE_DOMAIN}
DB_HOST=${DB_HOST:?} SECRET_REF=${SECRET_REF:?}  # platform-specific secret ref
# platform deploy...
```

---

### 3) Static Front-End + Serverless API

* **Edge:** same host derivation (`${STACK}.${BASE_DOMAIN}`), subpaths/subdomains for API.
* **Dev:** single local proxy routes `/api` → local function/container; no ports exposed.
* **Collision-proofing:** route prefixes include `${STACK}` or use per-stack hosts.

---

### 4) Functions (Lambda / Cloud Functions / Azure Functions)

* **Hostnames:** behind API Gateway / Front Door—use per-stack stage or subdomain.
* **Env/secrets:** always platform; local dev uses `--env-file` via emulator only.
* **Collision-proofing:** stage names or routes include `${STACK}`.

---

### 5) Data/Messaging Pipelines

* **Resource names:** topics/queues/streams include `${STACK}` (e.g., `${STACK}.events`).
* **Secrets/config:** env-injected; dev uses `.env` file, prod from platform.
* **No host ports** — still keep naming & ARM rules.

---

### 6) Docker Compose (Local Dev) — **updated v1.2**

* **Edge:** Traefik on shared `proxy` network; router rule `Host(${APP_HOST})`.
* **Identity:** Everything namespaced by `STACK`; containers/volumes scoped by project (`COMPOSE_PROJECT_NAME=${STACK}`).
* **Config:** Root `.env` with **concrete** values: `STACK=acctdemo`, `LOCAL_DOMAIN=localtest.me`, `APP_HOST=acctdemo.localtest.me`, `COMPOSE_PROJECT_NAME=acctdemo`.
* **Secrets:** Dev may use `.env` for non-secrets and local secrets; **never** in prod.
* **Images:** Fully qualified with explicit tags (optionally digests).
* **No host ports:** backends do not expose host ports; all traffic via Traefik.
* **No bridged-drive mounts:** all bind mounts are relative inside `~/code/<repo>` (Linux FS).

**Traefik (front door) example:**

```yaml
services:
  traefik:
    image: docker.io/library/traefik:v2.10
    command:
      - --providers.docker=true
      - --providers.docker.exposedbydefault=false
      - --entrypoints.web.address=:80
      - --entrypoints.websecure.address=:443
    ports: ["80:80","443:443"]
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - le-data:/letsencrypt
    networks: [proxy]
    labels:
      com.et-al.stack: ${STACK}
volumes:
  le-data:
    # name: ${COMPOSE_PROJECT_NAME}_le_data   # optional explicit namespacing
networks:
  proxy:
    external: true
```

**App behind Traefik:**

```yaml
services:
  web:
    image: nginx:stable
    networks: [proxy]
    labels:
      traefik.enable: "true"
      traefik.http.routers.${COMPOSE_PROJECT_NAME}.rule: Host(`${APP_HOST}`)
      traefik.http.routers.${COMPOSE_PROJECT_NAME}.entrypoints: websecure
      traefik.http.routers.${COMPOSE_PROJECT_NAME}.tls.certresolver: le
```

---

## Runtime Adapter Checklist

1. **Edge:** define how `${APP_HOST}` is bound (Ingress, LB, Gateway).
2. **Runtime:** container service / functions / k8s; confirm target port and health.
3. **Config:** map the standard env names to platform deployment knobs.
4. **Secrets:** map `WORDPRESS_DB_PASSWORD` to platform secret reference.
5. **Persistence:** set `${DB_HOST}` (socket path on GCP; TCP host:port elsewhere).
6. **Namespacing:** ensure every resource name includes `${STACK}`.
7. **Dev parity:** local proxy + `--env-file` ok in dev; prod forbids `--env-file`.
8. **Safety:** ARM toggle check at the top of prod scripts.
9. **Compose (dev) specifics — v1.2:** set **`COMPOSE_PROJECT_NAME=${STACK}`**, pin image tags, use label-based discovery, and forbid bridged-drive mounts.

---

## Single Front Door — Runtime Guard (SOP)

**Verify exactly one front door and correct wiring:**

```bash
# exactly one traefik service running?
[ "$(docker ps -q --filter 'label=com.docker.compose.service=traefik' | wc -l)" -eq 1 ] || {
  echo "Front door violation: not exactly one Traefik"; exit 1; }

# no other containers bind :80 or :443 directly
docker ps --format '{{.Names}} {{.Ports}}' | grep -E ':(80|443)->' | grep -v 'traefik' && {
  echo "Violation: another container binds :80 or :443"; exit 1; }

# app web attached to proxy network?
WEB_CID=$(docker ps -q --filter "label=com.docker.compose.project=${COMPOSE_PROJECT_NAME}" --filter "label=com.docker.compose.service=web")
docker inspect "$WEB_CID" -f '{{json .NetworkSettings.Networks}}' | grep -q '"proxy"' || {
  echo "App not attached to proxy network"; exit 1; }
```

---

## Portable Filesystem (WSL) Appendix

* **Repo location:** `~/code/<repo>` (ext4).
* **Git line endings:**

  ```bash
  git config --global core.autocrlf false
  git config --global core.eol lf
  ```

  Add `.gitattributes` enforcing LF for `*.sh`, `*.yml`, `*.jq`, etc.
* **Compose paths:** use **relative** mounts (`./…`) for content within the repo.
* **No `/mnt/c` mounts:** avoid drvfs performance/encoding issues.
* **Migration playbook:** copy repos from `/mnt/c` to `~/code`, normalize LF, rewrite absolute `/mnt/c` and `C:\...` references to relative paths, then recreate containers.

---

## Dev vs Prod Clarifications (unchanged from v1.1, reinforced)

* **Dev (Compose):** Root `.env` (ignored by git) holds concrete `STACK`, `LOCAL_DOMAIN`, `APP_HOST`, and `COMPOSE_PROJECT_NAME`. Optional `env/${STACK}.env` can feed scripts; keep it mirrored.
* **Prod (any runtime):** No `.env` files; no `--env-file`. Env & secrets injected by platform/CI. `APP_HOST=${STACK}.${BASE_DOMAIN}` computed by CI/platform; not stored in a file.

---

## Version History

* **v1.2 — 2025-10-13**
  Added **Deterministic Images** and **Portable Filesystem** core tenets; new non-negotiables (single front door per host, label-based discovery, no bridged-drive mounts, namespaced volumes, explicit project identity); updated Compose adapter; added **Single Front Door — Runtime Guard** SOP and **WSL Portable Filesystem** appendix.
* **v1.1 — 2025-10-04**
  Added Docker Compose (Local Dev) adapter; clarified `.env` usage.
* **v1.0 — Initial canon** (baseline EPC doctrine).

---

**End of EPC v1.2**
