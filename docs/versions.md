# Runtime versions and pinning policy

This document tracks the container image versions we pin across dev and prod Compose stacks, and the policy for updating them.

Policy
- Prefer stable/LTS majors, and always pin to the latest patch/minor for that major.
- Avoid `:latest` to keep reproducible environments.
- Review/update quarterly or on security advisories.

Pinned images (as of 2025-10-30)
- Keycloak: quay.io/keycloak/keycloak:26.4.2 (was 24.0)
- PostgreSQL: postgres:18.0 (no data yet; safe major bump; follow minors)
- Redis: redis:7.4.6 (stay on 7.x for broad client compatibility)
- Traefik: traefik:v3.5.4
- Grafana Alloy: grafana/alloy:1.11.3
- Grafana Tempo: grafana/tempo:2.9.0
- Grafana Loki: grafana/loki:3.5.7 (upgrade from 2.9.x)
- Grafana Mimir: grafana/mimir:2.17.2
- Grafana OSS: grafana/grafana-oss:12.2.1

Notes
- PostgreSQL: follow upstream guidance—always run the current minor in your chosen major.
- Keycloak 26.x: check release notes if you customize realms/providers; defaults continue to work for our dev/prod usage.
- Loki 3.x: default single-binary runs are compatible; revisit if you bring in custom config.
- Tempo/Mimir/Alloy: pinned to latest stable releases from upstream.

Update process
1) Bump tags in `infra/compose/**/docker-compose*.yml`.
2) Validate configs: `podman compose -f <file> config`.
3) Changelog skim for breaking changes; test dev stacks locally.
4) Roll through environments with a short smoke test (Traefik routes, Grafana dashboards, basic traces/logs/metrics).
