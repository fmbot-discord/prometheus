# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

This repo is the monitoring stack for the fmbot Discord bot — Prometheus + Grafana + cAdvisor + node-exporter + Pushgateway + Seq + postgres-exporter, deployed as a Docker Swarm stack. It started as a fork of `vegasbrianc/prometheus`; the README has since been rewritten to describe this stack specifically. Alerting is Grafana-managed (Grafana ships its own internal Alertmanager), so the upstream `prom/alertmanager` service is not deployed.

The parent `P:\fmbot\CLAUDE.md` covers the broader fmbot workspace and lists this folder under "Monitoring & Observability" — refer to it for context on what the bot itself emits metrics about.

## Deployment

Single-command swarm deploy (all services on the manager node, plus `mode: global` services on every node):

```bash
docker stack deploy -c docker-stack.yml prom   # deploy / update
docker stack rm prom                           # tear down
docker stack ps prom                           # status
docker service logs prom_<service_name>        # logs (e.g. prom_prometheus, prom_grafana)
```

**Required environment variables** (typically in `~/.bashrc` on the swarm manager):
- `SEQ_PASSWORD` — bcrypt hash for the Seq admin (`frikandel`), interpolated into `SEQ_FIRSTRUN_ADMINPASSWORDHASH`
- `GRAFANA_PASSWORD` — plaintext, interpolated into Grafana's `GF_SECURITY_ADMIN_PASSWORD` via `grafana/config.monitoring`
- `POSTGRES_EXPORTER_CONNSTRING` — Postgres DSN for the metrics-only `postgres_exporter` role, e.g. `postgresql://postgres_exporter:PASSWORD@host.docker.internal:5432/fmbot?sslmode=disable`. The role is created with `GRANT pg_monitor` — read-only access to stats/settings, no user data.

Without these set, Grafana/Seq start with empty credentials and postgres-exporter fails to connect.

**Required swarm secrets** (`docker secret create` once on the manager):
- `pagerduty_routing_key` — PagerDuty Events API v2 integration key. Mounted into the Grafana container at `/run/secrets/pagerduty_routing_key`; the provisioned PagerDuty contact point in `grafana/provisioning/alerting/contactpoints.yaml` references it via `$__file{...}`. Rotate with `docker secret rm` + recreate, then `docker service update --force prom_grafana`.

## Services & Ports

Defined in `docker-stack.yml`:

| Service | Image | Port(s) | Notes |
|---------|-------|---------|-------|
| prometheus | prom/prometheus:latest | 9090 | 180d retention, 8G mem / 3 CPU limit, manager-only |
| grafana | grafana/grafana | 3000 | Admin user is `frikandel`, manager-only, runs as uid 472. Holds all alerting (rules + contact points + notification policies) via internal Alertmanager. |
| cadvisor | gcr.io/cadvisor/cadvisor | 8080 | `mode: global` (every node) |
| node-exporter | prom/node-exporter | 9100 | `mode: global` |
| pushgateway | prom/pushgateway | 9091 | `mode: global` — referenced via `tasks.pushgateway` DNS |
| seq | datalust/seq | 7373 (UI), 5341 (ingest) | Structured log sink for Serilog from the bot |
| postgres-exporter | quay.io/prometheuscommunity/postgres-exporter | _none (swarm-internal only)_ | `mode: global`; reaches Postgres on the swarm host via `host.docker.internal` (see `extra_hosts:`). Scraped by Prometheus over the overlay network at `postgres-exporter:9187`. Feeds the `postgresql` Grafana dashboard. |

## Configuration Files

- **`prometheus/prometheus.yml`** — Scrape config. Jobs: `prometheus`, `cadvisor`, `node-exporter` (DNS SD via `tasks.node-exporter`), `pushgateway` (DNS SD, `honor_labels: true`), `postgres-exporter`. Global scrape interval 15s; node-exporter is 5s. The bot itself is **not** scraped here — the bot pushes to Pushgateway and ships logs to Seq.
- **`grafana/config.monitoring`** — Grafana env file. Sign-up disabled, feature toggles `kubernetesDashboards,dashboardNewLayouts` enabled.
- **`grafana/provisioning/datasources/datasource.yml`** — Auto-provisions the `Prometheus` datasource pointing at `http://prometheus:9090` as default. Don't add it via the UI; that produces a duplicate.
- **`grafana/provisioning/dashboards/dashboard.yml`** — Provisioner that imports any JSON dashboard placed alongside it under `/etc/grafana/provisioning/dashboards`.
- **`grafana/provisioning/alerting/Alerting group.yaml`** — Provisioned alert rules (Grafana-managed). Each top-level entry is a rule group; folder/group naming is reflected in the Grafana Alerting UI.
- **`grafana/provisioning/alerting/contactpoints.yaml`** — Provisioned contact points. PagerDuty receiver reads its integration key from the swarm secret via `$__file{/run/secrets/pagerduty_routing_key}` and targets the EU events endpoint. Notification routing (which alert goes to which contact point) is configured via the Grafana UI rather than provisioned, to avoid clobbering existing routes.

## Dashboards: Two Locations (Don't Confuse Them)

- **`grafana/provisioning/dashboards/*.json`** — auto-provisioned at Grafana startup. Drop a dashboard JSON here and it will appear after a service redeploy.
- **`dashboards/*.json`** — standalone dashboard exports (`Grafana_Dashboard.json`, `Grafana_Dashboard_prom_2.json`, `System_Monitoring.json`). These are **not** auto-provisioned — they're committed snapshots/manual-import sources. To make a dashboard live, copy or move it into `grafana/provisioning/dashboards/`.

When the user says "update the dashboard" (per recent commit history), check both locations.

## Editing Conventions

- This repo deploys to a real swarm — changes to `docker-stack.yml`, the prometheus config, or provisioning files take effect on the next `docker stack deploy`. Treat redeploys like a shared-infrastructure action.
- Prometheus config changes require a service restart (no live reload flag is set). `docker service update --force prom_prometheus` reloads after editing `prometheus/prometheus.yml`.
- `prometheus_data` and `grafana_data` are named volumes — bumping image tags is safe; deleting these volumes wipes 180 days of metrics and all Grafana state (saved dashboards edited via UI, users, API keys).
