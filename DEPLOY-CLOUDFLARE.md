# Cloudflare Tunnel deployment (origin IP never exposed)

This is the runbook for putting the stack online through a **Cloudflare Tunnel**,
so the home IP never appears in DNS and no router port is ever opened. It uses
two files added on top of upstream onl-docker:

- `docker-compose.cloudflare.yml` — adds `nginx` + `cloudflared`, drops host ports
- `nginx/templates/osu.conf.template` — Host-based routing to bancho/shiina

Nothing here edits upstream files, so `git pull upstream` stays clean.

> Throughout, `<domain>` is your domain. The real value lives only in `.env`
> (`DOMAIN=...`), which is gitignored — keep it out of tracked files and commits.

---

## 0. Prerequisites

- Domain is active in your Cloudflare account and its nameservers are moved to
  Cloudflare (DNS "Active").
- `.env` has `DOMAIN=<domain>` set (already done).
- Images build and the local stack runs (`make build`, then `make run`).

## 1. Cloudflare dashboard settings — DO THIS FIRST

The osu! client is **not a browser**. If Cloudflare challenges it, connections
fail silently and look like "server down." Before anything else:

- **Security → Bots → Bot Fight Mode: OFF.** This is the single most common
  cause of osu! not connecting.
- **Security → WAF:** no aggressive managed rules / no Browser Integrity Check /
  no CAPTCHA challenge on `c.<domain>`, `osu.<domain>`, `api.<domain>`.
- **SSL/TLS → Overview:** `Full`. (With a tunnel, edge↔origin TLS is handled by
  cloudflared; just don't use anything that forces extra challenges.)

## 2. Create the tunnel and get its token

Cloudflare **Zero Trust → Networks → Tunnels → Create a tunnel**:

1. Connector: **Cloudflared**. Name it (e.g. `osu`).
2. On the install screen, copy the **tunnel token** (the long string after
   `--token` in the shown command). Do **not** run their install command — our
   `cloudflared` container uses the token.
3. Put it in `.env` (gitignored):

   ```
   CF_TUNNEL_TOKEN=<the token>
   ```

## 3. Add public hostnames (auto-creates CNAMEs, never A records)

Still in the tunnel's config, add a **Public Hostname** for each subdomain below.
For every one: **Service type `HTTP`, URL `nginx:80`** (our nginx container routes
by Host header to the right backend). Cloudflare creates a *proxied CNAME* to the
tunnel automatically — no A record, no origin IP anywhere.

| Subdomain | Purpose |
|-----------|---------|
| *(root, blank)* | website (Shiina) |
| `c`  | bancho — chat / multiplayer (**prove this one first**) |
| `ce` | bancho |
| `c4` | bancho |
| `osu`| bancho — `/web`, score submission, etc. |
| `b`  | bancho |
| `api`| bancho / API |
| `i`  | bancho (redirected via `REDIRECT_OSU_URLS`) |
| `s`  | bancho (redirected via `REDIRECT_OSU_URLS`) |
| `a`  | avatars (nginx static) |
| `assets` | static assets (nginx static) |

(`tools/generate_cf_dns_records.sh` in the bancho.py-ex repo can bulk-generate
these if you'd rather import records than click — but with a tunnel, adding
public hostnames is what creates the CNAMEs, so the dashboard route is simplest.)

## 4. Start the stack with the tunnel overlay

```bash
cd ~/onl-docker
make run COMPOSE="docker compose -f docker-compose.yml -f docker-compose.cloudflare.yml"
```

(or detached: `docker compose -f docker-compose.yml -f docker-compose.cloudflare.yml up -d`)

## 5. Verify — prove `c.<domain>` before building anything on top

```bash
# tunnel came up and registered 4 edge connections:
docker logs onl-cloudflared --tail 30        # look for "Registered tunnel connection"

# bancho is reachable through the tunnel:
curl -sI https://c.<domain>/                  # expect a response from bancho, not a CF challenge page
```

If `curl` returns a Cloudflare HTML challenge instead of bancho, go back to
step 1 (Bot Fight Mode / WAF).

Then in-game: launch `osu!.exe -devserver <domain>`.
Note: osu!'s "force repair" relaunches in updater mode and **discards
`-devserver`** — that looks like the server is down but isn't.

## Why this keeps the origin IP hidden

- `cloudflared` only makes **outbound** connections to Cloudflare (port 7844).
  Nothing inbound; no router port forwarding.
- DNS entries are **CNAMEs to `<tunnel-id>.cfargotunnel.com`** — they contain no
  IP address. There is no A record pointing at the home connection.
- bancho/shiina publish **no host ports** (the overlay's `ports: !reset`), so
  even on the LAN they're only reachable inside the Docker network via nginx.
