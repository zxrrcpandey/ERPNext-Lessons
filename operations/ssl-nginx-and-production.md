# nginx, supervisor & SSL in production

How bench generates the production web/process config, why it silently produces the
wrong thing, and how to put Let's Encrypt on top without the next `bench setup nginx`
throwing it away.

**Applies to:** v15 and v16 unless a lesson is tagged otherwise. The mechanics live in
**bench**, not frappe, so they are near-identical across both — the source below was read
from bench `5.29.0` on `~/frappe-bench-v15`; the live build record is bench `5.31.0` on
Ubuntu 26.04.

**Evidence confidence:**

| Tag | Meaning |
|---|---|
| **[V]** | Verified in this session: read verbatim from bench/frappe source on disk, or executed |
| **[B]** | From a real build record (`SERVER_KANTISHIVA.md` = v16 Kantishiva 2026-08-17; `DEPLOYMENT_LESSONS.md` = v15 RD School Betul) |
| **[U]** | Unverified pattern — sound, but not executed on a Trustbit box |

Live result this document describes: **https://kantishiva.trustbit.cloud**, ERPNext v16,
HTTP 301 → HTTPS, Let's Encrypt cert valid to 2026-11-15, auto-renew with a deploy hook
that reloads nginx, all 7 supervisor processes RUNNING **[B]**.

---

## 0. What bench actually generates

Know these five paths before you debug anything **[V]**
(`bench/config/production_setup.py::setup_production`, `bench/utils/__init__.py::get_bench_name`):

| Path | Owner | Notes |
|---|---|---|
| `<bench>/config/nginx.conf` | bench, regenerated | the real file |
| `/etc/nginx/conf.d/<bench_name>.conf` | **symlink** → the above | `<bench_name>` is `basename` of the bench dir, i.e. `frappe-bench.conf` |
| `<bench>/config/supervisor.conf` | bench, regenerated | the real file |
| `/etc/supervisor/conf.d/<bench_name>.conf` | **symlink** → the above | see §5 — this one is *not always created* |
| `/etc/sudoers.d/frappe` | written by `bench setup production` | see §1.3 |

**The single most important consequence:** `/etc/nginx/conf.d/frappe-bench.conf` is a
symlink into the bench. Anything that rewrites it — certbot's installer mode, a hand
edit — is destroyed the next time `bench setup nginx` runs **[B]**. All persistent
configuration must go through **site_config.json / common_site_config.json**, which are
bench's inputs, never through the generated output.

---

## 1. `bench setup production` vs the individual commands

### 1.1 What `setup production` does

```bash
cd /home/frappe/frappe-bench
bench setup production frappe --yes
```

`<user>` is a positional argument, `--yes` the only option **[V]**
(`bench/commands/setup.py:102-107`). In order it **[V]**:

1. `setup_production_prerequisites()` — installs ansible / fail2ban / nginx / supervisor
   if missing (see §1.2, this is where it dies);
2. errors out if both `restart_supervisor_on_update` and `restart_systemd_on_update` are
   set: `You cannot use supervisor and systemd at the same time. Modify your common_site_config accordingly.`;
3. `check_supervisord_config(user)` — forces `[unix_http_server] chmod=0760`,
   `chown=<user>:<user>` into `supervisord.conf`;
4. `generate_supervisor_config()` → writes `<bench>/config/supervisor.conf`;
5. `make_nginx_conf()` → writes `<bench>/config/nginx.conf`;
6. `fix_prod_setup_perms()` — chowns **only** `logs/*` and `config/*` to the frappe user;
7. `remove_default_nginx_configs()` — deletes `/etc/nginx/conf.d/default.conf` and
   `/etc/nginx/sites-enabled/default`;
8. symlinks the two configs into `/etc/…` (with the caveat in §5);
9. reloads supervisor and nginx.

### 1.2 It aborts on `bench setup role fail2ban` — v15 + v16

**Symptom [B]:** on the Kantishiva Ubuntu 26.04 build, `bench setup production` aborted
at the fail2ban prerequisite.

**Root cause [V]** — `bench/config/production_setup.py:23-31`:

```python
def setup_production_prerequisites():
    if not which("ansible"):
        exec_cmd(f"sudo {sys.executable} -m pip install ansible")
    if not which("fail2ban-client"):
        exec_cmd("bench setup role fail2ban")
    ...
```

`bench setup role` runs an **Ansible playbook**. On a modern Ubuntu the `pip install
ansible` line fails first (PEP 668 — `/usr/lib/python3.14/EXTERNALLY-MANAGED` exists
**[B]**), so ansible is never installed, so the playbook cannot run.

**Fix — install the prerequisites yourself with apt and run the two generators
individually [B]:**

```bash
apt-get install -y nginx supervisor fail2ban
systemctl enable --now nginx supervisor fail2ban

cd /home/frappe/frappe-bench
bench setup supervisor --yes
bench setup nginx --yes
```

Then do steps 7–9 above by hand (§5 for the supervisor symlink, §7 for nginx).
On Kantishiva fail2ban was installed via apt with an sshd jail **[B]**.

**Note the side effect:** taking this path means you skip `remove_default_nginx_configs()`.
Delete the stock vhost yourself or it wins the `default_server` race and you get the
nginx welcome page:

```bash
rm -f /etc/nginx/sites-enabled/default /etc/nginx/conf.d/default.conf
```

### 1.3 `/etc/sudoers.d/frappe` — check it immediately

`bench setup production` writes a sudoers drop-in. A malformed sudoers file makes `sudo`
unusable **for every account on the box**. Run the syntax check from a shell you still
have open:

```bash
visudo -c            # must print "... parsed OK"
sudo -l -U frappe | tail -20
```

If it errors, delete `/etc/sudoers.d/frappe` from that still-open root session. Keep a
second SSH session open while doing production setup, full stop.

### 1.4 `--yes` is mandatory on a non-TTY SSH session — v15 + v16

**Symptom [B]:** `bench setup production|nginx|supervisor` "block on an interactive
overwrite prompt otherwise, which hangs a non-TTY SSH session."

**Root cause [V]** — the generators guard an existing file with `click.confirm`:

```python
# bench/config/nginx.py:20-24
if not yes and os.path.exists(conf_path):
    if not click.confirm("nginx.conf already exists and this will overwrite it. Do you want to continue?"):
        return
```
```python
# bench/config/supervisor.py
if not yes and os.path.exists(conf_path):
    click.confirm("supervisor.conf already exists and this will overwrite it. Do you want to continue?",
                  abort=True)
```

Both prompt strings are greppable. Note the two behave **differently** on "no": nginx
returns quietly (leaving the old config, looking like success), supervisor aborts.

**Rule: always pass `--yes`.** **[B]** — "Always pass `--yes` to
`bench setup production|nginx|supervisor`."

```bash
bench setup production frappe --yes
bench setup nginx --yes
bench setup supervisor --yes
```

The same class of bug bites elsewhere: `bench new-site` / `bench restore` without
`--db-root-password` hang on `getpass.getpass("MySQL root password: ")` **[V]**, and
`bench set-ssl-certificate` triggers the nginx overwrite prompt with **no `--yes` flag
available** (§4.3).

### 1.5 systemd instead of supervisor

bench uses supervisor unless `restart_systemd_on_update: true` is already in
`common_site_config.json`; `bench setup systemd` is the opt-in path. Setting both raises
the exception in §1.1 step 2. **[V]** Use supervisor unless a client has a reason not to
— it is what the build records and every troubleshooting note below assume.

---

## 2. `dns_multitenant` must be ON before nginx is generated

**This is the #1 silent SSL failure.** Turn it on first, always.

```bash
cd /home/frappe/frappe-bench
bench config dns_multitenant on
```

**[V]** — `bench/commands/config.py:34-35` writes `{"dns_multitenant": true}` into
`sites/common_site_config.json`.

**Symptom [B]:** "With it off, bench silently ignores `ssl_certificate` /
`ssl_certificate_key` in site_config, so SSL never applies." No error, no warning —
`nginx -t` passes, the site serves happily on plain HTTP.

**Root cause [V]** — `bench/config/nginx.py::prepare_sites`, lines 133-168. The
`ssl_certificate` keys are only ever *read* inside the `if dns_multitenant:` branch:

```python
for site in sites_configs:
    if dns_multitenant:
        ...
        elif site.get("ssl_certificate") and site.get("ssl_certificate_key"):
            sites["that_use_ssl"].append(site)
        else:
            sites["that_use_dns"].append(site_name)
    else:
        # port-based: assigns site["port"] = 80, 8001, 8002 …
        sites["that_use_port"].append(site)
```

With it off, every site lands in `that_use_port` and the template renders a plain
`listen <port>;` block with `server_name <site name>` — the certificate paths are never
looked at **[V]** (`bench/config/templates/nginx.conf:206-234`).

**Diagnostics.** With `dns_multitenant` off, `bench setup nginx` prints a port table
**[V]**:

```
Port configuration list:

Site kantishiva.trustbit.cloud assigned port: 80
```

If you see that block, SSL is not going to work. And with two sites sharing a port it
raises `Port conflicts found:` **[V]**.

`bench setup lets-encrypt` refuses outright **[V]**:

```
You cannot setup SSL without DNS Multitenancy
```

**Order that works:**

```bash
bench config dns_multitenant on     # 1. FIRST
bench setup nginx --yes             # 2. then generate
nginx -t && systemctl reload nginx  # 3. then reload
```

If you turned it on after generating, just regenerate — see §6, it is safe.

---

## 3. SSL with certbot

### 3.1 The rules

From the Kantishiva build record **[B]**, verbatim:

> **SSL: use `certbot certonly --nginx`, never `certbot --nginx`.**
> `/etc/nginx/conf.d/frappe-bench.conf` is a *symlink* to the bench-generated config;
> certbot's installer mode rewrites it and the next `bench setup nginx` blows it away.
> `certonly` doesn't touch server blocks — then register paths via
> `bench setup add-domain` / `bench set-ssl-certificate` so bench owns the config.
> Skip `bench setup lets-encrypt` entirely (unreliable by construction).
> Use **apt** certbot + python3-certbot-nginx; never mix with the snap.

**Correction to that quote:** do **not** use `bench setup add-domain` when the site name is
the domain (§4), and do **not** use `bench set-ssl-certificate` at all (§4.3 — no `--yes`).
The working sequence is `bench --site X set-config …` + `bench setup nginx --yes` (§3.3).

Three separate rules, each for its own reason:

| Rule | Why |
|---|---|
| `certonly`, not installer mode | installer mode edits server blocks; bench regenerates them; your SSL vanishes on the next `bench setup nginx` |
| `--nginx` **authenticator** (not `--standalone`) | `certonly --nginx` uses the plugin only to solve HTTP-01 — it inserts a temporary challenge block, reloads nginx, then reverts, so **no sustained downtime** (unlike `--standalone`, which needs nginx stopped at issuance *and at every renewal*) and no persistent server-block edit. It is not a literal no-op on the file: **if certbot is interrupted mid-issuance, run `bench setup nginx --yes` before trusting the generated file** — it is a symlink into the bench (§0) |
| apt, not snap | mixing the two leaves two certbots with two `/etc/letsencrypt` views and renewals that silently stop |

### 3.2 Issue the certificate

```bash
apt-get install -y certbot python3-certbot-nginx

certbot certonly --nginx \
  -d kantishiva.trustbit.cloud \
  --non-interactive --agree-tos --no-eff-email \
  -m ra.pandey008@gmail.com

ls -l /etc/letsencrypt/live/kantishiva.trustbit.cloud/
```

Preconditions, in this order — get them wrong and you burn Let's Encrypt attempts
(**5 duplicate certificates per domain per week**):

```bash
dig +short A kantishiva.trustbit.cloud    # must equal…
curl -s -4 https://ifconfig.me; echo      # …this
ufw status verbose                        # 80/tcp must be ALLOW
curl -sI http://kantishiva.trustbit.cloud/ | head -1   # nginx must already answer on :80
```

If issuance fails, **fix the cause, do not retry in a loop.**

**What `certonly --nginx` does to your nginx config, precisely:** it uses the plugin only
to solve the HTTP-01 challenge — inserting a temporary challenge server/location block into
the nginx configuration it manages, reloading nginx, then reverting it. No sustained
downtime, and no persistent server-block edit. But because
`/etc/nginx/conf.d/frappe-bench.conf` is a **symlink into the bench** (§0), an *interrupted*
`certonly --nginx` can leave that generated file modified. If certbot is killed or errors
out mid-issuance, run `bench setup nginx --yes` (plus `nginx -t`) before trusting the file.

### 3.3 Wire the paths into site_config, not into nginx

`certonly` deliberately reverted whatever it touched, so nothing is using the cert
yet. Register the paths where bench will read them:

```bash
cd /home/frappe/frappe-bench
bench --site kantishiva.trustbit.cloud set-config ssl_certificate     /etc/letsencrypt/live/kantishiva.trustbit.cloud/fullchain.pem
bench --site kantishiva.trustbit.cloud set-config ssl_certificate_key /etc/letsencrypt/live/kantishiva.trustbit.cloud/privkey.pem
bench --site kantishiva.trustbit.cloud set-config host_name           https://kantishiva.trustbit.cloud

bench setup nginx --yes
nginx -t && systemctl reload nginx
```

`bench --site X set-config <key> <value>` is frappe's command and only edits
`site_config.json`. `bench set-ssl-certificate <site> <path>` is bench's equivalent and
*also* regenerates nginx immediately — **do not use it**: it has no `--yes` and hangs a
non-TTY session on the overwrite prompt (§4.3).

**Why top-level keys and not `domains`:** see §4.

Confirm what bench emitted:

```bash
grep -nE 'ssl_certificate|listen 443|return 301|server_name' /etc/nginx/conf.d/frappe-bench.conf
```

You are looking for exactly one `listen 443 ssl;` block and exactly one `listen 80;`
block containing `return 301 https://$host$request_uri;` **[V]**
(`bench/config/templates/nginx.conf:14-17, 40-41, 166-177`).

### 3.4 Verify

```bash
curl -sI http://kantishiva.trustbit.cloud/  | head -3     # 301 → https
curl -sI https://kantishiva.trustbit.cloud/ | head -3     # 200
echo | openssl s_client -servername kantishiva.trustbit.cloud \
     -connect kantishiva.trustbit.cloud:443 2>/dev/null \
     | openssl x509 -noout -subject -issuer -dates
certbot certificates
certbot renew --dry-run
```

### 3.5 Renewal + the deploy hook

The apt certbot package ships a systemd timer (`certbot.timer`) that runs `certbot renew`
unattended **[U]** — confirm on the box with the `systemctl list-timers` line below rather
than assuming. Renewal replaces the files under `/etc/letsencrypt/live/…` but **nginx
keeps the old certificate in memory until it is reloaded** — so without a hook, the site
starts serving an expired cert ~90 days after install, on a day nobody is looking.

Kantishiva has "auto-renew scheduled, with a deploy hook that reloads nginx after
renewal" **[B]**. The hook goes in certbot's deploy-hook directory, which runs only when
a certificate was actually renewed:

```bash
cat > /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh <<'EOF'
#!/bin/sh
systemctl reload nginx
EOF
chmod +x /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh

systemctl list-timers certbot.timer --no-pager
certbot renew --dry-run          # must end with "all simulated renewals succeeded"
```

**[U]** on the exact script bytes — the build record states a deploy hook exists and
reloads nginx, but did not record the file. `/etc/letsencrypt/renewal-hooks/deploy/` is
certbot's documented convention.

Because the paths in `site_config.json` point at
`/etc/letsencrypt/live/<domain>/fullchain.pem` — a symlink certbot repoints on renewal —
**nothing in bench needs to change at renewal time**. That is the whole point of §3.3.

### 3.6 Why not `bench setup lets-encrypt`

"Skip `bench setup lets-encrypt` entirely (unreliable by construction)" **[B]**. Reading
`bench/config/lets_encrypt.py` **[V]** shows why:

* `run_certbot_and_setup_ssl()` opens with `service("nginx", "stop")` — an outage at
  issuance, and it does not restart nginx on some failure paths;
* with `--custom-domain` it writes the cert into the site's `domains` list, i.e. straight
  into the trap in §4;
* `setup_crontab()` installs a **root** cron
  `certbot renew -a nginx --post-hook "systemctl reload nginx"` at `0 0 */1 * *`, with
  the comment `Renew lets-encrypt every month` — a daily schedule with a monthly comment,
  which will confuse whoever reads the crontab next, and which duplicates apt certbot's
  own timer.

The wildcard path (`bench setup wildcard-ssl`) uses
`certbot certonly --manual --preferred-challenges=dns` **[V]** — genuinely interactive,
so unusable from a script.

### 3.7 Prior art from v15

RD School's `rdps.trustbit.cloud` used `certbot --nginx` (installer mode) **[B]**, with
the standing warning that "running `bench setup nginx` would drop SSL — re-run certbot if
that ever happens." That is the failure mode this whole section exists to eliminate.
**Do not copy the RD School pattern into new builds** — use §3.2/§3.3.

---

## 4. The duplicate `:80` server_name trap

**Applies when the site name IS the domain** — which for Trustbit deploys it usually is
(`kantishiva.trustbit.cloud`, `rdps.trustbit.cloud`, `mandi.trustbit.in`).

**Symptom [B]:** "`bench setup add-domain` registered the domain as an *alternate*,
producing **two `:80` blocks with the same server_name** — nginx uses the first, so the
HTTPS redirect was dead." The HTTPS site works; `http://` just never redirects.

### 4.1 The mechanism, exactly

`bench setup add-domain <domain> --site <site> --ssl-certificate … --ssl-certificate-key …`
appends to the site's `domains` list **[V]** (`bench/config/site_config.py:66-81`):

```json
{
  "domains": [
    {"domain": "kantishiva.trustbit.cloud",
     "ssl_certificate": "/etc/letsencrypt/live/kantishiva.trustbit.cloud/fullchain.pem",
     "ssl_certificate_key": "/etc/letsencrypt/live/kantishiva.trustbit.cloud/privkey.pem"}
  ]
}
```

Then `get_sites_with_config()` returns **two** entries for one site **[V]**
(`bench/config/nginx.py:203-243`) — the site itself, plus one per `domains` member:

| Entry | `ssl_certificate`? | Bucket | Rendered as |
|---|---|---|---|
| the site (`name = kantishiva.trustbit.cloud`, top-level config has no cert keys) | no | `that_use_dns` | plain **`listen 80;`** block, `server_name kantishiva.trustbit.cloud` |
| the `domains` member (`domain = kantishiva.trustbit.cloud`, has cert keys) | yes | `that_use_ssl` | `listen 443 ssl;` block **+ a second `listen 80;` block** with the same `server_name`, containing `return 301 https://…` |

The template emits `that_use_dns` **before** `that_use_ssl` **[V]**
(`templates/nginx.conf:206` then `:222`). nginx matches the **first** server block with
a matching `server_name` on that listen address — so the plain block wins and the redirect
block is dead code.

`nginx -t` still passes; nginx logs a warning of the form
`conflicting server name "<domain>" on 0.0.0.0:80, ignored` **[U]** (standard nginx
behaviour; not captured verbatim in the build record). This is a **warning, not an
error** — which is exactly why it gets missed.

### 4.2 The fix

**When the site name is the domain, do not use `domains` at all.** Put the cert keys at
the top level of `site_config.json` and delete `domains` **[B]**:

```bash
cd /home/frappe/frappe-bench
SITE=kantishiva.trustbit.cloud

bench --site $SITE set-config ssl_certificate     /etc/letsencrypt/live/$SITE/fullchain.pem
bench --site $SITE set-config ssl_certificate_key /etc/letsencrypt/live/$SITE/privkey.pem

# remove the domains entry if add-domain already created it
bench setup remove-domain $SITE --site $SITE
# or edit sites/$SITE/site_config.json and delete the "domains" key outright

bench setup nginx --yes
nginx -t && systemctl reload nginx
```

Now the site itself matches the `elif site.get("ssl_certificate") and
site.get("ssl_certificate_key")` branch, lands in `that_use_ssl`, `that_use_dns` is empty,
and you get **one** `:443` block and **one** `:80` redirect block **[V]**.

Confirm by counting:

```bash
grep -c 'listen 80;' /etc/nginx/conf.d/frappe-bench.conf     # must be 1
grep -n 'return 301' /etc/nginx/conf.d/frappe-bench.conf     # must be inside that one
```

`domains` is the right tool only when the site is served under a name **different** from
the site folder name, or under several names.

### 4.3 `bench set-ssl-certificate` has no `--yes`

`bench set-ssl-certificate <site> <path>` and `bench set-ssl-key` call
`set_site_config_nginx_property()`, which ends in `make_nginx_conf(bench_path=bench_path)`
— with `yes` defaulting to `False` **[V]** (`bench/config/site_config.py`). On a
non-interactive SSH session that hits the overwrite prompt from §1.4 with no flag to
suppress it.

Use `bench --site X set-config …` (frappe's command, config only, no regeneration)
followed by an explicit `bench setup nginx --yes`. That is the sequence in §3.3.

---

## 5. The supervisor conf symlink is not always created

**Symptom [B]:** "bench's supervisor config is not auto-linked". Nothing starts;
`supervisorctl status` shows no frappe programs at all.

**Root cause [V]** — in `setup_production()` the supervisor symlink is inside a
conditional, while the nginx symlink is not:

```python
# bench/config/production_setup.py:63-77
if conf.get("restart_supervisor_on_update"):          # <-- gate
    supervisor_conf = os.path.join(get_supervisor_confdir(), f"{bench_name}.conf")
    if not os.path.islink(supervisor_conf):
        os.symlink(os.path.abspath(os.path.join(bench_path, "config", "supervisor.conf")),
                   supervisor_conf)

if not os.path.islink(nginx_conf):                    # <-- no gate
    os.symlink(...)
```

And `bench setup supervisor` on its own **never** symlinks — `generate_supervisor_config()`
only writes `<bench>/config/supervisor.conf` **[V]**. So if you took the §1.2 path, or if
`restart_supervisor_on_update` is not set in `common_site_config.json`, you must link it
yourself.

**Fix [B]:**

```bash
ln -sfn /home/frappe/frappe-bench/config/supervisor.conf \
        /etc/supervisor/conf.d/frappe-bench.conf

supervisorctl reread
supervisorctl update
sleep 5
supervisorctl status
```

bench looks for the conf dir in this order **[V]**: `/etc/supervisor/conf.d`,
`/etc/supervisor.d/`, `/etc/supervisord/conf.d`, `/etc/supervisord.d`. On Ubuntu it is
the first.

### 5.1 What "everything running" looks like

Kantishiva: **7 processes — 2 redis, web, socketio, 2 workers, scheduler** **[B]**.
Program names, from the template **[V]**:

```
frappe-bench-frappe-web            gunicorn -b 127.0.0.1:<webserver_port> … frappe.app:application --preload
frappe-bench-node-socketio         node <bench>/apps/frappe/socketio.js
frappe-bench-frappe-schedule       bench schedule
frappe-bench-frappe-short-worker   bench worker --queue short[,default]
frappe-bench-frappe-long-worker    bench worker --queue long[,default,short]
frappe-bench-redis-cache           redis-server config/redis_cache.conf
frappe-bench-redis-queue           redis-server config/redis_queue.conf
```
groups: `frappe-bench-web`, `frappe-bench-workers`, `frappe-bench-redis`.

Two counts are normal: with `multi_queue_consumption` enabled there is **no**
`-frappe-default-worker` (short and long absorb the default queue) → 2 workers; without
it there are 3 **[V]** (`templates/supervisor.conf:29-45`). Kantishiva's 7 = 2 redis +
web + socketio + **2** workers + scheduler, i.e. the multi-queue shape **[B]**.

**Version difference in the socketio line.** bench 5.29 bakes the absolute node path into
the command (`command=/usr/bin/node <bench>/apps/frappe/socketio.js`) **[V]**. bench 5.31
reportedly emits `command=<bench_cmd> socketio` instead, resolving node at **runtime**
with `shutil.which("node")` inside the supervisor process's environment — that comes from
the v16 runbook research (checked against bench source by the researcher, **not**
re-verified here), so confirm with `grep socketio config/supervisor.conf` on the actual
box. Either way §5.2 applies; the difference is only *when* a missing node bites you.

### 5.2 Node must come from NodeSource, never nvm/fnm — v15 + v16

**Symptom [B]:** "a per-user nvm node makes `bench setup supervisor` silently drop the
socketio program." Realtime features (notifications, live list refresh, progress bars)
just do not work; nothing errors.

**Root cause [V]** — `generate_supervisor_config()` passes
`"node": which("node") or which("nodejs")` into the template, and **both** the socketio
program block and its membership of the `-web` group are wrapped in `{% if node %}`
(`templates/supervisor.conf:118-133`). If node is not on the PATH of whoever runs
`bench setup supervisor`, the program is omitted with no message.

nvm installs node into `~/.nvm/versions/...` and puts it on the PATH only via an
interactive shell rc. Root running `bench setup supervisor` sees no node.

**Fix:** install node system-wide from NodeSource so `/usr/bin/node` exists for every
user. Kantishiva runs Node **24.19.0** from NodeSource, because Ubuntu 26.04 only ships
22.22 and v16 requires `>=24` **[B]**.

```bash
which node                      # must be /usr/bin/node (or /usr/local/bin/node)
grep -c node-socketio /home/frappe/frappe-bench/config/supervisor.conf   # must be > 0
```

Also **[B]**: yarn must be **1.22.x classic** — "Berry/corepack breaks bench".

---

## 6. nginx won't start: `unknown log format "main"`

**Symptom [B]:**

```
nginx: [emerg] unknown log format "main" in /etc/nginx/conf.d/frappe-bench.conf
```

`systemctl reload nginx` fails, `nginx -t` fails, and if you restarted rather than
reloaded, the site is down.

> **Reconfirmed on bench 5.31.0 / Ubuntu 24.04**, including the asymmetry described below —
> `bench setup production` passed `nginx -t`, and a standalone `bench setup nginx --yes` one
> minute later failed with exactly this error. The click defaults
> (`--logging combined`, `--log_format main`) and `templates/nginx.conf:124` are unchanged
> from the 5.29.0 analysis. Note it fails **safe**: nginx keeps serving the
> already-loaded config, so a failed *reload* is not an outage — but the next reboot would
> be. Field record:
> [builds/2026-09-01-… §8](../builds/2026-09-01-bare-metal-four-site-cloudflare-tunnel.md).
> If you are also adding real-IP directives for a CDN or tunnel, put the `log_format` in
> the **same** `conf.d` prerequisite file — see
> [cloudflare-tunnel.md §7](cloudflare-tunnel.md).

**Root cause [V]** — two facts that only collide when you run `bench setup nginx` as a
standalone command:

1. `bench setup nginx`'s click option `--log_format` **defaults to the string `"main"`**
   (`bench/commands/setup.py:32-38`), and `--logging` defaults to `"combined"`, so the
   template renders `access_log /var/log/nginx/access.log main;`
   (`templates/nginx.conf:124`).
2. Debian/Ubuntu's `/etc/nginx/nginx.conf` **does not define a `log_format` named
   `main`** — that name comes from the nginx.org/RHEL stock config.

Note the asymmetry: `setup_production()` calls `make_nginx_conf(bench_path, yes=yes)`
with no `logging` argument at all, so `template_vars["logging"]` is never set and the
template emits **no `access_log` line** — meaning **`bench setup production` does not hit
this bug and `bench setup nginx` does** **[V]**. Kantishiva hit it precisely because
`setup production` had aborted (§1.2) and the generators were run individually **[B]**.

**Fix [B]** — define the format in a file that sorts *before* `frappe-bench.conf` inside
the `conf.d` include:

```bash
cat > /etc/nginx/conf.d/00-log-format.conf <<'EOF'
log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                '$status $body_bytes_sent "$http_referer" '
                '"$http_user_agent" "$http_x_forwarded_for"';
EOF

nginx -t && systemctl reload nginx
```

The filename and the approach are from the build record **[B]**; the format string is
nginx's stock `main` definition **[U]** — the build record did not capture the file's
contents. Any valid `log_format main …` works.

This survives `bench setup nginx` because it is a **separate file**, not an edit to the
generated one. That is the pattern for every nginx tweak you ever need to add.

**Alternative fix [V]** — tell bench not to name a format:

```bash
bench setup nginx --yes --logging none        # emits no access_log/error_log lines
# or
bench setup nginx --yes --log_format none     # emits `access_log <path> ;` (nginx default format)
```

Prefer the `00-log-format.conf` file: it is independent of anyone remembering the flag on
a later regeneration.

---

## 7. Other nginx-side gotchas

### 7.1 `chmod 755 /home/frappe` or assets 403 — v15 + v16

**Symptom [B]:** "`chmod 755 /home/frappe` or nginx can't serve assets." Pages load but
every `/assets/...` request 404s/403s and the UI is unstyled; the nginx error log shows
`Permission denied`.

**Root cause [V]:** the generated server block sets `root <bench>/sites;`
(`templates/nginx.conf:29`) and serves `/assets` with `try_files $uri =404;` — nginx runs
as `www-data` and must be able to **traverse** `/home/frappe`. Ubuntu's default home
permissions are `750`.

```bash
chmod 755 /home/frappe
namei -l /home/frappe/frappe-bench/sites/assets    # every component needs o+x
```

### 7.2 Root-run bench commands leave root-owned files — v15 + v16

`fix_prod_setup_perms()` chowns **only** `logs/*` and `config/*` **[V]**
(`bench/utils/system.py`). Everything else a root-run bench command touched stays
root-owned — 125 files on Kantishiva, which broke the frappe user at runtime with a
`PermissionError` on `logs/render-template.log` during a PDF render **[B]**.

```bash
chown -R frappe:frappe /home/frappe/frappe-bench     # after ANY root-run bench command
find /home/frappe/frappe-bench ! -user frappe | head -50
```

Full write-up: [backup-restore-and-migration.md §8.2](backup-restore-and-migration.md).

### 7.3 Firewall order

Add the allow rules **before** enabling ufw, and keep a second SSH session open. On
Kantishiva ufw allows 22/80/443 only, "SSH access re-verified *after* enabling" **[B]**.

```bash
ss -ltnp | grep sshd          # learn the REAL ssh port first
ufw allow 22/tcp; ufw allow 80/tcp; ufw allow 443/tcp
ufw default deny incoming; ufw default allow outgoing
ufw --force enable            # --force is what makes it non-interactive
ufw status verbose
```

Port 80 must stay open permanently — for the HTTP→HTTPS redirect and for certbot's
`--nginx` HTTP-01 renewals.

### 7.4 What should be listening

`ss -ltn` after a correct build: `3306` (MariaDB), `11000`/`13000` (bench redis),
`8000` (gunicorn), `9000` (socketio) all bound to **127.0.0.1 only**; `80`/`443` public.
On Kantishiva, MariaDB listening on `127.0.0.1:3306` only, no anonymous users, no remote
root, no `test` database — verified by query rather than assumed **[B]**.

---

## 8. Which commands are safe to re-run later

The question that matters at 11pm: *will this destroy the SSL config?*

| Command | Safe to re-run? | Effect on SSL |
|---|---|---|
| `bench setup nginx --yes` | **Yes** | Regenerates from site_config. SSL survives **iff** `dns_multitenant` is on and the cert paths are in site_config (§2, §3.3). Destroys any hand-edit of the generated file. |
| `bench setup supervisor --yes` | **Yes** | None. Does not touch nginx. Follow with `supervisorctl reread && supervisorctl update`. |
| `bench setup production <user> --yes` | Yes, but | Regenerates **both**, recreates symlinks, rewrites `/etc/sudoers.d/frappe`, and re-runs the fail2ban prerequisite that may abort (§1.2). Re-run `visudo -c` afterwards. |
| `bench --site X set-config ssl_certificate …` | **Yes** | Config only; no regeneration. |
| `bench set-ssl-certificate X <path>` | Yes, but | Regenerates nginx immediately **and prompts** — no `--yes` available (§4.3). |
| `bench config dns_multitenant on` | **Yes** | Config only. Regenerate nginx afterwards. |
| `certbot certonly --nginx -d …` | **Yes** | Reissues the cert in place. Inserts a temporary challenge block, reloads, then reverts — no persistent server-block edit. If it is interrupted, `bench setup nginx --yes` before trusting the file (§3.2). Watch the 5-per-week duplicate limit. |
| `certbot renew` / `certbot renew --dry-run` | **Yes** | Intended to run unattended. |
| `certbot --nginx` (installer mode) | **NO** | Rewrites the bench-symlinked config. Never run this. |
| `bench setup lets-encrypt` | **NO** | Stops nginx, may write into `domains` (§4), installs a duplicate root cron (§3.6). |
| `bench setup add-domain` when site name == domain | **NO** | Creates the duplicate `:80` block (§4). |
| `bench update` | **Careful** | Pulls apps and may regenerate configs. Never on prod without a maintenance window; take a backup first. |
| `bench restart` / `supervisorctl restart all` | **Depends** | On BBPL prod the bench redis group is FATAL/STOPPED (vestigial — the OS redis serves the site), so `restart all` is banned there; restart `web:` and `workers:` explicitly **[B]**. Check `supervisorctl status` before deciding. |

### The re-run drill

After **any** command in the "safe" column, run the same four checks:

```bash
nginx -t
grep -cE 'listen 80;' /etc/nginx/conf.d/frappe-bench.conf     # expect 1
curl -sI http://<domain>/  | head -1                          # expect 301
curl -sI https://<domain>/ | head -1                          # expect 200
supervisorctl status                                          # all RUNNING
```

---

## Quick reference — a correct production bring-up

Run as **root**. Every `bench` line below uses an **absolute path**, because root's PATH
does not include `/home/frappe/.local/bin`: use `/usr/local/bin/bench` if bench was
installed system-wide (pipx with `PIPX_BIN_DIR=/usr/local/bin`), or
`/home/frappe/.local/bin/bench` if it came from `uv tool install` as the frappe user. A
bare `bench` in the second case gives `bench: command not found` halfway through.

```bash
# --- prerequisites (apt, not bench's ansible roles) ---
apt-get install -y nginx supervisor fail2ban certbot python3-certbot-nginx
systemctl enable --now nginx supervisor fail2ban
rm -f /etc/nginx/sites-enabled/default /etc/nginx/conf.d/default.conf

# --- nginx log format, before anything generates a config ---
cat > /etc/nginx/conf.d/00-log-format.conf <<'EOF'
log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                '$status $body_bytes_sent "$http_referer" '
                '"$http_user_agent" "$http_x_forwarded_for"';
EOF

# --- bench config, THEN generation ---   (BENCH=/home/frappe/.local/bin/bench for Option B)
export BENCH=/usr/local/bin/bench
cd /home/frappe/frappe-bench
$BENCH config dns_multitenant on
$BENCH setup supervisor --yes
$BENCH setup nginx --yes

ln -sfn /home/frappe/frappe-bench/config/supervisor.conf /etc/supervisor/conf.d/frappe-bench.conf
supervisorctl reread && supervisorctl update && sleep 5 && supervisorctl status
chmod 755 /home/frappe
chown -R frappe:frappe /home/frappe/frappe-bench
nginx -t && systemctl reload nginx

# --- firewall, then TLS ---
ufw allow 22/tcp; ufw allow 80/tcp; ufw allow 443/tcp
ufw default deny incoming; ufw --force enable

certbot certonly --nginx -d <domain> --non-interactive --agree-tos --no-eff-email -m <email>

$BENCH --site <domain> set-config ssl_certificate     /etc/letsencrypt/live/<domain>/fullchain.pem
$BENCH --site <domain> set-config ssl_certificate_key /etc/letsencrypt/live/<domain>/privkey.pem
$BENCH --site <domain> set-config host_name           https://<domain>
$BENCH setup nginx --yes
chown -R frappe:frappe /home/frappe/frappe-bench    # after ANY root-run bench command
nginx -t && systemctl reload nginx

cat > /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh <<'EOF'
#!/bin/sh
systemctl reload nginx
EOF
chmod +x /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh
certbot renew --dry-run

# --- verify ---
visudo -c
curl -sI http://<domain>/ | head -1     # 301
curl -sI https://<domain>/ | head -1    # 200
curl -s  https://<domain>/api/method/frappe.ping   # {"message":"pong"}
```

Do **not** run `bench setup add-domain` on this site, and do **not** run
`certbot --nginx` without `certonly`.
