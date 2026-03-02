# Repository Organization — Monorepo vs Multi‑repo

This document finalizes how we organize code and operations for MVP and sets clear criteria to revisit later.

## TL;DR decision
- Choose a single monorepo for MVP (backend API, Angular web, iOS app, infra, docs). This best supports tightly coupled feature work, shared contracts, and simpler local dev/CI.

## Decision drivers
- Cross‑cutting features: many stories touch API + web + iOS simultaneously (scan → lookup → handover).
- Shared contracts: OpenAPI DTOs and event shapes must evolve together without drift.
- Small team cadence: one backlog and uniform governance outweigh polyrepo overhead now.
- Local dev: one `podman compose` environment and consistent scripts speed onboarding.

## When to reconsider (triggers)
- Teams scale to independent release cadences per surface (e.g., mobile outsourced or separate cadence).
- Compliance or IP boundaries require separate ACLs/secrets.
- CI time or artifact sizes become problematic for a single repo.

## Monorepo layout (proposed)

/
- api/ — .NET 10 solution
  - src/Api/
  - src/Application/
  - src/Domain/
  - src/Infrastructure/
  - tests/
  - Containerfile (or Dockerfile)
- web/ — Angular app (mobile‑first)
  - src/
  - e2e/
  - Containerfile (or Dockerfile)
- ios/ — Swift/SwiftUI app
  - Sources/
  - Tests/
- infra/
  - compose/app/ (api, web, keycloak, postgres, redis)
  - compose/observability/ (alloy, tempo, loki, mimir, grafana)
  - keycloak/ (realm export, seed scripts)
- docs/
  - spec.md, roadmap.md, db-*.*, api-endpoints.md, OPEN_TOPICS.md
- schemas/
  - openapi/ (generated spec + client SDKs)
  - db/ (EF migrations snapshots)
- .github/
  - workflows/ (CI, lint, test, build, reusable actions)
  - ISSUE_TEMPLATE/ (bug, story)
  - PULL_REQUEST_TEMPLATE.md
  - CODEOWNERS
  - copilot-instructions.md
- tools/
  - scripts for local dev, linting, codegen

Notes
- Optional: devcontainer/ for VS Code development container.
- Optional: nx.json only if we later add shared web libraries; not required initially.

## CI/CD blueprint

Workflows (split by area with path filters; reusable where possible):
- .github/workflows/ci-api.yml
  - Triggers: push/pull_request on paths: api/**, schemas/openapi/**, .github/**
  - Jobs: setup-dotnet, restore, build, test, generate OpenAPI, upload artifact, container build/push (if on main)
- .github/workflows/ci-web.yml
  - Triggers: paths: web/**, schemas/openapi/**, .github/**
  - Jobs: setup-node, install, lint, test, build, container build/push (if on main)
- .github/workflows/ci-ios.yml
  - Triggers: paths: ios/**, .github/**
  - Runner: macos-latest; Jobs: xcodebuild test, swiftlint/format; optional TestFlight via Fastlane on tags
- .github/workflows/ci-infra.yml
  - Triggers: paths: infra/**, .github/**
  - Jobs: compose config validation, yamllint, hadolint
- .github/workflows/ci-monorepo.yml
  - Triggers: paths: docs/**, tools/**, README.md
  - Jobs: markdownlint, link check

Path filter example (YAML snippet)
```yaml
on:
  pull_request:
    paths:
      - 'api/**'
      - 'schemas/openapi/**'
      - '.github/**'
```

Caching and concurrency
- .NET: actions/setup-dotnet cache; key includes global.json and *.csproj hashes.
- Node: actions/setup-node with npm/pnpm cache; key includes lockfile.
- iOS: cache DerivedData where practical; be mindful of macOS runner limits.
- Use `concurrency: group: ${{ github.workflow }}-${{ github.ref }} cancel-in-progress: true` to avoid duplicate runs.

Artifacts and releases
- Container images per service: ghcr.io/<org>/com-lib-api, com-lib-web.
- Tags: :sha, :main, and :phase-N (manual promotion).
- OpenAPI JSON published as build artifact and committed to `schemas/openapi/` via a controlled PR.

Secrets and environments
- Use GitHub Environments for staging/prod with required reviewers.
- Store Keycloak client secrets, DB passwords in Actions Secrets; never commit secrets.

## CODEOWNERS and governance

CODEOWNERS example
```
# Areas
/api/ @your-org/backend
/web/ @your-org/frontend
/ios/ @your-org/ios
/infra/ @your-org/infra
/docs/ @your-org/pm @your-org/tech-writers
```

Branching & PRs
- Trunk‑based with short‑lived feature branches: `feat/<area>/<topic>` (e.g., `feat/api/loan-request`).
- Require passing CI, at least 1 reviewer from relevant CODEOWNERS, semantic PR titles.
- Conventional Commits encouraged (e.g., `feat(api): add loan request endpoint`).

## OpenAPI and shared contracts
- Generate OpenAPI in API build; publish JSON to `schemas/openapi/`.
- Optional client SDK generation for Angular/Swift; commit generated clients under `web/src/generated/` and `ios/Sources/Generated/` to avoid drift.

## Local development
- `infra/compose/app`: brings up api, web, keycloak, postgres, redis.
- `infra/compose/observability`: Alloy, Tempo, Loki, Mimir, Grafana on shared `telemetry-net`.
- `.env` and `.env.local` supply local values; production via secrets/CI.

## Alternative now or later: Multi‑repo
- com-lib-api (backend + infra manifests)
- com-lib-web (Angular)
- com-lib-ios (iOS)
- com-lib-ops (observability/compose) [optional]

Pros
- Independent releases and ACLs; lighter clones and CI per repo.

Cons (for current stage)
- Cross‑repo changes are brittle; increased coordination tax.
- Harder local e2e dev; duplicated CI/glue.

## Migration path if we split later
1) Freeze main; ensure CI green.
2) Create split branches using git subtree (preserves history):
   - `git subtree split -P api -b split/api`
   - `git subtree split -P web -b split/web`
   - `git subtree split -P ios -b split/ios`
3) Push to new repos:
   - `git push git@github.com:org/com-lib-api split/api:main`
   - `git push git@github.com:org/com-lib-web split/web:main`
   - `git push git@github.com:org/com-lib-ios split/ios:main`
4) Optionally keep this repo as an orchestration repo for `infra/compose/*`.
5) Set up cross‑repo automation: dependabot for OpenAPI clients, release tags → dispatch builds.

## Answering the key question
- Does it make sense to keep every part separate now? No for MVP—the coordination cost outweighs benefits. We’ll keep everything together in one monorepo, with clear folder boundaries, CODEOWNERS, and path‑filtered CI so each area stays manageable. We’ll revisit when the split triggers above are met.
