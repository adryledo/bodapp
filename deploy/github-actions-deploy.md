# One-click deploy from GitHub Actions

The repository ships a CI/CD pipeline (`.github/workflows/deploy.yml`) that deploys
the app to your Hetzner VPS **over Tailscale** — no self-hosted runner, no registry,
no public SSH port. You click **"Run workflow"** on the Actions tab (or push to
`main`) and it:

1. Joins your tailnet from the runner with the **ephemeral** key
   `TAILSCALE_AUTHKEY` (temporary node; logs out when the run ends), then
   connects to the VPS **over Tailscale SSH** — `DEPLOY_HOST` is the
   server's MagicDNS name or `100.x` tailnet IP, because the Hetzner
   firewall accepts SSH only from the tailnet.
2. Ensures the repo is cloned (or clones it) into `REPO_DIR`.
3. Writes the server `.env` from a GitHub **secret** (`ENV_FILE`).
4. Runs `deploy/deploy.sh`: fetches latest code → builds the new image **while
   the old container keeps serving** → runs `prisma migrate deploy` →
   gracefully recreates **only the app container** (Postgres keeps running;
   the app is loopback-only on `127.0.0.1:3001`) → gates on
   `http://127.0.0.1:3001/healthz` and rolls the image back on failure.
   (It also forces `PUBLIC_BASE_URL=https://$CADDY_DOMAIN` into `.env` when
   that secret is set, so invite/QR links always match the domain.)

**Two pipelines, strictly separated:**

| Pipeline | Workflow | Owns |
|---|---|---|
| **Business logic** | `deploy.yml` (this one) | app image, migrations, the `bodapp-app` container, `.env`, app health gate + image rollback |
| **Infrastructure** | [`infra.yml`](../.github/workflows/infra.yml) — *"Infrastructure — Caddy (HTTPS)"* | Caddy install/config/reload, `/etc/caddy/Caddyfile`, certificates, end-to-end TLS gate |

The infrastructure workflow never builds or restarts the app, and this
workflow never touches Caddy (it does not even reference `/etc/caddy`).
`infra.yml` triggers on **workflow_dispatch** or on pushes to `main` that
change `deploy/caddy/**`; it ships its assets to `/tmp` on the server, so the
two pipelines share no working-tree state. See
[`migrate-to-https.md`](migrate-to-https.md).

## 0. What you need beforehand

- A Hetzner VPS (see [`hetzner-setup.md`](hetzner-setup.md) for OS/Docker install).
- A **Tailscale tailnet** with the server joined to it — the Hetzner firewall
  accepts SSH only from the tailnet, and the runner joins it per run with the
  **ephemeral** `TAILSCALE_AUTHKEY` secret (logs out at the end of the run).
- The repo pushed to GitHub (this app: `adryledo-hermes/bodapp`).
- Your real values for every secret in `.env.example`.

## 1. One-time server setup (do this once, on the VPS)

```bash
# 1. Docker Engine + compose plugin already installed (see hetzner-setup.md).

# 2. Clone the repo where the deploy will live (default /opt/bodapp):
sudo apt install -y git
sudo mkdir -p /opt/bodapp && sudo chown "$USER" /opt/bodapp
git clone https://github.com/adryledo-hermes/bodapp.git /opt/bodapp

# 3. Pre-create the photo-storage dir and make it writable by the container's
#    non-root user (uid 1001):
mkdir -p /opt/bodapp/storage/photos && sudo chown -R 1001:1001 /opt/bodapp/storage
```

> The pipeline writes `.env` automatically each deploy from the GitHub secret, so
> you do **not** need to create `.env` manually on the server.

## 2. Create a deploy SSH key (local machine)

> **Transport:** SSH/scp now travel **over Tailscale** — the runner joins
> your tailnet with `TAILSCALE_AUTHKEY` and `DEPLOY_HOST` is the box's
> tailnet address (MagicDNS name or `100.x` IP; the firewall has no public
> port-22 rule). The keypair below is still used when the server runs
> normal `sshd`; with the **Tailscale SSH server**
> (`sudo tailscale set --ssh` on the box) authorization is by tailnet
> identity/ACL and the key is simply ignored.

Generate a dedicated keypair for the workflow (do **not** reuse your personal key):

```bash
ssh-keygen -t ed25519 -C "bodapp-github-actions" -f ~/.ssh/bodapp_deploy -N ""
```

- **Public key** → add to the VPS user you'll SSH as (e.g. `deploy` or your user):

  ```bash
  ssh-copy-id -i ~/.ssh/bodapp_deploy.pub <user>@hetzner.tail1234.ts.net   # tailnet address
  # or manually append the .pub contents to ~/.ssh/authorized_keys
  ```

- **Private key** (`~/.ssh/bodapp_deploy`) → paste into the GitHub secret
  `DEPLOY_SSH_KEY` below.

## 3. Configure a GitHub environment and its secrets

The workflow reads its secrets from a GitHub **environment** named `prod`
(job-level `environment: prod`). Create it and add the secrets there.

**Create the environment:** **Settings → Environments → New environment** → name it
`prod` → **Save**. (Optionally set *Deployment branches* to `main` and add *Required
reviewers* if you want an approval gate before deploys.)

Then under **Settings → Environments → prod → Environment secrets → Add secret**,
add each of these:

| Secret | Value |
|---|---|
| `TAILSCALE_AUTHKEY` | **Ephemeral** tailnet auth key — Tailscale admin console → *Settings → Keys → Generate ephemeral auth key* (tick **Ephemeral**). The runner joins the tailnet as a temporary node each run and logs out at the end. Required by **both** workflows. |
| `DEPLOY_HOST` | **Tailnet address** of the VPS: MagicDNS name (e.g. `hetzner.tail1234.ts.net`) or `100.x` IP — **not** the public IP; the firewall accepts SSH only from the tailnet |
| `DEPLOY_USER` | SSH user (e.g. `deploy` or `root`) — with the Tailscale SSH server, authorization is by tailnet identity/ACL |
| `DEPLOY_SSH_KEY` | **Private** key content of `~/.ssh/bodapp_deploy` (the PEM/OpenSSH text). Needed for normal `sshd`; ignored by the Tailscale SSH server (harmless to keep passing) |
| `ENV_FILE` | The **entire contents of your server `.env`** — copy the `.env` you'd create from `.env.example` with all real values (DATABASE_URL/POSTGRES_*, TWILIO_ACCOUNT_SID/AUTH_TOKEN/PHONE_NUMBER, SESSION_SECRET, …). Keep this in sync with what the app needs. `PUBLIC_BASE_URL` is derived from `CADDY_DOMAIN` automatically; delete any obsolete `APP_PORT` line. |

**HTTPS — set both and the Infrastructure pipeline owns Caddy end-to-end** (install, Caddyfile render from the secrets, `caddy validate`, start/reload, end-to-end `https://…/healthz` gate — no manual Caddy steps):

> Ready-to-fill catalog of exactly the infrastructure pipeline's secrets:
> [`infra.env.example`](infra.env.example).

| Secret | Example | Notes |
|---|---|---|
| `CADDY_DOMAIN` | `app.yourdomain.com` | Public hostname. Consumed by **both** pipelines: `deploy.yml` forces `PUBLIC_BASE_URL=https://$CADDY_DOMAIN` into the server `.env` on every app deploy (invite/QR links never drift); `infra.yml` renders it into the Caddyfile and drives the TLS gate. |
| `CADDY_EMAIL` | `ops@yourdomain.com` | ACME / Let's Encrypt contact email — **`infra.yml` only** (certificate expiry notices). |

Optional:

| Secret | Default | Notes |
|---|---|---|
| `DEPLOY_PORT` | `22` | SSH port if non-standard |
| `REPO_DIR` | `/opt/bodapp` | Where the repo lives on the VPS |
| `REPO_URL` | `https://github.com/adryledo-hermes/bodapp.git` | Override if forks/copies |

## 4. Deploy

- **Manual / one-click:** open the **Actions** tab → select **"Deploy to Hetzner"** →
  **Run workflow** → choose branch (default `main`) → **Run workflow**.
- **Automatic:** every push to `main` triggers a deploy (remove the `push:` trigger
  in the workflow if you want manual-only).

## 5. Verify

**Business logic (`deploy.yml`):** the run prints the health gate
(`http://127.0.0.1:3001/healthz`) and the loopback-only port assertion.

**Infrastructure (`infra.yml`):** the run prints the Caddy provisioning
output and finishes with a hard **end-to-end TLS gate** on
`https://$CADDY_DOMAIN/healthz` — any HTTP status proves DNS, firewall,
certificate and proxy are fine (a 502 just means the app container is down,
which `deploy.yml` owns).

Then in a browser:
- `https://<your-domain>/login` — panel loads over TLS with a valid certificate.
- `http://<your-domain>` — must **redirect** to https.
- `PUBLIC_BASE_URL` is derived from `CADDY_DOMAIN` on every app deploy (no
  need to keep it in `ENV_FILE`; it is overridden). The app port is fixed at
  loopback `127.0.0.1:3001`; `APP_PORT` no longer exists.

## Troubleshooting

- **`docker: command not found` / compose plugin** → install Docker Engine + the
  compose plugin on the VPS first (hetzner-setup.md).
- **SSH permission denied** → over the tailnet the same rules apply: if the
  server runs `sshd`, confirm `DEPLOY_SSH_KEY` is the **private** key and its
  public half is in `DEPLOY_USER`'s `authorized_keys`; with the Tailscale SSH
  server, check the tailnet ACLs allow this node → `<server>:22` and that
  `DEPLOY_HOST` / `DEPLOY_USER` are right.
- **`FATAL: TAILSCALE_AUTHKEY secret is not set`** → add it to the `prod`
  environment: an **ephemeral** key from the Tailscale admin console →
  *Settings → Keys → Generate ephemeral auth key*.
- **`FATAL: cannot reach <host>:<port> over the tailnet` (join preflight)** →
  the runner joined but the server side is wrong: box not on the tailnet,
  `DEPLOY_HOST` still the **public** IP (must be MagicDNS name / `100.x`),
  Tailscale SSH not enabled (`sudo tailscale set --ssh`) or `sshd` down, or
  ACLs deny it. Compare `tailscale status` on the box with the run log.
- **`Tailscale SSH requires an additional check` + `Connection … port 22
  timed out` (exit 255)** → the tailnet's `ssh` ACL matched with
  **`"action": "check"`** (interactive approval) — CI can never open that
  `login.tailscale.com` URL. The workflow's auth probe now fails fast and
  prints the fix. In the **admin console → Access controls → ssh**, add an
  **`accept` rule above the `check` rule** (first match wins):

  ```json
  {"action": "accept", "src": ["autogroup:member"], "dst": ["autogroup:self"], "users": ["root", "autogroup:nonroot", "*"]}
  ```

  Adjust: `dst` → the server's tag if it is tagged; `users` must include
  your `DEPLOY_USER`; tighten `src` to `tag:ci` if you create the ephemeral
  key with that tag. Alternative: no Tailscale SSH at all
  (`sudo tailscale set --ssh=false`) + `sshd`/`authorized_keys` — the
  workflow already passes `DEPLOY_SSH_KEY`.
- **App comes up but health probe fails** → the pipeline prints logs on
  failure and rolls back the previous image; SSH in and run
  `cd /opt/bodapp && docker compose logs --tail=100 app`.
- **`502 Bad Gateway` from https://** → Caddy is up but the app container is
  down/unhealthy: `cd /opt/bodapp && docker compose ps && docker compose logs --tail=100 app`.
- **HTTPS gate red in `infra.yml` / `curl: (60)` certificate error** →
  TLS end-to-end failed: confirm the `CADDY_DOMAIN`/`CADDY_EMAIL` secrets
  match the DNS A record, the firewall allows 80/443
  ([`hetzner-firewall.md`](hetzner-firewall.md)), then run
  `caddy validate --config /etc/caddy/Caddyfile` and
  `journalctl -u caddy -n 100` — see [`migrate-to-https.md`](migrate-to-https.md).
- **`infra.yml` failed on port 80 during a migration** → the legacy app
  container still held :80 because `infra.yml` ran before the app swap; once
  `deploy.yml` is green, *Re-run failed jobs* on `infra.yml`.
- **`ENV_FILE` secret not available / empty on manual run from a non-default branch** →
  the job is guarded with `if: github.ref == 'refs/heads/main'`, so a manual "Run workflow"
  from another branch is skipped (not executed). Always deploy `main`. If you need other
  branches, set **Deployment branches** on the `prod` environment and relax that guard.
- **`.env` not matching DB** → `DATABASE_URL` and `POSTGRES_PASSWORD` inside
  `ENV_FILE` must agree with each other (compose re-interpolates from the same vars).
