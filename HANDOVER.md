# Handover: Nepal Open University Thesis Repository (DSpace 9.2)

This document is for whoever is deploying this repository for the first time. It assumes you're
comfortable with Linux and Docker, but assumes **no prior DSpace knowledge**. Follow it in order —
each section depends on the one before it.

There are two repositories:

* **This one (backend)** — the DSpace REST API, Java, runs in Docker.
* **Frontend** — the Angular user interface. Runs as a native Node.js process (not Docker).

---

## 0. Read this before anything else: dead hostnames must be fixed

This deployment was previously exposed through temporary Cloudflare Tunnels for testing. Those
tunnels are no longer running, and their hostnames **do not resolve to anything**. The values are
still sitting in tracked config files. Nothing will start correctly — you'll get connection errors,
CORS failures, or a blank frontend — until you replace them.

| File | Line | Current (dead) value | Replace with |
|---|---|---|---|
| `docker-compose.yml` | 20 | `dspace__P__server__P__url: https://courage-appears-albuquerque-petition.trycloudflare.com/server` | The real public URL for the REST API, e.g. `https://repository.example.edu/server`. For local-only testing, `http://localhost:8080/server`. |
| `docker-compose.yml` | 21 | `dspace__P__ui__P__url: https://smoke-trainer-vienna-tournament.trycloudflare.com` | The real public URL for the UI, e.g. `https://repository.example.edu`. For local-only testing, `http://localhost:4000`. |
| `frontend/config/config.yml` | 9 | `host: courage-appears-albuquerque-petition.trycloudflare.com` (under `rest:`) | The **hostname only** (no protocol) matching whatever you put in `dspace.server.url` above, e.g. `repository.example.edu`, or `localhost` for local-only testing. |

**These three values must describe the same deployment.** If `dspace.server.url` in
`docker-compose.yml` says `repository.example.edu` but `rest.host` in `frontend/config/config.yml`
still says something else, the frontend will fail to talk to the backend and you'll see a 500 error
or a blank page (see [Troubleshooting](#troubleshooting)).

There is a fourth, harmless occurrence of one of these dead hostnames: inside
`frontend/src/assets/pdfjs/web/viewer.mjs`, in a `HOSTED_VIEWER_ORIGINS` list. The code that reads
that list is unreachable (see warning 1 below), so it has no effect on anything — you don't need to
touch it, it's just a leftover string.

---

## 1. Two warnings that matter more than anything else in this document

**Warning 1 — the PDF viewer is hand-patched. Do not update or re-download it.**

`frontend/src/assets/pdfjs/` is a vendored copy of the Mozilla pdf.js viewer, and it has been
hand-edited to disable printing, text copying, and downloading from within the viewer, so that
items open in a view-only mode. These edits live inside `viewer.mjs` (and are supported by
`viewer.css`). If anyone re-downloads a fresh copy of pdf.js into this folder, or updates it via
some automated tool, **the view-only behavior reverts silently** — there is no error, no warning,
nothing in the logs. The viewer will simply work like stock pdf.js again (full print/download/copy
available) and nobody will notice until someone points it out. If pdf.js ever needs to be upgraded,
the person doing it must re-apply these edits by hand and test the result, not just drop in a new
release.

**Warning 2 — view-only is a frontend restriction only. It is not access control.**

The view-only viewer only stops casual users clicking "download" or "print" in the UI. The
underlying bitstream (the actual PDF file) is still served at its normal DSpace REST content URL to
anyone who requests that URL directly — with a browser, `curl`, a script, whatever. Nothing on the
backend currently blocks that. If you need real enforcement (e.g. embargoed or restricted-access
theses), you need **both**:
1. A backend-side block on the bitstream content endpoint (so DSpace itself refuses the request), and
2. DSpace resource policies on the relevant items/bitstreams (so DSpace knows who's allowed to read
   them in the first place).

The current setup does neither. Anyone with a direct link to a bitstream can download the full PDF
regardless of what the frontend viewer shows them.

---

## 2. Prerequisites (Ubuntu 22.04 / 24.04 LTS)

Install these before cloning anything.

* **Docker Engine + the Compose plugin**, from Docker's official APT repository — **not** the
  `docker.io` package from Ubuntu's own repos, which is often too old to support the `docker compose`
  subcommand used throughout this document. Follow Docker's official install guide for Ubuntu:
  https://docs.docker.com/engine/install/ubuntu/
* **Add your deploying user to the `docker` group** so you don't need `sudo` for every command, and
  log out/in (or reboot) for it to take effect:
  https://docs.docker.com/engine/install/linux-postinstall/
* **Node.js**, for building/running the frontend. The frontend requires Node `v20.x`, `v22.x`, or
  `v24.x`, and npm `>= v10.x`. Use NodeSource's setup script or your preferred Node version manager —
  do not rely on Ubuntu's default `apt` Node package, which is typically too old:
  https://github.com/nodesource/distributions

## 3. Clone both repositories

```bash
git clone -b local-customizations https://github.com/Supriya090/nou-dspace-backend.git
git clone -b local-customizations https://github.com/Supriya090/nou-dspace-frontend.git
```

* `nou-dspace-backend` — the Java/REST API. Everything in this document that mentions
  `docker compose` is run from inside this directory.
* `nou-dspace-frontend` — the Angular UI. Everything about building/serving the frontend is run
  from inside this directory.

Both repos track a `local-customizations` branch — that's the one you want, not `main`.

## 4. Fix the config values

Repeating the table from the top of this document — do this now, before starting anything:

| File | Line | Current (dead) value | Replace with |
|---|---|---|---|
| `docker-compose.yml` | 20 | `dspace__P__server__P__url` = trycloudflare URL | Your real REST API URL |
| `docker-compose.yml` | 21 | `dspace__P__ui__P__url` = trycloudflare URL | Your real UI URL |
| `frontend/config/config.yml` | 9 | `rest.host` = trycloudflare hostname | Hostname matching `dspace.server.url` above |

## 5. Start the backend stack

From inside `nou-dspace-backend`:

```bash
# 1. Start the database and search index first — dspace depends on both being ready
docker compose up -d dspacedb dspacesolr

# 2. Give them a few seconds to initialize, then start the DSpace application itself.
#    --build is required here: docker-compose.yml both builds locally (via Dockerfile.test)
#    AND references a public image name. Without --build, compose may just pull the
#    public image instead of building your customized Dockerfile.test (which includes
#    ImageMagick/Ghostscript — see the filter-media section below).
docker compose up -d --build dspace

# 3. Watch the logs and wait for startup to finish
docker compose logs -f dspace
```

Watch for a line containing `Started ServerBootApplication` in the log output — startup time varies
by hardware, from under a minute up to a couple of minutes. Do not proceed until you see it. Press
`Ctrl+C` to stop following the logs (the container keeps running in the background).

Once it's up, create the initial administrator account (you'll be prompted for email, password,
first/last name):

```bash
docker exec -it dspace /dspace/bin/dspace create-administrator
```

## 6. Build and serve the frontend

From inside `nou-dspace-frontend`:

```bash
npm install
npm run build:prod
npm run serve:ssr
```

(`npm start` does both of the last two steps in one command, if you prefer.) By default this
listens on `0.0.0.0:4000` (see `frontend/config/config.yml`).

Run this as a persistent service — a plain foreground command dies the moment your SSH session
ends. Use whatever your team already standardizes on (systemd unit, `pm2`, etc.); this document
doesn't prescribe one.

## 7. Reverse proxy and TLS

This part depends on your own infrastructure, so it's kept general. Whatever reverse proxy you use
(nginx, Apache, Caddy, etc.) needs to:

* Terminate TLS on your public hostname (e.g. via Let's Encrypt/certbot).
* Proxy requests for the UI to `127.0.0.1:4000`.
* Proxy requests under `/server` to `127.0.0.1:8080`.

Whatever public hostname you configure here **must be the same hostname** you put into
`dspace.server.url`, `dspace.ui.url`, and `config.yml`'s `rest.host` in step 4. If they don't match,
you'll hit the 500 error described in [Troubleshooting](#troubleshooting).

## 8. Set up the filter-media cron job

Thumbnails and full-text indexing (for search) are **not** generated when a file is uploaded — they
are generated by a scheduled background task called `filter-media`. Without this cron job, every
item will show "No Thumbnail Available" forever and full-text search won't find content inside
PDFs.

Add this to the crontab of whatever host is running the Docker containers (`crontab -e`):

```
*/15 * * * * docker exec dspace /dspace/bin/dspace filter-media
```

This must run on the Docker host itself (it uses `docker exec` to reach into the running `dspace`
container) — it will not work from inside the frontend server, or from some other machine.

This depends on **ImageMagick and Ghostscript** being present inside the `dspace` container. As of
this handover, they are installed via `Dockerfile.test` (see the `local-customizations` commit
history) — if you ever rebuild the image from a different Dockerfile or an unmodified upstream
image, confirm they're still present (`docker exec dspace which convert gs`), or `filter-media` will
fail silently with no thumbnails and no obvious error.

## 9. Verification checklist

A working deployment looks like this:

- [ ] `docker compose ps` (in the backend repo) shows `dspace`, `dspacedb`, and `dspacesolr` all `Up`.
- [ ] `docker compose logs dspace` contains `Started ServerBootApplication`.
- [ ] `curl -s http://localhost:8080/server/api` returns JSON (not a connection error, not a 500).
- [ ] The frontend process is running and reachable on port 4000 (or through your reverse proxy).
- [ ] The homepage shows "Nepal Open University Thesis Repository" text — if you still see generic
      "DSpace 9" text, the frontend build didn't pick up the customizations, or you're looking at
      the wrong deployment.
- [ ] You can log in with the administrator account created in step 5.
- [ ] Uploading a test item with a PDF attached, then either waiting up to 15 minutes for the cron
      job or running `docker exec dspace /dspace/bin/dspace filter-media` manually, produces a
      thumbnail.
- [ ] Clicking "View" on that item's file opens the PDF in-browser, in view-only mode (no working
      print, download, or text copy).

## 10. Troubleshooting

**500 error, or the frontend fails to load / hangs**
Almost always a mismatch between `docker-compose.yml`'s `dspace.server.url`/`dspace.ui.url` and
`frontend/config/config.yml`'s `rest.host`. They must describe the same hostname. Check both files
match, then see the next item.

**I edited `docker-compose.yml` but nothing changed**
`docker compose up -d` reuses the existing container as-is if one is already running — it does
**not** pick up new environment variables from an edited compose file. After editing
`docker-compose.yml`, you must fully recreate the container:
```bash
docker compose down
docker compose up -d --build dspace
```

**"No Thumbnail Available" on an item with an uploaded file**
1. Confirm the `filter-media` cron job (step 8) is actually installed and running (`crontab -l`).
2. Run it manually and watch for errors: `docker exec dspace /dspace/bin/dspace filter-media`.
3. Confirm ImageMagick and Ghostscript are present: `docker exec dspace which convert gs`.
4. If both tools are present and `filter-media` runs without errors but thumbnails still don't
   appear, check for ImageMagick "not authorized" / policy errors in
   `docker exec dspace tail -100 /dspace/log/dspace.log.$(date +%Y-%m-%d)` — Debian/Ubuntu-based
   ImageMagick installs commonly ship a default security policy
   (`/etc/ImageMagick-6/policy.xml`) that blocks PDF rasterization out of the box. This wasn't
   something we hit in testing, but if you rebuild the image on a different base or a newer
   ImageMagick version, it's the most common cause of exactly this symptom.

**`docker compose` warns about orphan containers**
This means there are containers running that used to be defined in this compose file (or a related
one) but no longer are — usually harmless leftovers from an earlier `up`. Clean them up with:
```bash
docker compose down --remove-orphans
```

**`docker compose up` fails with "Pool overlaps with other one on this address space", "container
name ... already in use", or "port is already allocated"**
All three mean something else on this host is already using the fixed subnet (`10.50.0.0/16`),
fixed container names (`dspace`, `dspacedb`, `dspacesolr`), or fixed published ports (`8080`, `8000`,
`5432`, `8983`) that `docker-compose.yml` hardcodes — most commonly a previous deployment attempt on
the same machine that was never torn down, or an unrelated Docker project that happens to use the
same subnet.
1. Check what's already using them: `docker network ls`, `docker ps -a`, `ss -tlnp`.
2. If it's a stale attempt you don't need anymore, remove it (`docker compose down` from that
   project's directory, or `docker rm`/`docker network rm` — check first that you're not removing
   something still in use).
3. If it needs to coexist with something else on the same host, edit this repo's copy of
   `docker-compose.yml`: change the `subnet:` value (and the matching `proxies.trusted.ipranges`,
   which must always agree with it), the `container_name:` values, and/or the `published:` ports to
   values that are free on this host.

---

## Also note, before going to production

* **`docker-compose.yml` uses the default PostgreSQL credentials** (`dspace` / `dspace`, matching
  DSpace's own built-in default `db.username`/`db.password`). Change these before any real
  deployment. To do it properly:
  1. Change `POSTGRES_USER`/`POSTGRES_PASSWORD` under the `dspacedb` service in `docker-compose.yml`.
  2. Add matching `db__P__username`/`db__P__password` environment variables under the `dspace`
     service (they aren't currently set at all, which is why the built-in `dspace`/`dspace` default
     is in effect).
  3. **PostgreSQL only applies `POSTGRES_PASSWORD` the first time it initializes an empty data
     directory.** If you change these values after the stack has already run once, you must also
     wipe the database volume for the new password to take effect: `docker compose down -v` (this
     deletes all existing data — only do this before you have real content in the repository).

* **`frontend/config/config.yml` is tracked by git**, not ignored. Any local edits you make to it
  (including the hostname fix in step 4) will show up as a modified file, and will conflict with any
  future `git pull` from this branch if the upstream copy of `config.yml` also changes. Be aware of
  this before pulling updates — you may need to stash or manually merge your local values.
