---
title: "Three repos, one image, a tag, and a bump: how the finance stack ships"
description: "An exhaustive walkthrough of how Kindred, plaid-sync, and finance-hub are built, stored, and deployed onto a single homelab host -- DNS, TLS, GHCR, the bump recipe, the env_file trap, and why postgres still needs DAC_OVERRIDE."
date: "2026-05-10"
tags: ["Homelab", "Docker", "Traefik", "Tailscale", "GitHub Actions", "GHCR", "SOPS"]
draft: true
---

The personal-finance corner of my homelab is three repos and three containers on a single host. They share a deployment pattern down to the filenames: one image per repo, one tag per release, one `just bump <tag>` to roll it. Nothing auto-deploys. Everything is reachable only on the tailnet, except for one webhook receiver that has to be public because Plaid can't dial into Tailscale.

This post walks through the whole thing end to end. What gets built where, where the image lives, how DNS resolves to ares, what Traefik does with the request, what middleware decides whether to let it through, what runs on disk, and what `just bump v0.3.0` actually executes when I run it. The intent is that someone reading this could rebuild it from scratch, or that future-me could remember which knob to turn six months from now.

There are some tradeoffs I'll flag along the way. The biggest is that I picked manual deploys over Watchtower or webhooks or Portainer. For a single operator with low release cadence (a few times a week at most) the simplicity is worth more than the automation.

## The three repos

- **`pike00/Kindred`** -- a personal CRM, public on GitHub, deployed at `kindred.lab.example.com`. FastAPI + arq worker + React/Vite frontend + Postgres 18 + Redis + Meilisearch. Source lives at `~/projects/personal-crm`.
- **`pike00/plaid-sync`** -- a Plaid-to-beancount sync engine, private. Deployed at `plaid-sync.lab.example.com` for the UI and `plaid-sync-hooks.lab.example.com` (public) for webhooks. Source at `~/projects/plaid-sync`.
- **`pike00/finance-hub`** -- a beancount reconciliation UI sitting on top of the ledger, private. Deployed at `finance-hub.lab.example.com`. Source at `~/projects/finance-hub`.

Each repo is its own thing. They share data only through the filesystem: plaid-sync writes flagged beancount entries into `~/Documents/Finance/Ledger`, finance-hub reads and rewrites them, both via host bind mounts. There's also a TCP connection on the docker network from plaid-sync to the finance-hub postgres for staging tables. No shared image, no shared volume, no shared database credentials in the same `.env.sops`.

The deployment configuration for all three lives in a fourth repo, the homelab repo (`~/Documents/Homelab`), under `apps/kindred/`, `apps/plaid-sync/`, and `apps/finance-hub/`. Each app directory has the same five files: `docker-compose.yml`, `.env`, `.env.sops`, `justfile`, and a `manual-dumps/` directory for the pg-dumps that `just bump` produces. The compose files in the homelab repo are not the same as the ones in the source repos -- the source repos carry `compose.yml` (canonical local prod), `compose.dev.yml` (loopback-friendly dev), `compose.worktree.yml` (per-worktree review apps), and `Dockerfile.prod`. The homelab repo only has the deployment overlay: which image tag to roll, which Traefik labels to attach, which host paths to bind in.

This split is deliberate. The application code doesn't know which host it runs on or which domain it serves. The homelab repo knows that ares serves it at `kindred.lab.example.com` and that `~/Documents/Finance` is the canonical ledger path. Moving the stack to a different host would touch the homelab repo only.

## DNS: two paths to the same Traefik

Everything except `plaid-sync-hooks` resolves to ares's Tailscale IP, `100.119.100.85`. There are two paths that get a client there.

**Tailnet clients** resolve via NextDNS, which I run as my recursive resolver for every device on the tailnet. The NextDNS Terraform module pins three rewrites:

```hcl
rewrite {
  domain  = "kindred.${var.lab_domain}"
  address = var.homelab_tailnet_ip
}
rewrite {
  domain  = "plaid-sync.${var.lab_domain}"
  address = var.homelab_tailnet_ip
}
rewrite {
  domain  = "finance-hub.${var.lab_domain}"
  address = var.homelab_tailnet_ip
}
```

The rewrite block makes the host AND any subdomain resolve to the tailnet IP. So `feature-x.kindred.lab.example.com` and `slug-1234.plaid-sync.lab.example.com` (worktree review apps; see below) also point at ares.

**Off-tailnet clients** -- for example, a phone that briefly drops off the tailnet -- get the same answer via Cloudflare DNS. The `khanpikehome.com` zone holds an unproxied A record for each name:

```hcl
resource "cloudflare_dns_record" "kph_kindred" {
  zone_id = local.lab_zone_id
  name    = "kindred"
  content = "100.119.100.85"
  type    = "A"
  ttl     = 3600
  proxied = false
}
```

`proxied = false` is the important part. If the record were proxied through Cloudflare, the traffic would try to enter the Cloudflare tunnel, fail to reach the internal route, and 404. Unproxied means the public DNS answer is the Tailscale CGNAT address. Anything not on the tailnet can't reach it: 100.64.0.0/10 is unroutable on the public internet. The DNS record leaks the topology -- "there is a service here at this name" -- but the IP is useless without tailnet membership. This is what `tier: tailnet_dns_service` means in the homelab's services registry.

There's also an explicit `kindred.dev` record on the same pattern, so I can run the dev stack of Kindred at `kindred.dev.lab.example.com` from any tailnet device, not just localhost. The dev tier was the cleanest thing to come out of building this -- being able to point my phone at a dev stack to test mobile rendering, without exposing it to the internet, is something I'd been hand-rolling with `ngrok` and `ssh -L` for years.

Cloudflare DNS handles one more thing for these zones: it runs the DNS-01 challenge for Let's Encrypt wildcard issuance. Traefik's `cloudflare` cert resolver is configured with an API token that has zone-level DNS edit permission. When the first request to a previously-unseen subdomain comes in, Traefik (a) requests a wildcard cert for `*.kindred.lab.example.com` from Let's Encrypt, (b) creates the `_acme-challenge.kindred.lab.example.com` TXT record via the Cloudflare API, (c) waits for propagation, (d) collects the cert, (e) caches it. The same cert covers every Kindred subdomain after that. Same wildcard pattern for `*.plaid-sync.lab.example.com` and `*.finance-hub.lab.example.com`.

## The one public name: `plaid-sync-hooks`

Plaid's webhook delivery system needs to reach my homelab to notify me of new transactions. Plaid cannot reach a tailnet address. So one route has to be public, and it has to be the smallest possible public surface area.

```hcl
resource "cloudflare_dns_record" "kph_plaid_sync_hooks" {
  zone_id = local.lab_zone_id
  name    = "plaid-sync-hooks"
  content = local.tunnel_cname
  type    = "CNAME"
  ttl     = 1
  proxied = true
}
```

This is a proxied CNAME to a Cloudflare Tunnel. Traffic from Plaid hits Cloudflare's edge, gets tunneled to a `cloudflared` container on ares (which itself only joins `pikenet-public` and `pikenet-private`, never the monitoring net -- there was a multi-day debugging session caused by violating this rule), and lands on Traefik. Traefik routes it based on the host header to a separate plaid-sync router:

```yaml
- "traefik.http.routers.plaid-sync-hooks.rule=Host(`plaid-sync-hooks.lab.example.com`) && PathPrefix(`/api/webhooks`)"
- "traefik.http.routers.plaid-sync-hooks.middlewares=rate-limit@file"
- "traefik.http.routers.plaid-sync-hooks.service=plaid-sync-svc"
```

Note three things. First, the host rule includes `PathPrefix(/api/webhooks)`, so anything else -- a probe at `/`, an attempt at `/admin`, a scan at `/.git/config` -- gets a 404 from Traefik before it ever reaches the plaid-sync container. Second, there's only a rate-limit middleware, no auth -- Plaid's webhook contract is "the payload is signed; verify the signature." That verification happens in the application. Third, both routers (`plaid-sync` for the UI and `plaid-sync-hooks` for webhooks) point at the same backing `plaid-sync-svc`, so the same container serves both worlds with different host rules.

## The middleware that makes "tailnet-only" mean something

The DNS layer keeps non-tailnet clients from finding the host. But anyone *with* a tailnet connection could still hit those routes, and the homelab has more tailnet devices than I'd like to fully trust with admin tools. So every tailnet service gets a Traefik IP allowlist middleware named `admin-ipallowlist@file`:

```yaml
http:
  middlewares:
    admin-ipallowlist:
      ipAllowList:
        sourceRange:
          - "100.64.0.0/10"     # the whole CGNAT range, i.e. anything on a tailnet
          - "172.20.1.1/32"     # pikenet-public gateway
          - "172.20.2.1/32"     # pikenet-private gateway
          - "172.20.3.1/32"     # pikenet-monitoring gateway
          - "172.20.20.1/32"    # traefik-socket-net gateway
```

The CGNAT range is the whole point: only requests whose source IP looks like a tailnet client are allowed. The `172.20.*.1/32` entries are docker bridge gateways. Traefik sees those addresses when traffic enters via the host-bound listener and gets SNAT-ed by `ts-postrouting` to the docker bridge before reaching the container. Without those entries, every tailnet request would be 403'd by Traefik's IP allowlist after passing tailscaled, which is a fun debugging experience.

This is what the kindred docker-compose label list looks like in `apps/kindred/docker-compose.yml`:

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.docker.network=pikenet-private"
  - "traefik.http.routers.kindred.rule=Host(`kindred.lab.example.com`)"
  - "traefik.http.routers.kindred.entrypoints=websecure"
  - "traefik.http.routers.kindred.tls=true"
  - "traefik.http.routers.kindred.middlewares=admin-ipallowlist@file"
  - "traefik.http.services.kindred.loadbalancer.server.port=8000"
```

Same shape for finance-hub. plaid-sync has two routers (the UI one with admin-ipallowlist, the hooks one with just the rate limiter), but everything else is identical.

## The build path, version A: GitHub Actions on a self-hosted runner

Kindred and finance-hub build their images on a GitHub Actions self-hosted runner that I run on ares as a systemd user service. The trigger is a tag push: `v0.3.0`, `v0.3.1-rc.1`, etc. The workflow lives at `.github/workflows/release.yml` and is the same in both repos. Here's the kindred one in full:

```yaml
name: Release
on:
  push:
    tags:
      - "v*.*.*"

permissions:
  contents: read
  packages: write

jobs:
  build:
    runs-on: [self-hosted, linux]
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - name: Login to GHCR
        run: |
          mkdir -p "$RUNNER_TEMP/docker-config"
          token=$(printf '%s:%s' "${{ github.actor }}" "${{ secrets.GITHUB_TOKEN }}" | base64 -w0)
          printf '{"auths":{"ghcr.io":{"auth":"%s"}}}' "$token" \
            > "$RUNNER_TEMP/docker-config/config.json"
          echo "DOCKER_CONFIG=$RUNNER_TEMP/docker-config" >> "$GITHUB_ENV"
      - uses: docker/build-push-action@v5
        with:
          context: .
          file: Dockerfile.prod
          push: true
          tags: |
            ghcr.io/pike00/kindred:${{ github.ref_name }}
            ghcr.io/pike00/kindred:sha-${{ github.sha }}
          cache-from: type=local,src=/tmp/buildx-cache
          cache-to: type=local,dest=/tmp/buildx-cache,mode=max
```

A few things in here matter.

**Self-hosted runner**, not GitHub-hosted. Private repos on free-tier GitHub bill for runner minutes; my container builds were running about 8 minutes each, multiple times per week. The math came out to roughly $7/month to keep using GitHub-hosted runners on the private repos. A self-hosted runner on ares costs zero dollars (the box is already on) and is also faster, because the buildx layer cache lives on the same disk as the next run. Setup is a 10-minute affair: download the runner tarball, register it with a one-time token from the repo settings, install as a systemd user service. The runner is configured to only run jobs from these specific repos.

**Direct write to `~/.docker/config.json`**, instead of using the `docker/login-action` or `gh auth setup-git` style helpers. On a desktop with a logged-in session, docker login would normally call out to `pass` or `secretservice` to store the credential. On a headless self-hosted runner there's no D-Bus session and no keyring, so any credential helper hangs. The workaround is to skip helpers entirely and write the `base64(actor:token)` blob directly into a per-job `config.json`, then point `DOCKER_CONFIG` at it. Pattern stolen from the old `docker-credential-osxkeychain` workaround days. Took me an evening to figure out the first time.

**Tag with both the version and the commit SHA**. The version tag (`v0.3.0`) is the human-facing handle for `just bump`. The `sha-<sha>` tag is an immutable handle for rollback or forensics; even if someone deletes and re-pushes a tag (which I don't, but it's possible), the SHA tag uniquely identifies the artifact.

**No `latest` tag for finance-hub; `latest` does float on kindred** (skipping `-rc` tags). The decision to float `latest` is downstream-coupled: things like Renovate or Watchtower would consume `latest`, and Watchtower would auto-redeploy. Since I do manual bumps anyway, `latest` is mostly cosmetic. Finance-hub drops it; kindred keeps it because the kindred SPA's `package.json` and the Vite build embed the git SHA into the footer of the running app, and `latest` is a useful sanity check that the image I'm pulling is the one I think it is.

**Local buildx cache** at `/tmp/buildx-cache`. This lives on the runner's host filesystem and survives between runs of the same workflow because the runner is persistent. Cold builds are about 4 minutes; warm builds are about 90 seconds.

## The build path, version B: host-side `just publish`

plaid-sync builds the same way -- same Dockerfile shape, same GHCR push -- but I drive the build from my workstation rather than from GitHub Actions. The recipe lives in the plaid-sync repo's `justfile` as `just publish <tag>`. I haven't moved this to GHA yet because the build was working fine and it would have meant pushing a private repo's image through the runner, which is fine but I never got around to wiring up a release workflow.

```bash
just publish v0.3.0
```

Internally it:

1. Validates the tag matches the `vMAJOR.MINOR.PATCH` regex.
2. Verifies the git tag exists locally.
3. Resolves the commit SHA for the tag.
4. Creates a `docker-container` driver buildx builder if one doesn't exist (the default `docker` driver doesn't support registry-backed cache export).
5. Builds with `--build-arg APP_VERSION=v0.3.0`, so `/health` returns the right version string at runtime.
6. Pulls and pushes a registry-backed build cache at `ghcr.io/pike00/plaid-sync:buildcache` so a build from a different host (a fresh laptop, calypso) can reuse layers.
7. Pushes the version tag, the `sha-<short>` tag, and `latest`.

The registry-backed cache is the one nice thing about the host-side path that the GHA path doesn't have. `cache-to: type=registry,ref=...,mode=max` writes intermediate stage layers (`frontend-builder`, `deps-builder`) as part of the cache image, so a completely cold builder on a different host still benefits from the prior build's frontend layer. The local buildx cache that the GHA workflow uses is faster on the same runner but worthless to anyone else.

If I ever move plaid-sync onto the self-hosted runner, I'll probably keep the registry-backed cache, because it lets me do builds on willbook when ares is busy. There's no fundamental reason not to do both.

## Dockerfile.prod, the shape

All three repos ship a `Dockerfile.prod` at the repo root. They're not identical, but they share a template.

```dockerfile
# Stage 1: frontend with bun
FROM oven/bun:1 AS frontend-builder
WORKDIR /build/web
COPY web/package.json web/bun.lock ./
RUN bun install --frozen-lockfile
COPY web/ ./
ARG VITE_API_BASE=
ENV VITE_API_BASE=${VITE_API_BASE} NODE_ENV=production
RUN bun run build

# Stage 2: python runtime
FROM python:3.13-slim AS runtime
ENV PATH="/app/api/.venv/bin:$PATH" STATIC_DIR=/app/static
COPY --from=ghcr.io/astral-sh/uv:0.9.26 /uv /uvx /bin/
WORKDIR /app/api
COPY api/pyproject.toml api/uv.lock ./
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --frozen --no-dev
COPY api/ ./
COPY --from=frontend-builder /build/web/dist /app/static
EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

The interesting things:

**bun for the frontend.** This is an ADR in finance-hub's docs that I won't reproduce in full. Short version: bun's `install` is genuinely faster than npm or pnpm in CI, the `bun.lock` is small and stable, `bun run build` calls vite under the hood so the build output is identical, and the `oven/bun:1` image already ships with bun preinstalled.

**uv for python.** I install uv from the upstream multi-arch image (`ghcr.io/astral-sh/uv:0.9.26`) rather than `pip install uv` -- it saves a layer and the install is a single file copy. The `uv sync --frozen --no-dev` step is inside a `--mount=type=cache` layer so the wheel cache survives across builds.

**The frontend ships inside the same image as the API.** FastAPI's `StaticFiles` mounts `/app/static` at `/` and serves the built SPA from there; the API is at `/api/*`. There's no separate frontend container in production. The frontend container in `compose.dev.yml` is bun running `vite dev` against the source tree with HMR, but that path doesn't exist in the prod image.

**plaid-sync's Dockerfile.prod has an extra trick:** the python builder stage produces a `.venv` at `/app/.venv` which gets copied into the runtime stage, but the runtime stage *does not* have uv installed. uv lives only in the deps-builder stage; the runtime image is smaller by ~53MB without it. The kindred and finance-hub Dockerfiles keep uv in the runtime image, which is technically wasteful but means alembic migrations can be run inside the container via `uv run`. plaid-sync runs alembic from FastAPI's lifespan handler at startup, so it doesn't need uv at runtime.

**plaid-sync also runs as `1000:1000` from the start.** The `mkdir -p /app/...` + `chown -R 1000:1000 /app` happens before the `USER` directive, and every subsequent `COPY` uses `--chown=1000:1000`, so each layer is born with the right ownership. The earlier version of the Dockerfile had a `chown -R` at the end that duplicated the venv into a follow-up layer; switching to per-COPY `--chown` saved ~159MB on the final image.

## GHCR: the storage layer

Images land at `ghcr.io/pike00/<repo-name>`. Three packages:

- `ghcr.io/pike00/kindred`
- `ghcr.io/pike00/plaid-sync`
- `ghcr.io/pike00/finance-hub`

Each tag is immutable in practice (I never overwrite). For each release I get:

- `:v0.3.0` -- the human-facing version tag
- `:sha-abc1234` -- the immutable commit tag
- `:latest` -- floats to the most recent non-`rc` tag (kindred only)
- `:buildcache` -- the registry-backed build cache (plaid-sync only)

Public visibility on the package is set per-image: kindred is public, plaid-sync and finance-hub are private. GHCR inherits visibility from the parent repository by default, and there's a small one-time UI dance to flip it the first time the image is pushed.

The thing that always bites me on GHCR: the package permissions are separate from the repo permissions. If I want a non-personal token (say, a deploy token) to pull a private image, I have to grant it explicit access at the package level even if it already has read access to the repo. For my single-operator setup this doesn't come up -- I pull as `pike00` everywhere -- but the model is worth knowing about.

## The deploy unit: `just bump <tag>`

The actual deploy step runs on ares against the homelab repo. Three flavors of the same recipe; here's finance-hub's:

```bash
bump tag:
    #!/usr/bin/env bash
    set -euo pipefail
    echo "==> bump finance-hub to {{tag}}"
    just pg-dump
    sed -i "s|^IMAGE_TAG=.*|IMAGE_TAG={{tag}}|" .env
    just -f ../../infra/just/compose.just stack="$(pwd)" scripts_dir="$(pwd)/../../infra/scripts" pull
    just -f ../../infra/just/compose.just stack="$(pwd)" scripts_dir="$(pwd)/../../infra/scripts" up --wait
    sleep 5
    docker compose exec -T app curl -fsS http://localhost:8000/api/health \
      || (echo "Health check FAILED for {{tag}}; investigate before rolling back." && exit 1)
    echo "==> finance-hub {{tag}} healthy"
```

Five steps. First, pg-dump. This is enforced by the homelab-wide `upgrade-container-image` skill that any agent running on this machine is supposed to invoke when it touches a pinned image tag: dump before bump, full stop, no exceptions. The dump is a `pg_dump -Fc` (custom format, restorable with `pg_restore`) landing in `apps/finance-hub/manual-dumps/<timestamp>.dump`. It's not the daily backup -- that runs via a separate `backup-volumes.service` -- but it is the "I want to know this moment is recoverable" snapshot, sized in tens of megabytes for these stacks.

Second, `sed -i` rewrites the `IMAGE_TAG=...` line in `apps/finance-hub/.env`. This is the only mutable line in `.env`. The other secrets (`POSTGRES_PASSWORD`) live in `.env.sops` and are decrypted on the fly by the next step.

Third, `pull` and `up --wait` go through the shared `infra/just/compose.just` recipe. That recipe does three things before `docker compose up`:

```bash
sops exec-file --no-fifo --input-type dotenv --output-type dotenv "{{env_sops}}" \
    'docker compose -f "{{compose_file}}" --env-file {} config' | "$bind_check"
sops exec-file --no-fifo --input-type dotenv --output-type dotenv "{{env_sops}}" \
    'docker compose -f "{{compose_file}}" --env-file {} config' | "$stale_check"
sops exec-file --no-fifo --input-type dotenv --output-type dotenv "{{env_sops}}" \
    'docker compose -f "{{compose_file}}" --env-file {} up -d {{args}}'
```

`sops exec-file` writes the decrypted plaintext to a FIFO (or temp file on systems without FIFOs), passes the path as `{}` to the inner command, and removes it on exit. The plaintext lives in memory for the duration of `docker compose up` and is gone the moment the recipe exits. The two `_check` scripts (`check-bind-mounts.py`, `check-stale-mounts.py`) are pre-flight guards that I added after a couple of incidents where stale bind-mount paths from a worktree directory crept into running containers and silently broke things. They render the compose config and refuse to proceed if any bind source is missing or if a running container has a different bind source than the rendered config.

`up --wait` blocks until every service's healthcheck reports green. For finance-hub that's `pg_isready` on postgres and a python urlopen on the app's `/api/health`. The healthchecks use a `start_period: 60s` so the app has time to run alembic migrations on first boot.

Fourth, after `up --wait` returns, the recipe pauses 5 seconds (margin for Traefik to discover the new container's network membership) and runs a final `curl` from inside the container to its own `/api/health`. This is belt-and-suspenders -- the docker healthcheck already passed -- but it catches the case where the healthcheck definition itself is broken (network reachability bug, wrong path, etc.) by hitting the endpoint through a different code path.

Fifth, the recipe exits 0. The container is now running the new tag and there's a pg-dump on disk that captures the state just before the roll.

Rollback is the same recipe minus the pg-dump:

```bash
just rollback v0.2.9
```

Sed rewrites the tag, `up --wait` rolls back. The previous pg-dump is still there if the rollback needs to be paired with a database restore.

## SOPS, env_file, and the trap I keep almost falling into

There's a footgun in Docker Compose that took out a couple of my early stacks: `env_file:` does not understand SOPS. The encrypted file is read as a literal binary blob, and the resulting environment variables look like `ENC[AES256_GCM,...]`. The container boots, the application looks up `POSTGRES_PASSWORD`, sees a base64'd ciphertext, and tries to use it as the password. Postgres fails to authenticate with cryptic errors. You stare at logs for an hour.

The fix is to never use `env_file:` with `.env.sops`. Instead, decrypt at the shell level with `sops exec-file` and let it inject variables via the docker compose `--env-file` flag:

```bash
sops exec-file --no-fifo --input-type dotenv --output-type dotenv .env.sops \
  'docker compose --env-file {} up -d'
```

The compose file references the variables via `${POSTGRES_PASSWORD}` interpolation. Docker substitutes them at parse time. The plaintext is never written to disk and never appears in `docker inspect` output (only the resolved env vars do, and those are unavoidable -- they're in the container).

The compose files for these stacks therefore look like:

```yaml
environment:
  POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
```

Not:

```yaml
env_file:
  - .env.sops  # DO NOT DO THIS
```

It's the kind of mistake that fails consistently and obviously the first time. The trap is that it also feels like the right answer, especially if you came from a "12-factor app, env_file is fine" background.

## Hardening defaults: cap_drop and the postgres exception

Every container runs with `cap_drop: [ALL]` and `security_opt: [no-new-privileges:true]`. Most also run with `read_only: true` and a small tmpfs at `/tmp`:

```yaml
cap_drop:
  - ALL
security_opt:
  - no-new-privileges:true
read_only: true
tmpfs:
  - /tmp:noexec,nosuid,size=128m
```

This is enough for most images that already run as a non-root user. It is not enough for postgres.

The official `postgres:18-alpine` image runs its entrypoint as root, then drops to the `postgres` user using a privilege-switching helper (it's a tiny C binary in the image). That helper calls `setuid()` and `setgid()`. Both system calls require the SETUID and SETGID capabilities. `cap_drop: [ALL]` strips them. The entrypoint silently fails the privilege drop and the container exits with no useful error.

The fix is to give postgres back exactly the caps it needs:

```yaml
cap_drop:
  - ALL
cap_add:
  - CHOWN
  - SETUID
  - SETGID
  - DAC_OVERRIDE
  - FOWNER
```

CHOWN is for the initial `chown -R postgres:postgres /var/lib/postgresql/data/pgdata`. SETUID/SETGID are for the privilege drop. DAC_OVERRIDE and FOWNER let initdb work around the file ownership on a fresh volume.

This is a real example of a hardening best practice colliding with a real-world image's startup sequence. The other place it bites: redis with `appendonly yes`. Redis sets a `umask 0077` and creates its append-only dir with `0700` permissions, owned by the redis user. When the container then needs to traverse that dir at runtime, the kernel's discretionary-access-control check fails -- and `cap_drop: [ALL]` removed DAC_OVERRIDE, the cap that would otherwise let root bypass DAC. Either pre-fix permissions on the volume before bringing redis up or add DAC_OVERRIDE back. I do the second on kindred-redis (with `SETUID` and `SETGID`); finance-hub doesn't have redis.

A related rule: with `cap_drop: [ALL]`, the container's UID must match the volume owner exactly, since there's no DAC_OVERRIDE fallback. plaid-sync's bind mount of `/home/will/Documents/Finance/Ledger` to `/ledger:rw` only works because the container runs as `1000:1000` and the host directory is owned by `will:will` who is `1000:1000`. If I ever changed the runtime UID, every existing volume would refuse writes and the failure mode would be "files just don't appear, no error in logs."

## Networks: shared and per-stack

There are three shared docker networks on this host:

- `pikenet-public` -- where `cloudflared-tunnel` and Traefik's public-facing routes live. Only services that need to be reachable from the internet (i.e. `plaid-sync-hooks`'s upstream) sit on this.
- `pikenet-private` -- where Traefik attaches to most services. Every tailnet-only route goes through here.
- `pikenet-monitoring` -- where Prometheus scrapes. Services join this only if they expose `/metrics`.

And per-stack internal networks for the data tier: `pikenet-internal-kindred`, `pikenet-internal-plaid-sync`, `pikenet-internal-finance-hub`. Postgres, Redis, and Meilisearch live on these and only these. They never get a port published to the host, and Traefik can't reach them because Traefik isn't a member of those networks.

This is the pattern that makes it safe to run a postgres container with the password `finance_hub` (the default in `.env.example`): the database is not reachable from anywhere except the application container that shares its internal network. If I wanted to inspect the DB from my laptop, I'd have to either `docker exec` into the host or temporarily attach a network shim.

The `traefik.docker.network=pikenet-private` label is what tells Traefik which network to route to when a service is attached to multiple networks (the case for `web` and `app`, which are on both the private and the internal network). Without it Traefik picks one at random, and "at random" is sometimes the internal network where Traefik isn't a member.

## The bind mount: finance-hub writes back to the ledger

```yaml
volumes:
  - ${LEDGER_HOST_ROOT:?set LEDGER_HOST_ROOT in .env to ~/Documents/Finance}:/finance:rw
```

`LEDGER_HOST_ROOT` is set in `apps/finance-hub/.env` to `/home/will/Documents/Finance`. Inside the container the path is `/finance`. The mount is read-write because finance-hub's reconciliation flow writes back to `Ledger/bean/main.bean` and the per-account `documents.bean` files.

This is the only host bind mount that the finance-hub container has. Postgres data lives in a named volume; the app config lives in the image. The bind mount is purely for the ledger.

plaid-sync has the same shape: `/home/will/Documents/Finance/Ledger:/ledger:rw`. plaid-sync writes `! `-flagged entries directly into the curated bean files; finance-hub's reconciliation UI is what un-flags them later. The two services don't need a shared database for this; the filesystem is the API.

Two things to watch with this pattern. First, the host `LEDGER_HOST_ROOT` is on the same physical disk as `/home`, which is bind-mounted into `~/Documents` (a separate concern), which is itself part of the `pike00/documents-vault` git repo. Anything finance-hub writes ends up on disk and shows up in `git status` in the vault. There's an `~/Documents/Homelab/CLAUDE.md` rule against writing backup artifacts under `~/Documents/` for exactly this reason -- but the ledger itself is *supposed* to be in the vault, since it's the source of truth I want backed up everywhere. So this case is fine; just don't generalize.

Second, the `rclone sync --delete-excluded` job that backs up `~/Documents/` to Backblaze and AWS runs every hour. If finance-hub temporarily writes a file and then deletes it, the deletion mirrors to the cloud. Most of the time this is what I want. The rare time I'd want it not to (rapid scratchwork inside `~/Documents/Finance/`) the answer is to write to `/tmp` inside the container instead of `/finance`.

## The dev tier: review apps via worktree

This isn't strictly part of the deploy story, but it's the most useful thing that came out of building the deploy pattern. Both plaid-sync and finance-hub have a `compose.worktree.yml` in the source repo that boots a per-worktree dev stack reachable at `https://<slug>.plaid-sync.lab.example.com`, where `<slug>` is the basename of the worktree directory. The wildcard DNS rewrite makes any subdomain resolve to ares. Traefik mints a cert under the wildcard SAN. Each worktree gets its own postgres, its own data volume, its own Traefik router. There are no host port collisions because no service binds a host port.

```bash
cd .worktrees/feature-x
just up    # boots https://feature-x.plaid-sync.lab.example.com
```

The dev stack uses `compose.worktree.yml`, not `compose.dev.yml`. The difference: `compose.dev.yml` is for local-only development (loopback ports for e2e tests, mounted source for hot-reload); `compose.worktree.yml` is for review apps that someone else (or me on my phone) can hit through the wildcard DNS. Both exist in parallel because they solve genuinely different problems and conflating them in one file produces something that does neither well.

Kindred doesn't have a worktree compose yet; it's on the list. The blocking work is that Kindred's setup flow generates a one-time admin token on first boot and prints it to the logs, and I haven't worked out the right UX for that across many ephemeral review apps.

## What this leaves

The end state is that each release is a tag push (`git tag v0.3.0 && git push origin v0.3.0`), a wait for GHCR to come back green, and a `just bump v0.3.0` from `~/Documents/Homelab/apps/<service>/`. The bump produces a pg-dump and a healthcheck-gated rollover. Rollback is the same recipe with the previous tag.

What this doesn't have:

- **Auto-deploy.** Watchtower exists. Webhooks exist. Portainer exists. I evaluated all three and went with manual bumps. The reasoning: my release cadence is at most a few times a week; the act of typing `just bump` is the moment I actually want to be paying attention to a deploy (because that's also when the pg-dump happens, that's when I want to look at the logs, and that's when I'd notice a regression). Adding auto-deploy would mean two failure modes -- the deploy itself, and the trigger -- where I currently have one.
- **A staging environment.** I have a dev tier (worktree review apps) and a prod tier. There is no separate staging environment that mirrors prod. For a single-user, single-host system the cost of staging is high relative to its value; for most of the bugs I hit, the worktree review app is enough.
- **Blue-green deploys.** A `just bump` is a docker-compose `up -d` with `--wait`. There's a brief window (single-digit seconds) where the old container has stopped and the new one is starting. Traefik sees the new container's healthcheck before the old container is reaped. For an HTTP API used by my household over Tailscale, this is fine. If this were a payment processor with strict SLAs it wouldn't be.
- **Secrets in a vault.** Secrets live in `.env.sops`, encrypted with my age key (post-quantum hybrid ML-KEM-768 + X25519, per [the earlier post on this setup](/blog/sops-age-docker-compose)). There's no separate Vault or KMS step. If I lost the age key, every `.env.sops` in the homelab and every cloud backup would be undecryptable. The age key is backed up to Bitwarden and on paper. The same single key gates everything.

The thing I keep being slightly surprised by is how small the moving-parts list is. A `Dockerfile.prod`, a `release.yml`, a `docker-compose.yml`, an `.env.sops`, a `bump` recipe, an `admin-ipallowlist` middleware, an unproxied A record. Six files per service, give or take. The complexity is in what each one knows: the cert resolver knows how to talk to Cloudflare, the IP allowlist knows which CIDRs are tailnet, the `up` recipe knows how to wire sops into docker-compose. But the structural overhead is low, and it generalizes -- adding a fourth service to this shape would be an afternoon, almost all of it spent writing the Dockerfile.

The pattern wasn't designed up front. It accumulated. Kindred established the GHA-on-self-hosted-runner shape and the `just bump`/`pg-dump` discipline. plaid-sync was earlier and uses the host-side `just publish` because that's what I had at the time; it's on the list to move onto the same shape. finance-hub copied kindred verbatim, then I noticed how copy-pasted the docker-compose hardening boilerplate was getting and extracted a couple of the shared things into `infra/just/compose.just`. The pre-flight bind-mount check came out of a bad afternoon where Traefik kept binding to a placeholder directory. The DAC_OVERRIDE for postgres came out of an even worse afternoon.

If you're building something similar, the parts I'd most recommend stealing: the manual `bump` recipe with a mandatory pg-dump, the IP allowlist middleware on the CGNAT range, the unproxied A records pointing at the Tailscale IP, the per-stack internal docker network for data, the sops-via-`exec-file` pattern. None of them is novel on its own. Together they make a deploy boring, and boring is what you want.
