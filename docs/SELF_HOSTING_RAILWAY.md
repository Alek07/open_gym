# Deploying openGym on Railway

This runs openGym as **two Railway services from one repo**, the same split as the Docker Compose
stack:

- **`web`** — nginx, built from [`web/Dockerfile`](../web/Dockerfile). The **public** service: it
  serves the app and proxies `/api` to the api service over Railway's private network.
- **`api`** — Node, built from [`api/Dockerfile`](../api/Dockerfile). **Private** (no public
  domain); the web service reaches it at `<name>.railway.internal`.

Passkey (WebAuthn) login needs the app and `/api` on **one origin**, so only `web` is public —
the browser only ever talks to the web domain, and nginx forwards `/api` to `api` internally.
Don't give `api` a public domain and don't point the frontend at a second host; that would be
cross-origin and break passkeys and the session cookie.

Config-as-code is committed: [`railway.json`](../railway.json) (web) and
[`api/railway.json`](../api/railway.json) (api). Each service reads the config at its own **Root
Directory** — so the Root Directory setting below is what makes Railway pick up the right one.

## Why this needed a tiny code change

Railway's private network is **IPv6-first** and its internal DNS lives at `[fd12::10]`, but
`web/nginx.conf.template` shipped with nginx's resolver pinned to `ipv6=off` — so on Railway every
`/api` lookup would fail and hang. The resolver's `ipv6`/`valid` flags are now env-configurable
(`RESOLVER_IPV6`, `RESOLVER_VALID`), defaulting to the old Docker values so nothing else changes.
On Railway you set `RESOLVER_IPV6=on` (below).

---

## 1. api service (private)

1. **New Project → Deploy from GitHub repo**, pick your openGym repo. Railway creates one service;
   this will be `api`.
2. Service → **Settings**:
   - **Root Directory:** `api` (so Railway uses `api/railway.json` + `api/Dockerfile`).
   - Rename the service to **`api`** (Settings → Service name) — this becomes its private hostname
     `api.railway.internal`, which the web service points at.
   - **Networking:** do **not** generate a public domain. Leave only private networking on.
3. **Variables** (Settings → Variables):

   | Variable | Value | Why |
   |---|---|---|
   | `RP_ID` | your **web** domain, e.g. `opengym-production.up.railway.app` | Passkeys bind to the public hostname — hostname only, no scheme/port/path |
   | `ORIGIN` | `https://` + that same domain | Must match the web origin exactly |
   | `TRUST_PROXY` | `1` | Lets the sign-in throttle / audit log read the real client IP from the forwarded headers |

   `PORT` defaults to `3000` and `DATA_DIR` to `/data`; you don't need to set them. You can't fill
   `RP_ID`/`ORIGIN` until you know the web domain (step 2) — set them, or redeploy, once you do.
4. **Volume** (Settings → Volumes → New Volume), mount path **`/data`**. The API stores *everything*
   there — accounts, passkey credentials, the session-signing `secret`, VAPID keys, per-user state,
   uploads. **Without a volume it's all wiped on every redeploy.** The image runs as root, so
   there's no volume-permission workaround needed.

The api binds to `::` (dual-stack), so it's reachable over Railway's IPv6 private network.

## 2. web service (public)

1. In the same project: **New → GitHub Repo → (same repo)**. A second service appears; this is
   `web`.
2. Service → **Settings**:
   - **Root Directory:** empty / repo root (so Railway uses the root `railway.json` +
     `web/Dockerfile`).
   - **Networking → Generate Domain.** The result (e.g. `opengym-production.up.railway.app`) is the
     domain you used for `RP_ID`/`ORIGIN` on the api service. Set its **target port to `8080`**
     (the port nginx listens on, below).
3. **Variables:**

   | Variable | Value | Why |
   |---|---|---|
   | `NGINX_PORT` | `8080` | Port nginx listens on. Not 80: on Railway's runtime the unprivileged nginx can't bind a port below 1024. Match the domain's target port |
   | `BACKEND` | `api.railway.internal` | The private hostname of the api service (its service name + `.railway.internal`) |
   | `PORT` | `3000` | The api's port — this is the *backend* port nginx proxies to, not the public one |
   | `RESOLVER` | `[fd12::10]` | Railway's internal DNS. **Brackets required** or nginx reads the colons as a port |
   | `RESOLVER_IPV6` | `on` | Railway private DNS is IPv6; the default `off` would fail every lookup |
   | `RESOLVER_VALID` | `1s` | Re-resolve quickly so a redeployed api's new address is picked up |

   > On the web service, `PORT=3000` is the **backend** port (the api), not the port nginx listens
   > on — that's `NGINX_PORT`. This is why you set the domain's target port to `8080` by hand
   > rather than relying on Railway's `PORT` autodetection.

4. Deploy. Open the web domain; the first account you create is the instance owner.

---

## Per-service auto-deploy (watch paths)

So a push only rebuilds the service it affects, each `railway.json` sets `build.watchPatterns`:

- **api** → `api/**`
- **web** → `frontend/**`, `web/**`, `api/coach/core/**`, `catalogue/media/**` (everything
  `web/Dockerfile` actually builds from — not just `web/`, or a frontend change wouldn't redeploy
  the app).

Two things to know: watch paths **only gate GitHub push-triggered deploys** (a manual `railway up`
always deploys), so both services must be **connected to the GitHub repo** and this config must be
on the branch Railway deploys. And patterns are repo-root-relative and the feature can be finicky
with config-as-code — verify with a test push, and if a service ignores them, set the same paths in
its dashboard **Settings → Build → Watch Paths**.

## Optional settings

Everything in [`.env.example`](../.env.example) works as a Railway **variable** on the matching
service:

- **api service:** `FIRST_USER_ADMIN`, `ADMIN_UIDS`, `INVITE_ONLY`, `ALLOW_GUEST`, `SESSION_DAYS`,
  `DEFAULT_LANG`, `PASSWORD_LOGIN`, `AUDIT_*`, `VAPID_SUBJECT`, the `MEDIA_*` limits.
- **web service:** `MEDIA_UPLOAD_MAX` (default `48m`; keep it above the api's `MEDIA_VIDEO_MAX_MB`).

## Enabling the AI Coach (later)

The Coach is off by default and the deploy above runs the no-AI `api/Dockerfile` target. There are
two ways to turn it on; the first needs no redeploy.

### Path A — a hosted provider (recommended on Railway)

Anthropic, OpenAI, Gemini and any OpenAI-compatible endpoint (Ollama, OpenRouter, …) are driven
over plain HTTPS with an API key and need **nothing beyond the default image**. So on the deploy
you already have:

1. On the **api** service, optionally set `VAPID_SUBJECT`/timeouts and, as a hard kill switch,
   leave `COACH_DISABLED` **unset** (set it to `1` to force the Coach off regardless of the admin
   toggle).
2. Open the app, go to **Settings → Admin**, turn the Coach on and connect a provider with its API
   key. The key is encrypted into `/data/coach.json` — already on your volume, so it survives
   redeploys.
3. Optional tuning (api service variables): `COACH_JOB_TIMEOUT_MS` (raise only for a slow endpoint;
   never goes below one minute).

No Dockerfile change, no rebuild.

### Path B — the Claude Agent SDK or Codex CLI inside the container

Only needed if you want those runtimes running *in* the api container (not just a hosted API key).
It builds the `coach` Dockerfile target (~300 MB: Agent SDK, Codex, bubblewrap). Two Railway
caveats:

- **Build target.** `api/railway.json` builds the default (last) stage. To build `coach` you'd set
  a Railway-specific Dockerfile or build target for the api service — ask and I'll wire it.
- **Codex needs a second volume.** The Codex provider caches its login at `/coach-auth`, kept
  separate from `/data` on purpose — but **Railway allows only one volume per service**, so Codex
  is awkward here. The Agent SDK alone is fine; Codex is the sticking point. If you need Codex,
  the Docker Compose stack (which mounts both `./data` and `./coach-auth`) is the better home.

See [AI coach](AI_COACH.md) for what the Coach can see and change.

## Troubleshooting

- **`/api` calls 502 or hang** → check `web`'s `RESOLVER=[fd12::10]` (with brackets),
  `RESOLVER_IPV6=on`, and that `BACKEND` matches the api service's exact name +
  `.railway.internal`. The api service must be in the **same project/environment**.
- **Blank page / assets 404** → the domain's target port must be `8080` (= `NGINX_PORT`).
- **Passkey won't register ("origin mismatch")** → `RP_ID`/`ORIGIN` on the **api** service must be
  the **web** domain (`RP_ID` host-only, `ORIGIN` with `https://`). After moving to a custom domain,
  update both and redeploy — existing passkeys are tied to the old hostname.
- **Everyone signed out after a deploy** → the `/data` volume isn't attached (or lost `secret`).

## Run it locally first (optional)

The two-container stack is the same shape, so a local `docker compose up` is a good smoke test
before Railway (see [Self-hosting](SELF_HOSTING.md)).
