# Frappe/ERPNext v15 on Ubuntu 24.04 — what changes

`v15/install-and-gotchas.md` is written against **Ubuntu 22.04**, and says so — its
compatibility table explicitly flags `tzdata-legacy` as *"Only Ubuntu 24.04+ … a v15 box on
24.04 **would** be [affected]"*. This file fills that gap: everything that differs when a
**v15** bench runs on **24.04 (Noble)**.

Established on a real build — bench **5.31.0**, Frappe **15.119.1**, ERPNext **15.120.0**,
Python **3.12.3**, MariaDB **10.11.14**, Node **20.20.2** — recorded in
[builds/2026-09-01-…](../builds/2026-09-01-bare-metal-four-site-cloudflare-tunnel.md).

**Read this alongside [install-and-gotchas.md](install-and-gotchas.md), not instead of it.**
Everything there still applies unless contradicted below.

**Confidence tags:** `[verified]` run or read in source during this build ·
`[source: X]` quoted · `[unverified]` plausible, unconfirmed.

---

## Summary

| Trap | Severity | 22.04? |
|---|---|---|
| §1 `HOME_MODE 0750` breaks nginx serving `/assets` and `/files` | **high — silent, misleading** | no, 24.04+ only |
| §2 socketio `BACKOFF` — new failure mode, corrects the existing doc | high | bench-version, not OS |
| §3 `bench init` needs `uv` even on v15 | blocks install | bench-version, not OS |
| §4 Python 3.12 is fine for v15 (and why v16 would not be) | informational | — |
| §5 MariaDB 10.11 vs the removed `innodb-file-format` keys | blocks MariaDB start | worse on 24.04 |
| §6 `tzdata-legacy` for `Asia/Calcutta` | breaks scheduler | 24.04+ only |
| §7 Multi-tenant `table_open_cache` | perf, silent | any |

---

## 1. `HOME_MODE 0750` — nginx cannot read `/home/frappe`

**The most valuable finding on this platform, and the least obvious.**

### Symptom

Sites appear to work. Pages return **HTTP 200**, `/api/method/ping` answers
`{"message":"pong"}`, the HTML is correct — but **every stylesheet and script 404s** and the
Desk renders unstyled. It looks exactly like a CDN, proxy or asset-build fault.

```
[crit] 25268#25268: *67 stat()
  "/home/frappe/frappe-bench/sites/assets/erpnext/dist/css/erpnext_email.bundle.RDU6AXP6.css"
  failed (13: Permission denied), client: 127.0.0.1, ...
```

### Cause

**Ubuntu 24.04 changed `adduser`'s default home-directory mode from `0755` to `0750`.**
**[verified]**

```console
$ grep '^HOME_MODE' /etc/login.defs
HOME_MODE	0750

$ stat -c '%a %n' /home/sysadmin
750 /home/sysadmin
```

```console
$ namei -l /home/frappe/frappe-bench/sites/assets/erpnext/dist/css/x.css
drwxr-xr-x root   root   /
drwxr-xr-x root   root   home
drwxr-x--- frappe frappe frappe          ← no world execute
drwxrwxr-x frappe frappe frappe-bench
```

nginx workers run as **`www-data`**, which is not in the `frappe` group, so they cannot
**traverse** `/home/frappe` at all.

### Why the symptom misleads

**HTML comes from gunicorn via `proxy_pass` — nginx never touches the filesystem for it.**
Only `/assets` and `/files` are read from disk. So the application is completely healthy and
only static delivery is broken, which points suspicion at everything except permissions.

Every Frappe install guide predates this change and assumes `0755`.

### Fix

```bash
sudo chmod o+rx /home/frappe
sudo systemctl reload nginx
```

### Check it on every 24.04 build

```bash
stat -c '%a %n' /home/frappe                                  # want 755
sudo -u www-data test -x /home/frappe && echo OK || echo BROKEN
```

**This also breaks user-uploaded files** (`/files`, `/private/files`), not just bundled
assets — attachments 404 identically. If you only test the Desk you may not notice until a
client reports a missing document.

> Alternative if you would rather not widen the home directory:
> `usermod -aG frappe www-data` plus group-execute on the path. `chmod o+rx` is what
> standard Frappe deployments assume and what bench's nginx template is written against.
> **[unverified — the group route was not tested]**

---

## 2. CORRECTION — socketio fails loudly on bench 5.31.0

**Supersedes** `install-and-gotchas.md` §2.1 and its summary-table row
*"Node from NodeSource, never nvm (supervisor loses socketio)"*, **for bench 5.31.0**.

That guidance says nvm-node is invisible at config-generation time so the socketio program
is **"silently omitted"** — dead realtime with no error anywhere.

**Observed on bench 5.31.0: the program was generated and failed loudly instead.**

```
frappe-bench-web:frappe-bench-node-socketio   BACKOFF
```

```
Error: Cannot start socketio: node not found and the python backend is unavailable.
Install node or a frappe version with frappe.realtime.server.
```

### Mechanism — bench resolves node twice

**(a) Generation time**, deciding whether to write the stanza: **[verified]**

```python
# bench/config/supervisor.py:49-52
"node": which("node") or which("nodejs"),
"socketio_enabled": config.get("socketio_backend", "node") == "python"
or bool(which("node") or which("nodejs")),
```

**(b) Runtime**, when supervisor launches it: **[verified]**

```python
# bench/commands/socketio.py
node = which("node") or which("nodejs")
if not node:
    raise click.ClickException(
        "Cannot start socketio: node not found and the python backend is"
        " unavailable. ...")
os.execv(node, [node, os.path.join("apps", "frappe", "socketio.js")])
```

The generated stanza bakes in **no absolute node path**: **[verified]**

```ini
[program:frappe-bench-node-socketio]
command=/home/frappe/.local/bin/bench socketio
```

### The trap: the recommended idiom causes it

```bash
sudo -H env "PATH=$PATH" bench setup production frappe --yes
```

That `env "PATH=$PATH"` wrapper — needed so sudo's `secure_path` does not hide `bench` —
**also exports the nvm node directory into the generation environment**. So lookup (a)
succeeds and the stanza *is* written; supervisor then runs it in its own minimal
environment, where lookup (b) fails.

| node visible at generation? | outcome |
|---|---|
| No (plain `sudo bench setup production`) | stanza omitted → **silent** dead realtime |
| Yes (`sudo -H env "PATH=$PATH" …`) | stanza written → runtime failure → **BACKOFF** |

Both are the same root cause. The loud one is strictly better — and following current best
practice is what converts the silent bug into the visible one.

### Fix

The standing advice (**system-wide node, not nvm**) remains correct. If node is already in
nvm, expose it rather than rebuilding:

```bash
NODEBIN=$(dirname "$(command -v node)")     # with nvm sourced
for b in node npm npx yarn; do sudo ln -sf "$NODEBIN/$b" /usr/local/bin/$b; done
sudo supervisorctl restart all
```

Verify the way **supervisor** sees it, not your login shell:

```bash
sudo -i node --version
sudo supervisorctl status | grep socketio     # RUNNING, not BACKOFF/STARTING
```

### `socketio_backend: python` is not an escape hatch on v15

bench 5.31.0 supports a node-free Python backend
(`bench/commands/socketio.py:47`), but **Frappe 15.119.1 does not ship
`frappe.realtime.server`** — `apps/frappe/frappe/realtime.py` exists with no `server`
submodule — so it warns and falls back to node. **[verified]** Relevant on newer Frappe only.

---

## 3. `bench init` needs `uv` — on v15 too

```
FileNotFoundError: [Errno 2] No such file or directory: 'uv'
ERROR: There was a problem while creating frappe-bench
```

`install-and-gotchas.md` notes uv as bench 5.28+ behaviour. Worth stating plainly: **this is
a property of the bench version, not the Frappe branch**, so a `--frappe-branch version-15`
build on a current bench hits it. **[verified on bench 5.31.0]**

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
export PATH="$HOME/.local/bin:$PATH"
bench init frappe-bench --frappe-branch version-15 --python /usr/bin/python3.12
```

Or `BENCH_DISABLE_UV=1` to restore the venv+pip path, as already documented. Installing uv
is lower-friction on a new box; the env var matters when reproducing an older toolchain.

The error appears **after** several minutes of cloning and names only the missing binary, so
it reads like a bench bug rather than a missing dependency.

---

## 4. Python 3.12 is correct for v15 — and this is where v15/v16 diverge

Ubuntu 24.04 ships **Python 3.12.3**, inside v15's supported range. No `uv python install`,
no deadsnakes, no PPA. **[verified]**

```bash
bench init frappe-bench --frappe-branch version-15 --python /usr/bin/python3.12
```

The contrast matters when choosing an OS: **Frappe v16 declares
`requires-python = ">=3.14,<3.15"` — a hard pin, not a floor** — which is why v16 pushes you
to Ubuntu 26.04 or a `uv`-managed interpreter. See
[v16/install-ubuntu-2604.md](../v16/install-ubuntu-2604.md).

**Practical consequence:** 24.04 + v15 is a clean, low-friction pairing. Do not combine a
v15→v16 port with an infrastructure migration — stand the platform up on v15, prove the
operational story (backups, restore, monitoring, exposure), then port against staging.

---

## 5. MariaDB 10.11 — and the config that stops it starting

24.04 ships **MariaDB 10.11.14**, comfortably above v15's 10.6.6+ requirement. No external
repo needed. **[verified]**

`install-and-gotchas.md` §2.2 already warns that the classic Frappe `my.cnf` breaks modern
MariaDB. Restating because it is fatal and still widely copy-pasted: **`innodb-file-format`
and `innodb-large-prefix` were removed in MariaDB 10.3** and mysqld **refuses to start** with
them present.

A working `/etc/mysql/mariadb.conf.d/99-frappe.cnf` for 10.11:

```ini
[mysqld]
bind-address                   = 127.0.0.1

# MUST be in place BEFORE the first `bench new-site`
character-set-client-handshake = FALSE
character-set-server           = utf8mb4
collation-server               = utf8mb4_unicode_ci

innodb_file_per_table          = 1
innodb_flush_log_at_trx_commit = 1
max_allowed_packet             = 256M

[mysql]
default-character-set = utf8mb4
[client]
default-character-set = utf8mb4
```

**Root must use password auth, not `unix_socket`**, or `bench new-site` cannot create
databases as the `frappe` user: **[verified]**

```bash
sudo mariadb -e "ALTER USER 'root'@'localhost' IDENTIFIED VIA mysql_native_password USING PASSWORD('…'); FLUSH PRIVILEGES;"
```

Expect a harmless startup warning: *"MariaDB version 10.11.x is more than 10.8 which is not
yet tested with Frappe Framework"*. It is a warning, not a failure — do not chase it.

---

## 6. `tzdata-legacy` — the trap the existing doc predicted

`install-and-gotchas.md` correctly flags this as 24.04-only. Confirming the shape: 24.04
moved deprecated tz aliases (`Asia/Calcutta`, `Asia/Katmandu`) into a separate package, so a
site configured with a legacy alias raises `ZoneInfoNotFoundError` and the scheduler dies.

```bash
sudo apt-get install -y tzdata-legacy
./env/bin/pip install tzdata
bench restart
```

**Avoid it entirely** by using the canonical name — `Asia/Kolkata`, not `Asia/Calcutta` — in
both the OS and System Settings:

```bash
sudo timedatectl set-timezone Asia/Kolkata
```

Set the OS timezone **before** creating sites; every log line, scheduled job and backup
timestamp depends on it, and a box left on UTC makes incident timelines needlessly hard.

### 6.1 Hit for real — and it is not v15-only, nor scheduler-only

Reproduced on 2026-09-01, on **v16 / Ubuntu 26.04**, during the **setup wizard**. Two
corrections to the framing above: **[verified]**

**It is 24.04 *and later*, not "24.04 only."** 26.04 inherits the same `tzdata` /
`tzdata-legacy` split. A fresh v16 box is equally exposed.

**The setup wizard is where most people meet it**, not the scheduler. Frappe's country
selector offers the **deprecated alias** `Asia/Calcutta` for India, so choosing India on a
clean install submits a key `zoneinfo` cannot resolve:

```
"timezone":"Asia/Calcutta"
...
zoneinfo._common.ZoneInfoNotFoundError: 'No time zone found with key Asia/Calcutta'
ModuleNotFoundError: No module named 'tzdata'
```

**The failure cascades in a way that hides it.** `System Settings.save()` raises, then
`frappe.log_error()` raises *for the same reason* while trying to record it — because
`log_error` → `insert()` → `set_user_and_timestamp()` → `now()` → `ZoneInfo(...)`. So
nothing lands in the Error Log and the user gets a raw traceback:

```
File "apps/frappe/frappe/desk/page/setup_wizard/setup_wizard.py", line 137, in process_setup_stages
    frappe.log_error(title=f"Setup failed: {message}")
  ...
  File "apps/frappe/frappe/utils/data.py", line 373, in now_datetime
    return datetime.datetime.now(ZoneInfo(get_system_timezone())).replace(tzinfo=None)
```

**It is latent on any 24.04+ box until someone runs a wizard.** The four v15 demo sites on
the reference host had been serving traffic for hours with `Asia/Calcutta` unresolvable —
nothing had asked for a timezone yet. Checked directly:

```console
$ ./env/bin/python -c "from zoneinfo import ZoneInfo; ZoneInfo('Asia/Calcutta')"
zoneinfo._common.ZoneInfoNotFoundError: 'No time zone found with key Asia/Calcutta'
```

**So patch it at build time on every 24.04+ box, before anyone opens a wizard:**

```bash
sudo apt-get install -y tzdata-legacy
./env/bin/pip install tzdata          # Python-level fallback
bench restart
```

Verify against the exact failing path rather than just the import:

```bash
bench --site <site> console
>>> import frappe
>>> frappe.db.set_single_value("System Settings","time_zone","Asia/Calcutta"); frappe.db.commit()
>>> from frappe.utils.data import now_datetime; now_datetime()
datetime.datetime(2026, 9, 1, 9, 36, 37, 750837)      # resolves — fix confirmed
```

The wizard fails **cleanly** — `setup_complete` stays `0`, zero companies are created, and
`time_zone` is left `NULL`, so simply re-run it after patching. No cleanup needed.

---

## 7. Multi-tenant tuning — `table_open_cache` on one bench

Not OS-specific, but it surfaces on any multi-site bench and is absent from this repo.

A single ERPNext v15 site is roughly **744 InnoDB tables**. **[verified — measured]** With
`innodb_file_per_table` on (correct, and the default), **four sites is ~3,000 tables** —
permanently above MariaDB's default `table_open_cache` of 2000, so the cache thrashes.

```ini
[mysqld]
table_open_cache      = 4000
table_definition_cache = 4000
```

Raise the file-descriptor limit to match, or the setting is silently capped:

```bash
sudo mkdir -p /etc/systemd/system/mariadb.service.d
printf '[Service]\nLimitNOFILE=65535\n' | sudo tee /etc/systemd/system/mariadb.service.d/limits.conf
sudo systemctl daemon-reload && sudo systemctl restart mariadb
```

### Buffer pool — ignore "70–80% of RAM"

That figure assumes a **dedicated database server**. On a unified box, gunicorn and the RQ
workers need their share, and an oversized pool gets workers OOM-killed under load.

Measured reality: an ERPNext v15 site's tablespace is around **118 MB**; four sites together
were **well under 1 GB**. Sizing the pool to the *data* rather than to a percentage of RAM:
**8 G on a 32 GB box** left ~14.5 GB steady-state usage with plenty of headroom.
**[verified]**

### Worker counts

```bash
bench config set-common-config -c gunicorn_workers 5
bench config set-common-config -c background_workers 2
```

Two things people get wrong:

- **`bench set-config -g` writes a string.** It only coerces `true`/`false`/`0`/`1`, so
  `-g gunicorn_workers 5` stores `"5"` and `bench setup supervisor` then does arithmetic on
  it and **crashes**. Use `bench config set-common-config -c`, which runs
  `ast.literal_eval`. **[verified]**
- **`background_workers: 2` yields 4 processes.** v15 enables `multi_queue_consumption`,
  rendering **two** worker programs (short + long, no default worker), each with
  `numprocs = background_workers`. Observed exactly: **[verified]**

```
frappe-bench-workers:frappe-bench-frappe-short-worker-0   RUNNING
frappe-bench-workers:frappe-bench-frappe-short-worker-1   RUNNING
frappe-bench-workers:frappe-bench-frappe-long-worker-0    RUNNING
frappe-bench-workers:frappe-bench-frappe-long-worker-1    RUNNING
```

`gunicorn_workers = 2×cpu+1` is also wrong on a small box: ERPNext requests are
**database-bound**, so 9 workers at ~350 MB each on 4 threads is idle Python competing for
CPU. **5 on a 4-thread machine** was the right call.

---

## 8. Post-`setup production` gates

Four checks that catch the failure modes above immediately rather than at demo time:

```bash
# 1. supervisor config actually symlinked
ls -l /etc/supervisor/conf.d/           # want frappe-bench.conf -> <bench>/config/supervisor.conf

# 2. processes running — and socketio not in BACKOFF (§2)
sudo supervisorctl status

# 3. nginx knows every site
grep -c '<your-site-pattern>' config/nginx.conf
sudo nginx -t

# 4. each site actually answers
for s in demo1 demo2 demo3 demo4; do
  echo "$s: $(curl -s -H "Host: ${s}.example.com" http://127.0.0.1/api/method/ping)"
done

# 5. and the assets are readable (§1) — the check people skip
while IFS= read -r a; do
  printf '%s %s\n' "$(curl -s -o /dev/null -w '%{http_code}' -H 'Host: demo1.example.com' "http://127.0.0.1$a")" "$a"
done < <(curl -s -H 'Host: demo1.example.com' http://127.0.0.1/login \
         | grep -oE '/assets/[^"'"'"' ]+\.(css|js)' | sort -u) | grep -v '^200' || echo "assets OK"
```

An empty `supervisorctl status` means `restart_supervisor_on_update` was not set **before**
`setup production` — a 502 with no obvious cause. Set both of these first:

```bash
bench config dns_multitenant on
bench config restart_supervisor_on_update on
```

---

## See also

- [install-and-gotchas.md](install-and-gotchas.md) — the v15 base document (22.04-centric)
- [builds/2026-09-01-…](../builds/2026-09-01-bare-metal-four-site-cloudflare-tunnel.md) — where all of the above was found
- [operations/cloudflare-tunnel.md](../operations/cloudflare-tunnel.md) — publishing without inbound ports
- [operations/ssl-nginx-and-production.md](../operations/ssl-nginx-and-production.md) — `log_format`, certbot, nginx ownership
