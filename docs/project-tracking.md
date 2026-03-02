# Project Tracking — GitHub Project + Issues

This guide outlines how to manage delivery using GitHub Projects and Issues, aligned with our roadmap.

## 1) Create the GitHub Project (v2)
- Create an organization-level project (recommended) or repository-level if preferred.
- Template: Board or Table. Suggested fields:
  - Status (Backlog, Ready, In Progress, In Review, Done)
  - Priority (P0–P3)
  - Size (S/M/L)
  - Phase (0–11)
  - Area (api, web, ios, infra, docs)
  - Target (alpha, beta, mvp)

## 2) Labels
- `area/api`, `area/web`, `area/ios`, `area/infra`, `area/docs`
- `type/feature`, `type/bug`, `type/chore`
- `priority/p0`, `priority/p1`, `priority/p2`, `priority/p3`
- `good-first-issue`, `help-wanted`

## 3) Issue templates
Create under `.github/ISSUE_TEMPLATE/`:
- `story.yml` — for user stories
  - Fields: Context, Acceptance Criteria, API/DB touchpoints, Out of Scope, QA notes
- `bug.yml` — for defects
  - Fields: Steps to Repro, Expected/Actual, Logs, Environment

## 4) PR template
- `.github/PULL_REQUEST_TEMPLATE.md`
  - Sections: Summary, Screenshots, Linked Issues, Testing, Risk, Checklist (CI green, migrations applied, docs updated)

## 5) Automation
- Auto-add new issues/PRs to the Project via workflow:
  - `.github/workflows/project-auto-add.yml`
- Enforce labels and semantic PR titles with a linter action.
- Require at least 1 reviewer; block on failing checks.

## 6) Backlog seeding (from roadmap)
Create issues for each Story [S] and Task [T] in `docs/roadmap.md`. Suggested titles:
- `[S][P1][Phase 1] OIDC login with Keycloak (web)`
- `[S][P1][Phase 1] AppAuth login (iOS)`
- `[S][P1][Phase 1] API shell with health and OTEL`
- ...continue for each story in Phases 2–11…

## 7) Definition of Done
- Feature behind CI‑passing PR, reviewed and merged
- Acceptance criteria met; linked issues closed
- Telemetry added where applicable; docs updated

## 8) Releases and milestones
- Milestones: Phase completion (e.g., Phase 4: Lending core)
- Tags: lightweight release tags at meaningful checkpoints

## 9) Triage cadence
- Weekly triage: review board, unblock items, re‑prioritize, prune backlog

## 10) Useful views
- By Phase: columns grouped by Phase
- By Area: swimlanes across api/web/ios
- Cycle time: enable insights to monitor lead time and throughput
