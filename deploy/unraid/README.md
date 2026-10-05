# Running Ghostfolio on Unraid (wolverine)

Ghostfolio runs on the Unraid server as a Compose Manager stack, reachable only
over Tailscale at `https://ghostfolio.<tailnet>.ts.net`. Updates are
hands-off:

```
push to main (jonnymo5/ghostfolio)
  └─► GitHub Actions: .github/workflows/unraid-image.yml
        builds linux/amd64, boot-tests it against Postgres + Redis, pushes
        ghcr.io/jonnymo5/ghostfolio:latest + :sha-<hash>
          └─► Watchtower on wolverine (ghcr.io/jonnymo5/watchtower, built from
              the jonnymo5/watchtower fork) notices the new :latest digest
              within 5 min, pulls it, and recreates the ghostfolio-server
              container
```

Never build the image locally on the Mac for wolverine — Apple Silicon
produces arm64 images.

## GitHub Actions in this fork

| Workflow           | State    | Why                                                                 |
| ------------------ | -------- | ------------------------------------------------------------------- |
| `build-code.yml`   | active   | Lint, format check, tests and production build on every PR          |
| `unraid-image.yml` | active   | Builds and publishes the image wolverine runs                       |
| `docker-image.yml` | disabled | Upstream's multi-arch build for Docker Hub; wolverine is amd64 only |
| `extract-locales`  | disabled | Opens translation PRs on every push to main; upstream housekeeping  |

Workflows are disabled in GitHub (`gh workflow disable -R jonnymo5/ghostfolio
<name>`), not deleted, so merging upstream never conflicts. A workflow file
that upstream adds later starts out enabled — check after each upstream merge
with `gh workflow list -R jonnymo5/ghostfolio --all`.

## Why Tailscale

A Tailscale sidecar gives Ghostfolio its own tailnet machine (`ghostfolio`)
with a real Let's Encrypt certificate, so it works over HTTPS at home and away
without exposing anything to the internet or the LAN. Every device that uses
it (Mac, phone) needs the Tailscale app, signed in to the same tailnet.

The sidecar reaches Ghostfolio by container name
(`http://ghostfolio-server:3333`) over the stack's private Docker network;
ghostfolio-server publishes no ports. The app container must not be named
`ghostfolio`: that is the sidecar's own hostname, so the proxy would reach the
sidecar itself and return 502.

## One-time setup

### 1. Tailscale auth key

In https://login.tailscale.com/admin, MagicDNS, HTTPS Certificates and the
`tag:container` tag are already set up from the Actual stack (see
`deploy/unraid/README.md` in `jonnymo5/actual-budget`). Generate a new auth
key under **Settings → Keys**: not reusable, **pre-approved**, tags:
`tag:container`.

### 2. GHCR image visibility

After the first successful run of `unraid-image`, open
https://github.com/users/jonnymo5/packages/container/ghostfolio/settings and
set visibility to **public** (the repo is public and AGPL, so nothing changes
by publishing the image). Wolverine then pulls without a login.

Because anyone can pull it, **nothing environment-specific may ever go into
the image**: no hostnames, tokens, or data. Those belong in this compose file
and its `.env`.

### 3. Prepare storage on wolverine

```sh
mkdir -p /mnt/maincache/appdata/ghostfolio/postgres /mnt/maincache/appdata/ghostfolio/tailscale
```

Postgres sets the ownership of its data folder itself on first start. In the
Unraid UI, the `appdata` share should be **cache: only**.

### 4. Watchtower

Already running from the Actual setup (`deploy/unraid/README.md` in
`jonnymo5/watchtower`). Nothing to do — it picks up any container with the
`com.centurylinklabs.watchtower.enable=true` label.

### 5. Start Ghostfolio

With the **Compose Manager** plugin:

1. Add a new stack `ghostfolio`; paste in `deploy/unraid/docker-compose.yml`.
2. In the stack's `.env`, set:

   ```sh
   TS_AUTHKEY=tskey-auth-...
   ROOT_URL=https://ghostfolio.<tailnet>.ts.net
   POSTGRES_PASSWORD=...   # openssl rand -hex 32
   REDIS_PASSWORD=...      # openssl rand -hex 32
   ACCESS_TOKEN_SALT=...   # openssl rand -hex 32
   JWT_SECRET_KEY=...      # openssl rand -hex 32
   ```

   Use hex values (as `openssl rand -hex 32` makes): `POSTGRES_PASSWORD` goes
   into a connection URL, where characters like `@`, `/` or `:` break it.
   Keep a copy of `.env` somewhere safe. **Never change `ACCESS_TOKEN_SALT`**
   — the security tokens you log in with are derived from it, so changing it
   locks you out. `POSTGRES_PASSWORD` is only applied when the database is
   first created; changing it later needs an `ALTER USER` in Postgres too.

3. Compose Up. The first start takes a minute or two while the database
   migrations run.
4. In the Tailscale admin console, the machine `ghostfolio` should appear.
   Open `https://ghostfolio.<tailnet>.ts.net` — the first request takes a few
   seconds while the certificate is issued.
5. Click **Get Started** to create the first user, which becomes the admin.
   Save the security token it shows; it is your login.

After the first login, delete `TS_AUTHKEY` from `.env`; Tailscale keeps its
login in `/mnt/maincache/appdata/ghostfolio/tailscale`, and the stack starts
fine without the key. If that folder is ever lost, the container starts but
stays logged out (`docker logs ghostfolio-tailscale` shows `NeedsLogin`):
generate a new auth key, put it back in `.env`, and Compose Up.

## Day-2 operations

- **Update Ghostfolio**: merge to `main` (or run the `unraid-image` workflow
  manually). Watchtower deploys it within 5 minutes. Watch it happen with
  `docker logs -f watchtower` on wolverine.
- **Pull in upstream changes**: when you want them,

  ```sh
  git fetch upstream && git merge upstream/main && git push origin main
  ```

  (one-time: `git remote add upstream https://github.com/ghostfolio/ghostfolio.git`).
  The push triggers a build like any other. Afterwards, check for newly added
  workflows (see above).

- **Pull requests**: `gh pr create` in a fork targets upstream by default;
  always pass `--repo jonnymo5/ghostfolio`.
- **Rollback**: Ghostfolio runs database migrations on every start, so back
  up first (see below). Then pin the previous tag in the compose file —
  `image: ghcr.io/jonnymo5/ghostfolio:sha-<hash>` (tags are listed on the
  GHCR package page) — and Compose Up. Watchtower leaves pinned tags alone
  until you switch back to `:latest`. If the newer version had already
  migrated the database, restore the backup taken before it.
- **Pause auto-updates**: set the ghostfolio-server label to
  `com.centurylinklabs.watchtower.enable=false` and Compose Up.
- **Update Tailscale or Redis**: pinned by version and digest and never
  auto-updated. Bump the tag + digest in the compose file, then Compose Up.
- **Update Postgres**: minor versions (15.x) are the same as above. A major
  version (e.g. 15 → 16) needs a dump and restore into a fresh data folder;
  don't just change the tag.
- **Backups**: the reliable backup is a database dump —

  ```sh
  docker exec ghostfolio-postgres pg_dump -U ghostfolio -Fc ghostfolio > /mnt/user/backups/ghostfolio-$(date +%F).dump
  ```

  (restore with `pg_restore --clean -U ghostfolio -d ghostfolio`). Run it
  before upstream merges and on a schedule (e.g. the User Scripts plugin).
  Copying `/mnt/maincache/appdata/ghostfolio/postgres` with the appdata backup
  plugin also works, as long as it stops the container first. For a portable
  copy of your activities, Ghostfolio can also export them as JSON from the
  **Portfolio → Activities** page.

- **Away from home**: nothing to do — the tailnet URL works anywhere Tailscale
  is connected. Do NOT port-forward or enable Funnel; the serve config
  explicitly disables Funnel.
