[![Infrastructure — Caddy (HTTPS)](https://github.com/adryledo/bodapp/actions/workflows/infra.yml/badge.svg)](https://github.com/adryledo/bodapp/actions/workflows/infra.yml)
[![Deploy to Hetzner](https://github.com/adryledo/bodapp/actions/workflows/deploy.yml/badge.svg)](https://github.com/adryledo/bodapp/actions/workflows/deploy.yml)

# Bodapp

Wedding management + invitation platform for **v1** on a Hetzner VPS.

Two surfaces:
- **Private couple panel** — dashboard, guests (editable Pokémon-card view + photos), seating planner (move tables, drag guests onto chairs), decoration/gift layer, task board, invitations manager.
- **Public invitation portal** — QR/link → phone → SMS OTP → personalized invitation + RSVP.

## v1 scope locks
- Multi-phone per invitation (couple/family): phone validated against the invitation's accepted phones *before* sending OTP.
- Invitations are created **manually per guest/couple/group** in the panel (Invitations manager); each gets its own QR code.
- Seating: always-visible chairs around each table — dropping a guest on a chair seats them AND fixes their seat number in one action; tables are draggable; every table auto-creates its attached centerpiece decoration.
- OTP via **Twilio SMS only** (no WhatsApp Business number).
- Couple's **bank account** shown on invitations.
- Photo upload: couple in v1, guests in v1.1 (guest photos now supported).
- **No menu selection**, no CSV import, no 2FA, no GDPR tooling (all deferred to v2).
- HTTPS via **Caddy** (host systemd service, automatic Let's Encrypt) in front of the app, which is loopback-only on `127.0.0.1:3001`; invitations distributed by QR/link.

## Stack
Next.js 16 (App Router, webpack build), TypeScript, Tailwind + shadcn/ui, Framer Motion, Prisma + PostgreSQL, dnd-kit, Twilio, `qrcode`, next-intl, Docker Compose, Caddy (reverse proxy / TLS).

## Get started
```bash
cp .env.example .env   # fill real values
docker compose up -d    # start Postgres (if not already running)
npm install
npx prisma migrate dev
npm run dev
```

## Build & verify
```bash
npm run build   # uses webpack (Turbopack OOM-kills on the 1.9GB target box)
npm run lint
npm run typecheck
```

## Features (v1.2.1)
- **Guests**: editable Pokémon-style cards with **profile photo** (shown only on the card); **relationship context dropdown** (Familia, Amigos, Trabajo… + "Otro" free text); **allergy multiselect** (common options + free text); **music split** — genres multiselect + free-text favourite song. Search/filter by name, RSVP, allergy, table.
- **Seating**: always-visible chairs — drop a guest on a chair to seat them AND fix seat number in one action; re-seat a guest within the same table by dropping on another chair; tables draggable (incl. mobile, `touch-action: none`); **table size scales clearly with capacity** (round: 44+5/seat, rect: 100+8/seat × 44+3/seat); shape/capacity changes no longer reset position or capacity (PATCH writes only the changed field); duplicate-seat + capacity + conflict warnings.
- **Invitations**: per-guest/group invitations created manually (**already-invited guests are disabled** in the picker); each invitation is created **from the Invitation Template** (image + base copy) **adding the guests' names**; each can be further personalised with a **live preview**, image and text overrides, countdown, day schedule, directions, accommodation and its **QR underneath** (the old QR panel is gone; `/qr` redirects to `/invitaciones`). The template editor (`/invitacion`) shares the same fields: image, names, message, **date/time pickers**, venue, dress code, schedule, directions, accommodation and colours.
- **Dashboard** (`/panel`): guests/tables/invitations/tasks summary + next pending task.
- **Decorations**: decorations **attach to a table when dropped on it** (rendered at the table's position) and **move together when the table moves**; drop on empty canvas to detach; every table auto-creates its centerpiece.
- **Photos**: the **wedding gallery shows ONLY wedding photos** — guest profile photos and invitation images (same storage) are excluded from it. The invitation editor shows a **live preview** (frame + image + text) under the current data.
- **Tasks**: Kanban + calendar views, pre-seeded Spanish wedding checklist.
- **Public portal**: QR → phone → SMS OTP → personalised invitation + RSVP. For private/testing deployments, set `REQUIRE_GUEST_OTP="false"` to open invitation links without sending SMS (explicit security trade-off; default remains OTP).

## Usage
1. Open the app; at `/` choose **Log in** (existing couple) or **Create a new couple** (self-registration at `/setup`).
2. Navigate: **mobile** uses a hamburger → slide-in drawer; **desktop** shows the links inline in the header.
3. **Guests** — add/edit guests (profile photo, context, allergies, music). Cards flip for details.
3. **Tables** — create tables, drag them around, drop guests on chairs.
4. **Invitations** — create per-group invitations from the template, open one to personalise image/text/logistics and download its QR.
5. **Tasks / Decorations / Photos** — plan and track the wedding.

## Roadmap
- ✅ **v1.2.2** — Editorial wedding invitation redesign inspired by fine-paper stationery: no decorative frames, cream/sage/stone palette, serif typography, live countdown, schedule, directions, accommodation and date/time pickers; template and per-invitation editor share the same fields.
- ✅ **v1.2.1** — Mobile nav drawer; guest context dropdown + allergy multiselect + music split (genres + favourite song); seating fixes (same-table re-seat, table size by capacity, PATCH-only-changed-fields so position/capacity never reset); profile photo vs wedding gallery separation; invitation panel upgrades (disable already-invited, rich per-invitation editor with image/text, QR inline under each invitation, QR panel removed).
- ✅ **v1.2** — Dashboard, per-guest seat numbers + draggable tables, attachable decorations (auto centerpiece), invitations per group, guest editing + photos, Playwright harness.
- ✅ **v1.1** — Multi-couple self-registration, welcome screen, configurable session expiry, logout, cookie/OTP hardening.
- ⬜ **v1.3** — Seating-map PDF export, guest relations UI, CSV import, OTP delivery dashboard, email notifications, 2FA, PWA.

## Testing
- **Unit tests:** `npm test` (vitest — pure libs: slugs, seating, OTP, i18n parity, …).
- **E2E UI tests (Playwright):** `npm run test:e2e` — see [`tests/e2e/README.md`](tests/e2e/README.md) (DB-free smoke on the welcome page; DB-backed flows added per feature).

Deployment instructions: [`deploy/hetzner-setup.md`](deploy/hetzner-setup.md).

## Deploy (Docker Compose on Hetzner VPS, HTTPS via Caddy)

- **One-click CI/CD deploy:** GitHub Actions → [`.github/workflows/deploy.yml`](.github/workflows/deploy.yml) (**Tailscale SSH deploy**: the runner joins the tailnet with the ephemeral `TAILSCALE_AUTHKEY` secret and reaches the box over Tailscale — click *Run workflow* on the Actions tab, or auto on push to `main`). Configure the required secrets + one-time server setup in [`deploy/github-actions-deploy.md`](deploy/github-actions-deploy.md).
- **Manual runbook:** [`deploy/hetzner-setup.md`](deploy/hetzner-setup.md).
- **Reverse proxy / HTTPS:** **Caddy runs on the host as a systemd service**
  (never inside the app container), on its **own separate pipeline** —
  [`infra.yml`](.github/workflows/infra.yml) *"Infrastructure — Caddy (HTTPS)"* —
  driven by the `CADDY_DOMAIN` + `CADDY_EMAIL` GitHub secrets:
  [`deploy/caddy/setup-caddy.sh`](deploy/caddy/setup-caddy.sh) installs it
  (once) and reconfigures it idempotently, then an end-to-end TLS gate.
  Config: [`deploy/caddy/Caddyfile`](deploy/caddy/Caddyfile); details:
  [`deploy/migrate-to-https.md`](deploy/migrate-to-https.md); firewall (only
  80/443): [`deploy/hetzner-firewall.md`](deploy/hetzner-firewall.md).
- **Image:** multi-stage `Dockerfile` — `node:20-alpine`, webpack build
  (`next build --webpack`), Next.js standalone `server.js`, runs as non-root,
  listens on port **3001**, published **loopback-only** (`127.0.0.1:3001`).
- **Compose:** `docker-compose.yml` stacks `postgres` (healthchecked) +
  `migrate` (one-shot) + `app` (healthchecked on `/healthz`, graceful
  `stop_grace_period`); photos mount `./storage:/app/storage`.
- **Pipelines (strictly separated):** [`deploy.yml`](.github/workflows/deploy.yml)
  = business logic — builds while the old container serves, migrates, swaps
  **only the app container**, gates on `/healthz`, asserts the loopback bind,
  rolls back the image on failure. [`infra.yml`](.github/workflows/infra.yml)
  = infrastructure — Caddy config/certs + TLS gate. Neither touches the
  other's domain. Both reach the VPS over **Tailscale**: the runner joins
  the tailnet with the ephemeral `TAILSCALE_AUTHKEY` secret, so the Hetzner
  firewall can accept SSH only from the tailnet (no public port 22).

```bash
cp .env.example .env          # fill real values (see runbook Step 4)
mkdir -p storage/photos && sudo chown -R 1001:1001 storage
docker compose up -d --build
docker compose run --rm migrate       # apply prisma/migrations
docker compose run --rm app npx --no-install prisma db seed   # optional demo couple
```

