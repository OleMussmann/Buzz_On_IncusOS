# Buzz relay on Incus

Self-hosted [Buzz](https://github.com/block/buzz) — Block's human+agent collaboration
platform, built on Nostr — deployed on an IncusOS host via
[`incus-compose`](https://github.com/lxc/incus-compose), reachable only over Tailscale.

Every message, reaction, workflow step and git event is a signed Nostr event in a single
event log. Agents are members with their own keypairs and audit trails, not bots with
permission flags. This repo deploys **the relay only** — the piece that has to be always-on.
Agents connect to it from wherever they run.

## Architecture

| Service | Image | Role |
|---|---|---|
| `postgres` | `docker.io/postgres:17-alpine` | Event store + full-text search |
| `redis` | `docker.io/redis:7-alpine` | Pub/sub fan-out between relay connections |
| `minio` | `quay.io/minio/minio` | S3-compatible object store for media (Blossom protocol) |
| `minio-init` | `quay.io/minio/mc` | One-shot: creates the media bucket, ensures it is not public, then exits |
| `relay` | `ghcr.io/block/buzz` | The relay itself — WebSocket (Nostr), REST API, web UI, git hosting |

Only `relay` is reachable from outside the Incus project, and only over Tailscale.
`postgres`, `redis`, `minio` and `minio-init` are internal-only.

The relay listens on three ports:

| Port | Serves |
|---|---|
| `3000` | Relay: WebSocket, REST API, web UI. This is what clients connect to and what `RELAY_URL` must name. |
| `8080` | Health probes only — `/_liveness`, `/_readiness`. Not a UI. |
| `9102` | Prometheus metrics. |

## Prerequisites

- Incus 7.2+ on the target host (7.2 is the floor for NAT proxy mode), reachable over
  HTTPS — `incus-compose` requires `core.https_address` even for local use, since it
  caches images in a separate project and copies them into yours on `up`.
- [`incus-compose`](https://github.com/lxc/incus-compose) 1.1.0+ on the client machine you
  manage Incus from (not on the IncusOS host itself).
- An Incus HTTPS remote pointed at the IncusOS host, set as the **default** remote
  (`incus remote switch <name>`; check with `incus remote get-default`).
- OCI registry remotes for **both** `docker.io` and `ghcr.io` — see the naming gotcha below.

### Remote naming: names must match the registry hostname, including `.io`

`incus-compose` resolves each image's leading path segment (e.g. `ghcr.io` in
`ghcr.io/block/buzz`) against an Incus remote of that exact name. It is not a hostname
lookup — it's a literal string match against your configured remote names:

```
incus remote add --protocol oci docker.io https://docker.io
incus remote add --protocol oci ghcr.io https://ghcr.io
```

Shortened names like `ghcr` will not resolve, even pointing at the right URL. You'll get:

```
image source error getting image server for ghcr.io: The remote "ghcr.io" doesn't exist
```

If you already added them under shortened names, rename rather than editing image refs:

```
incus remote rename ghcr ghcr.io
```

Verify with `incus remote list` before running `incus-compose up`.

Note that the two tools spell the same image differently. `compose.yaml` uses
`ghcr.io/block/buzz:<tag>` with a **slash**, because incus-compose splits the leading path
segment off and resolves it against your remotes. Raw `incus` commands take
`<remote>:<image>`, so the same image is `ghcr.io:block/buzz:<tag>` with a **colon**. Using
the slash form directly with `incus launch` fails.

## Repository structure

```
.
├── compose.yaml                  # Service definitions (Compose spec)
├── compose.incus.yaml            # Incus-specific overrides — Tailscale-scoped ports
│                                 # (gitignored, holds your real tailnet IP)
├── compose.incus.yaml.example    # Template — copy and fill in
├── versions.env                  # Image pins (tracked)
├── .env.example                  # Template — copy to .env and fill in real secrets
└── .env                          # Real secrets (gitignored — never commit)
```

## Configuration

Copy `.env.example` to `.env` and replace every `CHANGE_ME`. Generate the random secrets
with:

```
openssl rand -hex 32
```

If `openssl` isn't on your PATH:

```
python3 -c 'import os;print(os.urandom(32).hex())'
```

Either command fills the five placeholders spelled `CHANGE_ME_OPENSSL_RAND_HEX_32`:

| Placeholder appears at | |
|---|---|
| `BUZZ_GIT_HOOK_HMAC_SECRET` | Signs git hook callbacks |
| `POSTGRES_PASSWORD` | Sets *and* authenticates the Postgres user |
| `REDIS_PASSWORD` | Sets *and* authenticates Redis (`--requirepass`) |
| `BUZZ_S3_ACCESS_KEY` | Becomes `MINIO_ROOT_USER` |
| `BUZZ_S3_SECRET_KEY` | Becomes `MINIO_ROOT_PASSWORD` |

**Run the command five times — once per line.** They share a placeholder because they share
a *generator*, not a value. Pasting one output into all five would give Postgres, Redis and
MinIO the same credential, so a leak of any one of them is a leak of all three.

The two identity values are not in this group — they come from `buzz-admin generate-key`
(see below), take different halves of two different runs, and are spelled
`CHANGE_ME_GENERATE_KEY_RUN1_PUBLIC` and `CHANGE_ME_GENERATE_KEY_RUN2_SECRET` accordingly.
`CHANGE_ME_TAILSCALE_IP` is the one placeholder that genuinely *is* the same value in all
four places it appears.

Hex-only output sidesteps `.env` parsing pitfalls — `$` triggers variable interpolation and
`#` starts a comment. `-hex 32` yields 64 hex characters, which is the shape
`BUZZ_RELAY_PRIVATE_KEY` and `BUZZ_GIT_HOOK_HMAC_SECRET` expect; the same command is fine
for the Postgres, Redis and S3 values.

**You do not need an S3 account or a pre-made bucket.** Despite the names,
`BUZZ_S3_ACCESS_KEY` and `BUZZ_S3_SECRET_KEY` are not credentials for an external provider —
MinIO runs inside this stack and is configured *with* them (`MINIO_ROOT_USER` /
`MINIO_ROOT_PASSWORD` in `compose.yaml`). You invent the values; MinIO adopts them on first
boot. MinIO requires at least 3 characters for the user and 8 for the password, so a
64-hex string satisfies both. The bucket named by `BUZZ_S3_BUCKET` is created by the
`minio-init` one-shot on every `up`, which is why `relay` gates on it via
`service_completed_successfully`.

Pointing Buzz at a real external S3 provider is *not* configurable through `.env` here —
`BUZZ_S3_ENDPOINT` is fixed to `http://minio:9000` with path-style addressing in
`compose.yaml`, matching upstream's own constraint. Upstream directs you to the Helm chart
for that case.

Copy `compose.incus.yaml.example` to `compose.incus.yaml` and replace
`IncusOS_IP_ADDRESS` with your host's Tailscale IP. Binding to the Tailscale address
specifically — rather than `0.0.0.0` — is what keeps the relay off the public internet and
off the LAN, so don't drop the `host_ip` prefix.

All three port mappings use **long-form ports with `x-incus-compose.nat: true`** (kernel NAT
proxy mode, incus-compose 1.1.0+), which is faster than the default userspace proxy.

**`RELAY_URL` must match byte-for-byte what clients connect with, scheme included.** A
trailing slash, or `http://` where the client uses `ws://`, breaks NIP-42 auth in ways that
look like a credentials problem rather than a URL problem. For a tailnet-only deployment
that means `ws://<tailscale-ip>:3000`.

### Identities and keys

Buzz's whole premise is that every participant — human or agent — is a Nostr identity. There
are two classes of key here, and both are unrecoverable if lost.

**Relay-level, set before the first `up`:**

| Variable | What it is |
|---|---|
| `BUZZ_RELAY_PRIVATE_KEY` | The relay's own signing identity. 64-char hex. |
| `RELAY_OWNER_PUBKEY` | Your personal identity as relay owner. 64-char hex — **not** an `npub`. The owner cannot be removed from the member list later, so get this right the first time. |

Generate a keypair with the `buzz-admin` binary that ships inside the relay image. This is
a chicken-and-egg: both variables are required before the first `up`, so the relay isn't
running yet. Use a throwaway container.

`incus launch` has no `-- <command>` form (that's Docker's syntax) — you launch, then exec.
And the image's entrypoint is `buzz-relay`, which exits immediately without a database, so
override it or the instance dies before you can exec into it:

```
# Plain `incus`, not `incus-compose incus` — the `buzz` project doesn't exist yet.
# Note `ghcr.io:block/buzz` — a COLON after the remote name, not a slash.
incus launch ghcr.io:block/buzz:<tag> buzz-keygen \
    --project default -c oci.entrypoint="sleep infinity"

incus exec buzz-keygen --project default -- buzz-admin generate-key

incus delete -f buzz-keygen --project default
```

If `sleep infinity` isn't present in the image, `tail -f /dev/null` does the same job.

**Remote separator: `:` here, `/` in `compose.yaml`.** Raw `incus` takes
`<remote>:<image>`, so the reference is `ghcr.io:block/buzz:<tag>`. `compose.yaml` writes
the same image as `ghcr.io/block/buzz:<tag>` with a slash, because incus-compose resolves
the leading path segment against your remotes itself. Both are correct in their own place;
copying one form into the other's context fails.

**Which `<tag>`?** Use the one this stack is pinned to, so the `buzz-admin` you run is the
same build as the relay you're about to deploy:

```
grep BUZZ_TAG versions.env
```

To find out whether a newer build exists — the tag is `sha-` plus the first 7 characters of
the upstream commit the image was built from:

```
TOKEN=$(curl -s "https://ghcr.io/token?scope=repository:block/buzz:pull&service=ghcr.io" \
          | python3 -c 'import sys,json;print(json.load(sys.stdin)["token"])')
AMD=$(curl -s -H "Authorization: Bearer $TOKEN" \
        -H "Accept: application/vnd.oci.image.index.v1+json" \
        https://ghcr.io/v2/block/buzz/manifests/main \
        | python3 -c 'import sys,json;print([m["digest"] for m in json.load(sys.stdin)["manifests"] if m["platform"]["architecture"]=="amd64"][0])')
CFG=$(curl -s -H "Authorization: Bearer $TOKEN" \
        -H "Accept: application/vnd.oci.image.manifest.v1+json" \
        "https://ghcr.io/v2/block/buzz/manifests/$AMD" \
        | python3 -c 'import sys,json;print(json.load(sys.stdin)["config"]["digest"])')
curl -sL -H "Authorization: Bearer $TOKEN" "https://ghcr.io/v2/block/buzz/blobs/$CFG" \
  | python3 -c 'import sys,json;r=json.load(sys.stdin)["config"]["Labels"]["org.opencontainers.image.revision"];print("sha-"+r[:7])'
```

`incus-compose-update` does the equivalent for you against the `x-update` block, so this is
for when you want to look without running an update check. Either way, read the diff before
bumping — the 7-hex suffix is a real commit:
`https://github.com/block/buzz/compare/<current>...<candidate>`

Once the stack is up, use the running relay instead — no override needed:

```
incus-compose exec relay -- buzz-admin generate-key
```

`generate-key` prints both halves as **64-char hex** (verified against
`crates/buzz-admin/src/main.rs`: `public_key().to_hex()` and
`secret_key().display_secret()`), so no bech32 conversion is needed for either variable.

**Run it twice — the relay and the owner are different identities.**

| Run | Take | Goes in |
|---|---|---|
| Relay identity | Secret key | `BUZZ_RELAY_PRIVATE_KEY` in `.env`. The public half isn't used anywhere — clients learn it from the relay. |
| Owner identity | Public key | `RELAY_OWNER_PUBKEY` in `.env` |
| ↳ same run | Secret key | Your password manager or Nostr client — **never** `.env` |

Using one keypair for both makes the relay sign as you, collapsing the distinction between
"the relay said this" and "the owner said this". Generate two.

**Ignore the variable name `generate-key` prints.** It suggests `BUZZ_PRIVATE_KEY`, which is
the *agent-side* variable `buzz-acp` reads — the command is a generic keypair generator and
doesn't know what you're generating for. For the relay the variable is
`BUZZ_RELAY_PRIVATE_KEY`; for an agent it really is `BUZZ_PRIVATE_KEY`.

Note also that `add-member` cannot assign the `owner` role — the owner is set solely by the
`RELAY_OWNER_PUBKEY` config value, which is why it can't be revoked from the member list.

**Where key material lives.** `.env` is gitignored and is the only file in this repo that
should ever hold a private key. Back it up somewhere durable *outside* this repo — a
password manager entry is the minimum. Losing `BUZZ_RELAY_PRIVATE_KEY` means every client
sees the relay as a different, untrusted identity; losing the owner key means you can no
longer administer membership on a relay whose owner cannot be replaced.

**Per-agent keys.** Every agent that joins gets its own keypair — that is what makes its
actions attributable in the audit log. Issue one per agent, never share one across agents:

```
# 1. Generate a keypair for the agent
incus-compose exec relay -- buzz-admin generate-key

# 2. Register its public key as a member
incus-compose exec relay -- buzz-admin add-member --pubkey <npub-or-hex> --role member

# 3. Confirm
incus-compose exec relay -- buzz-admin list-members
```

The agent's *private* key goes into that agent's own environment (see below), not into this
repo.

## Usage

```
# Start everything (detached — always use -d for anything long-running).
# Both env files: pins live in versions.env, secrets in .env.
incus-compose --env-file .env --env-file versions.env up -d

# Check status (add -a to include stopped instances — minio-init is expected
# to be stopped once it has completed)
incus-compose --env-file .env --env-file versions.env ps -a

# Follow logs
incus-compose --env-file .env --env-file versions.env logs -f relay

# Render the merged, interpolated config without touching the host —
# the fastest way to check a change before applying it
incus-compose --env-file .env --env-file versions.env config

# Stop and remove containers (keeps volumes and cached images)
incus-compose --env-file .env --env-file versions.env down

# Also delete volumes — this drops every message, all media and all git state
incus-compose --env-file .env --env-file versions.env down --volumes

# Full teardown: remove the whole Incus project
incus-compose --env-file .env --env-file versions.env down --project
```

Setting `INCUS_COMPOSE_ENV_FILE='.env,versions.env'` removes the need to pass `--env-file`
on every invocation.

Prefer `incus-compose` over plain `incus` for day-to-day work — it scopes every operation to
the `buzz` project, so a mistake can't reach unrelated projects on the host. For anything it
doesn't wrap, `incus-compose incus ...` runs a raw incus command in that same project
context.

## Verifying it works

```
# Liveness (health port)
curl -fsS http://<tailscale-ip>:8080/_liveness

# Readiness — this is the one that goes green only after migrations finish
curl -fsS http://<tailscale-ip>:8080/_readiness

# Metrics
curl -s http://<tailscale-ip>:9102/metrics | head

# The relay itself
curl -fsS http://<tailscale-ip>:3000/
```

Confirm the migrations actually ran (rather than the relay booting against an empty schema):

```
incus-compose exec postgres -- psql -U buzz -d buzz -c '\dt'
```

Check the pinned image is the one running:

```
incus-compose exec relay -- buzz-relay --version
```

## Connecting an agent

Any agent that speaks [ACP](https://github.com/zed-industries/agent-client-protocol) can join
via the `buzz-acp` harness; Hermes has a native adapter and doesn't need it. This section is
deliberately generic — it names variables, not values.

1. **Generate and register a keypair for the agent** (see *Identities and keys* above). Each
   agent gets its own; that's what makes the audit log meaningful.
2. **Point the harness at this relay.** `buzz-acp` reads:

   | Variable | Value |
   |---|---|
   | `BUZZ_RELAY_URL` | Exactly what `RELAY_URL` is set to in `.env` — byte-for-byte |
   | `BUZZ_PRIVATE_KEY` | That agent's private key |
   | `BUZZ_ACP_AGENT_COMMAND` | The ACP adapter to drive, e.g. `opencode acp` |

3. **Adapters** exist as presets or npm packages for Claude Code, Codex, OpenCode and others;
   check [`crates/buzz-acp`](https://github.com/block/buzz/tree/main/crates/buzz-acp) for the
   current list rather than trusting a copy here.

Nothing about the relay needs to change to add an agent — registration is a `buzz-admin
add-member` call, not a config edit or a restart.

## Incus-specific gotchas

These are the things that will bite you if you edit `compose.yaml`. Each fails in a way that
doesn't obviously point at its cause.

- **`PSQL_PAGER: cat` on `postgres` is load-bearing.** incus-compose gives the entrypoint a
  TTY, so `psql` running the initdb scripts pipes its output into `less` and blocks forever.
  Without it, initdb never finishes — with no error, just a hang.
- **Don't declare the ports in `compose.yaml`.** Compose *merges* port lists across files
  rather than replacing them, so declaring a port in both files produces two proxy devices
  with the same name, and the userspace entry wins the collision — breaking start with
  `Connect IP "127.0.0.1" must be one of the instance's static IPv4 addresses`. All three
  ports are declared only in `compose.incus.yaml`.
- **`command:` APPENDS to the image entrypoint; it does not replace the image's CMD.**
  This is the biggest behavioural difference from Docker Compose and it bites any service
  you override `command:` on. The redis image resolves to
  `docker-entrypoint.sh redis-server`, so upstream's
  `command: ["redis-server", "--appendonly", ...]` produces
  `docker-entrypoint.sh redis-server redis-server --appendonly ...` — redis reads that
  second `redis-server` as a config-file path and dies with
  `Fatal error, can't open config file '/data/redis-server'`. Pass **only the flags**.
  Check what you'll actually get with:

  ```
  incus-compose incus config show <service>-1 | grep oci.entrypoint
  ```

  `image.oci.entrypoint` is the image's own; `oci.entrypoint` is that plus your `command:`.
  Note `entrypoint:` behaves the *other* way — it genuinely replaces, which is why
  `minio-init` uses it.
- **`service_completed_successfully` does not actually gate startup.** incus-compose logs
  `start instance <one-shot>-1: done` when the container *launches*, not when it *exits* —
  and the dependent service never even prints a "Waiting for dependency" line for it.
  Measured on a fresh deploy: `relay` probed the media bucket at `13:12:17` while
  `minio-init` created it at `13:12:25.80`, 8.9 s later. The relay died with
  `git conformance probe failed: … NoSuchBucket`, which reads like a MinIO misconfiguration
  and is really an ordering failure. `depends_on: … condition: service_healthy` **is**
  honoured (redis, postgres and minio all gated correctly) — it is specifically the
  completion condition that doesn't. Any one-shot you actually need to finish first has to
  be gated some other way.
- **incus-compose owns volume ownership, and it insists on root.** It provisions custom
  volumes as `initial.uid=0, initial.gid=0, security.shifted=true`, and refuses to start any
  instance whose attached volume differs:
  `volume configuration mismatch: expected security.shifted=true / UID mismatch, expected 0
  got 1000`. That message names no volume, and it aborts the *whole* `up`, so every service
  shows as stopped. A hand-created uid-1000 volume is rejected outright; a uid-0 one leaves
  an image that runs as a non-root user unable to write its own data
  (`/data/git/.pack-cache could not be created: Permission denied`). The relay image runs as
  `buzz` (uid 1000), so this stack pins it to container-root via `user: "0:0"` — matching
  the relay to the volume is the only side of the standoff you control. Container-root is
  not host-root; Incus idmaps it. It must be `user:`, not `x-incus: {oci.uid, oci.gid}`:
  from incus-compose 1.3 the expected volume owner comes from `user:` or the image's USER
  and ignores oci.uid, so the x-incus form fails the other way round (`UID mismatch,
  expected 1000 got 0`).
- **Postgres 18 moved its data directory.** `VOLUME` went from
  `/var/lib/postgresql/data` (PG17) up a level to `/var/lib/postgresql`, and the default
  `PGDATA` became `/var/lib/postgresql/18/docker` — a major-version-scoped subdirectory.
  Upstream Buzz's bundle still uses the PG17 paths. Mounting at the old target on PG18 fails
  at container spawn with `Failed to open target mountpoint /var/lib/postgresql/data for
  detached idmapped mount`, because Incus cannot create a missing nested mount target — and
  incus-compose reports it as the unrelated-looking UID mismatch above. This stack mounts at
  `/var/lib/postgresql` and leaves `PGDATA` unset so the image supplies its own default.
- **A failed service aborts `up` before service-name DNS is written.** incus-compose
  publishes bare service names by writing `address=/<service>/<ip>` records into the
  network's `raw.dnsmasq` config, and it does that late in `up`, once instances have
  addresses. If any service fails to start, `up` stops short and **no service can resolve
  any other by name** — which surfaces as unrelated-looking errors elsewhere, e.g.
  `dial tcp: lookup minio: i/o timeout` in `minio-init`. Before chasing a DNS problem,
  check whether some *other* service failed first:

  ```
  incus network show <network> | grep -A20 raw.dnsmasq   # empty == up never finished
  ```
- **`minio-init`'s command must be a single-element list, not a folded `>` string.**
  `sh -euc` takes the whole script as one argument, but Compose splits a bare string command
  on whitespace — which silently truncates the script at the first `&&`, leaving only
  `mc alias set …` and never creating the bucket. `incus-compose config` shows the split.
- **The relay healthcheck is the risky one.** `ic-healthd` latches the first failed probe
  into `unhealthy` and never re-probes, and with `restart: unless-stopped` a permanently
  failing probe becomes a restart loop rather than a passive red light. First boot runs
  migrations, hence `start_period: 120s` where upstream uses 30s (upstream assumes Docker's
  re-probing). Nothing depends on this healthcheck — **if it latches, delete the block.**
  The same reasoning is why the health probes on `postgres`, `redis` and `minio` — which
  `relay` genuinely gates on — all carry generous `start_period` values.
- **No `networks:` block, deliberately.** Upstream declares a `buzz-net` bridge because
  Docker Compose otherwise drops everything on the default bridge. incus-compose creates a
  dedicated bridge per compose project automatically — one `ic-*` network per stack, visible
  in `incus network list`. Declaring networks here would create a second, redundant bridge.
- **Stacks cannot reach each other by service name.** Each incus-compose stack gets its own
  Incus project *and its own bridge on its own subnet*, so there is no cross-stack DNS. This
  is why Buzz runs its own Postgres and Redis rather than sharing another stack's — see
  *Design choices*.
- **Volume names are project-scoped.** incus-compose creates custom volumes as `vol-<name>`,
  so this stack's `vol-pgdata` and another stack's `vol-pgdata` are distinct volumes in
  distinct Incus projects. Convenient, but easy to misread when running raw `incus` commands
  — always pass `--project buzz` or go through `incus-compose incus`.

## Design choices

- **Buzz runs its own Postgres and Redis** rather than sharing the ones another stack on this
  host already runs. Each incus-compose stack is a separate Incus project on a separate
  bridge, so sharing would mean either publishing Postgres on the tailnet or hand-attaching a
  second NIC outside incus-compose's model. Against that, a second Postgres costs ~130 MiB
  idle and a Redis ~20 MiB. Independent lifecycles, upgrades and blast radius are worth far
  more than the RAM: a `down --volumes` on an unrelated stack must never be able to take the
  agent-conversation history with it.
- **Stock `postgres:17-alpine`, not a prebuilt Buzz image.** The relay carries its own
  embedded SQLx migrations and runs them on boot with `BUZZ_AUTO_MIGRATE=true`, so no
  schema needs baking into the image.
- **Closed relay mode** (`BUZZ_REQUIRE_AUTH_TOKEN` + `BUZZ_REQUIRE_RELAY_MEMBERSHIP`) is on.
  Tailscale already limits who can reach the port; membership limits who can *use* the relay
  once they can. Both, not either.
- **The whole upstream surface is provisioned** — chat, media, git hosting and workflows,
  including the git data volume and HMAC secret — even though only channels and DMs are used
  initially. Keeping the stack shaped like upstream's bundle makes it far easier to diff
  against, and avoids a migration later.
- **Resource limits and log rotation** (`cpus`, `mem_limit`, `logging.options.max-size`) are
  set deliberately, not decoratively — unbounded logs on a long-running self-hosted stack
  will eventually fill the disk.

## TLS

This deployment is plain `ws://` over Tailscale: the tailnet is the security boundary and
nothing terminates TLS. That is a deliberate starting point, **not a verified one** — the
Buzz iOS and Android apps may refuse a non-`wss://` relay, and this has not been tested.

If they do, there are two ways out, and both change `RELAY_URL` (which must then be updated
byte-for-byte everywhere, including every agent's `BUZZ_RELAY_URL`):

- **Caddy** — upstream ships `compose.caddy.yml` for exactly this, with a `!reset` on the
  relay's direct port so only Caddy is exposed. It expects a public domain and Let's Encrypt;
  pointing it at a `.ts.net` name instead is the adaptation needed.
- **Tailscale Serve** — terminates TLS on the host with a `.ts.net` cert and proxies to the
  relay. Fewer containers, but it's host-level config living outside this stack.

## Keeping this in sync with upstream

Buzz's own [`deploy/compose/`](https://github.com/block/buzz/tree/main/deploy/compose) bundle
is the source of truth; this repo is a hand-adapted version of it. Buzz is **pre-1.0 with
daily builds**, so upstream changes to required env vars, service dependencies or default
commands will not propagate here on their own. Diff against upstream's `compose.yml`
periodically rather than assuming this file stays current.

Deliberate divergences from upstream, so a diff doesn't re-litigate them each time:

| Upstream | Here | Why |
|---|---|---|
| `name: buzz-prod` | `name: buzz` | Becomes the Incus project name |
| `env_file: .env` on `relay` | Explicit `environment:` entries | `.env` stays purely an interpolation source, and `incus-compose config` renders what actually reaches the container |
| `networks: buzz-net` | omitted | incus-compose creates one bridge per project |
| `ports:` on `relay` | moved to `compose.incus.yaml` | Compose merges port lists; duplicates collide |
| `entrypoint:` + folded string on `minio-init` | single-element `command:` list | Compose word-splits bare strings, truncating at `&&` |
| relay `start_period: 30s` | `120s` | `ic-healthd` doesn't re-probe after a failure |
| no `PSQL_PAGER` | `PSQL_PAGER: cat` | incus-compose gives the entrypoint a TTY |
| `BUZZ_IMAGE` env var | `BUZZ_DIGEST` in `versions.env` | Matches the pinning scheme used across these stacks |
| `postgres:17-alpine`, `PGDATA=…/data/pgdata`, volume at `…/postgresql/data` | `postgres:18-alpine`, `PGDATA` unset, volume at `/var/lib/postgresql` | PG18 relocated both; upstream's paths are PG17-era |
| relay runs as image default (`buzz`, uid 1000) | `user: "0:0"` | incus-compose provisions volumes root-owned and rejects anything else |

### Image pinning

`versions.env` carries the pins. The relay one deserves an explanation:

**There are no semver tags for the relay image.** Verified against the registry on
2026-08-26: `v0.5.2`, `0.5.2` and `v0.4.26` all 404. The `v*` GitHub releases publish no
correspondingly-tagged relay image, and every release since 2026-07-31 is `desktop-v0.5.x` —
the Tauri desktop app, not the relay. What exists is `main`, `latest`, and several hundred
`sha-<7>` per-commit builds.

**`sha256-*` tags are not images.** GHCR publishes Sigstore attestation bundles under a tag
named `sha256-<digest-of-the-image>` — SLSA provenance, not a rootfs. Since `main`'s digest
is reported as `sha256:<hex>`, the tag `sha256-<same-hex>` looks like the pinned equivalent
and is in fact its signature. Roughly half the tags in this repository are these sidecars.
The runnable per-commit tag is `sha-<7>`, with one dash and seven hex characters. To go from
a digest to its tag, read `org.opencontainers.image.revision` from the image config and take
the first 7 characters.

**Do not reach for `:latest`.** It is stale — built 2026-08-08, and labelled
`org.opencontainers.image.version=0.2.1` while upstream's release line reads v0.5.x. It is
*not* an alias for `main`; the two resolve to different digests. Anyone assuming otherwise
silently runs a months-old relay.

**`sha256-*` tags are not images either** — see the note above; they're Sigstore attestation
bundles.

**`sha-<7>` tags cannot be ranked.** A commit hash has no order of its own, and ghcr.io's
tags API returns names only, with no push times — so `x-update: {order: pushed}` has nothing
to sort by here. A wrong pick is a downgrade, and a downgrade is not harmless: an older relay
refuses to start against a database migrated by a newer one:

```
migration 20 was previously applied but is missing in the resolved migrations
```

and if the schema happens to be compatible, it silently runs old code.

So every pin in `versions.env` is a **digest**, with the human-readable tag left literal in
`compose.yaml` (`image:label@${DIGEST}`) — the same shape Search_Providers uses for
`nuq-postgres`. `:main` is rebuilt from HEAD, so its digest is always genuinely current, and
`mode: digest` follows it. To recover the commit behind a digest, read the image's
`org.opencontainers.image.revision` label with the snippet above.

**Postgres's major version is deliberately literal** (`postgres:18-alpine@${DIGEST}`) and
must stay out of `versions.env`. An earlier `tag_re: '^1[7-9]-alpine$'` let the updater
offer 17→18 as a routine bump; PG18 binaries refuse to start against a PG17 data directory,
so a major upgrade is a `pg_upgrade`/dump-restore project. Keeping the major out of the pins
file means `apply-all` can never perform one by accident.

Pin updates are managed by
[`incus-compose-update`](https://github.com/OleMussmann/incus-compose-update). `STACK_DIRS`
should include this directory.

## Backups

`down --volumes` drops everything, and Buzz is the durable record of agent conversations.
Four volumes hold state:

| Volume | Holds |
|---|---|
| `vol-pgdata` | Every event: messages, reactions, workflow steps, reviews, membership |
| `vol-miniodata` | Uploaded media |
| `vol-gitdata` | Hosted git repositories |
| `vol-redisdata` | Pub/sub only — reconstructible, not worth backing up |

Snapshot Postgres and the object/git volumes from the **same maintenance window**; a
Postgres dump paired with a later media snapshot will reference blobs that the dump doesn't
know about.

### Backups are host-side — this repo makes none

**Nothing here creates a snapshot.** There is no `snapshots.*` key in `compose.yaml`, no
cron unit, no `x-` extension that sets one up. Deploying this stack gives you **no** data
protection until you arrange it on the Incus host yourself, and `down --volumes` is then
unrecoverable.

Both options below cover the four volumes above. Option B is the better fit for this stack,
because it snapshots them together.

Names on the host are not the names in `compose.yaml`: incus-compose prefixes each volume
with `vol-`, and the Incus project is `buzz` — taken from `compose.yaml`'s `name:` key, not
from the directory. Volumes the image declares itself, rather than this file, show up as
`vol-auto-<service>-<path>`.

Every `incus` command below needs to be pointed at the right project. Running them as
`incus-compose incus <args>` does that for you; plain `incus` needs an explicit `--project`.
Substitute your own storage pool for `<pool>` (`incus storage list` — commonly `default`).

#### Option A — let Incus snapshot each volume on a schedule

Incus can do this on its own, on any storage driver: ZFS and btrfs give cheap copy-on-write
snapshots, LVM thin snapshots, and the `dir` driver falls back to a full copy.

```
incus-compose incus storage volume set <pool> vol-pgdata \
  snapshots.schedule=@daily \
  snapshots.expiry=4w
```

Then check what a volume actually carries, and what has been taken:

```
incus-compose incus storage volume show <pool> vol-pgdata
incus-compose incus storage volume snapshot list <pool> vol-pgdata
```

Two behaviours here cost more time than they should:

- **`@daily` is not midnight, and not the same moment for every volume.** Incus expands it
  to `<minute> <hour> * * *`, where both fields are a *stable pseudo-random* value derived
  from the volume's internal database id — deliberate load-spreading, "scheduled time
  obfuscation" in the upstream source. So each volume fires once a day at its own fixed but
  arbitrary time, and after you set a schedule it can take a full 24 h before every volume
  has a first snapshot. An empty `snapshot list` an hour after setup is expected, not a
  fault. If you need a predictable window, give a cron expression instead of the alias:
  `snapshots.schedule="30 3 * * *"`.
- **Expiry units are case-sensitive.** `S`econds, `M`inutes, `H`ours, `d`ays, `w`eeks,
  `m`onths, `y`ears — `2m` is two months, `2M` is two minutes. `snapshots.expiry` applies to
  hand-taken snapshots too, unless you pass `--no-expiry`.

#### Option B — `incus-compose backup`

incus-compose can snapshot a project's data volumes into a separate backup project in one
pass, which is the easier answer when several volumes have to be consistent with one another
— exactly the maintenance-window problem described above:

```
incus-compose backup create --name pre-upgrade
incus-compose backup list
incus-compose backup verify <timestamp>
incus-compose backup restore <timestamp>
incus-compose backup delete --keep-last 7
```

It has no scheduler of its own — drive it from cron or a systemd timer. `--live` snapshots
without stopping anything, which buys you a crash-consistent copy rather than a clean one.
See `incus-compose backup --help`.

#### Taking and restoring one by hand

Worth doing before any upgrade, whichever option you run:

```
incus-compose incus storage volume snapshot create <pool> vol-pgdata pre-upgrade
```

Restoring needs the volume idle — a custom volume in use by a running instance cannot be
rolled back, so stop the stack first:

```
incus-compose down
incus-compose incus storage volume snapshot restore <pool> vol-pgdata pre-upgrade
incus-compose up -d
```

#### Snapshots are not backups

They live on the same pool as the data they protect. A dead disk, a destroyed pool or a
mistaken `incus project delete` takes both. For anything you would genuinely miss, get a
copy off the host:

```
incus-compose incus storage volume export <pool> vol-pgdata volume.tar.gz
incus-compose incus storage volume copy <pool>/vol-pgdata <remote>:<pool>/vol-pgdata
```

`.env` is *not* covered by any of this — it lives only on your client machine, and it holds
the relay and owner private keys. Back it up separately.
