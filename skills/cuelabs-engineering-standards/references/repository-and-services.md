# Repository and service standard

## Contents

- Canonical structure and naming
- Community health and shared configuration
- Service layouts and language naming
- Deployment and local containers
- Cleanup rules
- Existing-repository and new-repository procedures

## Canonical structure

```
api/
  common/            Go backend — auth + core API. ALWAYS named "common".
  <service-name>/    Additional services, named by FUNCTION not language:
                     e.g. measure (Python pose estimation), observability
                     (Python ingest), intake + process (Node upload gateway
                     + Python processing, a real multi-service pipeline;
                     see "Multi-service pipelines" below).
web/                 Next.js marketing site + dashboard
mobile/
  flutter/           Primary cross-platform app (Dart)
  android/           Native Android (Kotlin) — placeholder until built
  ios/               Native iOS (Swift) — placeholder until built
deploy/
  docker/            Extra container assets (compose itself lives at the repo root)
  helm/              ONE standard-form chart that deploys ALL services (incl. Envoy) to k8s
  terraform/         Cluster-agnostic IaC: installs the Helm chart via kubeconfig
docs/                overview.md, setup.md (+ optional api/, architecture.md)
scripts/             Developer / CI helper scripts
```

Plus the root files listed under **Community health & config** below.

### Naming rule (important)
`api/common` is the shared Go backend in every repo. Every **other** service is
named by what it *does* (`measure`, `observability`, `intake`, `process`),
**never** by its language (`go`/`python`/`nodejs`). Only create a service
directory when a real service exists — do not add empty placeholder directories.

### Multi-service pipelines (more than one non-common service)
Most repos need only `api/common` + one other service. A repo needs **more**
than that when a real multi-stage pipeline exists — e.g. a document flow
where a Node gateway receives the upload, a Python service does the actual
extraction/decision work, and Go persists the result. In that shape:
- **Go (`api/common`) is the CRUD/data owner ONLY** — it persists what other
  services decide, it never makes classification/detection decisions itself.
  Anything that's a lookup-then-write against data Go already owns (create
  the row, update the status) is CRUD; anything that decides a value (which
  category, is this a duplicate, is this anomalous) is not, even if it
  happens to touch the database as a side effect.
- **The processing service (e.g. `process`) owns every decision, no
  persistence of its own.** It receives raw input plus whatever reference
  data it needs (existing categories, fingerprints, historical aggregates)
  from Go, computes the result, and sends the full decision back for Go to
  write. It never opens its own connection to Go's datastore.
- **The gateway service (e.g. `intake`) is thin** — request validation and
  handing off to the processing service. It owns no persisted state of its
  own; the CRUD service is still the one creating/tracking the job record.
- Real, standing services for all three — never a Cloud Function standing in
  for one of them. Serverless functions are reserved for probe/health-check
  endpoints only (see `cloud-and-ci.md`).

## Community health & config (required root files)

**These files must have parity across all CueLABS™ repos** — byte-identical,
sourced from [`assets/templates/`](../assets/templates/). Only `README.md`, `CHANGELOG.md`,
and `.github/dependabot.yml` are repo-specific (repo overview, its own history,
and its own manifest scoping); everything else in the table below — including the
compose-driven, mobile-ready `Makefile` — is identical across repos.

| File | Purpose |
|------|---------|
| `README.md` | Overview, architecture diagram, repo structure, getting started, links |
| `CONTRIBUTING.md` | Fork/branch flow, Conventional Commits, review, layout |
| `CODE_OF_CONDUCT.md` | Contributor Covenant 2.1 |
| `SECURITY.md` | Private vulnerability reporting; secret-handling rules |
| `CODEOWNERS` | Default reviewers |
| `CHANGELOG.md` | Keep a Changelog format |
| `LICENSE` | Project license |
| `.gitignore` | Must NOT ignore `.dockerignore`; must ignore `.env*`, secrets |
| `.dockerignore` | Root byte-identical; plus one per build context (repo-specific, see `assets/templates/dockerignore.*`) |
| `.editorconfig` | Shared editor settings (tabs for Go) |
| `Makefile` | Compose-driven standard targets plus guarded `mobile-goldens`; identical across repos |
| `.env.example` | Root env template: per-service sections, dev-safe defaults, `NEXT_PUBLIC_*` block |
| `.github/dependabot.yml` | **Scoped per manifest**, grouped per ecosystem |
| `.github/PULL_REQUEST_TEMPLATE.md` | PR checklist |
| `.github/ISSUE_TEMPLATE/` | bug_report, feature_request, config.yml |

**What's in `assets/templates/`:** ready-to-copy `LICENSE`, `CODEOWNERS`,
`CONTRIBUTING.md`, `CODE_OF_CONDUCT.md`, `SECURITY.md`, `PULL_REQUEST_TEMPLATE.md`,
`ISSUE_TEMPLATE/*`, `Makefile`, a `dependabot.example.yml`, an `env.example`,
`prettierignore.web`,
Docker templates (`Dockerfile.go`, `Dockerfile.web`, `Dockerfile.python`,
`Dockerfile.node`, `docker-compose.example.yml`, per-context `dockerignore.*`),
the standard-form
Helm chart skeleton (`helm/`), and cluster-agnostic terraform (`terraform/`). Dotfile templates are stored
**without** a leading dot so they stay visible and are never applied to this repo
by accident — when adopting them, copy `assets/templates/gitignore` → `.gitignore`,
`assets/templates/dockerignore.root` → `.dockerignore`, and `assets/templates/editorconfig` →
`.editorconfig`; copy `assets/templates/prettierignore.web` →
`web/.prettierignore`.

**Changelog discipline**: `CHANGELOG.md` follows Keep a Changelog strictly —
ONE `### <Bucket>` heading per bucket per section, in canonical order (Added,
Changed, Deprecated, Removed, Fixed, Security). New entries merge into the
existing bucket heading under `[Unreleased]`; grep for it before writing a new
one. Every entry carries its `(#NNN)` PR ref: open the PR first, then amend the
entry with the real number.

`dependabot.yml` is the one config that is **not** identical across repos: it
lists one `updates` entry per real manifest directory (`gomod /api/common`,
`pip /api/<python-service>`, `npm /web`, `pub /mobile/flutter`), grouped per
ecosystem, and **must not** point at dead/deprecated directories. It has **no**
`github-actions` entry — CI workflow conventions and shared jobs are org canon
kept byte-identical across repos and updated deliberately in canon passes,
never by a per-repo bot. Product files may add only explicitly ratified
surface jobs/workflows (for example the mobile workflows); the tag-gated
`release.yml` remains a shared fleet file when it lands.

## Service structure (production)
When bootstrapping or standardizing a service, **migrate existing code into
these layouts** — don't scaffold empty projects.

**Every service directory** (each `api/*`, `web`) carries its own internals,
consistent across repos: `README.md` (layout/run/config/test), `.gitignore`,
`.dockerignore` (see `assets/templates/dockerignore.*`), and `.env.example` with the
service's native-run variables (compose users rely on the root `.env`).
Go module paths are canonical: `github.com/cuesoftinc/<repo>/api/common`.

**Go (`api/common`):**
```
cmd/server/main.go            entrypoint: slog JSON, config → deps → graceful shutdown
internal/config/              typed env config (fail-fast on missing secrets)
internal/handler/             thin HTTP/gRPC handlers
internal/middleware/          slog logging, request-id, CORS allowlist, recovery
internal/model/  internal/service/  internal/repository/  internal/router/
internal/util/   internal/proto/ (generated)
```
Singular package names. `/health` + `/ready`, `$PORT` (8080), structured `slog`,
graceful shutdown on SIGINT/SIGTERM. gRPC services multiplex gRPC + `/health`
over `$PORT` via h2c.

**Python (`api/<service>`, FastAPI):**
```
app/main.py app/config.py     FastAPI + lifespan (load models/clients once)
router/  service/  model/  repository/   (+ domain pkgs, e.g. ml/, analysis/, database/)
```
Singular folders. `lifespan` (never `@app.on_event`), `/health` + `/ready`, uvicorn,
non-root image, pinned deps.

**Node (`api/<service>`, NestJS):**
```
src/main.ts                   bootstrap: CORS via CORS_ORIGINS, $PORT
src/app.module.ts              root module
src/health/                    /health + /ready
src/<feature>/                 one module per real feature, added when it exists
```
`$PORT` (per the port convention), `/health` + `/ready`, non-root `node`
user in the image. This is an **API service** (upload gateway, webhook
receiver, etc.) — distinct from `web` (the Next.js frontend), which keeps
its own layout below. Never call this service's directory `node` or
`nodejs`; name it by function per the naming rule above.

**Next.js (`web`):**
```
src/app/                      routes — home at `/`, product dashboard at `/dashboard`
src/{auth,components,config,controllers,design,generated,lib,mocks,models}
src/proto/                    generated protocol clients, only when needed
```
Minimal root `layout.tsx` (html/body — plus a CSS-in-JS registry only where the
repo uses one); the home page
renders its own shell; `/dashboard` gets a nested `layout.tsx`. `@/*` → `./src/*`,
`output: "standalone"`.

**Flutter (`mobile/flutter`):** feature-first per the mobile standard
(see the Mobile section below — the single source for the tree):
```
lib/main.dart + lib/main_dev.dart          (per-flavor entrypoints)
lib/src/features/<feature>/{presentation,domain,data}
lib/src/{app,routing,core}                 (core/ui = the design system)
```
Prefer `package:` imports over relative so moves are mechanical.

### File & folder naming (uniform across repos)
Same language ⇒ same conventions; each project keeps its own *features*.

| Language | Folders | Files |
|----------|---------|-------|
| Go | lowercase, **singular** package (`handler`, `service`, `model`, `repository`, `router`, `util`, `config`, `middleware`, `proto`) | `snake_case.go` |
| Python | lowercase, **singular** (`router`, `service`, `model`, `repository`) | `snake_case.py` |
| Node API (NestJS, `api/<service>`) | lowercase, one folder per module (`health/`, `<feature>/`) | **kebab-case** (`file-upload.controller.ts`, `file-upload.service.ts`, `file-upload.module.ts`) |
| Next / TS (`web`) | **kebab-case** (`shared-layouts/`, `change-password/`) | **Components PascalCase** (`NavBar.tsx`); modules/hooks/styles/types **kebab-case** (`use-auth.ts`, `home-context.ts`, `nav-bar.styles.ts`); Next reserved lowercase (`page.tsx`, `layout.tsx`, `route.ts`) |
| Dart | `snake_case` | `snake_case.dart` |

Generated code (`proto/`, `*_pb.*`, grpc-web clients) keeps its generated names — never rename it.

## Deployment and local containers (summary)

The canonical text lives in `$cuelabs-delivery-standard`
(`containers-and-deploy.md`); activate it when writing Dockerfiles, compose,
Helm, or Terraform. The structural rules:

- ONE `deploy/helm` chart deploys every service (Envoy config included, never
  a standalone `deploy/envoy/`); `deploy/terraform` installs it through
  kubeconfig with no cloud-specific providers.
- A root `docker-compose.yml` plus the shared compose-driven `Makefile`
  (`make up` / `down` / `logs` / `build`) run the stack locally.
- Host ports: `api/common` → 8080, `web` → 3000, each additional API service
  → 8081, 8082, … in the order it was added.
- Templates for every service type live in `assets/templates/`.

## Cleanup rules (when standardizing)
Remove (safe — not application code):
- **Non-canonical GitHub Actions workflow files**: preserve the ratified
  `.github/workflows/build-and-test.yml`, the deferred tag-gated `release.yml`
  when present, and ratified surface workflows such as the mobile
  `mobile-goldens.yml` and `mobile-e2e.yml`; remove obsolete, duplicate,
  misplaced, or unratified workflow files.
- Buggy/one-off scripts (e.g. old `refactor-structure.sh`).
- Stale planning/aspirational docs that no longer match reality.
- Generated artifacts committed by mistake (e.g. model output images),
  committed build binaries, `tmp/` output.
- Dead `.gitkeep` files in directories that now hold real content.

Never remove:
- **Application code**, service assets/models, or test fixtures.
- Placeholder `.gitkeep`s in genuinely-empty standard dirs (`deploy/*`,
  `mobile/android`, `mobile/ios`, `scripts`).

## Procedure A — standardize an existing repo
1. Branch `chore/standardize-structure`.
2. Move services into place with `git mv` (preserves history). `api/common`
   (Go) stays put — its module path is unaffected. Rename functional services
   (e.g. `reliability-service` → `observability`).
3. `git grep -n <old-path>` and update every reference (Dockerfiles, compose,
   Makefile, docs, CI). Go modules with logical/bare module names are unaffected
   by folder moves; only path-based module names need a `go.mod` + import
   rewrite.
4. Add the community-health/config files from `assets/templates/` (the
   bundled CLI's `apply` copies the missing ones).
5. Create any missing standard dirs (`deploy/{docker,helm,terraform}`, `scripts`)
   with `.gitkeep` placeholders.
6. Apply the cleanup rules above.
7. Verify: build/imports intact, `git status` clean, no app code deleted.
   Open a PR (do not self-merge without review).

## Procedure B — bootstrap a new repo
1. Create the structure above (only the services you actually have).
2. `web`: `npx create-next-app@<version>` (see versions).
3. `api/common`: `go mod init github.com/cuesoftinc/<repo>/api/common` + Gin.
4. `mobile/flutter`: `flutter create`.
5. Add all community-health/config files from `assets/templates/`.
6. Wire `deploy/helm` to deploy every service.
