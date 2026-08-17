# ERPNext v16 on Ubuntu 26.04 LTS — ordered install runbook

**Applies to: v16 only** (unless a stage is tagged otherwise). Built and verified on
`kantishiva.trustbit.cloud` (Hostinger VPS, 1 vCPU / 3.9 GB / 48 GB, Ubuntu 26.04
"resolute", 2026-08-17). Source of truth: `Trustbit Kantishiva/SERVER_KANTISHIVA.md`,
corroborated against frappe/bench source and frappe v16 CI.

Anything **not** actually run on that box is labelled `[UNVERIFIED]` or
`[SOURCE-VERIFIED, NOT RUN HERE]`. Do not silently promote those to fact.

## Stack this runbook produces

| Component | Version | Note |
|---|---|---|
| Python | 3.14.4 (system `/usr/bin/python3`) | v16 pins `requires-python >=3.14,<3.15` |
| Frappe | 16.31.0 | branch `version-16` |
| ERPNext | 16.32.1 | branch `version-16` |
| MariaDB | 11.8.6 | Ubuntu *main*; v16 CI tests against exactly 11.8 |
| Redis | 8.0.5 | system package for the binaries; bench runs its own instances |
| Node | 24.19.0 | NodeSource; Ubuntu 26.04 ships 22.22, below v16's `engines >=24` |
| yarn | 1.22.22 (classic) | Berry/corepack breaks bench |
| bench CLI | 5.31.0 | `/usr/local/bin/bench` |
| uv | 0.12.5 | **hard requirement** of bench 5.31 |

## Conventions

You are `root` over SSH. Anything that must run as the bench owner is wrapped as
`sudo -u frappe -H bash -lc '...'` with an explicit `export PATH=...` inside — Ubuntu's
`~/.bashrc` returns early for non-interactive shells, so a PATH line written there is
**not** picked up. Export your secrets once before starting:

```bash
export DOMAIN=kantishiva.trustbit.cloud          # the site name AND the vhost name
export LE_EMAIL=you@example.com
export DB_ROOT_PW='...'                          # no single quotes/backslashes inside
export ADMIN_PW='...'
test -n "$DB_ROOT_PW" && test -n "$ADMIN_PW" && echo 'passwords present' || echo 'ABORT'
```

---

## Stage 0 — Preflight

```bash
id -u                                            # must print 0
grep -E '^(VERSION_ID|VERSION_CODENAME)=' /etc/os-release
/usr/bin/python3 --version
nproc; free -m; df -h /
timedatectl set-timezone UTC
apt-get update -qq && apt-get install -y dnsutils curl
dig +short A "$DOMAIN"; curl -s -4 https://ifconfig.me; echo
```

**Verify:** `VERSION_ID="26.04"`, `Python 3.14.x`, and the `dig` answer is byte-identical
to the `ifconfig.me` answer. If DNS does not match, stop now — Let's Encrypt (Stage 13)
will fail and burn rate-limit attempts (5 duplicate certs per domain per week).

### Why Ubuntu 26.04 specifically

Frappe v16's `pyproject.toml` sets `requires-python = ">=3.14,<3.15"`. That is a hard
floor **and** a hard ceiling, so the interpreter is not negotiable:

| Ubuntu | System python3 | Usable for v16 |
|---|---|---|
| 22.04 LTS | 3.10 | No |
| 24.04 LTS | 3.12 | No |
| **26.04 LTS** | **3.14.4** | **Yes — the only Ubuntu shipping it natively** |

On 24.04 you would have to bolt on a deadsnakes/uv-managed 3.14 and then keep the bench
venv, the system python and apt's python in sync forever. 26.04 also happens to ship
MariaDB **11.8.6** in *main*, which is exactly the version v16 CI tests against — so the
distro package is correct and you must **not** add MariaDB's own apt repo (it would give
you 12.x and trip `check_compatible_versions()`).

Do **not** run `uv python install 3.14 --default` even though the official docs show it:
it installs a second CPython and drops a `python3` shim into `~/.local/bin` that shadows
`/usr/bin/python3`. The system 3.14.4 already satisfies the pin.

---

## Stage 1 — Swap, FIRST, before anything memory-hungry

**Do this before installing a single package.** This is not padding.

```bash
swapon --show                                    # must print nothing
test ! -e /swapfile || echo 'ABORT: /swapfile already exists'
findmnt -no FSTYPE /                             # ext4 -> fallocate is fine
fallocate -l 4G /swapfile || dd if=/dev/zero of=/swapfile bs=1M count=4096 status=none
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
grep -q '^/swapfile ' /etc/fstab || echo '/swapfile none swap sw 0 0' >> /etc/fstab
printf 'vm.swappiness=10\nvm.vfs_cache_pressure=50\n' > /etc/sysctl.d/99-frappe.conf
sysctl -p /etc/sysctl.d/99-frappe.conf
```

**Verify:** `swapon --show` lists `/swapfile file 4G`; `free -m` shows Swap ≈ 4096;
exactly one `/swapfile` line in `/etc/fstab` so it survives reboot.

**Danger:** only ever `mkswap` a plain new file. Pointed at a block device
(`/dev/vda1`) it destroys that device's contents.

### Why (this is the lesson, not the command)

`bench build` is the memory peak of the entire install, and on 1 vCPU it is where the
OOM killer takes out the build — or worse, takes out `sshd` or `mysqld` instead. Two
facts from `frappe/build.py` on `version-16` make this unavoidable:

```python
def get_safe_max_old_space_size():
    import psutil
    total_memory = psutil.virtual_memory().total / (1024 * 1024)
    safe_max_old_space_size = max(1024, int(total_memory * 0.75))

def get_node_env():
    return {"NODE_OPTIONS": f"--max_old_space_size={get_safe_max_old_space_size()}"}
```

1. **`psutil.virtual_memory().total` is PHYSICAL RAM. Swap is not counted.** On a
   3911 MB box frappe tells V8 it may grow to `--max_old_space_size=2933` — on a machine
   that also runs MariaDB, Redis and gunicorn. V8 will not GC hard until it approaches
   2933 MB; the kernel OOM killer fires long before that.
2. **Exporting `NODE_OPTIONS` yourself has no effect.** `frappe/commands/__init__.py`
   does `env = dict(environ, **env)` — frappe's computed value is applied *last* and
   overwrites yours. You cannot lower the ceiling from the shell.

So the only levers you actually control are: swap, stopping MariaDB during the build,
and splitting the build per app (`--apps frappe` / `--apps erpnext`).

Also new in v16: `bench build` runs `--run-build-command`, and `erpnext/package.json`
now has `"build": "cd banking && yarn build"` — a **React 19 + Vite 8 + Tailwind 4**
production build that did not exist in v15 (v15's erpnext `package.json` has no
`scripts` block at all). esbuild is a separate Go process and is not constrained by the
heap; Vite/Rollup runs *inside* the Node heap, so the banking build is the step that
actually consumes the 2933 MB.

Symptoms of getting this wrong: a bare `Killed` and exit code 137, or
`FATAL ERROR: Reached heap limit Allocation failed - JavaScript heap out of memory`.

Expect **20–45 minutes** for a full build on 1 vCPU. It is not hung —
`frappe/commands/__init__.py` passes `preexec_fn=set_low_prio`, applying `nice(19)` and
`ionice(IOPRIO_CLASS_IDLE)` on Linux deliberately.

---

## Stage 2 — Base packages and build toolchain

```bash
export DEBIAN_FRONTEND=noninteractive
apt-get update
apt-get -y -o Dpkg::Options::=--force-confdef -o Dpkg::Options::=--force-confold upgrade
apt-get install -y git curl wget ca-certificates gnupg lsb-release \
  software-properties-common build-essential pkg-config \
  python3 python3-dev python3-venv python3-setuptools \
  libssl-dev libffi-dev zlib1g-dev cron unzip tzdata-legacy
apt-get install -y libmariadb-dev

# certbot, python3-certbot-nginx and redis-server all live in *universe*; only MariaDB
# is in main. A minimal/cloud image may ship with universe disabled, and then Stage 4
# and Stage 13b fail with `Unable to locate package` at the two worst moments.
add-apt-repository -y universe && apt-get update

systemctl enable --now cron
```

`tzdata-legacy` is here deliberately: without it the **setup wizard dies at stage 1**
with `ZoneInfoNotFoundError: 'No time zone found with key Asia/Calcutta'` (Stage 15 has
the full diagnosis). It costs nothing now and saves a debugging cycle later.

**Verify:**

```bash
ls -l /usr/include/python3.14/Python.h     # must exist
pkg-config --version                       # 2.5.x (transitional pkg for pkgconf)
pkg-config --exists mariadb && echo 'mariadb.pc FOUND'
gcc --version | head -1
systemctl is-active cron
```

If `apt-get upgrade` pulled a new kernel, **reboot now** — never mid-`bench init`.

### Why the compiler is mandatory: mysqlclient

Frappe v16 added `mysqlclient==2.2.7` as an unconditional runtime dependency (v15 had
only PyMySQL). **mysqlclient publishes Windows wheels plus an sdist and nothing else —
there has never been a manylinux wheel for any 2.2.x release.** It therefore compiles
from C source on every Linux install, always. Every *other* C extension in frappe v16
(pyarrow 25, duckdb 1.4.3, Pillow 12.2, orjson, lxml, cryptography, nh3, …) ships a
cp314 or abi3 manylinux wheel, so nothing else builds and no Rust toolchain is needed.

bench added a matching hard guard — `check_pkg_config()` fires when
`get_current_frappe_version() >= 16`. Missing pieces produce these exact errors:

| Missing | Error |
|---|---|
| pkg-config | `Exception: pkg-config is not installed. Please install it before proceeding.` |
| libmariadb-dev | `Exception: Can not find valid pkg-config name. Specify MYSQLCLIENT_CFLAGS and MYSQLCLIENT_LDFLAGS env vars manually` |
| python3-dev | `fatal error: Python.h: No such file or directory` |
| build-essential | `error: command 'gcc' failed` |

`mysqlclient` is one small C file (`_mysql.c`) — seconds of compile, not a pyarrow-scale
build. Do **not** try to force `mysqlclient==2.2.8` in to "get 3.14 support"; the PR that
added 3.14 to mysqlclient touched only CI workflow files, so 2.2.7's source builds fine,
and forcing 2.2.8 fights frappe's `==` pin and gets reverted by `bench setup requirements`.

---

## Stage 3 — MariaDB 11.8 and the config that must NOT be copied from the old wiki

```bash
export DEBIAN_FRONTEND=noninteractive
apt-get install -y mariadb-server mariadb-client
systemctl enable --now mariadb
mariadb --version
```

### The exact `99-frappe.cnf`

```bash
cat > /etc/mysql/mariadb.conf.d/99-frappe.cnf <<'EOF'
# Frappe/ERPNext v16 required MariaDB settings.
# Mirrors frappe/frappe_docker overrides/compose.mariadb.yaml (image mariadb:11.8),
# which is the exact server version frappe version-16 CI tests against.
#
# DO NOT add innodb-file-format or innodb-large-prefix here: both were REMOVED
# in MariaDB 10.3.1 and mysqld will refuse to start on 11.8.
# innodb-file-per-table has defaulted to ON since 5.6 and is not needed.
# Filename is 99- so it sorts AFTER 50-server.cnf: last definition wins.

[mysqld]
character-set-server = utf8mb4
collation-server     = utf8mb4_unicode_ci
skip-character-set-client-handshake
innodb_buffer_pool_size = 512M

[mysql]
default-character-set = utf8mb4
EOF

systemctl restart mariadb
systemctl is-active mariadb
```

`innodb_buffer_pool_size = 512M` is sizing judgement for a 3.9 GB box, not a Frappe
requirement. Everything above it is.

**Verify:**

```bash
mariadb -e "SHOW VARIABLES WHERE Variable_name IN
 ('version','character_set_server','collation_server','skip_name_resolve');"
ss -ltn | grep 3306          # must be 127.0.0.1:3306, never 0.0.0.0
```

Expect `version 11.8.x`, `character_set_server utf8mb4`,
`collation_server utf8mb4_unicode_ci`, `skip_name_resolve OFF`.

### Why not the classic Frappe `my.cnf` — applies to **both v15 and v16**

Nearly every ERPNext guide still tells you to paste a `my.cnf` containing:

```
innodb-file-format=barracuda
innodb-large-prefix=1
```

Both variables were **removed in MariaDB 10.3.1**. On 11.8 `mysqld` exits at startup:

```
[ERROR] /usr/sbin/mariadbd: unknown variable 'innodb-file-format=barracuda'
```

and the database service does not come back. A bad option is only detected **on
restart**, so it can lie dormant for weeks until a reboot takes the site down. Always
run `systemctl restart mariadb && systemctl is-active mariadb` after touching any
`.cnf`, and read `journalctl -u mariadb -n 50 --no-pager` if it fails.

### Why the filename is `99-`, not `50-`

`/etc/mysql/mariadb.conf.d/` is read in **alphabetical order and the last definition
wins**. Ubuntu ships its own `50-server.cnf`. `50-frappe.cnf` sorts *before* it
(`f` < `s`), so any overlapping setting in `50-server.cnf` silently overrides yours and
you get a config file that looks applied but isn't. `99-frappe.cnf` always wins. Trust
`SHOW VARIABLES`, never the file.

### Why `collation-server` must be stated explicitly (v16 / MariaDB 11.8)

MariaDB 11.8 changed the default collation to `utf8mb4_uca1400_ai_ci` (MDEV-25829).
`skip-character-set-client-handshake` makes the server force `collation_server` onto
every connection — so shipping that flag *without* `collation-server` leaves connections
running as `uca1400_ai_ci` while Frappe's tables are `utf8mb4_unicode_ci`, producing:

```
(1267, "Illegal mix of collations (utf8mb4_uca1400_ai_ci,IMPLICIT) and (utf8mb4_unicode_ci,IMPLICIT) for operation '='")
```

The three settings are a matched set. Also note v16 **removed** the old hard my.cnf
validation from `setup_db.py` — it now only warns on version, so a misconfigured charset
sails silently through `bench new-site`. Nothing will catch this for you.

Do **not** set any `sql_mode` (frappe has zero `sql_mode` references; `ANSI_QUOTES` in
particular breaks Frappe's raw SQL), and do **not** change the default auth plugin to
ed25519/PARSEC — PyMySQL cannot speak them.

### Harden + root auth

`mariadb-secure-installation` has no batch flags, so run the SQL equivalent. Since
MariaDB 10.4 privileges live in `mysql.global_priv`; `mysql.user` is a non-updatable view.

```bash
mariadb <<'SQL'
DELETE FROM mysql.global_priv WHERE User='';
DELETE FROM mysql.global_priv WHERE User='root' AND Host NOT IN ('localhost','127.0.0.1','::1');
DROP DATABASE IF EXISTS test;
DELETE FROM mysql.db WHERE Db='test' OR Db='test\\_%';
FLUSH PRIVILEGES;
SQL

mariadb <<SQL
ALTER USER 'root'@'localhost' IDENTIFIED VIA unix_socket OR mysql_native_password USING PASSWORD('$DB_ROOT_PW');
FLUSH PRIVILEGES;
SQL
```

**Verify — both paths must work before you continue:**

```bash
mariadb -e "SELECT 'socket auth still works' AS check_1;"
MYSQL_PWD="$DB_ROOT_PW" mariadb -u root -h 127.0.0.1 -P 3306 \
  -e "SELECT current_user() AS check_2, 'tcp password auth works' AS ok;"
```

If the TCP one fails, stop — bench cannot create the site database and you will get
`(1045, "Access denied for user 'root'@'localhost'")` from `bench new-site`.

### Why the `unix_socket OR mysql_native_password` form

A plain `ALTER USER ... IDENTIFIED BY '...'` **replaces** the `unix_socket` plugin.
Passwordless `sudo mariadb` stops working and distro maintenance paths that assume
socket auth break. The `VIA ... OR ...` form registers *both* plugins, tried in
declaration order: OS root logging in over the socket authenticates as root with no
password, and bench (running as the `frappe` OS user, where unix_socket fails) falls
through to the password. Both halves were verified on Kantishiva.

Passwords go in on **stdin via heredoc**, not `-e`, so they never appear in `ps`.

Security posture on Ubuntu 26.04, checked by query rather than assumed: MariaDB listens
on `127.0.0.1:3306` only, no anonymous users, no remote root, no `test` database — the
package ships secure by default, so the statements above were effectively no-ops.

---

## Stage 4 — Redis

```bash
apt-get install -y redis-server redis-tools          # universe — enabled in Stage 2
redis-server --version                 # 8.0.5 on 26.04 (universe); bench needs >= 6
systemctl disable --now redis-server   # bench runs its own instances
ss -ltn | grep ':6379' || echo 'port 6379 free (expected)'
```

Bench does not use the distro service: it generates `config/redis_cache.conf` (port
**13000**) and `config/redis_queue.conf` (port **11000**) and runs them under supervisor.
`redis_socketio` is force-synced to equal `redis_cache` — it is a legacy alias, not a
third server. We install the package only for the `redis-server` / `redis-cli` binaries,
which bench's supervisor template invokes by name.

`[UNVERIFIED]` If a future release replaces redis with valkey, bench's `which("redis-server")`
will fail; compatibility symlinks would be needed. Not the case on 26.04.

---

## Stage 5 — Node 24 from NodeSource (never nvm)

```bash
export DEBIAN_FRONTEND=noninteractive
install -d -m 0755 /etc/apt/keyrings
curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key \
  | gpg --dearmor -o /etc/apt/keyrings/nodesource.gpg
chmod a+r /etc/apt/keyrings/nodesource.gpg
echo 'deb [signed-by=/etc/apt/keyrings/nodesource.gpg] https://deb.nodesource.com/node_24.x nodistro main' \
  > /etc/apt/sources.list.d/nodesource.list
apt-get update
apt-get install -y nodejs
npm install -g yarn@1.22.22
```

**Verify:**

```bash
node -v            # v24.x  (24.19.0 on Kantishiva)
yarn -v            # 1.22.x
command -v node    # /usr/bin/node  — a SYSTEM path, not /home/...
sudo which node    # must also resolve
```

### Why not the Ubuntu package

Ubuntu 26.04 ships `nodejs` **22.22.1**, and frappe v16's `package.json` declares
`"engines": {"node": ">=24"}`. bench runs `yarn install --check-files` with no
`--ignore-engines`, and yarn 1.x treats an engines mismatch as fatal:

```
error frappe-framework@: The engine "node" is incompatible with this module. Expected version ">=24". Got "22.22.1"
error Found incompatible module.
```

Frappe's own guard is useless here — `build.py:287` only warns
`if node_version.major < 18`, so it will happily let you start a build on Node 20/22 that
then dies deep inside the bundle with opaque JS syntax errors.

### Why NOT nvm/fnm — the silent failure

bench resolves node with `which("node")` when writing the supervisor config
(`bench/config/supervisor.py`: `"node": which("node") or which("nodejs")`), and the
template wraps the entire socketio program **and its group membership** in `{% if node %}`:

```jinja
{% if node %}
[program:{{ bench_name }}-node-socketio]
...
[group:{{ bench_name }}-web]
programs={{ bench_name }}-frappe-web {%- if node -%} ,{{ bench_name }}-node-socketio {%- endif%}
```

`bench setup production` runs under sudo, whose `secure_path` is
`/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin` — it **cannot see**
`~/.nvm/versions/node/v24.x/bin/node`. Result: the socketio program is silently omitted.
You get a site that loads perfectly with **dead realtime** — no notifications, no
progress bars, no list auto-refresh — and no error anywhere. `/usr/bin/node` from
NodeSource is inside `secure_path` and identical for root, `frappe` and supervisor.

`[SOURCE-VERIFIED, NOT RUN HERE]` bench 5.31 changed the supervisor line from
`command={{ node }} apps/frappe/socketio.js` to `command={{ bench_cmd }} socketio`,
which resolves node via `shutil.which("node")` at **runtime inside the supervisor
process**. That makes a system-wide node even more important, not less. Check after
Stage 10 with `grep -A3 node-socketio /etc/supervisor/conf.d/frappe-bench.conf`.

### Why yarn classic 1.22.x only

- frappe v16's `yarn.lock` header is literally `# yarn lockfile v1`.
- `package.json` has no `packageManager` field; `.yarnrc.yml` does not exist.
- bench runs `yarn install --check-files` — a flag that **does not exist in Yarn 2+**.
- frappe's `esbuild.js` runs `yarn install --frozen-lockfile` — renamed `--immutable` in Berry.

`npm i -g yarn` still gives classic (npm dist-tag `latest: 1.22.22`; Berry ships as
`@yarnpkg/cli`). **Do not run `corepack enable`** — Node 24 still bundles corepack with
shims disabled, and enabling it lets a corepack shim shadow `/usr/bin/yarn` and fail on
the classic flags. Ubuntu's own `yarnpkg` package is 4.1.0 — wrong major, do not use it.

---

## Stage 6 — The `frappe` service user

```bash
useradd -m -s /bin/bash frappe || echo 'user frappe already exists'
usermod -aG sudo frappe
passwd -l frappe                    # no password login; we enter via sudo -u frappe
chmod 755 /home/frappe
id frappe; stat -c '%a %U %n' /home/frappe
```

**Verify:** `stat -c '%a' /home/frappe` prints **755**.

The chmod is not cosmetic — **applies to both v15 and v16**. Modern Ubuntu creates home
directories mode `750`, and nginx (as `www-data`) then cannot traverse into
`/home/frappe/frappe-bench/sites` to serve static assets. Symptom: the site loads but
every CSS/JS request 403s and Desk renders unstyled.

---

## Stage 7 — bench CLI under PEP 668

`/usr/lib/python3.14/EXTERNALLY-MANAGED` exists on 26.04, so `pip install frappe-bench`
as root fails with:

```
error: externally-managed-environment
```

**Never** use `--break-system-packages` and never delete `EXTERNALLY-MANAGED`. On this
box the system `python3` **is** 3.14 and apt, `python3-apt` and `unattended-upgrades`
all depend on it; polluting it can break package management with no easy recovery.

### bench 5.31 requires `uv` — this is the one that stops people dead

bench 5.28+ shells out to a **bare `uv` on PATH** (`uv venv env --seed --python ...`,
`uv pip install -e ...`) and does *not* install it. uv is not packaged in Ubuntu 26.04.
Without it, `bench init` dies at venv creation with:

```
FileNotFoundError: [Errno 2] No such file or directory: 'uv'
```

Two supported ways to satisfy this.

**Option A — pipx, system-wide (what was actually done on Kantishiva):**

```bash
apt-get install -y pipx
export PIPX_HOME=/opt/pipx PIPX_BIN_DIR=/usr/local/bin
pipx install uv                      # MUST be installed separately - see below
pipx install frappe-bench
```

The `PIPX_HOME=/opt/pipx PIPX_BIN_DIR=/usr/local/bin` pair is the point: it puts `bench`
at `/usr/local/bin/bench` so the `frappe` user, root and supervisor all see the same
binary. **pipx alone is a trap**: pipx links only the *requested* package's console
scripts, so bench's bundled `uv~=0.11.6` lands inside
`/opt/pipx/venvs/frappe-bench/bin/uv` and never reaches PATH. Installing `uv` as its own
pipx package is what fixes it. `[SOURCE-VERIFIED]` The alternative escape hatch is
`export BENCH_DISABLE_UV=1`, which makes bench fall back to `python3 -m venv` (and makes
`python3-venv` mandatory).

**Option B — `uv tool install`, as the frappe user (what the official docs and frappe
v16 CI do):**

```bash
sudo -u frappe -H bash -lc 'curl -LsSf https://astral.sh/uv/install.sh | sh'
sudo -u frappe -H bash -lc 'export PATH=$HOME/.local/bin:$PATH; uv tool install frappe-bench --python /usr/bin/python3'
sudo -u frappe -H bash -lc "grep -q '.local/bin' ~/.profile || printf 'export PATH=\"\$HOME/.local/bin:\$PATH\"\n' >> ~/.profile"
```

Here the standalone `uv` you installed to run the command is already at
`~/.local/bin/uv`, so PATH is satisfied by construction. Cost: `bench` lives in
`/home/frappe/.local/bin`, which is **not on root's PATH** — every root-run bench command
then needs its absolute path (`/home/frappe/.local/bin/bench`) or `sudo env "PATH=$PATH" bench ...`.

**Verify (either option):**

```bash
command -v uv && uv --version              # 0.12.5 on Kantishiva
command -v bench && bench --version        # 5.31.0
```

Floor: bench **5.25.0** is where the v16 codepath (`check_pkg_config()`) first appears;
**5.28.0** is where uv became the default. Do not go below 5.28.0; install 5.31.0+.

Do not hand-upgrade setuptools inside the bench tool venv — bench 5.31.0 pins
`setuptools>=71,<82` because it still imports distutils via the setuptools shim
(`bench/utils/bench.py`: `from distutils.version import LooseVersion`, and distutils left
the stdlib in 3.12). If `bench update` ever raises `ModuleNotFoundError` for `distutils`
or `pkg_resources`, that pin has been violated — reinstall bench cleanly.

---

## Stage 8 — `bench init` on version-16

```bash
sudo -u frappe -H bash -lc 'export PATH=$HOME/.local/bin:/usr/local/bin:$PATH; cd $HOME && \
  bench init frappe-bench --frappe-branch version-16 --python /usr/bin/python3 --skip-assets --verbose'

# python-side tz fallback, installed now rather than after the wizard crashes (Stage 15)
/home/frappe/frappe-bench/env/bin/pip install tzdata
chown -R frappe:frappe /home/frappe/frappe-bench
```

**Verify:**

```bash
sudo -u frappe -H bash -lc 'cd $HOME/frappe-bench && git -C apps/frappe rev-parse --abbrev-ref HEAD'
sudo -u frappe -H bash -lc 'cd $HOME/frappe-bench && ./env/bin/python --version && readlink -f env/bin/python'
sudo -u frappe -H bash -lc 'cd $HOME/frappe-bench && ./env/bin/python -c "import MySQLdb, frappe; print(MySQLdb.version_info); print(frappe.__version__)"'
```

Expect branch `version-16`, python `3.14.x` resolving to `/usr/bin/python3.14`, a
`MySQLdb` version tuple (proof the mysqlclient C build worked) and frappe `16.x`.

### Why `--frappe-branch version-16` is mandatory, not cosmetic

`--frappe-branch` is an alias of `--version` (same dest `frappe_branch`), default `None`.
In `bench/utils/system.py`, a `None` branch clones `https://github.com/frappe/frappe.git`
at its **default branch, which is `develop`** (v17-dev). You get an unsupported moving
target, and ERPNext v16 then fails its own dependency check
(`frappe = ">=16.21.0,<17.0.0"`). Nothing warns you at init time — the only tell is
`git rev-parse --abbrev-ref HEAD` printing `develop`.

`--python /usr/bin/python3` is explicit because bench's default is the bare string
`"python3"`, which resolves against whatever PATH happens to be.

`--skip-assets` does **not** skip yarn: `yarn install --check-files` runs unconditionally
for any app with a `package.json`; only `build_assets()` is gated. That is why Node had
to exist in Stage 5. What `--skip-assets` buys you is deferring the memory peak to a
separately restartable step.

---

## Stage 9 — ERPNext + assets

```bash
sudo -u frappe -H bash -lc 'export PATH=$HOME/.local/bin:/usr/local/bin:$PATH; \
  cd $HOME/frappe-bench && bench get-app erpnext --branch version-16 --skip-assets'
```

Then build. Run it alone, with nothing else competing:

```bash
sudo -u frappe -H bash -lc 'cd $HOME/frappe-bench && bench build --production --apps frappe'
sudo -u frappe -H bash -lc 'cd $HOME/frappe-bench && bench build --production --apps erpnext'
```

**If the build is still OOM-killed** (`Killed`, exit 137, or
`FATAL ERROR: Reached heap limit`), free the ~500 MB MariaDB is holding for the duration:

```bash
systemctl stop mariadb    # [SOURCE-VERIFIED, NOT RUN HERE] - optional; bench build needs
                          # no DB. Kantishiva did not need this. Do NOT do it on a box
                          # that is already serving a site.
sudo -u frappe -H bash -lc 'cd $HOME/frappe-bench && bench build --production --apps erpnext'
systemctl start mariadb
```

**Verify:**

```bash
sudo -u frappe -H bash -lc 'cd $HOME/frappe-bench && ls -la sites/assets/assets.json && \
  ls sites/assets/frappe/dist/js | head -5 && ls sites/assets/erpnext/dist/js | head -5'
dmesg -T | grep -i -m5 'killed process'     # should show nothing from this run
git -C /home/frappe/frappe-bench/apps/erpnext rev-parse --abbrev-ref HEAD
```

The build is idempotent and restartable — a failure here costs minutes, not the whole
environment. `--production` sets `minify` and `NODE_ENV=production`; it does **not**
disable sourcemaps (`esbuild.js` hardcodes `sourcemap: true`), so budget the memory and
disk for them.

`[UNVERIFIED]` Whether per-app builds compose `assets.json` correctly was not traced end
to end. If the site later shows 404s on assets, fall back to a single
`bench build --production` with swap active.

---

## Stage 10 — Site creation and install-app

Bench's redis instances are not running yet, and frappe touches the cache during site
creation, so start them temporarily or you get
`Error 111 connecting to 127.0.0.1:13000. Connection refused.`

```bash
sudo -u frappe -H bash -lc 'cd $HOME/frappe-bench && \
  redis-server config/redis_cache.conf --daemonize yes && \
  redis-server config/redis_queue.conf --daemonize yes'
redis-cli -p 13000 ping && redis-cli -p 11000 ping

sudo -u frappe -H bash -lc "export PATH=\$HOME/.local/bin:/usr/local/bin:\$PATH; cd \$HOME/frappe-bench && \
  bench new-site $DOMAIN --db-root-username root --db-root-password '$DB_ROOT_PW' \
  --admin-password '$ADMIN_PW' --install-app erpnext --set-default --verbose"

redis-cli -p 13000 shutdown nosave 2>/dev/null || true
redis-cli -p 11000 shutdown nosave 2>/dev/null || true
```

**Verify:**

```bash
sudo -u frappe -H bash -lc "cd \$HOME/frappe-bench && bench --site $DOMAIN list-apps"
mariadb -e "SHOW DATABASES;" | head -20
ss -ltn | grep -E ':13000|:11000' || echo 'temp redis stopped (expected)'
```

Expect `frappe` and `erpnext` at 16.x, a new `_<hash>` database, and
`sites/$DOMAIN/site_config.json` present.

Flag notes (verified against v16 `frappe/commands/site.py`): `--db-root-username` /
`--db-root-password` superseded the `--mariadb-root-*` names; `--no-mariadb-socket` is
deprecated in favour of `--mariadb-user-host-login-scope`; `--set-default` is real and
writes `sites/currentsite.txt`. There is **no environment-variable alternative** for the
secrets — only `--db-socket` has an envvar (`MYSQL_UNIX_PORT`). Do not invent
`MYSQL_ROOT_PASSWORD`. Passwords are briefly visible in `ps` and land in shell history;
prefix with a space under `HISTCONTROL=ignorespace` and clear history afterwards.

**Never re-run `bench new-site --force` against a live site — it drops the database.**

Sizing before you generate supervisor config (bench defaults `gunicorn_workers` to
`cpu_count()*2+1` = 3 on 1 vCPU, at ~250–400 MB RSS each with `--preload`):

```bash
sudo -u frappe -H bash -lc 'cd $HOME/frappe-bench && bench set-config -g gunicorn_workers 2'
sudo -u frappe -H bash -lc 'cd $HOME/frappe-bench && bench set-config -g background_workers 1'
```

---

## Stage 11 — Production setup (supervisor + nginx)

> **Stages 11–16 write `/usr/local/bin/bench` (Stage 7 Option A). If you took Option B,
> substitute `/home/frappe/.local/bin/bench` in every root-run line** — that directory is
> **not** on root's PATH, and a bare `bench` gives you `bench: command not found` halfway
> through production setup, with no supervisor or nginx config generated.

```bash
export DEBIAN_FRONTEND=noninteractive
apt-get install -y nginx supervisor fail2ban
systemctl enable --now nginx supervisor fail2ban
rm -f /etc/nginx/sites-enabled/default

# MUST be before any nginx generation - see Stage 13
sudo -u frappe -H bash -lc 'cd $HOME/frappe-bench && bench config dns_multitenant on'
```

### `bench setup production` aborts on fail2ban

On Kantishiva, `bench setup production` **aborted at `bench setup role fail2ban`** —
that step shells out to bench's Ansible playbooks, and Ansible was not installed.
(`bench setup production` itself does not need Ansible; only `bench setup role` does.
Separately, `setup_production_prerequisites()` will try
`sudo <python> -m pip install ansible` when ansible is absent, which then fails on PEP 668
with `error: externally-managed-environment`.)

**Workaround used, and the one to use again — install fail2ban from apt and run the two
generators individually:**

```bash
cd /home/frappe/frappe-bench

# nginx's log_format MUST exist before this generates a config - see below
cat > /etc/nginx/conf.d/00-log-format.conf <<'EOF'
log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                '$status $body_bytes_sent "$http_referer" '
                '"$http_user_agent" "$http_x_forwarded_for"';
EOF

/usr/local/bin/bench setup supervisor --yes     # Option B: /home/frappe/.local/bin/bench
/usr/local/bin/bench setup nginx --yes          # Option B: /home/frappe/.local/bin/bench
```

**Always pass `--yes`.** Without it both commands block on
`nginx.conf already exists and this will overwrite it. Do you want to continue?`, which
hangs a non-TTY SSH session forever.

### The supervisor symlink bench does not create

The symlink is gated on `restart_supervisor_on_update` in `common_site_config.json`
(`bench/config/production_setup.py:63-77`) while the nginx one is not, and
`bench setup supervisor` never symlinks at all — `generate_supervisor_config()` only
writes `<bench>/config/supervisor.conf`. `restart_supervisor_on_update` is **unset on a
fresh bench**, so even a fully successful `bench setup production` skips this step. Create
it by hand whichever path you took, and re-check it after any future
`bench setup production`:

```bash
ln -sfn /home/frappe/frappe-bench/config/supervisor.conf /etc/supervisor/conf.d/frappe-bench.conf
supervisorctl reread
supervisorctl update
sleep 5
supervisorctl status
```

The nginx side is the same pattern: `/etc/nginx/conf.d/frappe-bench.conf` is a **symlink**
to `/home/frappe/frappe-bench/config/nginx.conf`. Remember that — Stage 13 depends on it.

```bash
ln -sfn /home/frappe/frappe-bench/config/nginx.conf /etc/nginx/conf.d/frappe-bench.conf
```

### nginx will not start: `unknown log format "main"` (pre-empted above — read this if you skipped it)

```
nginx: [emerg] unknown log format "main" in /etc/nginx/conf.d/frappe-bench.conf:NN
```

`bench setup nginx`'s click options default to `--logging combined` and
`--log_format main` (`bench/commands/setup.py:28-41`), `config/nginx.py:49-53` sets
`template_vars["logging"]` whenever `logging != "none"`, and
`config/templates/nginx.conf:124` then renders `access_log /var/log/nginx/access.log
main;`. But **Debian/Ubuntu's `/etc/nginx/nginx.conf` defines no `log_format main`** —
that format only exists in nginx's upstream sample config, which Debian does not ship.

**This is not v16-specific and not Kantishiva-specific: v15's bench 5.29.0 has identical
defaults.** Only `bench setup production` escapes it, because it calls
`make_nginx_conf(bench_path, yes=yes)` with **no `logging` argument**, so
`template_vars["logging"]` is never set and no `access_log` line is emitted. Kantishiva hit
it precisely because `setup production` had aborted on fail2ban and the generators were run
individually.

nginx resolves directives in inclusion order, so the fix is a file that sorts *before*
`frappe-bench.conf` inside `conf.d/`:

```bash
cat > /etc/nginx/conf.d/00-log-format.conf <<'EOF'
log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                '$status $body_bytes_sent "$http_referer" '
                '"$http_user_agent" "$http_x_forwarded_for"';
EOF
nginx -t && systemctl reload nginx
```

`[UNVERIFIED — exact bytes]` The *fix* (a `00-log-format.conf` in `conf.d/`) is what was
done on Kantishiva; the literal format string used there was not recorded. The block
above is nginx upstream's standard `main` definition. Any valid `log_format main {...};`
in the `http` context satisfies bench.

### chown after ANY root-run bench command — applies to **both v15 and v16**

```bash
chown -R frappe:frappe /home/frappe/frappe-bench
```

On Kantishiva, root-run bench commands left **125 root-owned files** inside the bench.
The site looked fine until runtime, when the `frappe`-owned workers hit files they could
not write — e.g. during PDF rendering:

```
PermissionError: [Errno 13] Permission denied: '/home/frappe/frappe-bench/logs/render-template.log'
```

Make this reflexive: every time you run bench as root (production setup, `setup nginx`,
`setup supervisor`), `chown -R frappe:frappe` immediately afterwards. Then
`supervisorctl restart all`.

**Verify the whole stage:**

```bash
supervisorctl status                                   # all 7 programs RUNNING
nginx -t
grep -nE 'server_name|listen|root ' /etc/nginx/conf.d/frappe-bench.conf | head -20
curl -sI -H "Host: $DOMAIN" http://127.0.0.1/ | head -3
curl -s  -H "Host: $DOMAIN" http://127.0.0.1/api/method/frappe.ping
visudo -c                                              # must print 'parsed OK'
```

Expect 7 supervisor programs on a single-site bench — 2 redis (cache, queue), web
(gunicorn), node-socketio, 2 workers, scheduler — all `RUNNING`, no `FATAL`/`BACKOFF`;
`nginx -t` "syntax is ok"; `frappe.ping` returning `{"message":"pong"}`.

If `bench setup production` did write `/etc/sudoers.d/frappe`, `visudo -c` is the safety
check — a malformed sudoers file makes `sudo` unusable for **every** account on the box.
Keep the root session open until it passes.

---

## Stage 12 — Firewall (order matters absolutely)

```bash
apt-get install -y ufw
ss -ltnp | grep -E ':22|:2222'          # confirm the real sshd port FIRST
ufw allow 22/tcp  comment 'ssh'
ufw allow 80/tcp  comment 'http'
ufw allow 443/tcp comment 'https'
ufw default deny incoming
ufw default allow outgoing
ufw --force enable
ufw status verbose
```

**Verify:** `ufw status verbose` shows `Status: active` with 22/80/443 allowed and
default deny incoming. **Then open a second SSH session before closing the first.** On
Kantishiva SSH access was re-verified *after* enabling — do the same.

`ufw --force enable` with no SSH allow rule drops your session, and most VPS providers
give you no console fallback. `--force` is what makes `enable` non-interactive (it skips
the "this may disrupt existing ssh connections" prompt).

Ports 80 and 443 must be open before Stage 13's certbot run. Do **not** open anything
else: gunicorn (8000), socketio (9000), MariaDB (3306) and redis (11000/13000) must stay
on loopback — bench's default `use_redis_auth` is `False`, so an exposed redis is
unauthenticated.

```bash
ss -tlnp | grep -v 127.0.0.1            # should show only 22/80/443
```

Do not bother with `bench setup firewall` — it calls
`run_playbook("roles/bench/tasks/setup_firewall.yml", ...)` and needs Ansible plus the
easy-install playbook tree.

---

## Stage 13 — SSL

### 13a. `dns_multitenant` must be ON before nginx generation

Already done in Stage 11, but verify it stuck:

```bash
grep dns_multitenant /home/frappe/frappe-bench/sites/common_site_config.json   # "dns_multitenant": true
```

**Why — applies to both v15 and v16.** In `bench/config/nginx.py::prepare_sites()`, the
dns_multitenant-off branch appends the site to `sites["that_use_port"]` and the template
renders `server_block(...)` with **no `ssl_certificate` / `ssl_certificate_key` arguments
at all**. The macro's SSL branch (`{% if ssl_certificate and ssl_certificate_key %}`) and
the generated `return 301 https://$host$request_uri;` block are therefore unreachable.

Consequence: with it off, bench **silently ignores** the `ssl_certificate` keys in
site_config and can never emit an HTTPS block. There is no error message anywhere. This
is the number one cause of "HTTPS just won't come up". (The reason people think it works
without the flag: a single site coincidentally gets `listen 80; server_name <site>;`
because the port allocator hands site #1 port 80. That is HTTP only.)

### 13b. Issue the certificate with `certonly`, never installer mode

```bash
apt-get install -y certbot python3-certbot-nginx   # both in *universe* — enabled in Stage 2
which certbot                                   # /usr/bin/certbot, NOT /snap/bin/certbot
certbot --version

certbot certonly --nginx -d "$DOMAIN" --non-interactive --agree-tos -m "$LE_EMAIL"
ls -l /etc/letsencrypt/live/$DOMAIN/
```

**Why `certonly --nginx` and never `certbot --nginx`:**
`/etc/nginx/conf.d/frappe-bench.conf` is a **symlink** to the bench-generated
`config/nginx.conf`. certbot's *installer* mode edits that file in place through the
symlink — and `make_nginx_conf()` does a bare `open(conf_path, "w")` from the Jinja
template, so the next `bench setup nginx` or `bench setup production` destroys every
certbot edit with no warning.

`certonly --nginx` uses the plugin **only** to solve HTTP-01: it inserts a temporary
challenge server/location block into the nginx configuration it manages, reloads nginx,
then reverts — so there is no sustained downtime (unlike `--standalone`, which needs nginx
stopped at issuance *and at every renewal*) and no persistent server-block edit. It is not
a literal no-op on the file, though: **if certbot is interrupted mid-issuance, run
`bench setup nginx --yes` before trusting the generated file.**

**Why apt certbot, not snap:** mixing a snap certbot with apt plugins produces
`The requested nginx plugin does not appear to be installed`. Ubuntu 26.04 ships current
packages (certbot 4.0.0, python3-certbot-nginx 4.0.0 — both in *universe*), so use apt for
both and never install the snap. Also: on a 1 vCPU box, skip snapd entirely.

**Ordering:** the nginx plugin needs an existing `:80` server block matching `$DOMAIN` to
solve the challenge — so Stage 11 must be complete and
`curl -sI http://$DOMAIN/` must return a Frappe response, not the nginx welcome page,
before you run certbot.

**Skip `bench setup lets-encrypt` entirely.** Reading `bench/config/lets_encrypt.py`: it
renders `/etc/letsencrypt/configs/<site>.cfg` with `authenticator = standalone` and calls
`service("nginx","stop")` first (hard downtime), but `setup_crontab()` then installs
`certbot renew -a nginx --post-hook "systemctl reload nginx"` — an authenticator mismatch
requiring `python3-certbot-nginx`, which bench never installs. Its cron comment says
"every month" while `job.setall("0 0 */1 * *")` is daily. Unreliable by construction.

### 13c. Wire the cert in — and the `add-domain` trap

**When the site name IS the domain** (the normal single-site case), put the paths
straight into that site's `site_config.json`:

```bash
cd /home/frappe/frappe-bench
sudo -u frappe -H bash -lc "cd \$HOME/frappe-bench && bench --site $DOMAIN set-config ssl_certificate     /etc/letsencrypt/live/$DOMAIN/fullchain.pem"
sudo -u frappe -H bash -lc "cd \$HOME/frappe-bench && bench --site $DOMAIN set-config ssl_certificate_key /etc/letsencrypt/live/$DOMAIN/privkey.pem"
sudo -u frappe -H bash -lc "cd \$HOME/frappe-bench && bench --site $DOMAIN set-config host_name           https://$DOMAIN"
/usr/local/bin/bench setup nginx --yes          # Option B: /home/frappe/.local/bin/bench
chown -R frappe:frappe /home/frappe/frappe-bench
nginx -t && systemctl reload nginx
```

**Use `set-config`, not `bench set-ssl-certificate` / `bench set-ssl-key`.** Those two
regenerate nginx immediately — `set_site_config_nginx_property()` ends in
`make_nginx_conf(bench_path=bench_path)` with `yes` defaulting to `False`
(`bench/config/site_config.py:51-59`) — and **neither has a `--yes` flag**
(`bench/commands/utils.py:53-68`: both take only positional args). On the non-TTY root SSH
session this runbook assumes they hang forever on `nginx.conf already exists and this will
overwrite it. Do you want to continue?`, at the exact moment the site is mid-reconfiguration.
`bench --site X set-config` touches only `site_config.json`; the single explicit
`bench setup nginx --yes` afterwards does the regeneration under your control. This is also
what the Kantishiva fix ended up being — "set `ssl_certificate` / `ssl_certificate_key`
directly in site_config".

**Do NOT use `bench setup add-domain` for a name that is already the site name.** On
Kantishiva that registered the domain as an *alternate*, and the generated config then
contained **two `:80` server blocks with the same `server_name`**. nginx uses the first
match, so the HTTPS redirect block in the second was dead — HTTP kept serving plain and
nothing complained. The fix was to drop `domains` from site_config and set
`ssl_certificate` / `ssl_certificate_key` directly, exactly as above. `add-domain` is for
*genuinely additional* hostnames only.

**Verify:**

```bash
grep -nE 'ssl_certificate|listen 443|return 301' /etc/nginx/conf.d/frappe-bench.conf
grep -c 'listen 80' /etc/nginx/conf.d/frappe-bench.conf     # exactly ONE per server_name
curl -sI http://$DOMAIN/  | head -3                          # 301 -> https
curl -sI https://$DOMAIN/ | head -3                          # 200
echo | openssl s_client -servername $DOMAIN -connect $DOMAIN:443 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates
```

### 13d. Renewal

```bash
mkdir -p /etc/letsencrypt/renewal-hooks/deploy
printf '#!/bin/sh\nsystemctl reload nginx\n' > /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh
chmod +x /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh
systemctl list-timers certbot.timer
certbot renew --dry-run
```

Without the deploy hook, renewal succeeds and nginx keeps serving the **old expired
certificate** until something else reloads it. Kantishiva's cert is valid to
**2026-11-15** with auto-renew scheduled and this hook in place.

After any future `bench update`, re-run the `grep -c 'listen 443'` check before assuming
SSL survived regeneration.

---

## Stage 14 — PDF: all three engines

v16 has **three** PDF paths and you will meet all of them. The generator is chosen
**per Print Format**, not only globally — `printview.py` does:

```python
getattr(print_format, "pdf_generator", "wkhtmltopdf")
```

so flipping the global Print Settings value alone does **not** change existing formats.
There is also a v16 migration patch
(`sets_wkhtmltopdf_as_default_for_pdf_generator_field`) that explicitly writes
`"wkhtmltopdf"` onto every existing Print Format row, and
`frappe/utils/pdf.py::get_pdf()` — the path taken by all server-side PDF (email
attachments, `frappe.attach_print`, Auto Email Reports, `report_to_pdf`) — calls
pdfkit/wkhtmltopdf with **no Chrome branch at all**. Treat "v16 switched to Chrome" as
marketing; the code says otherwise.

### 14a. wkhtmltopdf — the jammy .deb (there is no 26.04 package)

`wkhtmltopdf/wkhtmltopdf` is archived (last push 2022-11-22); the newest
`wkhtmltopdf/packaging` release is **0.12.6.1-3**, whose newest Ubuntu target is
**jammy 22.04**. There is no noble/resolute build and **no `wkhtmltopdf` or `wkhtmltox`
package in Ubuntu 26.04's archive at all**. The jammy build installs and runs fine on
26.04 — its dependency closure resolves because `libssl3` is satisfied via `Provides:`
from `libssl3t64` and `libpng16-16` via `libpng16-16t64`.

```bash
cd /tmp
curl -fsSLO https://github.com/wkhtmltopdf/packaging/releases/download/0.12.6.1-3/wkhtmltox_0.12.6.1-3.jammy_amd64.deb
apt-get install -y ./wkhtmltox_0.12.6.1-3.jammy_amd64.deb    # apt, NOT dpkg -i, so Provides are honoured
wkhtmltopdf --version
which wkhtmltopdf
```

**Verify:** `wkhtmltopdf --version` prints `wkhtmltopdf 0.12.6.1 (with patched qt)`.
The "patched qt" part matters — an unpatched build breaks headers/footers, which is why
`apt install wkhtmltopdf` (where it exists at all) is the wrong answer. The deb installs
to `/usr/local/bin/wkhtmltopdf` + `/usr/local/lib/libwkhtmltox.so*`, both on the default
supervisor PATH.

Without it: `OSError: No wkhtmltopdf executable found`.

This is v16's shipped default and what ERPNext print formats are tuned for, so on
Kantishiva it was left as the Print Settings default.

`[SOURCE-VERIFIED, NOT RUN HERE]` If apt refuses the dependencies, `dpkg -x` the archive
and place `usr/local/bin/wkhtmltopdf` + `usr/local/lib/libwkhtmltox.so*` by hand, then
`ldconfig`.

### 14b. Chromium — self-downloads, but needs ~18 shared libraries

Chromium (Chrome for Testing 133) is fetched automatically by
`find_or_download_chromium_executable()` into `<bench>/chromium` — or you can point
`chromium_path` in `common_site_config.json` at an existing binary, which short-circuits
the download. Ubuntu 26.04 has **no chromium .deb**, only the `chromium-browser` snap
shim, whose confinement breaks a supervisor-spawned headless process — so let bench
download its own.

The download works on a minimal server. **Launching it does not**, until the runtime
libraries exist:

```
TimeoutError: Chromium took too long to start
```

(and underneath, typically `error while loading shared libraries: libnss3.so`). Install
the exact set that fixed it on Kantishiva:

```bash
apt-get install -y \
  libnss3 libnspr4 libatk1.0-0t64 libatk-bridge2.0-0t64 libatspi2.0-0t64 \
  libxcomposite1 libxdamage1 libxfixes3 libxrandr2 libxkbcommon0 \
  libgbm1 libasound2t64 libcups2t64 libdrm2 \
  libpango-1.0-0 libcairo2 libxshmfence1 libx11-xcb1
```

Note the `t64` suffixes — Ubuntu's 64-bit `time_t` transition renamed `libasound2`,
`libcups2`, `libatk1.0-0` and friends. Using the pre-t64 names gives
`Unable to locate package`.

**Verify:**

```bash
find /home/frappe/frappe-bench/chromium -type f -name '*headless_shell*' -o -name 'chrome*' | head
sudo -u frappe -H bash -lc 'ls -R $HOME/frappe-bench/chromium | head -20'
```

Then warm the download deliberately rather than letting a user's first print request
trigger a ~150 MB fetch inside a gunicorn worker.

`[SOURCE-VERIFIED, NOT RUN HERE]` bench/frappe expose `bench setup-chrome` for exactly
that warm-up, plus `chromium_path`, `chromium_version`, `chromium_download_url`,
`chromium_max_concurrent` and `chromium_start_timeout` in `common_site_config.json`. On a
1 vCPU box consider `bench set-config -g chromium_max_concurrent 1`, since each gunicorn
worker can spawn its own headless shell at ~150–250 MB. The core key is `chromium_path`
— **not** `chromium_binary_path`, which belongs to the `print_designer` app and is what
most forum posts wrongly quote.

### 14c. WeasyPrint 68.0 — a hard dependency you did not ask for

`WeasyPrint==68.0` (with `pydyf==0.12.1`) is **pinned in frappe v16's pyproject** and is
loaded through cffi. It is used by the Print Format Builder route
(`frappe.utils.weasyprint.download_pdf`). Nothing complains at install time; it fails the
first time something calls it:

```
OSError: cannot load library 'libpangoft2-1.0-0': libpangoft2-1.0-0: cannot open shared object file: No such file or directory
```

```bash
apt-get install -y libpangoft2-1.0-0 libpangocairo-1.0-0 libharfbuzz-subset0 \
                   libgdk-pixbuf-2.0-0 shared-mime-info
apt-get install -y libpango-1.0-0 libharfbuzz0b libcairo2 fontconfig libfontconfig1
```

**Verify:**

```bash
sudo -u frappe -H bash -lc 'cd $HOME/frappe-bench && ./env/bin/python -c "import weasyprint; print(weasyprint.__version__)"'
```

### 14d. Prove all three actually produce a PDF

On Kantishiva all three were verified by generating real PDFs and checking for a valid
`%PDF` header. Do the same before you call the install done:

Write a tiny helper and drive it with `bench execute`, then assert on the file:

```bash
cat > /home/frappe/frappe-bench/apps/frappe/frappe/pdf_smoke.py <<'PY'
def run():
	from frappe.utils.pdf import get_pdf
	with open('/tmp/wk.pdf', 'wb') as f:
		f.write(get_pdf('<h1>wkhtmltopdf smoke test</h1>'))
PY
chown frappe:frappe /home/frappe/frappe-bench/apps/frappe/frappe/pdf_smoke.py

rm -f /tmp/wk.pdf
sudo -u frappe -H bash -lc "cd \$HOME/frappe-bench && bench --site $DOMAIN execute frappe.pdf_smoke.run"
test -s /tmp/wk.pdf && head -c 4 /tmp/wk.pdf | grep -q '%PDF' && echo 'PDF OK' || echo 'PDF FAILED'

rm -f /home/frappe/frappe-bench/apps/frappe/frappe/pdf_smoke.py
```

**Do not pipe a heredoc into `bench console`.** `bench console` is IPython: it consumes the
stdin block but a Python exception inside it does **not** become a non-zero exit status, so
a failed `get_pdf()` leaves an empty or stale `/tmp/wk.pdf` and the following `head -c 4`
prints nothing while the step reads as passed. The `rm -f` + `test -s` + `%PDF` grep above
is what makes the check actually fail when the engine is broken. `bench execute` propagates
the traceback and a non-zero exit.

Repeat per engine by switching Print Settings **and** the individual Print Format's
`pdf_generator` field. Separately test an email-with-PDF-attachment flow — that path
always goes through wkhtmltopdf regardless of the Print Settings value, and it is the
failure people discover days after go-live.

---

## Stage 15 — Setup wizard crash: `Asia/Calcutta` (pre-empted in Stage 2 — read this if you skipped it)

On the Kantishiva build, `tzdata-legacy` and the venv `tzdata` were only installed *after*
this crash. Stages 2 and 8 above now install both up front, so a clean run of this runbook
never sees it. The diagnosis is kept because the same traceback appears on any Ubuntu
24.04+ box (v15 included) where the packages are missing.

First run of the setup wizard died at **stage 1** with:

```
zoneinfo._common.ZoneInfoNotFoundError: 'No time zone found with key Asia/Calcutta'
```

preceded in the traceback by:

```
ModuleNotFoundError: No module named 'tzdata'
```

**Root cause — three things at once, all specific to this stack:**

1. Frappe's country→timezone map still emits the **deprecated alias** `Asia/Calcutta`
   (renamed to `Asia/Kolkata` in the tz database years ago).
2. **Ubuntu 24.04+ moved every deprecated tz alias out of `tzdata` into a separate
   `tzdata-legacy` package**, not installed by default. So
   `/usr/share/zoneinfo/Asia/Kolkata` existed and `Asia/Calcutta` did not.
3. Python's `zoneinfo` falls back to the pip `tzdata` package when the system copy lacks
   a key — but pip `tzdata` was not in the bench venv either, hence the
   `ModuleNotFoundError` in the first traceback.

**Fix — both halves, belt and braces:**

```bash
apt-get install -y tzdata-legacy                      # system aliases
/home/frappe/frappe-bench/env/bin/pip install tzdata  # python fallback
supervisorctl restart all                             # workers cache zoneinfo
```

The `supervisorctl restart all` is not optional — workers cache zoneinfo lookups, so a
fix without a restart looks like it did nothing.

**Verify:**

```bash
/home/frappe/frappe-bench/env/bin/python -c \
  "from zoneinfo import ZoneInfo; print(ZoneInfo('Asia/Calcutta'), ZoneInfo('Asia/Kolkata'))"
```

Then retry the wizard, and save System Settings with the alias (the exact operation that
crashed) as the real confirmation.

**Retry is safe here.** The wizard failed on its *first* stage, so nothing was
half-created — no Company, no Fiscal Year, `setup_complete=0`. If it had failed at a
later stage you would be looking at a site reset instead.

---

## Stage 16 — Scheduler and backups

```bash
sudo -u frappe -H bash -lc "cd \$HOME/frappe-bench && bench --site $DOMAIN set-maintenance-mode off"
sudo -u frappe -H bash -lc "cd \$HOME/frappe-bench && bench --site $DOMAIN enable-scheduler"
sudo -u frappe -H bash -lc "cd \$HOME/frappe-bench && bench --site $DOMAIN scheduler status"
sudo -u frappe -H bash -lc "cd \$HOME/frappe-bench && bench --site $DOMAIN set-config developer_mode 0"
```

**Verify:** `scheduler status` prints `Scheduler is enabled`. An install with the
scheduler disabled looks completely healthy and silently does nothing — no emails, no
auto-repeat, no scheduled reports.

Backups. On Kantishiva they run **every 6 hours with files**, installed in
`crontab -u frappe`, and were verified by taking one. **Do not reach for
`bench setup backups` to get there** — see below.

```bash
crontab -u frappe -l    # bench init already installed this unless --no-backups was passed
crontab -u frappe -e    # append --with-files after 'backup'
crontab -u frappe -l
sudo -u frappe -H bash -lc "cd \$HOME/frappe-bench && bench --site $DOMAIN backup --with-files"
ls -lh /home/frappe/frappe-bench/sites/$DOMAIN/private/backups/ | tail -5
```

**The entry bench generates never includes `--with-files`.** `bench/bench.py` builds it as
`backup_command = f"cd {bench_dir} && {sys.argv[0]} --verbose --site all backup"` — database
only, no `public/` or `private/` files — on a 6-hourly schedule
(`comment="bench auto backups set for every 6 hours"`). And
`bench/utils/system.py:114-115` shows **`bench init` already called `bench.setup.backups()`**
unless you passed `--no-backups`, so running `bench setup backups` again just re-touches the
same entry. The 6-hourly-**with-files** state on Kantishiva was the result of editing that
crontab afterwards. If you skip the edit you get database-only backups while believing files
are covered. [verified in bench 5.29.0 source]

`[UNVERIFIED — exact cron line]` The literal crontab entry on Kantishiva was not
recorded. The equivalent hand-written line for the observed 6-hourly-with-files behaviour
is:

```cron
0 */6 * * * cd /home/frappe/frappe-bench && /usr/local/bin/bench --site all backup --with-files >> /home/frappe/frappe-bench/logs/backup.log 2>&1
```

Adjust the `bench` path to match your Stage 7 choice. Backups land in
`sites/<site>/private/backups` **on the same disk** — arrange off-box copies separately;
that is not covered here.

---

## Verify the whole install

Run every line. Anything that does not match is an open defect, not a cosmetic issue.

```bash
# --- identity of the stack -------------------------------------------------
sudo -u frappe -H bash -lc 'cd $HOME/frappe-bench && bench version'      # frappe 16.x, erpnext 16.x
git -C /home/frappe/frappe-bench/apps/frappe  rev-parse --abbrev-ref HEAD   # version-16
git -C /home/frappe/frappe-bench/apps/erpnext rev-parse --abbrev-ref HEAD   # version-16
git -C /home/frappe/frappe-bench/apps/frappe  rev-parse HEAD                # RECORD THIS
git -C /home/frappe/frappe-bench/apps/erpnext rev-parse HEAD                # RECORD THIS
node -v; yarn -v; bench --version; uv --version; mariadb --version

# --- processes -------------------------------------------------------------
supervisorctl status                       # 7 RUNNING: 2 redis, web, socketio, 2 workers, scheduler
grep -A3 node-socketio /etc/supervisor/conf.d/frappe-bench.conf   # block MUST exist
systemctl is-active nginx mariadb supervisor fail2ban cron
systemctl is-active redis-server   # MUST be inactive - bench runs its own (Stage 4)
supervisorctl status | grep redis  # frappe-bench-redis-cache + -redis-queue RUNNING
redis-cli -p 13000 ping; redis-cli -p 11000 ping                   # the real redis check
systemctl is-enabled nginx mariadb supervisor cron                 # survives reboot

# --- ownership -------------------------------------------------------------
find /home/frappe/frappe-bench ! -user frappe | head    # MUST be empty

# --- HTTP / TLS ------------------------------------------------------------
curl -sI http://$DOMAIN/  | head -3                      # 301 -> https
curl -sI https://$DOMAIN/ | head -3                      # 200
curl -s  https://$DOMAIN/api/method/frappe.ping          # {"message":"pong"}
curl -s  https://$DOMAIN/login | grep -o '<title>[^<]*</title>'
echo | openssl s_client -servername $DOMAIN -connect $DOMAIN:443 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates
certbot renew --dry-run                                  # simulated renewals succeeded
grep -c 'listen 80' /etc/nginx/conf.d/frappe-bench.conf  # one per server_name, not two

# --- realtime (the thing that fails silently) ------------------------------
tail -20 /home/frappe/frappe-bench/logs/node-socketio.error.log
#   then: open Desk in a browser, confirm notifications/progress bars work

# --- database --------------------------------------------------------------
mariadb -e "SHOW VARIABLES WHERE Variable_name IN ('version','character_set_server','collation_server');"
ss -ltn | grep 3306                                      # 127.0.0.1 only

# --- PDF, all three engines ------------------------------------------------
wkhtmltopdf --version                                    # "(with patched qt)"
sudo -u frappe -H bash -lc 'cd $HOME/frappe-bench && ./env/bin/python -c "import weasyprint; print(weasyprint.__version__)"'
ls -R /home/frappe/frappe-bench/chromium | head
#   then generate a real PDF per engine and check the file starts with %PDF

# --- timezone --------------------------------------------------------------
/home/frappe/frappe-bench/env/bin/python -c \
  "from zoneinfo import ZoneInfo; print(ZoneInfo('Asia/Calcutta'), ZoneInfo('Asia/Kolkata'))"

# --- scheduled work --------------------------------------------------------
sudo -u frappe -H bash -lc "cd \$HOME/frappe-bench && bench --site $DOMAIN scheduler status"
crontab -u frappe -l
sudo -u frappe -H bash -lc 'cd $HOME/frappe-bench && bench doctor'

# --- firewall / exposure ---------------------------------------------------
ufw status verbose                                       # 22/80/443, default deny in
ss -tlnp | grep -v 127.0.0.1                             # nothing else public

# --- health ----------------------------------------------------------------
free -m; swapon --show; df -h /
tail -20 /home/frappe/frappe-bench/logs/web.error.log
tail -20 /home/frappe/frappe-bench/logs/worker.error.log
journalctl -u nginx -n 20 --no-pager
```

**Then reboot once and re-run this entire block.** That is the only real proof of boot
persistence, and it is the cheapest test you will ever run. Take a VPS snapshot
immediately afterwards as your rollback target, and record both git HEADs — the
`version-16` branch moves weekly and `bench update` will carry you forward unattended.

Finally: change the Administrator password at first login, and configure an outgoing
email account before anyone relies on the system.

---

## Error string index (greppable)

| Error text | Stage | Cause |
|---|---|---|
| `error: externally-managed-environment` | 7 | PEP 668; use pipx or `uv tool install` |
| `FileNotFoundError: [Errno 2] No such file or directory: 'uv'` | 7 | bench 5.31 needs `uv` on PATH; pipx does not expose dependency scripts |
| `Exception: pkg-config is not installed. Please install it before proceeding.` | 2 | bench's v16 `check_pkg_config()` guard |
| `Exception: Can not find valid pkg-config name. Specify MYSQLCLIENT_CFLAGS and MYSQLCLIENT_LDFLAGS env vars manually` | 2 | `libmariadb-dev` missing; mysqlclient 2.2.7 is sdist-only |
| `fatal error: Python.h: No such file or directory` | 2 | `python3-dev` missing |
| `unknown variable 'innodb-file-format=barracuda'` | 3 | classic Frappe my.cnf on MariaDB ≥10.3 |
| `Illegal mix of collations (utf8mb4_uca1400_ai_ci,IMPLICIT) and (utf8mb4_unicode_ci,IMPLICIT)` | 3 | MariaDB 11.8 default collation; set `collation-server` |
| `error frappe-framework@: The engine "node" is incompatible with this module. Expected version ">=24". Got "22.22.1"` | 5 | Ubuntu's nodejs 22.22; use NodeSource 24 |
| `Error 111 connecting to 127.0.0.1:13000. Connection refused.` | 10 | bench redis not running during `new-site` |
| `(1045, "Access denied for user 'root'@'localhost'")` | 3/10 | root TCP password auth not set; use the `VIA ... OR ...` form |
| `nginx: [emerg] unknown log format "main"` | 11 | Debian/Ubuntu nginx.conf defines no `log_format main` |
| `nginx.conf already exists and this will overwrite it. Do you want to continue?` | 11 | missing `--yes` on a non-TTY session |
| `PermissionError: [Errno 13] Permission denied: '.../logs/render-template.log'` | 11 | root-owned files from a root-run bench command |
| `OSError: No wkhtmltopdf executable found` | 14a | not packaged on 26.04; install the jammy .deb |
| `TimeoutError: Chromium took too long to start` | 14b | headless Chrome runtime libs missing |
| `OSError: cannot load library 'libpangoft2-1.0-0'` | 14c | WeasyPrint 68.0 native deps missing |
| `zoneinfo._common.ZoneInfoNotFoundError: 'No time zone found with key Asia/Calcutta'` | 15 | deprecated alias moved to `tzdata-legacy` |
| `ModuleNotFoundError: No module named 'tzdata'` | 15 | pip `tzdata` missing from the bench venv |

## Known-good deviations and things deliberately NOT done

- **No MariaDB apt repo.** Ubuntu's 11.8.6 is exactly frappe's tested ceiling;
  `check_compatible_versions()` warns above 11.8, so 12.x would be a downgrade in support.
- **No `bench setup lets-encrypt`** (Stage 13b) and **no `bench setup firewall`**
  (Stage 12) — both need Ansible or are broken by construction.
- **No `uv python install 3.14 --default`** — it shadows `/usr/bin/python3`.
- **No `corepack enable`**, no Yarn Berry, no nvm.
- **No third-party apps.** The v15 apps `trustbit_mandi` and `item_creator` are **not**
  installed on Kantishiva: both target Frappe v15, and moving them to v16 is a real
  migration with breaking API changes (default sort order changed from `modified` to
  `creation`; `has_permission` hooks must now explicitly return `True`; document hooks can
  no longer commit; `/app` rerouted to `/desk`; Energy Points, Newsletter, Backup
  Integrations and Blog split into separate apps), not a reinstall. Get frappe + erpnext
  healthy and verified before adding anything.
