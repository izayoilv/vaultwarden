# vaultwarden

[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Vaultwarden](https://img.shields.io/badge/server-Vaultwarden-175DDC.svg)](https://github.com/dani-garcia/vaultwarden)
[![Vaultwarden](https://img.shields.io/badge/domain-vaultwarden.kudofools.dev-purple.svg)](https://vaultwarden.kudofools.dev)
[![Database](https://img.shields.io/badge/database-SQLite-6b9d4e.svg)](https://sqlite.org)
[![Image](https://img.shields.io/badge/image-1.37.2-2ea44f.svg)](https://hub.docker.com/r/vaultwarden/server)

Flux-managed [Vaultwarden](https://github.com/dani-garcia/vaultwarden) (Bitwarden-compatible) password manager on the kudofools k3s cluster.

## Features

- Vault: `vaultwarden.kudofools.dev` (Bitwarden clients point at this base URL)
- **Mesh-only access**: no public ingress — served through Caddy behind a `100.64.0.0/10` source guard, relayed by a rathole TCP tunnel from the cluster
- Signups and invitations disabled (`SIGNUPS_ALLOWED=false`, `INVITATIONS_ALLOWED=false`); accounts join via admin-panel invites only
- Admin token injected from OpenBao via External Secrets Operator
- Runs with read-only root filesystem, dropped capabilities, no privilege escalation

## Structure

```
clusters/default/
├── namespace.yaml          # vaultwarden namespace
├── pvc.yaml                # 1Gi local-path data volume (sqlite + attachments)
├── externalsecret.yaml     # admin token from OpenBao
├── deployment.yaml         # vaultwarden/server:1.37.2
├── service.yaml            # vaultwarden:80
└── kustomization.yaml
```

Exposure (wired outside this repo):

- `kudofools-infra/clusters/default/vaultwarden.yaml` — Flux `GitRepository` + `Kustomization`
- `kudofools-infra/opentofu/openbao.tf` — `vaultwarden` read policy on `kv/data/vaultwarden/*`
- `kudofools-infra/opentofu/cloudflare.tf` — DNS-only `vaultwarden` A record → public edge
- NetBird dashboard DNS override — `vaultwarden.kudofools.dev` → edge mesh IP (so mesh peers arrive with mesh source IPs)
- Relay `server.services.vaultwarden` → `0.0.0.0:15001`, relay client forwards `vaultwarden.vaultwarden.svc:80`
- Caddy `vaultwarden.kudofools.dev` → `rathole:15001`, mesh `remote_ip` guard

## Configuration

| Setting | Value | Notes |
|---|---|---|
| `DOMAIN` | `https://vaultwarden.kudofools.dev` | Required for invites/attachments URLs |
| `ROCKET_ADDRESS` / `ROCKET_PORT` | `0.0.0.0` / `80` | Container needs non-loopback bind |
| `SIGNUPS_ALLOWED` | `false` | No public registration, ever |
| `INVITATIONS_ALLOWED` | `false` | Users join via admin invite only (flip temporarily to onboard) |
| `SHOW_PASSWORD_HINT` | `false` | Don't leak hints on the login page |
| `ADMIN_TOKEN` | from `vaultwarden-secrets` | OpenBao `kv/vaultwarden/admin-token` → `admin-token` |
| Data | `/data` on PVC | sqlite db, attachments, `rsa_key*` |

Deployment-level:

| Setting | Value |
|---|---|
| Image | `docker.io/vaultwarden/server:1.37.2` |
| Health | `readinessProbe`/`livenessProbe` on `/alive` |
| Tmp | `emptyDir` on `/tmp` (Rocket multipart uploads under read-only root) |

## Setup

Full onboarding steps (seeding the admin token and the relay client stanza into OpenBao, ESO sync, admin login, inviting users) are in [SETUP.md](SETUP.md).

## Operations

| Task | Command |
|---|---|
| Check health | `kubectl get pods -n vaultwarden` |
| View logs | `kubectl logs -n vaultwarden deploy/vaultwarden` |
| Rotate admin token | patch `kv/vaultwarden/admin-token` in OpenBao → force ESO sync → rollout restart |
| Back up | `kubectl cp` or snapshot `/data` (db.sqlite3, attachments/, rsa_key*) — encrypted offsite copy |
| Reach admin panel | `https://vaultwarden.kudofools.dev/admin` (mesh peer only) |

## License

[MIT](LICENSE) © 2026 IzayoiLv
