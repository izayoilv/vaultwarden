# Setup

## Prerequisites

- kudofools cluster with Flux, OpenBao, ESO (see [kudofools-infra](https://forgejo.kudofools.dev/izayoilv/kudofools-infra))
- OpenBao root token + unseal keys (`~/.bao-keys.json` on the node)
- Edge relay with rathole server + socket-activated Caddy (mesh `remote_ip` guard)
- NetBird mesh membership for every client (there is no public access by design)

## 1. Seed secrets into OpenBao

All secrets are managed by OpenBao + ESO. No secrets are committed to Git.

```bash
ROOT_TOKEN=$(jq -r '.root_token' ~/.bao-keys.json)

# Vaultwarden admin token — paste it at https://vaultwarden.kudofools.dev/admin
VAULTWARDEN_ADMIN_TOKEN=$(openssl rand -base64 48)
echo "SAVE THIS - Vaultwarden admin login: $VAULTWARDEN_ADMIN_TOKEN"
kubectl exec -n openbao openbao-0 -- env BAO_TOKEN=$ROOT_TOKEN bao kv put kv/vaultwarden/admin-token \
  admin-token="$VAULTWARDEN_ADMIN_TOKEN"

# Relay client stanza for the vaultwarden tunnel (relay server.toml gets the same token).
# Appended to the existing client.toml that already carries the registry service.
RATHOLE_VW_TOKEN=$(openssl rand -base64 32)
echo "COPY INTO relay server.toml [server.services.vaultwarden]: $RATHOLE_VW_TOKEN"
kubectl exec -n openbao openbao-0 -- env BAO_TOKEN=$ROOT_TOKEN bao kv get -field=client.toml kv/rathole/client.toml > /tmp/rathole-client.toml
cat >> /tmp/rathole-client.toml <<EOF

[client.services.vaultwarden]
token = "$RATHOLE_VW_TOKEN"
local_addr = "vaultwarden.vaultwarden.svc:80"
EOF
kubectl exec -n openbao openbao-0 -- env BAO_TOKEN=$ROOT_TOKEN bao kv put kv/rathole/client.toml \
  client.toml="$(cat /tmp/rathole-client.toml)"
```

## 2. Harden the admin token (Argon2 PHC)

The seeded plaintext token works, but anyone able to read the k8s Secret sees your admin password as-is. Store a hash instead — you keep logging in with the plaintext, the server compares against the hash. Requires a TTY (`-it`, not just `-i`):

```bash
kubectl exec -it -n vaultwarden deploy/vaultwarden -- /vaultwarden hash
# paste the plaintext token when prompted (hidden) → copy the $argon2id$... output
ROOT_TOKEN=$(jq -r '.root_token' ~/.bao-keys.json)
read -rsp "Paste PHC string: " PHC; echo
kubectl exec -n openbao openbao-0 -- env BAO_TOKEN=$ROOT_TOKEN bao kv put kv/vaultwarden/admin-token admin-token="$PHC"
kubectl annotate externalsecret -n vaultwarden vaultwarden-secrets force-sync=$(date +%s) --overwrite
kubectl rollout restart deploy -n vaultwarden vaultwarden
```

`"$PHC"` in double quotes is safe — the `$`s in the hash are not re-expanded. Verify by logging in at `/admin` with the original plaintext afterwards.

## 3. Wire the relay side

```toml
# relay server.toml, then:
# systemctl --user daemon-reload; systemctl --user restart rathole
[server.services.vaultwarden]
token = "<RATHOLE_VW_TOKEN from step 1>"
bind_addr = "0.0.0.0:15001"
```

```caddy
# edge Caddyfile, then caddy reload.
# Mesh-only: anything not arriving with a 100.64.0.0/10 source gets 404.
vaultwarden.kudofools.dev {
	header {
		X-Frame-Options "SAMEORIGIN"
		X-Content-Type-Options "nosniff"
		Referrer-Policy "strict-origin-when-cross-origin"
		-Server
	}
	route {
		@netbird remote_ip 100.64.0.0/10
		handle @netbird {
			reverse_proxy rathole:15001
		}
		respond "Not Found" 404
	}
}
```

NetBird dashboard DNS override: `vaultwarden.kudofools.dev` → edge mesh IP, same as the dashboard entry — so mesh peers arrive with mesh source IPs and pass the guard.

## 4. Force ESO to sync the secrets into the cluster

```bash
kubectl annotate externalsecret -n vaultwarden vaultwarden-secrets force-sync=$(date +%s) --overwrite
kubectl annotate externalsecret -n rathole rathole-client force-sync=$(date +%s) --overwrite
kubectl rollout restart deploy -n rathole rathole-client
```

## 5. Verify mesh-only reachability

```bash
# From a meshed workstation (mesh DNS override active):
curl -k -s -o /dev/null -w "%{http_code}\n" https://vaultwarden.kudofools.dev/          # expect 200
curl -k -s -o /dev/null -w "%{http_code}\n" https://vaultwarden.kudofools.dev/alive     # expect 200
# From the public internet (or without the DNS override):
curl -k -s -o /dev/null -w "%{http_code}\n" https://vaultwarden.kudofools.dev/          # expect 404
```

## 6. Onboard users (invites only)

1. Open `https://vaultwarden.kudofools.dev/admin` from a mesh peer, log in with the token from step 1.
2. Admin → Users → **Invite User** (signups stay disabled; this is the only join path).
3. Invited users set their master password via the emailed link (requires `DOMAIN` + working `SMTP_*` if you add mail later — without SMTP, create the account in the admin panel and share the temporary password out-of-band, user changes it on first login).

## Rotating the admin token

1. Generate + patch `kv/vaultwarden/admin-token` in OpenBao (step 1 pattern).
2. Force ESO sync (step 3 pattern for `vaultwarden-secrets`) and restart Vaultwarden:
   ```bash
   kubectl rollout restart deployment -n vaultwarden vaultwarden
   ```

## Backups

`/data` holds `db.sqlite3`, `attachments/`, and `rsa_key*` — everything needed to restore. Snapshot the `vaultwarden-data-pvc` volume (or `kubectl cp` it out) on a schedule and keep an encrypted offsite copy. Without `rsa_key*`, existing clients cannot decrypt a restored db.
