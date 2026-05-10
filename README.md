# .fmbot monitoring stack

Prometheus + Grafana + Alertmanager + cAdvisor + node-exporter + Pushgateway + Seq, deployed as a Docker Swarm stack. Originally forked from `vegasbrianc/prometheus`.

The Grafana dashboard is not publicly accessible.

## Deploy

    docker stack deploy -c docker-stack.yml prom

## Remove

    docker stack rm prom

## Logs for a service

    docker service logs prom_prometheus

## Required env vars (e.g. in `~/.bashrc` on the swarm manager)

- `SEQ_PASSWORD` — bcrypt hash for the Seq admin user
- `GRAFANA_PASSWORD` — Grafana admin password
- `POSTGRES_EXPORTER_CONNSTRING` — Postgres DSN for postgres-exporter, e.g. `postgresql://postgres_exporter:PASSWORD@host.docker.internal:5432/fmbot?sslmode=disable`

## Required swarm secrets

Alertmanager reads the PagerDuty routing key from a Docker swarm secret. Create it once on the manager node before the first deploy:

    printf "%s" "<pagerduty-events-v2-integration-key>" | docker secret create pagerduty_routing_key -

To rotate: `docker secret rm pagerduty_routing_key`, recreate, then `docker service update --force prom_alertmanager`.
