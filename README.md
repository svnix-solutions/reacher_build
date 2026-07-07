# 🦎 Reacher Email Verification — Komodo Stack

A production, queue-based **email verification** stack built on
[Reacher / `check-if-email-exists`](https://github.com/svnix-solutions/check-if-email-exists)
(our fork of [reacherhq/check-if-email-exists](https://github.com/reacherhq/check-if-email-exists)).
Checks whether an email address exists **without sending any email** — syntax,
MX/DNS, disposable-address, SMTP reachability, catch-all, role account and more.

By default the stack runs the public `reacherhq/backend` image. Optionally, you
can run your own build compiled from the source repo
[`svnix-solutions/check-if-email-exists`](https://github.com/svnix-solutions/check-if-email-exists)
— see [Deploy with Komodo](#deploy-with-komodo).

## What's in the stack

| Service    | Image                     | Purpose                                                            |
| ---------- | ------------------------- | ----------------------------------------------------------------- |
| `rabbitmq` | `rabbitmq:4.0-management` | Job queue feeding the workers (bulk `/v1` endpoints). UI on `15672`. |
| `postgres` | `postgres:16-alpine`      | Stores bulk jobs and per-email verification results.              |
| `reacher`  | `reacherhq/backend:latest` | Reacher backend — HTTP API **and** queue worker. Serves `:8080`. |

- **`POST /v0/check_email`** — single, immediate verification (no queue, not throttled).
- **`POST /v1/check_email`** — single verification via the throttled `/v1` path.
- **`POST /v1/bulk`** + **`GET /v1/bulk/{job_id}[/results]`** — enqueue a batch;
  workers drain RabbitMQ and persist results to Postgres.
- **`GET /version`** — unauthenticated health/version endpoint.

Full API: [Reacher OpenAPI](https://docs.reacher.email/advanced/openapi).

## ⚠️ Outbound port 25

SMTP verification connects to remote mail servers on **TCP port 25**. Most cloud
providers (AWS, GCP, Azure, DigitalOcean, …) block outbound `:25` by default. If
so, verifications return `unknown`. Options:

- Run on a host/provider where outbound `:25` is open, **or**
- Route SMTP through a SOCKS5 proxy — set `REACHER_PROXY_*` in `.env`
  (see [proxy25.com](https://proxy25.com)).

Also set **`REACHER_HELLO_NAME`** / **`REACHER_FROM_EMAIL`** to a domain whose
reverse-DNS (PTR) points back at this host's public IP — it materially improves
result accuracy.

## Quick start (local Docker)

```bash
cp .env.example .env
# edit .env — set REACHER_HEADER_SECRET, RABBITMQ_PASS, POSTGRES_PASSWORD,
# REACHER_HELLO_NAME, REACHER_FROM_EMAIL

# Exposing through Cloudflare Tunnel? Create the shared network once
# (same name the cloudflared stack uses):
docker network create cloudflared   # matches CLOUDFLARE_TUNNEL_NETWORK

docker compose up -d
docker compose logs -f reacher
```

Smoke test (include `x-reacher-secret` only if you set `REACHER_HEADER_SECRET`):

```bash
curl http://localhost:8080/version

curl -X POST http://localhost:8080/v0/check_email \
  -H 'content-type: application/json' \
  -H 'x-reacher-secret: <your-secret>' \
  -d '{"to_email": "someone@gmail.com"}'
```

Bulk job:

```bash
# Enqueue
curl -X POST http://localhost:8080/v1/bulk \
  -H 'content-type: application/json' \
  -H 'x-reacher-secret: <your-secret>' \
  -d '{"input": ["a@example.com", "b@example.com"]}'
# → { "job_id": 1 }

# Poll status / fetch results
curl http://localhost:8080/v1/bulk/1 -H 'x-reacher-secret: <your-secret>'
curl http://localhost:8080/v1/bulk/1/results -H 'x-reacher-secret: <your-secret>'
```

## Deploy with Komodo

This repo holds only the deployable stack (`compose.yaml` + `.env.example`).
Komodo Resource Sync definitions (the `[[stack]]` / `[[build]]` / `[[procedure]]`
TOML) live in our separate Komodo resources repo, not here.

Point a Komodo **Stack** at this repo with `file_paths = ["compose.yaml"]`, set
`run_build = false` so it just pulls `${REACHER_IMAGE}` (default
`reacherhq/backend:latest`), and set the secrets — `REACHER_HEADER_SECRET`,
`RABBITMQ_PASS`, `POSTGRES_PASSWORD` — in the stack **Environment** in the UI.

To run your own image instead of the public one, build the backend from the
`svnix-solutions/check-if-email-exists` source repo (`backend/Dockerfile`) in a
separate Komodo Build and set `REACHER_IMAGE` to your pushed tag.

## Configuration

All configuration is via environment variables in `.env` (mapped to Reacher's
`RCH__*` variables in `compose.yaml`). Key ones:

| `.env` variable          | Default                | Notes                                                       |
| ------------------------ | ---------------------- | ----------------------------------------------------------- |
| `REACHER_IMAGE`          | `reacherhq/backend:latest` | Public image, or your own build.                        |
| `REACHER_HEADER_SECRET`  | *(empty)*              | `x-reacher-secret` value. Set it if the API is exposed.     |
| `REACHER_HELLO_NAME`     | `mail.example.com`     | SMTP EHLO name — match host PTR.                            |
| `REACHER_FROM_EMAIL`     | `verify@example.com`   | SMTP `MAIL FROM`.                                           |
| `REACHER_CONCURRENCY`    | `5`                    | Emails verified in parallel per instance.                  |
| `REACHER_REPLICAS`       | `1`                    | Scale workers. If > 1, drop the host port (use the tunnel). |
| `REACHER_MAX_RPM`/`_RPD` | `60` / `10000`         | Per-worker throttle on `/v1/*`.                            |
| `REACHER_PROXY_*`        | *(empty)*              | SOCKS5 proxy for SMTP when `:25` is blocked.               |

Reacher exposes far more knobs (per-provider verification method overrides,
multiple named proxies, Sentry, headless Chrome for Yahoo/Hotmail B2C) via its
[`backend_config.toml`](https://github.com/svnix-solutions/check-if-email-exists/blob/main/backend/backend_config.toml).
The Docker image bundles Chrome + chromedriver for headless verification.

## Exposing through Cloudflare Tunnel

The `reacher` service joins the shared external **`cloudflared`** network (from
the `cloudflare_tunnel_build` stack), so the tunnel reaches it directly — no
published ports, no inbound firewall rules.

1. Make sure the `cloudflared` stack is running and the network exists:
   ```bash
   docker network create cloudflared      # once per host, if not already created
   ```
   Set `CLOUDFLARE_TUNNEL_NETWORK` in `.env` to match whatever name that stack uses.
2. In **Cloudflare Zero Trust → Networks → Tunnels → your tunnel → Public Hostnames**,
   add a hostname pointing at the reacher container:

   | Field     | Value                          |
   | --------- | ------------------------------ |
   | Subdomain | `reacher` (or your choice)     |
   | Domain    | `example.com`                  |
   | Service   | `http://reacher-backend:8080`  |

   The service host is the container name — `${COMPOSE_PROJECT_NAME}-backend`,
   i.e. `reacher-backend` with the default project name. Docker's embedded DNS
   resolves it on the `cloudflared` network.
3. Since the tunnel is the only ingress you need, you can drop the `ports:`
   mapping on the `reacher` service to keep the API fully network-internal.

> Because Cloudflare terminates TLS at its edge and the tunnel is outbound-only,
> keep `REACHER_HEADER_SECRET` set — it's the API's only auth layer. Consider a
> Cloudflare Access policy on the hostname for an extra gate.

## Scaling

Each `reacher` instance is both an API and a worker. To add throughput, raise
`REACHER_REPLICAS` (or run the stack on multiple Komodo servers pointing at the
same RabbitMQ + Postgres). When `REACHER_REPLICAS > 1`, remove the `ports:`
mapping on the `reacher` service and reach it through the `cloudflared` network
— a published host port can't be shared across replicas. Give each SMTP-verifying
instance its own outbound IP (or proxy) and keep the per-day throttle sane to
avoid your IPs being blocked by mail providers.

## Notes

- Named volumes `rabbitmq_data` and `postgres_data` persist across restarts.
- The `tunnel` network is declared **external** — create it (or start the
  `cloudflared` stack) before deploying, or point `CLOUDFLARE_TUNNEL_NETWORK` at
  your existing tunnel network.
- Reacher is dual-licensed (AGPL-3.0 / commercial). Review the
  [license terms](https://github.com/svnix-solutions/check-if-email-exists/blob/main/LICENSE.md) before commercial use.
