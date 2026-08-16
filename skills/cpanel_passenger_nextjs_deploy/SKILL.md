---
name: cpanel_passenger_nextjs_deploy
description: Deploy a Next.js + Prisma/MySQL app to shared cPanel hosting (CloudLinux Node.js Selector / Phusion Passenger) — the specific build-vs-server split, CI artifact traps, and cPanel-only failure modes that make this stack fight the platform if you skip them.
---

# cPanel / Passenger Next.js + Prisma Deployment

## Goal
Shared cPanel hosting (InterServer, CloudLinux Node.js Selector, Passenger/LiteSpeed)
looks like it should run any Node app, but it has hard resource limits and platform
conventions that a normal Next.js/Prisma deploy silently violates. This skill is the
concrete gotcha catalog from actually shipping one (cloud.bongshai.com), so the next one
skips straight to a working deploy instead of re-discovering each failure by trial and
error against a live production DB.

## The one rule that avoids most of this
**Never run `next build`, `prisma generate`, `prisma migrate`, or `npm install` for
production on the cPanel server itself.** Build entirely in GitHub Actions (Linux,
uncapped memory), assemble a ready-to-run bundle there, and only `npm install` (for
`node_modules`, via cPanel's own UI — see below) and start the app on the server. Shared
hosting's per-process memory cap is real and will OOM a full framework build; see Gotcha 6.

## Gotcha catalog

Each of these produced a real, confusing failure in production before being root-caused.
Symptom → root cause → fix, in the order you're likely to hit them.

### 1. `next build` succeeds locally, CI fails on `Cannot find name 'LayoutProps'`
Next.js's typed-routes feature writes ambient types (`LayoutProps`, `PageProps`,
`RouteContext`) to `.next/types/` as a side effect of `next dev` or `next build` — but
`tsc --noEmit` run as a separate CI step, before a build has ever populated that folder,
fails to find them. **Fix**: run `npx next typegen` explicitly before type-checking in CI.

### 2. CI fails on `Missing required environment variable: DB_HOST` even though the build has no reason to touch the DB
`next build`'s page-data-collection phase *imports* every route module to read its
config exports (`runtime`, `dynamic`, etc.) — it does not call the handlers, but the
import alone runs any module-scope code. A `PrismaClient` (or any DB client)
constructed at module scope crashes the whole build in an environment with no DB
credentials (e.g. CI). **Fix**: wrap the client in a lazy `Proxy` so the real client is
only constructed on first property access, which only happens inside an actual request:
```ts
function getClient(): PrismaClient {
  if (!globalThis.__prisma) globalThis.__prisma = createClient();
  return globalThis.__prisma;
}
export const prisma = new Proxy({} as PrismaClient, {
  get(_target, prop, _receiver) {
    const client = getClient();
    const value = Reflect.get(client as object, prop, client);
    return typeof value === "function" ? value.bind(client) : value;
  },
});
```
Verify by temporarily removing all DB env vars locally and confirming `npm run build`
still succeeds.

### 3. Deployed site 404s on static files that are definitely in the repo (favicon aside)
`output: "standalone"` does **not** copy `public/` or `.next/static/` into
`.next/standalone/` — by design, so you assemble the real bundle yourself:
```yaml
- name: Assemble deploy bundle
  run: |
    mkdir -p deploy
    cp -r .next/standalone/. deploy/
    mkdir -p deploy/.next/static
    cp -r .next/static/. deploy/.next/static/
    cp -r public deploy/public
```
An empty `public/` directory is also invisible to git (git doesn't track empty dirs), so
if the project never had a real file in `public/`, the whole folder silently doesn't
exist in a fresh checkout either — commit a real placeholder (`robots.txt` works and is
useful anyway), not `.gitkeep`.

**Diagnosing this in production**: if a request to a `public/` file returns your app's
own custom 404 page (check for `X-Powered-By: Next.js` and your app's HTML in the body,
not a bare server 404), the request reached Node and Next's router didn't find the file
— the file is missing from the *server's* `public/` folder specifically, not a routing
bug. This is what a stale/partial deploy of just the `public/` folder looks like; a full
site otherwise working correctly (pages, API routes, `.next/static/*.js` chunks all
loading fine) but one particular static asset 404ing is the fingerprint.

### 4. `actions/upload-artifact` produces a zip that's missing `.next/` entirely
`upload-artifact@v4` excludes dotfiles and dot-prefixed directories **by default** —
and `.next/` is dot-prefixed. The action reports success, the zip downloads fine, it's
just missing the one directory that matters most. This produces symptoms that look like
a File Manager or archive-extraction bug (partial extraction, "corrupted zip") when the
zip itself was already incomplete before upload. **Fix**:
```yaml
- uses: actions/upload-artifact@v4
  with:
    path: deploy/
    include-hidden-files: true
```
**Diagnose**: unzip the downloaded artifact locally and diff its top-level contents
against what the "Assemble deploy bundle" step's own `find deploy -maxdepth 2` logged in
the Actions run — don't assume the zip matches the log.

### 5. MySQL tables silently created as MyISAM
Some shared-hosting MySQL configs default to MyISAM, not InnoDB, for new tables. This
breaks foreign key constraints *and* transactions/row-locking silently — no error,
FKs just don't enforce and `SELECT ... FOR UPDATE` doesn't lock. If the app relies on
transactional quota/inventory-style reservations, this is a correctness bug, not just a
perf one. **Fix**: after every `prisma db push`, convert any non-InnoDB table:
```ts
const tables = await prisma.$queryRawUnsafe<{TABLE_NAME:string,ENGINE:string}[]>(
  `SELECT TABLE_NAME, ENGINE FROM information_schema.TABLES WHERE TABLE_SCHEMA = DATABASE()`
);
for (const t of tables.filter(t => t.ENGINE !== "InnoDB")) {
  await prisma.$executeRawUnsafe(`ALTER TABLE \`${t.TABLE_NAME}\` ENGINE=InnoDB`);
}
```
Wire this into the `db:push` script itself (`prisma db push && tsx ensure-innodb.ts &&
prisma db push`) so it can never be forgotten after a schema change.

### 6. Prisma CLI (`generate`/`migrate`/`db push`) OOMs on the server with a WASM error
`RangeError: WebAssembly.Instance(): Out of memory: Cannot allocate Wasm memory for new
instance`. Prisma 7's schema engine is a build-time WASM module and it's memory-hungry;
shared-hosting per-process memory caps are usually well below what it needs. This is a
**hard resource limit**, not a config problem — there is no flag that fixes it. **Fix**:
never run these commands on the server. Run `prisma generate` in CI (Linux, generous
memory) and ship the already-generated client as part of the deploy bundle;
`db push`/`migrate` from your local machine or CI against the production DB directly
(the DB is reachable over the network independent of where the app runs).

### 7. CloudLinux Node.js Selector rejects an uploaded `node_modules`
CloudLinux's Node.js Selector manages `node_modules` as a symlink into its own nodevenv
and expects to populate it itself via "Run NPM Install" in the UI — it does not accept a
pre-built `node_modules` folder dropped in by zip extraction (uploading one produces an
app that won't start, with an error naming the symlink target). **Fix**: build
everything else in CI, but exclude `node_modules` from what you upload; extract just
`server.js` + `.next/` + `public/` + `package.json`, then use "Run NPM Install" from the
cPanel UI to let it populate its own managed `node_modules`.

### 8. "Run NPM Install" reports success but the app still won't start (`Cannot find module 'next'`)
On resource-constrained shared hosting, the install can be killed partway through while
still reporting a generic success in the cPanel UI. Check `stderr.log` (in the app's log
directory, not the on-screen "Run NPM Install" output) for the real error. **Fix**: just
re-run "Run NPM Install" again — npm resumes/skips already-installed packages, and a
second pass typically completes what the first one didn't finish.

### 9. A broken/dangling `node_modules` symlink can't be deleted or fixed in place
If the nodevenv symlink's target directory has itself gone missing (e.g. after an
"Application root" edit), File Manager can't delete or rename the dangling link — every
action errors referencing the missing target, and it can't be worked around from the UI.
Likewise, editing "Application root" on an existing app in place does **not** relocate
its venv and can leave it permanently broken ("Unable to find app venv folder"). **Fix**:
don't try to repair it — destroy the app in Node.js Selector and recreate it fresh at a
(new or same) application root. Recreation always provisions a correctly-linked venv.

### 10. cPanel's "Run JS script" UI dumps an unreadable wall of minified source as the error
When a bundled/minified CLI (e.g. Prisma's) throws, cPanel's error view can dump the
entire minified file as "context" around the error, burying the actual message. **Fix**:
never rely on that view for diagnosis — write a tiny script that runs the failing
command via `execSync` in a try/catch and writes just `error.message` /
`stdout`/`stderr` (truncated) to a plain `.txt` file you can open directly in File
Manager:
```js
import { execSync } from "node:child_process";
import { writeFileSync } from "node:fs";
function run(label, cmd) {
  try {
    writeFileSync("diagnose-output.txt",
      `=== ${label}: OK ===\n${execSync(cmd, {encoding:"utf8"}).slice(0,1500)}`,
      { flag: "a" });
  } catch (e) {
    writeFileSync("diagnose-output.txt",
      `=== ${label}: FAILED ===\n${e.message}\n\nstderr:\n${(e.stderr||"").toString().slice(0,2500)}`,
      { flag: "a" });
  }
}
```
Delete it once the deploy is healthy — it's a diagnostic tool, not part of the app.

### 11. cPanel's own "SSH Access" page authorizes a key, but the shell still refuses it
On InterServer (and shared cPanel hosting generally), shell/SSH access is **disabled at
the account level by default** — a setting separate from and invisible in cPanel's own
"SSH Access" page. You can generate/import a key there, it'll show status "authorized",
and `ssh -v` will show the server correctly offering and then rejecting that exact key
(`Permission denied (publickey,gssapi-keyex,gssapi-with-mic,password)` for every method,
not just yours) — because the account has no real shell at all, independent of key
config. **Fix**: this needs the host's support team to flip an account-level flag; ask
for "shell/SSH access enabled for account X" by name. Don't spend time re-generating or
re-importing keys chasing this — verbose SSH output (`ssh -v`) confirming the key was
correctly *offered* and still rejected for every auth method is the tell that it's not a
key problem. Before writing shell off entirely, though, check cPanel's browser-based
**Terminal** feature (Advanced section) — it's a genuinely separate capability from SSH
and isn't gated by this same restriction (see Gotcha 16), though it has its own failure
mode when the account is already resource-maxed.

### 12. cPanel's port 2083 can sit behind a bot-protection layer that blocks API tokens too
A valid `Authorization: cpanel user:token` API request can still get a 200 response
containing an HTML "One moment, please..." interstitial (auto-reloading via
`setTimeout`/`window.location.reload()`) instead of JSON — check the `Server` response
header: cPanel's own daemon doesn't say `Server: openresty`. This is a separate
reverse-proxy/anti-bot layer in front of the port, and it blocks scripted clients
(different hostname, different User-Agent, added browser-like headers, waiting between
requests — none of it helps, because it requires actual JS execution). **Don't** burn
time trying to spoof past it. **Fix**: use FTP/FTPS instead (see Gotcha 13) — it's a
different port/protocol, unaffected by this layer, and cPanel-created FTP accounts can be
scoped to a single subdirectory (e.g. the app root only) for least-privilege access.

### 13. Automating file sync without shell: FTPS + a resumable, idempotent uploader
When both SSH (Gotcha 11) and the API (Gotcha 12) are blocked, a cPanel FTP account
scoped to the app directory (**FTP Accounts** → Directory: the app root) still works and
sidesteps both. Node's `basic-ftp` package handles explicit FTPS cleanly:
```js
await client.access({ host, port: 21, user, password,
  secure: true, secureOptions: { rejectUnauthorized: false } }); // cert is usually self-signed
```
Sustained transfers over this kind of link are **not reliable** — expect `ECONNRESET` on
both control and data sockets partway through a large `uploadFromDir`. Don't treat that
as a one-shot operation. Instead, write the sync as a loop that: lists the remote tree
recursively, diffs by (relative path, size) against the local tree, uploads only what's
missing/mismatched, and reconnects-and-repeats until zero files remain pending. This
makes every failure a no-op retry instead of lost progress — verified converging in 4
rounds (391 files, 2 mid-transfer connection resets) against a real flaky link.

**Building without CI**: if the project has zero native/compiled dependencies (check for
things like `bcrypt` (use `bcryptjs` instead), `sharp`, or any package with a native
addon — this project deliberately avoided all of them, see the parent skill), a
standalone Next.js build produced on any OS is byte-identical in behavior on the Linux
server, since Prisma 7's runtime engine is WASM (not a native binary, unlike Prisma <7)
and Next's own output is plain JS. This means you can build locally and skip GitHub
Actions/artifact-download entirely for the sync path — confirm you're shipping the exact
build you think you are by comparing `.next/BUILD_ID` locally vs. the uploaded copy
byte-for-byte, don't just trust "the upload finished."

### 14. Don't delete the live `.next` before the replacement is verified complete
Passenger doesn't reload a running app just because files on disk changed underneath it
— it keeps serving from the already-loaded process until something restarts it (a
manual `tmp/restart.txt` touch, a crash, or an idle-timeout recycle). That's a safety net
*if* you haven't already deleted the old build: deleting `.next` first (to sidestep
uncertain overwrite/merge semantics — see Gotcha 3's cousin problem with the File Manager
UI) and then having the reupload fail partway leaves a window where any restart trigger
(including one outside your control) serves from a broken, incomplete `.next`. **Safer
pattern**: sync the new build into a fresh directory alongside the live one, verify it's
complete (Gotcha 13's `BUILD_ID` check), and only *then* remove the old one and touch
`restart.txt` — never delete-then-reupload in place.

### 15. Every restart leaks a duplicate worker process — Passenger doesn't cleanly replace the old one
Neither the UI's "Restart" button nor touching `tmp/restart.txt` reliably terminates
the previous worker before spawning a new one on this stack — confirmed with a direct
before/after process count on a real account (24 → 33 processes from a single restart,
zero reduction). Enough restarts over a session (a dozen-plus deploy iterations in one
sitting is not unusual) silently accumulates duplicates until the account's process
limit is hit. From the outside this looks like the site suddenly serving every request
in 15–25s with the DB pool exhausted (`active=0 idle=0`, every query timing out) — and
it can cascade into cPanel's own Terminal failing too (`cagefs_enter: Unable to fork` —
no free process slot to fork a new shell into). **Fix**: don't treat restarts as free —
batch multiple changes into one restart rather than one per change (see the parent
project's own note on this), and deploy the automated cleanup safety net in Gotcha 16
rather than relying on manual vigilance.

### 16. Automated leak cleanup for Gotcha 15: a cron job that kills every duplicate except the newest — no SSH needed
cPanel's browser-based **Terminal** (Advanced section) is a genuinely different
capability from SSH (Gotcha 11) — it isn't gated by the same account-level shell
restriction, and gives a real (if manually-relayed, not scriptable-from-outside) shell
prompt. The one catch: it needs a free process slot to even fork itself, so it fails
with the exact same `cagefs_enter: Unable to fork` error once the account is already
maxed out by Gotcha 15's leak — chicken-and-egg, not fixable from inside Terminal at
that point; needs the host's support team to force-clear something server-side once it
gets that bad.

The real fix is **cPanel Cron Jobs** — a separate, always-available feature,
independent of SSH/Terminal entirely — running a cleanup script periodically:
```bash
# ps aux shows each Node.js Selector process as "lsnode:/home/<user>/<app-path>" —
# find all matches, keep only the newest (last in ps output = most recently
# started), kill every earlier one.
SEARCHSTRING="lsnode:/home/<user>/<app-path>"
MATCHES=$(ps -aux | grep "$SEARCHSTRING" | grep -v grep)
prev_line=""
while IFS= read -r line; do
  [ -z "$line" ] && continue
  if [ -n "$prev_line" ]; then
    pid=$(echo "$prev_line" | awk '{print $2}')
    kill -SIGKILL "$pid" 2>/dev/null
  fi
  prev_line=$line
done <<<"$MATCHES"
```
Wire it into a cPanel Cron Job (`*/15 * * * * /bin/bash /path/to/cleanup.sh`) — once set
up this needs zero shell access to keep running, and the script itself can be uploaded
over the same scoped FTP account from Gotcha 13.

**Two gotchas hit setting this up, both worth avoiding next time**:
- Don't derive paths via `dirname "$(realpath "$0")"` inside the script — `realpath`
  isn't guaranteed present in a minimal cron execution environment (CageFS jail), and
  its absence fails the whole script silently before it writes a single log line.
  Hardcode absolute paths instead.
- **The `<app-path>` in `lsnode:/home/<user>/<app-path>` is the app's internal Node.js
  Selector identifier, which can differ from its actual filesystem/FTP directory name**
  — don't assume they match just because the FTP/deploy path "looks obviously right."
  Verify with a live `ps aux | grep lsnode` first. A search string that's silently wrong
  matches zero processes forever, and a dry-run reporting "nothing to clean" then looks
  identical to a real, healthy "genuinely nothing to clean" — always log the match
  *count* on every run, not just kill actions, so a silent false-negative doesn't get
  mistaken for a working safety net.

## Deployment checklist (order matters)
1. `output: "standalone"` in `next.config.ts`; lazy DB client (Gotcha 2).
2. CI workflow: `npm ci` → `prisma generate` → `next typegen` (Gotcha 1) → type-check →
   lint → `next build` → assemble `deploy/` bundle (Gotcha 3) → upload with
   `include-hidden-files: true` (Gotcha 4), **excluding** `node_modules`.
3. Push a schema change with `db push` from local/CI, never the server (Gotcha 6); run
   the InnoDB sweep after every push (Gotcha 5).
4. In cPanel Node.js Selector: create the app pointing at `.next/standalone/server.js`
   as the startup file; set all env vars in the UI (`DATABASE_URL`/DB_* vars,
   `AUTH_SECRET`, `AUTH_URL` as the external `https://` origin — Apache/LiteSpeed
   terminates TLS in front of the plain-HTTP Node process, so OAuth callback URLs must
   use the external URL, not `localhost`).
5. Download the latest **successful** Actions run's artifact (check the run's own log
   output, not just "green check"), extract everything except `node_modules` into the
   app root, overwriting `public/` and `.next/` fully rather than merging.
6. "Run NPM Install" from the UI (Gotcha 7); if the app still won't start, check
   `stderr.log` before assuming the install actually finished (Gotcha 8).
7. Restart the app from the UI (or `touch tmp/restart.txt` if you have file access but
   not the UI — same effect). Verify live: fetch a known `public/` file (confirms
   Gotcha 3/4 didn't regress), fetch `/api/auth/providers` (confirms OAuth env vars and
   callback URLs), and confirm a dashboard-style protected route redirects rather than
   500s for a logged-out request.

**If SSH and the API are both unavailable** (Gotchas 11–12), steps 5–7 can be done
end-to-end over a scoped FTP account instead of File Manager clicks — see Gotcha 13 for
the resumable-uploader pattern and Gotcha 14 for the safe delete/swap order.

## Post-deploy verification pattern
Don't trust "the page loads" alone — a stale partial deploy can look fine at a glance
(landing page, CSS, JS chunks all present) while one specific piece (e.g. `public/`) is
stale, because Next's own build-ID-stamped chunks all update together but manually-copied
static assets don't. Cross-check:
- `curl -sI https://<domain>/<known-public-file>` — status and `Content-Type` should
  match the real file, not your app's HTML 404 page.
- Hit `/api/auth/providers` and confirm each provider's `callbackUrl` is the production
  domain, not `localhost`.
- For the DB layer specifically, a short read-only script using the app's own Prisma
  client (run **locally** against the production DB via env vars, not on the server) is
  more trustworthy than clicking through the UI: table engines are all InnoDB, no
  suspiciously-old `RESERVED`/pending rows from a crashed upload, row counts sane.

## Related
- [web_performance_seo](../web_performance_seo/) for the CI/cache-busting side of static
  asset delivery once the app itself is live.
