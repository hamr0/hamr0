# Observability & Ops Playbook — pulselog + flightlog on a VPS

A generic, copy-into-any-project setup for **error capture, health checks, a
weekly growth digest, local backups, and off-box watchdogs** — using
[`flightlog`](https://github.com/hamr0/flightlog) and
[`pulselog`](https://github.com/hamr0/pulselog). Zero paid SaaS, zero
production dependencies, no Docker. Everything is a systemd timer + one JSONL
line per event.

> **How to use this doc.** Replace every `<PLACEHOLDER>` with your project's
> value. The running example is an app called `<APP>` on host `<VPS_IP>`,
> served at `<DOMAIN>`, running as the unprivileged user `<APP_USER>`, with
> code at `<APP_DIR>` (e.g. `/opt/<APP>`) and runtime state at `<DATA_DIR>`
> (e.g. `/var/lib/<APP>`). Alerts go to `<OPERATOR_EMAIL>`.

| Placeholder | Meaning | Example |
|---|---|---|
| `<APP>` | app / project name | `myapp` |
| `<APP_USER>` | unprivileged service user | `myapp` |
| `<APP_DIR>` | deployed code dir | `/opt/myapp` |
| `<DATA_DIR>` | runtime state (DB, `*.jsonl`) | `/var/lib/myapp` |
| `<DOMAIN>` | public domain | `myapp.com` |
| `<VPS_IP>` | VPS public IP | `203.0.113.10` |
| `<MAIL_HOST>` | HELO / rDNS name for mail | `mail.myapp.com` |
| `<DKIM_SELECTOR>` | OpenDKIM selector | `myapp2026` |
| `<OPERATOR_EMAIL>` | where alerts land | `you@gmail.com` |
| `<USER>` | human login user on the **backup host** (§8) | `alice` |

**Order of work:** §2 (mail — first, always) → §3 (flightlog in-app) → §4 (your
stats command) → §5–7 (pulselog + timers on the VPS) → §8 (backup host,
optional but recommended). §9 verifies each piece; §10 is for when it breaks.

> **Log-path note:** this doc writes `/var/log/maillog` (Fedora/RHEL). On
> Debian/Ubuntu use `/var/log/mail.log`; on journald-only systems use
> `journalctl -u postfix` instead.

---

## 0. Architecture — four independent layers

Each layer catches what the others miss. Layers 1–3 live on the VPS; layer 4
lives on a **separate always-on box** (a home server, a second cheap VPS) so it
survives the VPS being fully down.

```
                          ┌──────────────────────── VPS (<VPS_IP>) ────────────────────────┐
  app throws  ──▶ (1) flightlog  ──▶  <DATA_DIR>/errors.jsonl                                │
                          │                                                                  │
  every 15m  ──▶ (2) pulselog health   (systemd timer)  ── silent on green, emails on fail   │
  weekly     ──▶ (3) pulselog digest   (systemd timer)  ── stats + error rollup email        │
                          └──────────────────────────────────────────────────────────────────┘
                                                        ▲ reads maillog / DB over SSH
  ┌──────────── backup host (home server / 2nd VPS) ───┼───────────────────────────────────┐
  daily      ──▶ pulselog --backup   (systemd timer)   │  curated DB + certs, rotated        │
  every 5m   ──▶ (4) pulselog watch  (systemd timer) ──┘  uptime + cert + backup-fresh       │
                                                          + mail-delivery (optional)          │
  └──────────────────────────────────────────────────────────────────────────────────────────┘
```

| # | Layer | Runs where | Cadence | Catches |
|---|-------|-----------|---------|---------|
| 1 | **flightlog** | in-app | on error | uncaught exceptions, rejections, `capture()`d errors → `errors.jsonl` |
| 2 | **pulselog health** | VPS | 15 min | unit down, API 5xx, disk full, cert expiring, mail queue backed up |
| 3 | **pulselog digest** | VPS | weekly | growth metrics (WoW) + "≥N of the same error this week" |
| 4 | **pulselog watch + backup** | off-box | 5 min / daily | site down, cert expired, stale/failed backup, **mail delivery broken** |

Everything writes the **same JSONL dialect** (`{"ts","kind","app",…}`), so one
`tail`/`jq` spans errors, health, stats, and backups.

---

## 1. Prerequisites

- **Node.js** — flightlog needs ≥18 (ESM `import`) or ≥22.12 for CommonJS
  `require`. pulselog's `command`/`http`/backup modes need only Node ≥18; its
  bundled SQLite metric helpers need ≥22.5 (`--experimental-sqlite`). Match your
  app's runtime.
- **Install** (as runtime deps, so `npm ci --omit=dev` on the server still
  gets them):
  ```bash
  npm i flightlog pulselog          # in-app + on-VPS
  sudo npm i -g pulselog            # on the backup host (invoked as a CLI)
  ```
- **A mailer.** pulselog and your app send via `sendmail`. Have a working MTA
  (Postfix + OpenDKIM) **or** a `sendmail` shim (`msmtp` + `msmtp-mta`). See §2
  — this is the step everyone skips and then silently loses every alert.
- **A dedicated service user** (`<APP_USER>`) the app + timers run as. Keep it
  unprivileged: it will NOT be able to read `/var/log/maillog` (that matters in
  §8 and §10).

---

## 2. ⚠️ Mail deliverability FIRST — SPF, DKIM, PTR

**Do this before anything else.** pulselog/flightlog can be flawless and you'll
still get *nothing* if the VPS can't hand mail to the recipient. Gmail (and most
providers) reject unauthenticated mail outright. There are three independent
gates; you need all three green.

> **Golden rule: never put your mail host behind a proxy/CDN.** If `<MAIL_HOST>`
> (or the domain whose A record your SPF trusts) is proxied (e.g. Cloudflare
> "orange cloud"), it resolves to the CDN's IPs, not your VPS — which breaks SPF
> *and* reverse-DNS at once. A mail host must be **DNS-only** and point straight
> at `<VPS_IP>`. CDNs don't proxy SMTP/25 anyway, so proxying it buys nothing.

### 2a. PTR / reverse DNS (fixes Gmail `5.7.25`)
Set at your **VPS provider** (not your DNS host): `<VPS_IP>` → `<MAIL_HOST>`.
Then ensure `<MAIL_HOST>` forward-resolves *back* to `<VPS_IP>` (this is
forward-confirmed reverse DNS, FCrDNS). Verify:
```bash
dig +short -x <VPS_IP>            # → <MAIL_HOST>.
dig +short A <MAIL_HOST>          # → <VPS_IP>   (NOT a CDN IP)
```

### 2b. SPF (part of fixing `5.7.26`)
Publish a TXT record on `<DOMAIN>`. **Prefer a literal IP** so it doesn't depend
on any hostname's proxy state:
```
<DOMAIN>.   TXT   "v=spf1 ip4:<VPS_IP> -all"
```
(If you relay through Gmail/another provider, add their `include:` and use
`~all`.) Verify: `dig +short TXT <DOMAIN>`.

### 2c. DKIM (part of fixing `5.7.26`)
Set up OpenDKIM (Postfix milter) with selector `<DKIM_SELECTOR>` for `<DOMAIN>`,
then **publish the public key** — the step most often forgotten (the MTA signs
happily, the recipient just can't verify):
```bash
# on the VPS, after opendkim-genkey created <DKIM_SELECTOR>.txt:
cat /etc/opendkim/keys/<DOMAIN>/<DKIM_SELECTOR>.txt   # copy the p=… value
```
Publish as TXT at `<DKIM_SELECTOR>._domainkey.<DOMAIN>` = `v=DKIM1; k=rsa; p=…`.
Verify it's live **and** that signing happens:
```bash
dig +short TXT <DKIM_SELECTOR>._domainkey.<DOMAIN>          # → v=DKIM1;...
grep 'DKIM-Signature field added' /var/log/maillog | tail  # → s=<DKIM_SELECTOR>
```

### 2d. DMARC (recommended)
```
_dmarc.<DOMAIN>.   TXT   "v=DMARC1; p=none; rua=mailto:<OPERATOR_EMAIL>"
```

### 2e. Prove it end-to-end
```bash
printf 'Subject: mailtest\nFrom: noreply@<DOMAIN>\nTo: <OPERATOR_EMAIL>\n\nhi\n' \
  | /usr/sbin/sendmail -f noreply@<DOMAIN> <OPERATOR_EMAIL>
sleep 8
grep '<OPERATOR_EMAIL>' /var/log/maillog | tail -1     # want: status=sent (250 2.0.0 OK)
```
`status=bounced` with `5.7.25` → fix PTR (2a). `5.7.26` → fix SPF/DKIM (2b/2c).

---

## 3. flightlog — in-app error capture

Wire it once, as early in startup as possible (right after you load env/secrets).
It registers global handlers and hands you `capture()` / `captureSync()`.

**Long-lived process (your server):**
```js
import { install } from 'flightlog';
import path from 'node:path';

const { capture } = install({
  file: path.join(process.env.DATA_DIR || './data', 'errors.jsonl'),
  context: { app: '<APP>', proc: 'server', release: process.env.RELEASE },
});

// uncaught exceptions log synchronously then exit(1) (systemd restarts clean);
// unhandled rejections log and stay alive. For the errors you catch yourself:
try { await risky(); }
catch (e) { capture(e, { where: 'checkout' }); }   // fire-and-forget, never throws
```

**Short-lived process (cron job, mail pipe, CLI) — use `captureSync`** so the
line flushes before you exit:
```js
const { captureSync } = install({
  file: path.join(process.env.DATA_DIR || './data', 'errors.jsonl'),
  context: { app: '<APP>', proc: 'inbound' },
});
try { main(); }
catch (e) { captureSync(e, { where: 'inbound' }); process.exit(1); }
// captureSync returns { ok, errno? } — check it if the exit code should reflect
// whether the error actually landed on disk.
```

**The `where` field is your grouping key** for the weekly "≥N of the same error"
rollup (§6). Give each call site a stable, low-cardinality `where` (e.g.
`checkout`, `sweep-cleanup`) — not the error message.

> **Never spread untrusted objects into `context`/`capture()` extras.** Keys like
> `ts`/`kind`/`name`/`message`/`stack` would shadow core fields. Pass an
> allow-listed set of fields, never a raw request/payload object. Values are
> JSON-escaped (safe), but your own keys can clobber.

`errors.jsonl` is written `0600`. Point every process at the **same** file (or a
per-proc file) under `<DATA_DIR>`.

---

## 4. The stats command — generic growth metrics

The digest's job is "is it growing?" You provide the numbers via one command
that prints a **flat JSON object of named integers**. That's the only app-specific
code in the whole setup.

`bin/stats.js` (any language works; must print JSON to stdout and exit 0):
```js
// node --experimental-sqlite bin/stats.js --metrics-json
// prints e.g. {"users":128,"active_7d":54,"orders":2310}
import { DatabaseSync } from 'node:sqlite';
const db = new DatabaseSync(process.env.DB_PATH, { readOnly: true });
const n = (sql) => db.prepare(sql).get().n;
const metrics = {
  users:     n(`SELECT COUNT(*) AS n FROM users`),
  active_7d: n(`SELECT COUNT(*) AS n FROM users WHERE last_seen > date('now','-7 days')`),
};
db.close();
process.stdout.write(JSON.stringify(metrics) + '\n');
```

Rules that keep it privacy-clean and robust:
- **Counts only** — `SELECT COUNT(*)`. Nothing recorded that wasn't already a row.
- **Read-only** — open the DB read-only; never mutate from a stats run.
- **Exit non-zero if the source is missing** (e.g. DB not found) so pulselog
  records `null` for that metric instead of a silent zero.
- Not a database? Any command works: `wc -l`, a `curl … | jq`, etc. — as long as
  the final stdout is one flat JSON object of integers.

---

## 5. pulselog on-VPS: health checks

`pulselog.config.json` (health section) — stays **silent on green**, emails
`<OPERATOR_EMAIL>` on any failure. Every check takes an optional `timeoutMs`;
the top-level `retry` re-probes a blip within the same run before alerting.

```json
{
  "output": { "file": "<DATA_DIR>/health.jsonl", "maxBytes": 5000000, "heartbeat": false },
  "alert":  { "email": "<OPERATOR_EMAIL>", "from": "alerts@<DOMAIN>", "app": "<APP>" },
  "retry":  { "retries": 2, "retryDelayMs": 2000 },
  "checks": [
    { "type": "service", "name": "<APP>",   "unit": "<APP>.service" },
    { "type": "service", "name": "postfix", "unit": "postfix.service" },
    { "type": "service", "name": "nginx",   "unit": "nginx.service" },
    { "type": "http",    "name": "api",  "url": "http://127.0.0.1:3000/health", "expectStatus": 200, "timeoutMs": 5000 },
    { "type": "disk",    "name": "disk-root", "path": "/",          "maxPercent": 85 },
    { "type": "disk",    "name": "disk-data", "path": "<DATA_DIR>", "maxPercent": 85 },
    { "type": "ssl",     "name": "cert", "host": "<DOMAIN>", "port": 443, "warnDays": 14 },
    { "type": "command", "name": "mailq-depth", "command": "sh", "args": ["-c", "test $(mailq | grep -c '^[A-F0-9]') -lt 50"] }
  ]
}
```
Available check types: `service`, `http`, `tcp`, `ssl`, `disk`, `file-age`,
`command`. `command` is the escape hatch for anything else — it just needs exit 0
= healthy.

---

## 6. pulselog on-VPS: weekly digest (stats + "≥N of the same error")

Same config file, `digest` section. One weekly run: collect metrics, append **one
`kind:"stats"` line** to a history file, and email a week-over-week table plus a
flightlog error rollup.

```json
{
  "digest": {
    "app": "<APP>",
    "history": "<DATA_DIR>/stats.jsonl",
    "email": "<OPERATOR_EMAIL>",
    "from": "noreply@<DOMAIN>",
    "weeks": 4,
    "skipIfFlat": false,

    "metricsCommand": {
      "command": "/usr/bin/node",
      "args": ["--experimental-sqlite", "<APP_DIR>/bin/stats.js", "--metrics-json"],
      "timeoutMs": 10000
    },
    "metrics": [
      { "name": "users" },
      { "name": "active_7d" }
    ],

    "flightlog": { "file": "<DATA_DIR>/errors.jsonl", "groupBy": "where", "flagAtLeast": 20 }
  }
}
```

**"Email me if there are >20 of the same error"** = the `flightlog` block:
- `flagAtLeast: 20` → any group whose **7-day count reaches 20** is flagged in the
  digest (default is 20; set to whatever threshold you want).
- `groupBy: "where"` → groups by your flightlog `where` field (§3). Use `"name"`
  (the default) to group by error class/name instead.
- **Counts and names only** ever reach the email — never messages or stacks
  (those can carry PII). That privacy invariant is mutation-tested in pulselog.

**Metrics:** each `metrics[]` entry with no `command` of its own is filled **by
name** from the single `metricsCommand` JSON (one process spawn for all numbers).
Or give a metric its own `command` that prints one integer.

**Cadence vs. table are decoupled.** The digest run *is* the weekly snapshot;
`weeks` only controls how many rows the table shows. `skipIfFlat: true` suppresses
the email in weeks where every Δ=0 and nothing is flagged (leave it `false` if you
want a weekly proof-of-life). Preview any time without sending:
```bash
pulselog --digest --dry-run --config <APP_DIR>/pulselog.config.json
```

---

## 7. systemd units (VPS)

Health — every 15 min:
```ini
# /etc/systemd/system/<APP>-health.service
[Unit]
Description=<APP> health check (pulselog)
After=network-online.target
[Service]
Type=oneshot
User=<APP_USER>
Group=<APP_USER>
ExecStart=/usr/bin/node <APP_DIR>/node_modules/pulselog/bin/pulselog.js --config <APP_DIR>/pulselog.config.json
Nice=10
```
```ini
# /etc/systemd/system/<APP>-health.timer
[Unit]
Description=Run <APP> health check every 15 minutes
[Timer]
OnBootSec=5min
OnUnitActiveSec=15min
AccuracySec=1min
Persistent=true
[Install]
WantedBy=timers.target
```

Digest — weekly (Sunday 09:00 UTC):
```ini
# /etc/systemd/system/<APP>-stats-digest.service
[Unit]
Description=<APP> weekly stats digest (pulselog --digest)
After=network-online.target
[Service]
Type=oneshot
User=<APP_USER>
Group=<APP_USER>
WorkingDirectory=<APP_DIR>
EnvironmentFile=/etc/<APP>/env       # so stats.js sees DB_PATH/DATA_DIR
ExecStart=/usr/bin/node <APP_DIR>/node_modules/pulselog/bin/pulselog.js --digest --config <APP_DIR>/pulselog.config.json
NoNewPrivileges=true
ProtectSystem=true
ProtectHome=true
PrivateTmp=true
Nice=10
```
```ini
# /etc/systemd/system/<APP>-stats-digest.timer
[Unit]
Description=Send the <APP> weekly stats digest every Sunday
[Timer]
OnCalendar=Sun *-*-* 09:00:00 UTC
AccuracySec=15min
Persistent=true
[Install]
WantedBy=timers.target
```

Enable:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now <APP>-health.timer <APP>-stats-digest.timer
systemctl list-timers '<APP>-*'
```

> **Deploy gotcha:** your deploy script probably only does `git pull` + `npm ci`
> + `systemctl restart <APP>`. It does **not** install/reload these unit files or
> the config — do that by hand when you add or change them (`daemon-reload`), and
> make sure `pulselog` is a runtime dep so `npm ci --omit=dev` installs
> `node_modules/pulselog/bin/pulselog.js`.

---

## 8. Backup host (off-box): local backup + watch — optional but recommended

Runs on a **separate always-on machine**. One config drives both: `--backup`
runs the backup section; no flag runs the `checks`. This is the only layer that
survives the VPS being completely down.

`/etc/<APP>/pulselog.config.json` on the backup host:
```json
{
  "output": { "file": "/home/<USER>/<APP>-backups/health.jsonl", "maxBytes": 5000000 },
  "alert":  { "email": "<OPERATOR_EMAIL>", "from": "<OPERATOR_EMAIL>", "app": "<APP>-offbox" },
  "retry":  { "retries": 2, "retryDelayMs": 2000 },
  "checks": [
    { "type": "http",     "name": "site",        "url": "https://<DOMAIN>/health", "expectStatus": 200, "timeoutMs": 10000 },
    { "type": "ssl",      "name": "cert",        "host": "<DOMAIN>", "port": 443, "warnDays": 14 },
    { "type": "file-age", "name": "backup-fresh","path": "/home/<USER>/<APP>-backups", "maxAgeHours": 30, "pattern": ".tar.gz" },
    { "type": "command",  "name": "mail-delivery","command": "/usr/local/bin/<APP>-mail-check.sh", "timeoutMs": 20000 }
  ],
  "backup": {
    "app": "<APP>", "dir": "/home/<USER>/<APP>-backups", "name": "<APP>-backup",
    "command": "/usr/local/bin/<APP>-pull.sh", "timeoutMs": 600000,
    "keepLast": 7, "keepDays": 30, "minBytes": 1024,
    "history": "/home/<USER>/<APP>-backups/backup.jsonl",
    "email": "<OPERATOR_EMAIL>", "from": "<OPERATOR_EMAIL>"
  }
}
```
The `mail-delivery` check is the optional §8a watchdog — drop that line if you
skip it.

**Pull script** — the "fetch the sources" step, and (like `bin/stats.js` in §4)
the app-specific part you replace wholesale: the example below assumes SQLite in
WAL mode and root SSH — swap in whatever dumps *your* data into
`$PULSELOG_STAGE`. pulselog owns the rest: staging → tar → size-floor → atomic
publish → rotation → JSONL line → failure email:
```bash
#!/bin/bash
# /usr/local/bin/<APP>-pull.sh   (config via /etc/default/<APP>-backup)
set -euo pipefail
VPS_HOST="${VPS_HOST:-<VPS_IP>}"; VPS_USER="${VPS_USER:-root}"; SSH_KEY="${SSH_KEY:-$HOME/.ssh/<APP>_vps}"
: "${PULSELOG_STAGE:?must be invoked by pulselog --backup}"
SSH="ssh -i $SSH_KEY -o BatchMode=yes -o StrictHostKeyChecking=accept-new"
# DB + WAL sidecars = crash-consistent snapshot (SQLite WAL mode)
$SSH "$VPS_USER@$VPS_HOST" 'tar -cf - -C <DATA_DIR> <APP>.db <APP>.db-shm <APP>.db-wal' | tar -xf - -C "$PULSELOG_STAGE/"
$SSH "$VPS_USER@$VPS_HOST" 'tar -czf - -C /etc letsencrypt' > "$PULSELOG_STAGE/letsencrypt.tar.gz"
test -s "$PULSELOG_STAGE/<APP>.db"    # fail loud if the DB pull came back empty
```

### 8a. Mail-delivery watchdog (why it MUST be off-box)

A check that verifies "the VPS can still get mail to the recipient." It **cannot**
live on the VPS, for two reasons:
1. **Permissions** — the on-VPS health check runs as unprivileged `<APP_USER>`,
   which can't read root-owned `/var/log/maillog`.
2. **Circular alerting** — the failure it detects is "mail is broken," so its own
   alert would ride the same broken path and bounce. It can't page you for the
   one thing it watches.

The backup host has an **independent** mail path and already has SSH into the VPS.
Key on the *latest* delivery outcome so a stale bounce can't stick it red:
```bash
#!/bin/bash
# /usr/local/bin/<APP>-mail-check.sh   (env from /etc/default/<APP>-backup)
set -euo pipefail
VPS_HOST="${VPS_HOST:-<VPS_IP>}"; VPS_USER="${VPS_USER:-root}"; SSH_KEY="${SSH_KEY:-$HOME/.ssh/<APP>_vps}"
RECIPIENT="${MAIL_CHECK_RECIPIENT:-<OPERATOR_EMAIL>}"
last=$(ssh -i "$SSH_KEY" -o BatchMode=yes -o ConnectTimeout=10 "$VPS_USER@$VPS_HOST" \
  "grep -h 'to=<$RECIPIENT>' /var/log/maillog 2>/dev/null | grep -oE 'status=(sent|bounced)' | tail -1" \
  2>/dev/null || true)
[ "$last" != "status=bounced" ]   # green if sent or no attempts; red only if the latest bounced
```

**Units** (both `Type=oneshot`, `User=<the human user>`):
```ini
# /etc/default/<APP>-backup  — single source of VPS coordinates
VPS_HOST=<VPS_IP>
VPS_USER=root
SSH_KEY=/home/<USER>/.ssh/<APP>_vps
```
```ini
# <APP>-watch.service  (every 5 min via <APP>-watch.timer)
[Service]
Type=oneshot
User=<USER>
EnvironmentFile=-/etc/default/<APP>-backup
ExecStart=/usr/local/bin/pulselog --config /etc/<APP>/pulselog.config.json
```
```ini
# <APP>-backup.service  (daily via <APP>-backup.timer)
[Service]
Type=oneshot
User=<USER>
EnvironmentFile=/etc/default/<APP>-backup
ExecStart=/usr/local/bin/pulselog --backup --config /etc/<APP>/pulselog.config.json
```

> The backup host's own alerts need a working mailer too — the simplest is
> `msmtp` → Gmail, which makes Gmail sign the mail with clean reputation (set the
> config `from` to the authenticated Gmail address).

---

## 9. Verify everything

```bash
# flightlog: force an error, confirm a line lands
node -e "import('flightlog').then(f=>{const {captureSync}=f.install({file:'/tmp/e.jsonl'});captureSync(new Error('x'),{where:'test'})})"; cat /tmp/e.jsonl

# health: exit 0 + silent when green
node <APP_DIR>/node_modules/pulselog/bin/pulselog.js --config <APP_DIR>/pulselog.config.json; echo "exit=$?"

# digest: render without sending
node <APP_DIR>/node_modules/pulselog/bin/pulselog.js --digest --dry-run --config <APP_DIR>/pulselog.config.json

# mail: real send (§2e) → want status=sent
# timers: are they actually scheduled?
systemctl list-timers '<APP>-*'

# backup host:
sudo /usr/local/bin/pulselog --backup --config /etc/<APP>/pulselog.config.json
ls -la ~/<APP>-backups/*.tar.gz
```

---

## 10. Troubleshooting — the failure modes (learned the hard way)

| Symptom | Likely cause | Check / fix |
|---|---|---|
| Digest/alerts never arrive, but timers ran | **Mail bounces at recipient** | `grep '<OPERATOR_EMAIL>' /var/log/maillog \| grep bounced` → fix §2 |
| Bounce `5.7.25` | PTR missing or forward≠reverse (often a **proxied mail host**) | §2a; `dig A <MAIL_HOST>` must be `<VPS_IP>`, not a CDN IP |
| Bounce `5.7.26` | SPF and DKIM both fail | §2b/2c; `dig TXT <DKIM_SELECTOR>._domainkey.<DOMAIN>` must be non-empty |
| Timer never fired | unit not installed / not enabled | `systemctl list-timers '<APP>-*'`; install units + `daemon-reload` + `enable --now` |
| `pulselog.js: not found` on the VPS | `npm ci --omit=dev` skipped it | pulselog must be in **`dependencies`**, not `devDependencies` |
| pulselog refuses the config | 0.4.x **config-ownership gate** | config must be owned by the running user **or root**, and not group/world-writable |
| Digest metric shows `null` | metric command failed / DB missing | run the `metricsCommand` by hand; it must print flat JSON and exit 0 |
| Can't find last week's run in `journalctl` | journal rotated | the durable record is the JSONL (`stats.jsonl` / `health.jsonl`), not the journal |
| On-VPS mail-bounce check false-alarms | runs as `<APP_USER>` (can't read maillog) + circular | move it **off-box** (§8a) — don't grant the service user log access |

---

## 11. Security & privacy notes

- **pulselog config-ownership gate (≥0.4.0):** the CLI refuses a config that is
  group/world-writable or owned by a third party (it drives command execution).
  Root-owned-and-readable by a non-root service user is fine (0.4.1). Keep configs
  `0644 root` or owned by the running user.
- **Alert/digest mail is plain `sendmail`, unsigned by pulselog itself.**
  Deliverability rides entirely on your MTA (§2). Keep a **secondary signal** so a
  spam-foldered alert isn't your only notice: the JSONL lines, and the off-box
  `file-age` dead-man's-switch (§8).
- **Error messages/stacks never leave the box** via the digest — only counts +
  group names. Don't defeat this by putting PII in the flightlog `where`/`name`.
- **flightlog `context` is not clobber-protected** — pass allow-listed fields,
  never a raw request/payload object (§3).
- Both tools write JSONL `0600`. Back up `errors.jsonl`/`stats.jsonl` if you want
  history to survive a rebuild; neither is in the DB backup by default.

---

## Appendix — minimal file inventory

```
<APP_DIR>/
  bin/stats.js                  # your only custom code: prints {"metric":N,...}
  pulselog.config.json          # health + digest sections
  node_modules/{flightlog,pulselog}
/etc/<APP>/env                  # DB_PATH/DATA_DIR for the digest's stats.js
/etc/systemd/system/
  <APP>-health.{service,timer}
  <APP>-stats-digest.{service,timer}
<DATA_DIR>/
  errors.jsonl  health.jsonl  stats.jsonl

# backup host
/etc/<APP>/pulselog.config.json
/etc/default/<APP>-backup
/usr/local/bin/{<APP>-pull.sh,<APP>-mail-check.sh}
/etc/systemd/system/<APP>-{watch,backup}.{service,timer}
```

**One-line mental model:** flightlog records errors, pulselog watches health +
trends + backups, mail carries the alerts — and the mail path (§2) is the part
that will bite you, so verify it first and watch it from off-box.
