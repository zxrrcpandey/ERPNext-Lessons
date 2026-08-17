# Frappe / ERPNext v15 — install and production gotchas

Everything Trustbit knows about running **Frappe/ERPNext v15 in production**, consolidated
so v15 clients stay supportable while new builds move to v16.

**Sources** (all real deployments, no theory):

| Source | What it covers |
|---|---|
| `RD School Betul/DEPLOYMENT_LESSONS.md` | dev `rdschool.localhost`, old prod `168.144.155.237`, new prod `rdps.trustbit.cloud` — 8 incidents |
| `Trustbit Kantishiva/PROJECT_STATE.md` | `mandi.trustbit.in` v15 prod (Hostinger, Ubuntu 22.04) and its restore into a local bench |
| `Betul Bio Fuel Pvt Ltd/` | `erpbbpl.com` prod + `betulbiofuel.mkasystem.com` demo — v15 prod operating rules |
| `~/frappe-bench-v15` (live checkout) | frappe 15.100.0 / erpnext 15.97.0 source, read directly for this doc |
| bench 5.29.0 at `/opt/homebrew/lib/python3.11/site-packages/bench` | the CLI serving the v15 benches — read directly for this doc |

**Confidence tags used below**

- `[verified]` — run or read in source during this write-up, or recorded as an executed
  command in a primary deployment doc.
- `[source: X]` — quoted from a primary source, not independently re-run.
- `[unverified]` — plausible and useful, but nobody has confirmed it. Treat as a hypothesis.

---

## 0. CORRECTION — supersedes `RD School/DEPLOYMENT_LESSONS.md` Lesson 8

**Applies to: v15 and v16.**

RD School Lesson 8 says SSL on `rdps.trustbit.cloud` was issued with `certbot --nginx`.
**Do not use plain `certbot --nginx` on a bench-managed box.** Use:

```bash
certbot certonly --nginx -d <site-domain>
```

**Why.** `bench setup production` creates `/etc/nginx/conf.d/<bench_name>.conf` as an
**`os.symlink` to `<bench>/config/nginx.conf`** — verified in bench 5.29.0,
`bench/config/production_setup.py:76-79`:

```python
if not os.path.islink(nginx_conf):
    os.symlink(
        os.path.abspath(os.path.join(bench_path, "config", "nginx.conf")), nginx_conf
    )
```

certbot's **installer** mode (`--nginx` without `certonly`) rewrites the config file it
finds. Writing through that symlink edits the *bench-generated* `config/nginx.conf`.
The next `bench setup nginx` / `bench setup production` regenerates that file from
bench's Jinja template and the SSL directives vanish — a working HTTPS site drops to
plain HTTP with no warning.

`certonly --nginx` uses the plugin **only** to solve HTTP-01: it inserts a temporary
challenge block, reloads nginx, then reverts — no sustained downtime (unlike
`--standalone`) and no persistent server-block edit. It is not a literal no-op on the file,
so **if certbot is interrupted mid-issuance, run `bench setup nginx --yes` before trusting
the generated config.** [verified in bench source; failure mode
`[source: SERVER_KANTISHIVA.md §7]`]

**The correct wiring after `certonly`** — let bench own the config, so regeneration is safe:

```bash
bench config dns_multitenant on          # MUST come before bench setup nginx
bench --site <site> set-config ssl_certificate     /etc/letsencrypt/live/<domain>/fullchain.pem
bench --site <site> set-config ssl_certificate_key /etc/letsencrypt/live/<domain>/privkey.pem
bench --site <site> set-config host_name           https://<domain>
bench setup nginx --yes                  # needs /etc/nginx/conf.d/00-log-format.conf — see §2.5
sudo nginx -t && sudo systemctl reload nginx
```

**Use `set-config`, not `bench set-ssl-certificate` / `bench set-ssl-key`.** Those two
regenerate nginx immediately — `set_site_config_nginx_property()` ends in
`make_nginx_conf(bench_path=bench_path)` with `yes` defaulting to `False`
(`bench/config/site_config.py:51-59`) — and they have **no `--yes` flag** to pass
(`bench/commands/utils.py:53-68`: both take only positional args). On a non-TTY root SSH
session they hang forever on `nginx.conf already exists and this will overwrite it. Do you
want to continue?`, mid-reconfiguration. `bench --site X set-config` only edits
`site_config.json`; the one explicit `bench setup nginx --yes` after it does the
regeneration under your control. [verified in bench 5.29.0; this is also what
`SERVER_KANTISHIVA.md` ended up doing — "set `ssl_certificate` / `ssl_certificate_key`
directly in site_config"]

**Set BOTH keys — the certificate alone does nothing.** bench's nginx template gates the
whole SSL block on `{% if ssl_certificate and ssl_certificate_key %}`
(`bench/config/templates/nginx.conf:15,39-41`), so setting only the cert silently produces a
plain-HTTP vhost. The alternative
`bench setup add-domain <domain> --site <site> [--ssl-certificate PATH]
[--ssl-certificate-key PATH]` (`bench/commands/setup.py:326-331`) exists, but see the
duplicate-`server_name` trap below before reaching for it. [verified]

Two traps in that block, both hit on the Kantishiva build `[source: SERVER_KANTISHIVA.md]`:

- **`dns_multitenant` must be ON before nginx generation.** With it off, bench's template
  never reaches the per-site SSL branch, so `ssl_certificate` / `ssl_certificate_key` in
  `site_config.json` are silently ignored and SSL never applies. Confirmed present in
  bench 5.29.0 (`bench/config/nginx.py:119-150`) — same behaviour on v15 benches. [verified]
- **When the site name *is* the domain, skip `add-domain`.** It registers the domain as an
  *alternate*, producing **two `:80` server blocks with the same `server_name`**; nginx
  serves the first and the HTTPS redirect is dead. Set `ssl_certificate` /
  `ssl_certificate_key` directly in `site_config.json` and leave `domains` empty.

Also: **skip `bench setup lets-encrypt`** — unreliable by construction, and its behaviour on
current bench could not be verified. Use apt `certbot` + `python3-certbot-nginx`; never mix
with the snap. `[source: SERVER_KANTISHIVA.md §7]`

If SSL ever disappears after a bench command, re-run `bench setup nginx --yes` — the certs
are still on disk under `/etc/letsencrypt/live/`; only the vhost was lost.

---

## 1. The v15 stack actually running in production

| Component | Version | Where |
|---|---|---|
| frappe | **15.100.0** (branch `version-15`) | mandi prod + `~/frappe-bench-v15` [verified: `apps/frappe/frappe/__init__.py:54`] |
| erpnext | **15.97.0** (branch `version-15`) | same [verified: `apps/erpnext/erpnext/__init__.py:7`] |
| india_compliance | **15.25.4** (branch `version-15`) | same `[source: PROJECT_STATE.md]` |
| trustbit_mandi | 0.0.1, `main` @ `55cd6a7` | client app `[source: PROJECT_STATE.md]` |
| MariaDB | **10.6** | mandi prod (Ubuntu 22.04) `[source: PROJECT_STATE.md]` |
| Node | **18** baseline (RD School moved to **20 LTS** for LMS) | `[source: DEPLOYMENT_LESSONS.md L4]` |
| Python | 3.11.14 in the local bench venv | [verified: `env/bin/python -V`] |
| bench CLI | **5.29.0** | [verified: `bench --version`] |
| OS | Ubuntu 22.04 (mandi prod, hostname `Demo`) | `[source: PROJECT_STATE.md]` |

**Python ceiling — the single most important compatibility fact for v15:**

```
apps/frappe/pyproject.toml:7 → requires-python = ">=3.10,<3.14"
apps/erpnext/pyproject.toml:7 → requires-python = ">=3.10"
```
[verified in the 15.100.0 / 15.97.0 checkout]

So: Ubuntu 22.04 (Python 3.10) ✅, Ubuntu 24.04 (3.12) ✅,
**Ubuntu 26.04 (Python 3.14) ✗ — frappe v15 will not install.** A v15 client stays on
22.04/24.04. If the box must be 26.04, the job is a **v16 migration**, not a reinstall.
`[SERVER_KANTISHIVA.md records the same wall from the other side: v16 pins >=3.14,<3.15]`

**Assets: v15 builds are cheap, v16's are not.** `apps/erpnext/package.json` on 15.97.0 has
**no `scripts` block at all** — only `dependencies: {onscan.js}` [verified]. v16 added a
React 19 + Vite 8 + Tailwind 4 sub-build (`banking/`) that runs inside the Node heap and is
the OOM point on small VPSes, so there is no Vite sub-build and no 20–45 minute build peak
on v15.

**But the heap mechanism itself is NOT a v16 change — keep swap on a 1–2 GB v15 box.**
`get_safe_max_old_space_size()` = `max(1024, int(total_memory * 0.75))` off
`psutil.virtual_memory().total` (**physical RAM only, swap not counted**) is present in
v15 at `apps/frappe/frappe/build.py:293-306`, and `get_node_env()` (`build.py:289-290`)
is applied through `frappe.commands.popen`, which does `env = dict(environ, **env)`
(`frappe/commands/__init__.py:62-69`) — so frappe's computed `NODE_OPTIONS` overwrites
anything you export. [verified in the 15.100.0 checkout] What you may safely *not* carry
back is v16's **build time and memory budget**, not the swap advice.

**Redis: exactly two bench-managed instances** — `config/redis_cache.conf` and
`config/redis_queue.conf`. `redis_socketio` is a legacy alias force-synced to equal
`redis_cache`, not a third server. Defaults are 13000 / 11000; a bench created with a port
offset differs — `~/frappe-bench-v15` runs **13003 / 11003** [verified:
`sites/common_site_config.json`].

---

## 2. v15 install outline

Honest scope: **no source in this repo records a verbatim, end-to-end v15 apt+bench install
transcript.** What follows is the shape of the install with each step tagged. Anything
marked `[unverified]` must be confirmed against `--help` on the box before running.

### 2.1 Components that must exist before `bench init`

Not a copy-paste apt line — **no verified v15 apt invocation is on record.** These are the
components every one of our v15 boxes has:

MariaDB server (10.6 on 22.04) · redis-server · nginx · supervisor · Node 18 or 20 from
**NodeSource** (not nvm/fnm) · **yarn 1.x classic** · wkhtmltopdf (patched qt) · git ·
python3-dev + python3-venv + build-essential.

Two hard mechanical requirements, both verified in bench 5.29.0:

- **Node must be installed before `bench init`, even with `--skip-assets`.**
  `yarn install --check-files` runs unconditionally for any app with a `package.json`
  (`bench/app.py:960`); only the asset build is gated on `--skip-assets`. Missing yarn
  aborts with the literal string `Please install yarn using below command and try again.`
  (`bench/utils/bench.py:147`). [verified]
- **Node must come from a system-wide path.** bench resolves node with `which("node")` when
  generating the supervisor config; a per-user nvm node is invisible to the
  supervisor/sudo `secure_path`, and the **socketio program is silently omitted** from the
  generated config. Result: a site that looks fine but has dead realtime — no notifications,
  no progress bars, no list auto-refresh, and no error anywhere.
  `[source: SERVER_KANTISHIVA.md §3 + the researched runbook, quoting bench/config/supervisor.py:48-49]`

### 2.2 MariaDB config

Create `/etc/mysql/mariadb.conf.d/99-frappe.cnf` — named `99-` so it sorts **after**
Ubuntu's own `50-server.cnf` — containing only the utf8mb4 pair
(`character-set-server = utf8mb4`, `collation-server = utf8mb4_unicode_ci` under
`[mysqld]`, plus the client `default-character-set = utf8mb4`).

**Do NOT copy the classic Frappe `my.cnf` from old guides.** `innodb-file-format` and
`innodb-large-prefix` were **removed in MariaDB 10.3+** and stop the server from starting.
This was hit on 11.8 during the v16 build `[source: SERVER_KANTISHIVA.md §4]`; since the
removal predates 10.6, **the same guide will break a v15/10.6 box** [reasoning stated, not
re-tested on 10.6]. The explicit `collation-server` line matters on 11.x (default changed to
`utf8mb4_uca1400_ai_ci`); on 10.6 it is harmless and worth setting anyway.

### 2.3 Bench and site

```bash
# as the frappe user
bench init frappe-bench --frappe-branch version-15 --python /usr/bin/python3 --verbose
cd frappe-bench
bench get-app erpnext --branch version-15
bench get-app https://github.com/resilient-tech/india-compliance --branch version-15   # [unverified URL — confirm upstream]

bench new-site --help | head -40        # ← CONFIRM flag names, then run new-site
bench --site <site> install-app erpnext
```

- **`--frappe-branch version-15` is mandatory, not optional.** With no branch, bench clones
  frappe's **default branch (`develop`)** — verified in bench 5.29.0
  (`bench/commands/make.py:9-10` → `bench/utils/system.py:85-93`, `get_app(branch=frappe_branch)`).
  Verify after: `git -C apps/frappe rev-parse --abbrev-ref HEAD` → `version-15`. [verified]
- **`bench new-site` flag names moved in v15.** `--db-root-username` / `--db-root-password`
  superseded `--mariadb-root-username` / `--mariadb-root-password`, and
  `--no-mariadb-socket` was deprecated in favour of `--mariadb-user-host-login-scope`.
  Different bench builds differ — **run `bench new-site --help` first every time.**
  `[source: researched v16 runbook, flagged there as a v15 rename]`
- **Never re-run `bench new-site --force` on a live site: it drops the database.**
  `[source: researched runbook, listed as a data-destruction risk]`
- Passwords passed on the command line are visible in `ps` and land in shell history.
  Clear history afterwards; do **not** persist `mariadb_root_password` into
  `sites/common_site_config.json`.

### 2.4 Redis must be up before `install-app`

**Bench's redis must be running before `install-app` on a new box, or the install fails
silently.** `[source: DEPLOYMENT_LESSONS.md L8]` Frappe touches the cache during site
creation; with redis down the runbook records the failure as
`Error 111 connecting to 127.0.0.1:13000` `[source: researched runbook — v16 port default]`.

Starting bench's own redis by hand (this exact pair is what restarts the local v15 bench)
[verified: `PROJECT_STATE.md` "Running it again later", executed 2026-08-16]:

```bash
cd ~/frappe-bench-v15
redis-server config/redis_cache.conf --daemonize yes
redis-server config/redis_queue.conf --daemonize yes
bench serve --port 8004        # or `bench start` for the full dev stack
```

### 2.5 Production wiring

```bash
bench config dns_multitenant on                       # BEFORE nginx generation
cd /home/frappe/frappe-bench
bench setup production frappe --yes                   # supervisor + nginx + sudoers
# fallback if that aborts:
bench setup supervisor --yes

# MUST come before the standalone `bench setup nginx` — see the log_format note below
cat > /etc/nginx/conf.d/00-log-format.conf <<'EOF'
log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                '$status $body_bytes_sent "$http_referer" '
                '"$http_user_agent" "$http_x_forwarded_for"';
EOF

bench setup nginx --yes
sudo nginx -t                                         # must pass before any reload
sudo visudo -c                                        # must print "parsed OK"
sudo chmod 755 /home/frappe
```

- **The standalone `bench setup nginx` breaks nginx on Debian/Ubuntu without that
  `00-log-format.conf` file** — on v15's bench too, not only v16's:
  ```
  nginx: [emerg] unknown log format "main"
  ```
  `bench setup nginx`'s click options default to `--logging combined` and
  `--log_format main` (bench 5.29.0, `bench/commands/setup.py:28-41`);
  `bench/config/nginx.py:49-53` sets `template_vars["logging"]` whenever `logging !=
  "none"`; `config/templates/nginx.conf:124` then renders
  `access_log /var/log/nginx/access.log main;`. Debian/Ubuntu's `/etc/nginx/nginx.conf`
  defines no `log_format main` (that name is from nginx.org's stock config), so nginx
  refuses to start. Only `bench setup production` escapes it, because it calls
  `make_nginx_conf(bench_path, yes=yes)` with **no `logging` argument** —
  `template_vars["logging"]` is never set and no `access_log` line is emitted. Since §2.5's
  fail2ban fallback runs `bench setup nginx` standalone, **you will hit this on v15.**
  [verified in bench 5.29.0 source; the alternative is
  `bench setup nginx --yes --logging none`, but the separate file survives future
  regenerations and a forgotten flag does not]

- **Always pass `--yes`** to `bench setup production|nginx|supervisor`. They block on an
  interactive overwrite prompt otherwise, which **hangs a non-TTY SSH session**.
  `[source: SERVER_KANTISHIVA.md §8]`
- **`bench setup production` can abort at `bench setup role fail2ban`.** Verified in bench
  5.29.0, `bench/config/production_setup.py:27-28`:
  ```python
  if not which("fail2ban-client"):
      exec_cmd("bench setup role fail2ban")
  ```
  That role needs **Ansible**. On a box without it, install fail2ban from apt with an sshd
  jail *first*, then run `bench setup supervisor --yes` and `bench setup nginx --yes`
  individually. [verified in bench source; hit for real on the v16 build]
- **`chmod 755 /home/frappe` or nginx cannot serve assets.** Modern Ubuntu creates home
  directories mode 750; nginx runs as `www-data` and cannot traverse into
  `/home/frappe/frappe-bench/sites`. Symptom: the site loads but **every CSS/JS 403s**.
  `[source: DEPLOYMENT_LESSONS.md L8 + researched runbook for the mechanism]`
- **Supervisor conf must be symlinked into `/etc/supervisor/conf.d/`.**
  `bench setup production` does it (`production_setup.py:69-74`), but only
  `if not os.path.islink(...)` — if a plain file is sitting there it is left alone. If you
  ran only `bench setup supervisor`, link it yourself:
  `config/supervisor.conf` → `/etc/supervisor/conf.d/frappe-bench.conf`.
  `[DEPLOYMENT_LESSONS.md L8; SERVER_KANTISHIVA.md; bench source verified]`
- Then SSL per **§0** — `certbot certonly --nginx`, never `certbot --nginx`.

---

## 3. Lesson 1 — Fresh sites can silently miss ERPNext's install-time Custom Fields

**Applies to: v15 (mechanism also present in v16).** Incident 2026-07-21, fresh
`rdps.trustbit.cloud`.

**Symptom.** Opening any RFQ — and the same code path behind PO supplier-contact lookups —
crashes with:

```
pymysql.err.OperationalError: (1054, "Unknown column 'tabContact.is_billing_contact' in 'WHERE'")
```

**Root cause.** ERPNext's `after_install` hook creates a handful of Custom Fields
(`Contact.is_billing_contact`, `Address.tax_category`, `Address.is_your_company_address`,
`Email Account.company`). On that site the step silently did not complete. **Nothing
notices until a form that queries those columns is opened — weeks later.**

### Why `bench migrate` does NOT fix the record-missing variant

`bench migrate` syncs *schema* for fields that exist as **DocField / Custom Field records**.
When the Custom Field *record itself* is missing, migrate has nothing to sync and exits
clean. Two different failures wear the same error text:

| Variant | What is missing | Fix |
|---|---|---|
| **Record present, column missing** | DB column | `bench migrate` (this is the one that "worked" on the old dev box) |
| **Record missing** | the `Custom Field` row itself | re-run the app's install-time creator |

**Tell them apart first.** Filter the `Custom Field` list by `dt = Contact`, or from
`bench --site <site> console`:

```python
frappe.db.get_value("Custom Field", {"dt": "Contact", "fieldname": "is_billing_contact"})  # record?
frappe.db.has_column("Contact", "is_billing_contact")                                      # column?
```
(`frappe.db.has_column` exists at `apps/frappe/frappe/database/database.py:1335` on 15.100.0 [verified])

### The idempotent creators — safe to re-run any time

```bash
bench --site <site> execute erpnext.setup.install.create_print_setting_custom_fields
bench --site <site> execute erpnext.setup.install.create_address_and_contact_custom_fields
bench --site <site> execute erpnext.setup.install.create_custom_company_links
bench --site <site> clear-cache      # dev/demo site only — on production use the targeted form, §11
```
`[source: DEPLOYMENT_LESSONS.md L1 — run on rdps.trustbit.cloud]`

On a **production** site replace that last line with the targeted clear from
`bench --site <site> console` (§11):

```python
frappe.clear_cache(doctype="Contact")
frappe.clear_cache(doctype="Address")
from frappe.cache_manager import clear_global_cache; clear_global_cache()
```

**Function names vary by erpnext minor. Check before you run:**

```bash
grep "^def create_" apps/erpnext/erpnext/setup/install.py
```

On **erpnext 15.97.0** (mandi prod's pin) that returns exactly [verified, run this session]:

```
create_print_setting_custom_fields
create_custom_company_links
create_default_success_action
create_default_energy_point_rules
create_default_role_profiles
```

→ **`create_address_and_contact_custom_fields` does not exist on 15.97.0.** Running it there
fails. On that version `Email Account.company` (and `Communication.company`) come from
`create_custom_company_links` — verified by reading
`apps/erpnext/erpnext/setup/install.py:135-165`, which is exactly what
`DEPLOYMENT_LESSONS.md` warned about for older v15 (their dev box on 15.108).

### Where `is_billing_contact` actually comes from on 15.97.0 — and why migrate *does* help there

Read this session, and it changes the fix you reach for:

- `Contact.is_billing_contact` is shipped as a **customization JSON**, not by a creator
  function: `apps/erpnext/erpnext/erpnext_integrations/custom/contact.json`, and that file
  carries **`"sync_on_migrate": 1`** [verified].
- Same for Address: `apps/erpnext/erpnext/accounts/custom/address.json`, `sync_on_migrate: 1`.
- `frappe.modules.utils.sync_customizations()` is called from `frappe/migrate.py:145`
  (every migrate) and `frappe/installer.py:335` (install-app). At
  `frappe/modules/utils.py:119-122` it applies a file **only** when the JSON sets
  `sync_on_migrate` — otherwise only during install [verified].

**So on erpnext 15.97.0, `bench migrate` recreates the `Contact` / `Address` custom-field
records, but nothing recreates `Email Account.company` — only
`create_custom_company_links` does.** The general rule survives intact; the specific remedy
depends on your exact erpnext minor. **Determine the origin (creator function vs.
`custom/*.json`) on the version in front of you, then choose migrate vs. creator.**

### Rules

- After bootstrapping **any** new site, run the creators that exist on that version once,
  then verify `Contact.is_billing_contact` exists as a column.
- Put the column checks in your deploy health gate. RD School's `verify_deployment()` now
  checks `Contact.is_billing_contact`, `Address.tax_category`, `Email Account.company` on
  every deploy and **fails the gate** if any is missing. `[source: DEPLOYMENT_LESSONS.md L1]`
- Any user report of `Unknown column 'tabX.y'` → check for the **record** first.
  Record missing → re-run the app's install-time creator. Record present → `bench migrate`.

---

## 4. Lesson 2 — Installing ANY app re-syncs role profiles and strips roles

**Applies to: v15.** Incident 2026-07-20.

Installing the `lms` app **removed the `Employee` role from School users.**
`[source: DEPLOYMENT_LESSONS.md L2]`

**Root cause, verified in frappe 15.100.0 source** —
`apps/frappe/frappe/core/doctype/role_profile/role_profile.py`:

```python
def on_update(self):
    self.clear_cache()
    self.queue_action(
        "update_all_users",
        now=frappe.flags.in_test or frappe.flags.in_install,   # ← runs INLINE during an install
        enqueue_after_commit=True,
    )

def update_all_users(self):
    ...
    for user, roles in user_roles.items():
        if roles != role_profile_roles:
            user = frappe.get_doc("User", user)
            user.roles = []                       # ← every role wiped
            user.add_roles(*role_profile_roles)   # ← only the profile's roles restored
```

Any Role Profile that gets saved during an install — and erpnext ships
`create_default_role_profiles` in `setup/install.py` — fires `on_update`, which for **every
user whose `role_profile_name` points at it** blanks `user.roles` and re-adds only the
profile's roles. Roles granted outside the profile are gone. Because `now=` is true during
install, it happens synchronously inside the `install-app`, not in a background job you
might notice.

**Rule: after every `bench install-app <anything>`, reconcile and health-check.** RD School's
trio (their app's function names; each client's equivalents differ):

```bash
bench --site <site> execute rdschool.setup_data.assign_full_profile_roles
bench --site <site> execute rdschool.setup_data.rebuild_school_docperms
bench --site <site> execute rdschool.setup_data.ensure_company_user_permissions
bench --site <site> execute rdschool.setup_data.verify_deployment   # must print HEALTHCHECK PASS
```

The wider RD School rule: **always deploy via `bash apps/rdschool/deploy.sh <site>` — never
hand-run `migrate`/`build` alone.** The script pulls, migrates, builds, reconciles
roles/permissions, restarts, and refuses to pass unless `verify_deployment()` prints
`HEALTHCHECK PASS`. `[source: DEPLOYMENT_LESSONS.md]`

---

## 5. Lesson 3 — A failed `bench get-app` leaves a half-registered app

**Applies to: v15 (bench-level, so v16 too).** Incident 2026-07-20.

If `get-app` dies mid-way (e.g. a yarn failure), the app folder exists under `apps/` but is
**NOT** in `sites/apps.txt`. A later `bench build --app <app>` then dies with:

```
TypeError: paths[0] must be of type string
```

plus **exit 143**. Exit 143 is SIGTERM, so this **looks like an OOM kill and is not** — the
real cause is the half-registration. Do not go chasing memory. `[source: DEPLOYMENT_LESSONS.md L3]`

**Rules:**

- After any failed `get-app`, either finish the registration properly or **remove the app
  folder before retrying**.
- **Never append to `apps.txt` with `echo >>`.** The file may lack a trailing newline; doing
  this once produced the literal corrupted entry **`erpnextlms`** (two app names fused).
  Rewrite the whole file, or let bench manage it.

---

## 6. Lesson 4 — Node version pins (LMS)

**Applies to: v15.**

- Frappe v15 itself is fine on **Node 18 or 20**.
- RD School's server went **18 → 20 LTS (NodeSource)** to build LMS.
- **LMS is pinned at `v2.53.0` (detached HEAD).** Every newer tag requires **Node ≥ 22**, and
  `@iconify/utils` needs **≥ 20.12**.

**Rule: do not update the lms app past v2.53.0 without first moving the server to Node 22.**
`[source: DEPLOYMENT_LESSONS.md L4]`

---

## 7. Lesson 5 — Fixtures bake the exporting site's company into `link_filters`

**Applies to: v15.** Incident 2026-06-15.

`bench migrate` clobbers company-specific `link_filter`s with the **dev** company name
(`"RD School Betul"`) on a prod site whose company is `"RDPS Betul"` → **empty dropdowns**.

Fixed with an `after_migrate` hook (`relocalize_company_link_filters`) that auto-heals on
every migrate.

**Rule: never hardcode a company name anywhere — code, fixtures, or JS. Prod and dev
companies differ.** `[source: DEPLOYMENT_LESSONS.md L5]`

---

## 8. Lesson 6 — Exactly one fixtures entry per doctype

**Applies to: v15.** Incident 2026-06-15.

Two `custom_field` fixture entries in `hooks.py` **silently overwrite each other** — export
writes one file per doctype, and the second entry wins. RD School shipped **1 of 15 fields**
that way, with no error at any point.

**Rule:** exactly one fixtures entry per doctype. After any UI field change:

```bash
bench export-fixtures --app <app>
```

then **eyeball the JSON diff for lost rows.** `[source: DEPLOYMENT_LESSONS.md L6]`

---

## 9. Lesson 7 — Custom-DocPerm roles need explicit READ on supporting masters

**Applies to: v15.** Incident 2026-06-15.

ERPNext buying/stock/accounts forms auto-load supporting masters — Address, Contact, Terms,
Price List, Warehouse, taxes, Mode of Payment, etc. Roles built on **Custom DocPerms only**
have no read grant on those, so a tester hits:

```
No permission on <Master>
```

**Rule:** any such report means add the doctype to `SUPPORT_READ_DOCTYPES` in `setup_data.py`
and re-run `rebuild_school_docperms` on **dev *and* prod**. `[source: DEPLOYMENT_LESSONS.md L7]`

Related v15 permission trap from the BBPL app: a direct `INSERT` into `tabCustom DocPerm`
**drops the Standard DocPerm** — call `setup_custom_perms()` first.
`[source: Betul Bio Fuel CLAUDE.md, Lesson 169]`

---

## 10. Misc server gotchas (RD School Lesson 8)

**Applies to: v15.**

| Gotcha | Detail |
|---|---|
| SSL | **See §0 — `certbot certonly --nginx`.** `bench setup nginx` would drop SSL issued in installer mode; re-run certbot/`bench setup nginx` if that happens. RD School's `deploy.sh` never touches nginx. |
| Restart after install | **Restart bench after every app install.** A bench started before the install hits scheduler `ModuleNotFoundError`. |
| Redis before install | **Bench's redis must be running before `install-app`** on a new box, or the install fails silently. |
| Home dir perms | **`chmod 755 /home/frappe`** or nginx cannot serve assets. |
| Supervisor | **Supervisor conf must be symlinked into `/etc/supervisor/conf.d/`.** |

---

## 11. v15 production operating rules (Betul Bio Fuel)

**Applies to: v15.** These are prod-server rules from a live client, not app-development
lessons. Full app-level catalogue lives with that project.

### Standard deploy sequence

```bash
chown -R frappe:frappe apps/<app>        # BEFORE the pull — see below
git pull upstream <branch>
bench build --app <app>
bench --site <site> migrate
find apps/<app> -name __pycache__ -exec rm -rf {} +
supervisorctl restart frappe-bench-web: frappe-bench-workers:
```

- **`chown -R frappe:frappe` before the pull, not just `.git`.** Root-owned working-tree
  files from a prior direct/root operation **block the checkout and leave a PARTIAL tree**
  (version bumped, HEAD and code not). Verify with
  `find <app> ! -user frappe | grep -v __pycache__` → 0 rows. Recovery from a failed
  mid-checkout: `chown -R frappe:frappe <app>` then `git reset --hard upstream/<branch>`.
  `[source: BBPL, "Round 72" / v2.16.1 deploy gotcha]` The same class of bug bit the v16
  build — running bench commands as root left **125 root-owned files**, producing runtime
  `PermissionError` on `logs/render-template.log` during PDF render. `[source: SERVER_KANTISHIVA.md]`
- **Purge `__pycache__` after adding a new API file, or the new module is not picked up.**
  `[source: BBPL Lesson 180]`

### NEVER on a v15 production box

| Forbidden | Why |
|---|---|
| `bench clear-cache` — **and `bench --site <site> clear-cache`, which is the same frappe command** | It deletes **every redis key for the site** — verified at `apps/frappe/frappe/__init__.py:984-989`, which enumerates all keys and deletes everything not listed in the `persistent_cache_keys` hook. BBPL's recorded symptom is that it flushes all user sessions. *(Nuance found while writing this: v15 sessions fall back to `tabSessions` in the DB — `frappe/sessions.py:369` — so the exact logout symptom is `[unverified]` here. The rule stands regardless: it is a blunt instrument on a live site, and the targeted alternative is free.)* Use instead, from `bench console`: `frappe.clear_cache(doctype="X")` + `from frappe.cache_manager import clear_global_cache; clear_global_cache()`. **This prohibition covers `bench --site <site> clear-cache` too — it is the same command; every place in this repo that shows the site-scoped form is either a dev/demo site or must be swapped for the targeted form on production.** `[source: BBPL Lesson 234]` |
| `bench restart` / `supervisorctl restart all` | On that prod box the **bench redis supervisor group is FATAL/STOPPED** (vestigial — the OS redis instance serves the site), so a blanket restart hits it. Restart `frappe-bench-web:` and `frappe-bench-workers:` explicitly. **Check your own box's `supervisorctl status` before adopting this** — it is a property of that server, not of v15. `[source: BBPL, verified there 29 Jul 2026]` |
| `developer_mode 1` | Prod invariant. |
| `restart_supervisor_on_update: true` | Prod invariant. |
| `git add .` / `git add -a` in the app tree | Sweeps other sessions' WIP. Always `git add -- <exact paths>`, then verify `git diff --cached --name-only`. |
| `scp`-ing or editing the prod tree between commits | Git diverges from the repo and the next pull fails or silently reverts. `[source: BBPL Lesson 230]` |

**Prod config invariants to assert on every deploy:** `developer_mode = 0`,
`restart_supervisor_on_update = false`, `live_reload = false`.

**After editing `hooks.py`** (doc_events / scheduler_events / after_migrate /
permission_query_conditions / …), a supervisor restart is **not** enough — the merged hook
dict stays cached in redis. From `bench console`, after the restart:

```python
frappe.cache().delete_keys("hooks")
frappe.cache().delete_keys("app_hooks")
frappe.get_hooks("doc_events")   # verify
```
`[source: BBPL Lesson 252]`

**Deploy to demo first, always.** Even when the instruction is "deploy to production", the
same change must land on demo, be verified by the user there, and only then be promoted.
No exception for hotfixes, JS-only changes, or "tiny" data ops.
`[source: BBPL Rule 12, reinforced after the v2.13.1 hotfix skipped it]`

---

## 12. Restoring a v15 production site into a local bench

**Applies to: v15.** Executed 2026-08-16 for `mandi.trustbit.in` → `mandi.local`.
`[source: PROJECT_STATE.md — all steps ran, 0 errors]`

The whole point of this recipe: **it never needs the local MariaDB root password.**

1. **Pin the local apps to the exact production versions first** (`frappe` → `v15.100.0`,
   `erpnext` → `v15.97.0`). Note the git remote in these benches is named **`upstream`**,
   not `origin`.
2. Safety-dump the existing local DB to scratch before touching it.
3. Drop all tables in the existing local site DB and **import the production dump with the
   site's own DB user**. Frappe dumps carry **no `CREATE DATABASE` / `USE`**, so they import
   into any database — which is what lets you skip root entirely.
   (`bench restore` and `bench new-site` *create* a database and therefore *do* need the
   root password.)
4. Restore public/private files from the `*-files.tar` archives.
5. `bench --site <site> migrate`.

Verified afterwards: 17 Deals, 27 Deal Deliveries, 13 Vehicle Dispatches, 12 Sales Invoices,
2 Companies; all 9 client reports registered; `/api/method/ping` → `pong`.

**Restore gotchas:**

- A **MariaDB 10.6 dump imported cleanly into a local MariaDB 12.1** (Homebrew). Worth
  knowing, but do not assume it in the other direction.
- Homebrew may have both `mariadb` (running, serves `:3306`) and `mariadb@10.6` (client
  only). Use `/opt/homebrew/bin/mysql`.
- **A restore carries production passwords.** After the restore, every user in the local
  site has their production password. Treat the local copy as production data.
- Backups and server snapshots hold the live `db_password` and full customer data — keep
  them `.gitignore`d. Only the app source belongs on GitHub.
- The old mandi prod box used **root password SSH login with fail2ban active** — rapid failed
  logins ban the source IP for ~10 minutes.

**Backup command in use** (before any risky operation):

```bash
bench --site <site> backup --with-files
```

Kantishiva's v16 box runs this every 6 hours from `crontab -u frappe`; a v15 box should do
the same. `[source: SERVER_KANTISHIVA.md]`

---

## 13. v16-discovered gotchas — do they apply to v15?

Read this before back-porting anything from the Kantishiva v16 notes.

| v16 finding | v15? |
|---|---|
| `certbot certonly --nginx`, never `certbot --nginx` | **YES** — the symlink is created by bench, not by frappe. Verified in bench 5.29.0. See §0. |
| `bench config dns_multitenant on` before `bench setup nginx` | **YES** — same gating code in bench 5.29.0's `config/nginx.py`. |
| Always pass `--yes` to `bench setup production\|nginx\|supervisor` | **YES** — bench-level. |
| `bench setup production` aborts on `bench setup role fail2ban` (needs Ansible) | **YES** — verified in bench 5.29.0 `config/production_setup.py:27-28`. |
| Root-run bench commands leave root-owned files that break the frappe user | **YES** — independently recorded on BBPL v15 prod (Round 72). |
| Node from NodeSource, never nvm (supervisor loses socketio) | **YES** — bench resolves node with `which()` at config-generation time. |
| Classic Frappe `my.cnf` breaks MariaDB (`innodb-file-format`, `innodb-large-prefix`) | **Likely** — those keys were removed in 10.3+, so 10.6 is affected. Not re-tested on 10.6. |
| `FileNotFoundError: 'uv'` from `bench init` | **NO for v15's needs** — uv is bench 5.28+ default behaviour. A v15 build on a modern bench can set `BENCH_DISABLE_UV=1` to restore the venv+pip path (`bench/utils/__init__.py::use_uv`). `[source: researched runbook]` |
| PEP 668 / `EXTERNALLY-MANAGED` blocking `pip install frappe-bench` | Depends on the OS, not the frappe version. Not an issue on 22.04. |
| `ZoneInfoNotFoundError: 'Asia/Calcutta'` (tzdata-legacy) | **Only Ubuntu 24.04+** — the deprecated tz aliases moved to a separate `tzdata-legacy` package there. A v15 box on 22.04 is unaffected; a v15 box on 24.04 **would** be. Fix: `apt-get install -y tzdata-legacy` + `env/bin/pip install tzdata` + restart workers. |
| nginx `unknown log format "main"` | **YES** — bench 5.29.0 has the same defaults (`bench setup nginx` → `--logging combined`, `--log_format main`, `bench/commands/setup.py:28-41`; `bench/config/nginx.py:49-53` sets `template_vars["logging"]` whenever `logging != "none"`; `config/templates/nginx.conf:124` renders `access_log /var/log/nginx/access.log main;`). Only `bench setup production` escapes it, because it calls `make_nginx_conf(bench_path, yes=yes)` with no `logging` argument. So the standalone `bench setup nginx --yes` prescribed as §2.5's fail2ban fallback **does** hit it on v15. Fix: `/etc/nginx/conf.d/00-log-format.conf` (sorts before `frappe-bench.conf`) — see §2.5. |
| PDF via `chrome` generator / `bench setup-chrome` | **NO** — v16 only. v15 is wkhtmltopdf, full stop. |
| Heavy React/Vite asset build, swap mandatory | **NO** — erpnext 15.97.0's `package.json` has no `scripts` block. Verified. |
| `mysqlclient` / `pkg-config` hard requirement | **NO** — that dependency is new in v16. |
| v15 apps run on v16 | **NO.** erpnext v16 pins `frappe >=16.21.0,<17.0.0` and `requires-python >=3.14`, while v15 frappe pins `>=3.10,<3.14`. Moving `trustbit_mandi` / `item_creator` to v16 is a real migration with breaking API changes, not a reinstall. `[source: SERVER_KANTISHIVA.md + researched runbook]` |

---

## 14. Greppable error index

| Error string (literal) | Where it appears | Cause → fix |
|---|---|---|
| `pymysql.err.OperationalError: (1054, "Unknown column 'tabContact.is_billing_contact' in 'WHERE'")` | opening an RFQ; PO supplier-contact lookup | ERPNext install-time Custom Field missing → §3 (decide record-missing vs column-missing first) |
| `Unknown column 'tabX.y'` (general form) | any form | same class → §3 |
| `TypeError: paths[0] must be of type string` | `bench build --app <app>` | app folder present but absent from `sites/apps.txt` after a failed `get-app` → §5 |
| exit code `143` on `bench build` | same | SIGTERM, **not** OOM — same cause as above → §5 |
| `ModuleNotFoundError` (scheduler) | bench started before an app install | restart bench after every `install-app` → §10 |
| `No permission on <Master>` | ERPNext buying/stock/accounts forms | Custom-DocPerm role lacks READ on a supporting master → §9 |
| `Please install yarn using below command and try again.` | `bench init` / `bench get-app` | yarn 1.x classic not installed; `--skip-assets` does not skip yarn → §2.1 (`bench/utils/bench.py:147`) |
| `Error 111 connecting to 127.0.0.1:13000` | `bench new-site` / `install-app` | bench redis not running → §2.4 `[source: researched runbook, v16 port default]` |
| `erpnextlms` (in `sites/apps.txt`) | not an error — a corrupted line | `echo >>` onto a file with no trailing newline → §5 |
| `zoneinfo._common.ZoneInfoNotFoundError: 'No time zone found with key Asia/Calcutta'` / `ModuleNotFoundError: No module named 'tzdata'` | setup wizard stage 1, Ubuntu **24.04+** only | `tzdata-legacy` not installed → §13 |
| `unknown log format "main"` | `nginx -t` / nginx start, after a standalone `bench setup nginx` | bench-generated `access_log ... main` with no matching `log_format` in Debian/Ubuntu's `nginx.conf`. **Applies to v15's bench 5.29.0 as well as v16** — the `--logging combined` / `--log_format main` defaults are identical; only `bench setup production` avoids it → §2.5, §13 |
| `PermissionError` on `logs/render-template.log` | PDF render as the frappe user | root-owned files from a root-run bench command → `chown -R frappe:frappe` → §11 |

---

## 15. Open items / not verified

- No end-to-end v15 apt package list is on record anywhere in our docs. §2.1 lists the
  components, not a verified `apt-get install` line.
- The `india_compliance` get-app URL in §2.3 is `[unverified]` — confirm the upstream remote
  before using it. The pinned version (15.25.4, branch `version-15`) *is* verified.
- The MariaDB 10.6 config warning (§2.2) is reasoned from the 10.3 removal, verified on 11.8,
  **not** re-tested on 10.6.
- `bench clear-cache` → "logs everyone out": the redis-key deletion is verified in source;
  the session-logout consequence is BBPL's field observation and is contradicted by a DB
  fallback path in `frappe/sessions.py`. The prohibition stands either way.
- No v15 box in these sources has been rebuilt from scratch since the v16 lessons landed.
  The first v15 build that follows §2 should have its transcript folded back into this file.
