# What changed from v15 → v16

**Audience:** you know v15 well. This file is only the delta, and only the parts that
change what you type or what breaks. For the ordered build, use
[v16/install-ubuntu-2604.md](install-ubuntu-2604.md); for keeping a v15 client alive,
[v15/install-and-gotchas.md](../v15/install-and-gotchas.md).

**Sources and how facts are tagged.** Three kinds of evidence, never blurred:

| Tag | Means |
|---|---|
| `[KANTISHIVA]` | Actually run and observed on the v16 build (`kantishiva.trustbit.cloud`, Ubuntu 26.04, 2026-08-17). Highest authority for v16. `Trustbit Kantishiva/SERVER_KANTISHIVA.md` |
| `[V15-LOCAL]` | Read directly out of the v15 checkout at `~/frappe-bench-v15` (frappe **15.100.0**, erpnext **15.97.0**). File + line given. |
| `[SOURCE-VERIFIED]` | Read out of frappe/bench source or frappe CI by the research pass, but **not** executed by us. `tasks/wr2hqc5dv.output` |
| `[UNVERIFIED]` | Believed, not checked. Confirm before you rely on it. |

Two claims that circulate internally are **wrong** and are corrected below: WeasyPrint is
not new in v16 (§7), and the Node-heap cap is not new in v16 (§8).

---

## 1. The two stacks side by side

| | **v15** (what we run) | **v16** (Kantishiva) |
|---|---|---|
| frappe `requires-python` | `>=3.10,<3.14` `[V15-LOCAL apps/frappe/pyproject.toml:7]` | `>=3.14,<3.15` `[SOURCE-VERIFIED]` |
| erpnext `requires-python` | `>=3.10` `[V15-LOCAL apps/erpnext/pyproject.toml:7]` | `>=3.14` `[SOURCE-VERIFIED]` |
| Python in practice | 3.11.14 local bench venv; prod on Ubuntu 22.04 | **3.14.4**, the system `/usr/bin/python3` `[KANTISHIVA]` |
| Practical OS | Ubuntu 22.04 / 24.04 | **Ubuntu 26.04 LTS** only `[KANTISHIVA]` |
| frappe / erpnext | 15.100.0 / 15.97.0 | 16.31.0 / 16.32.1 `[KANTISHIVA]` |
| Node `engines` | `">=18"` `[V15-LOCAL apps/frappe/package.json:19]` | `">=24"` `[SOURCE-VERIFIED]` |
| Node in practice | 18.20.8 local; 20 LTS at RD School (for LMS) | **24.19.0** from NodeSource `[KANTISHIVA]` |
| yarn | 1.22.x classic | 1.22.22 classic — unchanged, still **not** Berry `[KANTISHIVA]` |
| esbuild pin | `"^0.14.29"` `[V15-LOCAL apps/frappe/package.json:45]` | `"^0.14.29"` → resolves 0.14.54 — unchanged `[SOURCE-VERIFIED]` |
| erpnext front-end build | **none** — `package.json` has no `scripts` block `[V15-LOCAL]` | React 19.2 + Vite 8.0 + Tailwind 4.3 sub-app (`banking/`) `[SOURCE-VERIFIED]` |
| bench CLI | 5.29.0 in use | 5.31.0; **5.25.0** is the floor for the v16 codepath `[SOURCE-VERIFIED]` |
| env builder | venv + pip (uv only if bench ≥5.28) | **uv** — `uv venv env --seed`, `uv pip install -e` `[KANTISHIVA]` |
| MySQL driver | `PyMySQL==1.1.1` only `[V15-LOCAL pyproject.toml:22]` | `PyMySQL==1.1.2` **+ `mysqlclient==2.2.7`** `[SOURCE-VERIFIED]` |
| Compiler needed to install? | No | **Yes** — mysqlclient is sdist-only `[SOURCE-VERIFIED]` |
| MariaDB tested range | 10.6 → 10.8 `[V15-LOCAL setup_db.py:97-110]`; prod runs 10.6 | 10.6 → **11.8**; CI runs `mariadb:11.8` `[SOURCE-VERIFIED]` / 11.8.6 shipped `[KANTISHIVA]` |
| my.cnf validation at `new-site` | present | **removed** — version warning only `[SOURCE-VERIFIED]` |
| redis client pin | `redis~=4.5.5`, `hiredis~=2.2.3` `[V15-LOCAL]` | `redis~=7.1.0`, `hiredis~=3.3.0`; server must speak RESP3 `[SOURCE-VERIFIED]` |
| rq | `rq==1.15.1` `[V15-LOCAL]` | `rq 2.6.1` `[SOURCE-VERIFIED]` |
| PDF engines | wkhtmltopdf (pdfkit) | wkhtmltopdf (still default) **+ Chromium/CDP** `[KANTISHIVA]` |
| WeasyPrint | `WeasyPrint==68.0` already present `[V15-LOCAL pyproject.toml:28]` | `WeasyPrint==68.0` — **not a v16 addition** `[SOURCE-VERIFIED]` |
| DB backends | mariadb, postgres | mariadb, postgres, **sqlite** (`--db-type sqlite`) `[SOURCE-VERIFIED]` |
| socketio supervisor line | absolute node path baked in at generate time | `command={{ bench_cmd }} socketio`, node resolved at **runtime** `[SOURCE-VERIFIED]` |

One-line summary: **v16 is a platform bump, not a feature bump.** Nearly every new
failure mode below traces back to one decision — the Python pin.

---

## 2. Python 3.14 is a hard pin, and it picks your OS

**Applies to: v16.**

```
frappe version-16   requires-python = ">=3.14,<3.15"   [SOURCE-VERIFIED]
frappe 15.100.0     requires-python = ">=3.10,<3.14"   [V15-LOCAL apps/frappe/pyproject.toml:7]
```

Both ends are exclusive of the other. **v15 will not install on Ubuntu 26.04 and v16 will
not install on 22.04/24.04 without a hand-built interpreter.** There is no shared OS.
Ubuntu 26.04 "resolute" is the only Ubuntu that ships 3.14 (3.14.4) `[KANTISHIVA]`, which
is why the Kantishiva box is on 26.04 — the OS was chosen by the framework, not the
other way round.

**The escape hatch, if a client's box cannot be 26.04:** the official docs install a
uv-managed interpreter with `uv python install 3.14 --default`
`[SOURCE-VERIFIED, NOT RUN HERE]`. We deliberately did *not* use it on 26.04 because it
drops a `~/.local/bin/python3` shim that PATH-shadows `/usr/bin/python3` and silently
changes which interpreter `bench init` builds against. On an older OS it is the only
route, but treat "v16 on 24.04" as untested by us and price it as such.

### Fallout 1 — PEP 668, because the system Python *is* 3.14

`/usr/lib/python3.14/EXTERNALLY-MANAGED` exists, so installing bench as root fails:

```
error: externally-managed-environment
```

Never `--break-system-packages`, never delete `EXTERNALLY-MANAGED`: on 26.04 that same
interpreter is what `apt`, `python3-apt` and `unattended-upgrades` run on
`[SOURCE-VERIFIED]`. Install bench in its own venv — see §4.

### Fallout 2 — the tz alias trap (Ubuntu 24.04+, hit at Kantishiva)

The setup wizard died at stage 1 `[KANTISHIVA]`:

```
zoneinfo._common.ZoneInfoNotFoundError: 'No time zone found with key Asia/Calcutta'
ModuleNotFoundError: No module named 'tzdata'
```

Frappe's country→timezone map still emits the deprecated alias `Asia/Calcutta`, and
Ubuntu 24.04+ moved every deprecated alias out of `tzdata` into **`tzdata-legacy`**, which
is not installed by default. Python's `zoneinfo` would fall back to the pip `tzdata`
package, which was not in the bench venv either. Fix both halves:

```bash
apt-get install -y tzdata-legacy                      # system aliases
/home/frappe/frappe-bench/env/bin/pip install tzdata  # python fallback
supervisorctl restart all                             # workers cache zoneinfo
```

This is an OS-version bug, not a v16 bug — a v15 box on 24.04 would hit it too. It lands
in the v16 column purely because v16 forces you onto a new OS.

---

## 3. Node ≥ 24

**Applies to: v16** (the nvm/supervisor rule applies to both).

v15 declares `"node": ">=18"` `[V15-LOCAL apps/frappe/package.json:19]`; v16 declares
`">=24"` `[SOURCE-VERIFIED]`. Ubuntu 26.04's own `nodejs` package is **22.22**, below the
floor — so NodeSource is mandatory on the very OS v16 forces you onto `[KANTISHIVA]`.

Yarn 1.x errors on an `engines` mismatch and bench runs `yarn install --check-files`
without `--ignore-engines`, so a too-old Node fails `bench init` at the asset step rather
than at a nice preflight check. And `--skip-assets` **does not skip yarn** — only the
build itself is gated `[SOURCE-VERIFIED: bench/app.py:959-964]`:

```
Please install yarn using below command and try again.
```

### NodeSource, never nvm/fnm — the silent one

`bench/config/supervisor.py:49` does `"node": which("node") or which("nodejs")` and the
template wraps the whole `[program:*-node-socketio]` block **and its `[group:*-web]`
membership** in `{% if node %}` `[SOURCE-VERIFIED]`. `bench setup production` runs under
sudo, whose `secure_path` cannot see `~/.nvm/...`. Result: the site loads perfectly and
websockets are dead — no notifications, no progress bars, no list auto-refresh, no error
anywhere `[KANTISHIVA recorded the same rule]`.

```bash
sudo which node                                   # must print /usr/bin/node BEFORE setup
grep -A3 node-socketio /etc/supervisor/conf.d/frappe-bench.conf   # must exist AFTER
```

**v16-only twist:** bench ≤5.22 baked the absolute node path into `supervisor.conf`; bench
5.31 emits `command={{ bench_cmd }} socketio` and resolves node with `shutil.which()` at
**runtime, inside supervisor's environment** `[SOURCE-VERIFIED]`. The failure therefore
moves from config-generation time to process-start time — a PATH that was fine when you
generated the config can break the socketio process on a later reboot. Same fix: a
system-wide `/usr/bin/node`.

Do **not** run `corepack enable`; a corepack yarn shim shadows `/usr/bin/yarn` and fails
on classic flags `[SOURCE-VERIFIED]`.

---

## 4. bench 5.28+ builds the env with uv, not pip

**Applies to: v16 mandatory; v15 too if you use a modern bench.**

The first thing that stops people dead on a fresh v16 box `[KANTISHIVA]`:

```
FileNotFoundError: 'uv'
```

bench 5.31 declares `uv~=0.11.6` as a dependency and **shells out to a bare `uv` on
PATH** — `uv venv env --seed --python <py>` and `uv pip install -e apps/frappe`
`[SOURCE-VERIFIED: bench/bench.py:371, bench/app.py:939]`. It does not install uv for you,
and uv is not packaged in Ubuntu 26.04.

| bench tag | What changed `[SOURCE-VERIFIED, by tag bisection]` |
|---|---|
| 5.25.0 | first tag with the frappe-≥16 codepath (`check_pkg_config()`) — the true floor for v16 |
| 5.25.4 | uv available, opt-in via `BENCH_USE_UV` |
| **5.28.0** | **uv becomes the default**; `BENCH_DISABLE_UV=1` reverts |
| 5.31.0 | what Kantishiva runs |

**The pipx trap:** `pipx install frappe-bench` installs bench's `uv` into
`~/.local/pipx/venvs/frappe-bench/bin/uv`, which is *not* on PATH — pipx only links the
requested package's console scripts. `bench init` then dies at venv creation
`[SOURCE-VERIFIED]`. Two working routes:

```bash
# Official / CI route [SOURCE-VERIFIED, NOT RUN HERE]
curl -LsSf https://astral.sh/uv/install.sh | sh
uv tool install frappe-bench --python /usr/bin/python3

# Kantishiva's route, reconstructed from SERVER_KANTISHIVA.md notes 1-2 — pipx for
# BOTH packages, with the env vars it names, so uv and bench land on a shared PATH.
# The env vars and the approach are [KANTISHIVA]; these two literal command lines are
# not a copied transcript, so check `command -v uv` and `command -v bench` after.
PIPX_HOME=/opt/pipx PIPX_BIN_DIR=/usr/local/bin pipx install uv
PIPX_HOME=/opt/pipx PIPX_BIN_DIR=/usr/local/bin pipx install frappe-bench
```

Result on that box: `bench` at `/usr/local/bin/bench` (5.31.0) and uv 0.12.5 `[KANTISHIVA]`.

Either is fine; the invariant is **`command -v uv` must succeed for the user running
bench**. The `PIPX_BIN_DIR=/usr/local/bin` bit matters because the `frappe` user has to
find the binaries too.

`BENCH_DISABLE_UV=1` falls back to `python3 -m venv`, which then makes the `python3-venv`
apt package mandatory `[SOURCE-VERIFIED: bench/utils/bench.py:53-59]`. That is the lever
for building a **v15** bench with a modern bench CLI.

Do not hand-upgrade setuptools inside the bench tool venv: bench 5.31.0 pins
`setuptools>=71,<82` because it still imports distutils through the shim
`[SOURCE-VERIFIED]`.

---

## 5. `mysqlclient==2.2.7` — a new core dependency that needs a compiler

**Applies to: v16 only.**

v15 ships **PyMySQL only** — `"PyMySQL==1.1.1"` and no mysqlclient anywhere in the
dependency list `[V15-LOCAL apps/frappe/pyproject.toml:22]`. v16 adds
`mysqlclient==2.2.7` alongside `PyMySQL==1.1.2` `[SOURCE-VERIFIED]`.

**mysqlclient publishes no Linux wheels — Windows wheels plus an sdist, and that has been
true for every 2.2.x release** `[SOURCE-VERIFIED: PyPI JSON API for 2.2.7]`. So it compiles
from source on every Linux install, every time. Its `setup.py` shells out to
`pkg-config --exists` against `["mysqlclient", "mariadb", "libmariadb",
"perconaserverclient"]`, which is why bench added a matching hard guard for frappe ≥16.

New hard prerequisites for `bench init`, none of which v15 needed:

```bash
apt-get install -y build-essential pkg-config python3-dev libmariadb-dev
ls -l /usr/include/python3.14/Python.h      # must exist
pkg-config --exists mariadb && echo OK      # must print OK
```

Error strings, all greppable:

```
Exception: pkg-config is not installed. Please install it before proceeding.
Exception: Can not find valid pkg-config name. Specify MYSQLCLIENT_CFLAGS and MYSQLCLIENT_LDFLAGS env vars manually
fatal error: Python.h: No such file or directory
error: command 'gcc' failed
```

Reassurance for a 1 vCPU box: it is one small C file (`_mysql.c`), seconds of compile.
**Every other** C extension in frappe/erpnext v16 — pyarrow 25, duckdb 1.4.3, Pillow 12.2,
orjson, lxml, cryptography 50, nh3, psutil — ships a cp314 or abi3 manylinux wheel, so
nothing else builds and **no Rust toolchain is needed** `[SOURCE-VERIFIED: PyPI JSON API
against v16's exact pins]`.

Escape hatch if 2.2.7 ever fails to build: `mysqlclient==2.2.8` added 3.14 support, at the
cost of deviating from frappe's `==` pin `[SOURCE-VERIFIED]`.

*Inference, stated as such:* the research pass could not prove 2.2.7 compiles on CPython
3.14 (it inferred it from frappe CI). The Kantishiva bench built and runs, and mysqlclient
is an unconditional dependency, so it must have compiled there — but no one read that line
of build output, so treat "it compiles on 3.14.4" as verified-by-outcome, not by transcript.

---

## 6. MariaDB: the tested window moved 10.8 → 11.8

**Applies to: both, different numbers.**

The check is `frappe/database/mariadb/setup_db.py::check_compatible_versions()`. v15's is
quoted verbatim from our own checkout `[V15-LOCAL setup_db.py:97-110]`:

```python
if version_tuple < (10, 6):
    "Warning: MariaDB version {version} is less than 10.6 which is not supported by Frappe"
elif version_tuple >= (10, 9):
    "Warning: MariaDB version {version} is more than 10.8 which is not yet tested with Frappe Framework."
```

v16 keeps the 10.6 floor and lifts the ceiling to 11.8 — it warns only below 10.6 or above
`(11, 8)` `[SOURCE-VERIFIED: full read of version-16 setup_db.py]`. v16 CI runs
`image: mariadb:11.8` across `_base-server-tests.yml`, `_base-ui-tests.yml` and
`_base-migration.yml`, and frappe_docker ships `mariadb:11.8` `[SOURCE-VERIFIED]`.

Practical consequences:

- **Use Ubuntu 26.04's own package.** It is 11.8.6 `[KANTISHIVA]` — exactly the tested
  version, no third-party repo. Adding MariaDB's own apt repo (as nearly every ERPNext
  guide tells you) gets you 12.x and trips
  `Warning: MariaDB version ... is newer than 11.8 which is not yet tested with Frappe Framework.`
  `[SOURCE-VERIFIED]`
- **11.8 is the ceiling, not the middle.** Do not let a future distro upgrade push the box
  to 11.9+/12.x until frappe widens the check.
- **v16 removed the my.cnf validation.** v15's `new-site` used to abort on a bad
  charset/collation config; v16's `setup_db.py` only compares versions and asserts nothing
  about collation `[SOURCE-VERIFIED]`. A misconfigured server now sails through
  `bench new-site` silently and surfaces later as data errors. Set the charset yourself
  and verify it.
- **11.8 changed the default collation** to `utf8mb4_uca1400_ai_ci` (MDEV-25829). Frappe's
  tables are `utf8mb4_unicode_ci`, so a bare `CHARACTER SET utf8mb4` with no `COLLATE`
  now yields the wrong collation and throws `Illegal mix of collations` (error 1267)
  `[SOURCE-VERIFIED]`. Hence `collation-server = utf8mb4_unicode_ci` must be stated
  explicitly on v16 where v15 could get away with the default `[KANTISHIVA]`.
- **`innodb_snapshot_isolation` defaults ON in 11.8** (was OFF in 10.6/11.4), surfacing
  `ER_CHECKREAD (1020)`. No action needed — frappe's `is_deadlocked()` matches
  `ER.CHECKREAD` and retries `[SOURCE-VERIFIED]`.
- **The old wiki my.cnf still kills the server** — `innodb-file-format` and
  `innodb-large-prefix` were removed in MariaDB **10.3.1**, so mysqld refuses to start on
  11.8 `[KANTISHIVA]` and, by the same reasoning, on any 10.6 box. This one is not a v16
  change; it is a MariaDB change that v16 boxes hit first because they are new builds.
- **Auth plugins are unchanged and still constrained:** PyMySQL cannot speak ed25519 or
  PARSEC, so leave the default plugin alone on both versions `[SOURCE-VERIFIED]`.

---

## 7. PDF — what actually changed, and what didn't

**Applies to: v16** (corrections apply to both).

> **Correction — WeasyPrint is NOT new in v16.** `WeasyPrint==68.0` is already a pinned
> dependency of frappe **15.100.0** `[V15-LOCAL apps/frappe/pyproject.toml:28]`, alongside
> `pydyf==0.11.0`, `cairocffi==1.5.1` and `tinycss2==1.5.1`. v16 keeps `WeasyPrint==68.0`,
> moves to `pydyf==0.12.1` and **drops** `cairocffi`/`tinycss2` `[SOURCE-VERIFIED]`. If a
> Trustbit doc says WeasyPrint arrived with v16, it is wrong. What *is* new in v16 is the
> Chromium/CDP generator.

| | v15 (15.100.0) | v16 |
|---|---|---|
| Default engine | wkhtmltopdf via pdfkit | wkhtmltopdf via pdfkit — **still the default** `[SOURCE-VERIFIED]` |
| `pdf_generator` field | on Print Format, `"options": "wkhtmltopdf"` only `[V15-LOCAL print_format.json:268-271]` | on **Print Settings and Print Format**, `"options": "wkhtmltopdf\nchrome"`, `"default": "wkhtmltopdf"` `[SOURCE-VERIFIED]` |
| Chrome implementation | **absent** — no `frappe/utils/pdf_generator/`, no `get_chrome_pdf`, no chromium references `[V15-LOCAL, by grep]` | shipped: `frappe/utils/pdf_generator/{browser,cdp_connection,chrome_pdf_generator,page,pdf_merge}.py`, chrome-headless-shell over CDP websockets `[SOURCE-VERIFIED]` |
| Provisioning command | — | **`bench setup-chrome`** (bench-level, not site-level) `[SOURCE-VERIFIED]` |
| WeasyPrint | pinned 68.0, present | pinned 68.0, used by the Print Format Builder (beta) route `frappe.utils.weasyprint.download_pdf` `[SOURCE-VERIFIED]` |

Note the v15 type hint `pdf_generator: Literal["wkhtmltopdf", "chrome"] | None = None`
already exists at `[V15-LOCAL frappe/utils/print_format.py:239]` — the *extension point*
predates the engine. Do not read that as v15 supporting Chrome; nothing implements it.

### The trap: the generator is chosen per Print Format

```python
# v16 frappe/utils/print_utils.py:53                       [SOURCE-VERIFIED]
pdf_generator = frappe.get_cached_value("Print Format", print_format, "pdf_generator") or "wkhtmltopdf"
# printview.py                                             [KANTISHIVA]
getattr(print_format, "pdf_generator", "wkhtmltopdf")
```

Plus a migration patch,
`frappe.printing.doctype.print_format.patches.sets_wkhtmltopdf_as_default_for_pdf_generator_field`,
which explicitly writes `"wkhtmltopdf"` onto every existing Print Format row
`[SOURCE-VERIFIED]`. **So flipping the global Print Settings value does not change existing
formats** `[KANTISHIVA]` — you must set it per format, or the client will report "the PDF
still looks the same".

Worse for planning: the whitelisted `download_pdf` endpoint hardcodes `wkhtmltopdf` when
unset, and `frappe/utils/pdf.py::get_pdf()` — the terminal path for **all** server-side
PDF: email attachments, `frappe.attach_print`, Auto Email Reports, `report_to_pdf` — calls
`pdfkit.from_string(...)` with no Chrome path at all `[SOURCE-VERIFIED]`. **You cannot
delete wkhtmltopdf from a v16 box and expect emailed PDFs to work.**

### What that costs on Ubuntu 26.04

- **wkhtmltopdf has no apt candidate on 26.04 at all** `[KANTISHIVA]`. Upstream is
  archived; the newest `wkhtmltopdf/packaging` release is `0.12.6.1-3` and its newest
  Ubuntu target is **jammy** `[SOURCE-VERIFIED]`. The jammy .deb installs and runs fine on
  26.04 — verified by generating real PDFs `[KANTISHIVA]`. Check it reports
  `0.12.6.1 (with patched qt)`.
- **Chromium self-provisions but not its libraries.** `find_or_download_chromium_executable()`
  downloads Chrome for Testing into `<bench>/chromium` (default 133.0.6943.35), or honours
  a `chromium_path` key in `common_site_config.json` `[KANTISHIVA + SOURCE-VERIFIED]`. On a
  minimal server it then dies with:

  ```
  TimeoutError: Chromium took too long to start
  ```

  Fixed by installing: libnss3, libnspr4, libatk1.0-0t64, libatk-bridge2.0-0t64,
  libatspi2.0-0t64, libxcomposite1, libxdamage1, libxfixes3, libxrandr2, libxkbcommon0,
  libgbm1, libasound2t64, libcups2t64, libdrm2, libpango-1.0-0, libcairo2, libxshmfence1,
  libx11-xcb1 `[KANTISHIVA]`. Note the `t64` suffixes — 26.04 renamed those packages.
- **WeasyPrint fails silently until something calls it** `[KANTISHIVA]`:

  ```
  OSError: cannot load library 'libpangoft2-1.0-0'
  ```

  Fixed with libpangoft2-1.0-0, libpangocairo-1.0-0, libharfbuzz-subset0,
  libgdk-pixbuf-2.0-0, shared-mime-info. Because WeasyPrint is pinned in v15 too, a v15
  box on a minimal 24.04 image can hit exactly this — it just never came up on 22.04.
- Do not point `chromium_path` at the snap: 26.04's `chromium` is a snap transitional
  package and a snap-confined browser spawned from a supervisor-managed gunicorn worker
  hits AppArmor problems `[SOURCE-VERIFIED]`.
- Chromium is spawned as a subprocess of **each** gunicorn/RQ worker. On a 1 vCPU box set
  `bench set-config -g chromium_max_concurrent 1` `[SOURCE-VERIFIED, NOT RUN HERE]`.
- Config keys are `chromium_path`, `chromium_version`, `chromium_download_url`,
  `chromium_max_concurrent`, `use_persistent_chromium`, `chromium_start_timeout` in core
  frappe. The `print_designer` app uses a *different* key, `chromium_binary_path` — blog
  posts quoting that name against core frappe are wrong `[SOURCE-VERIFIED]`.

**Recommendation, as run at Kantishiva:** install both. Keep wkhtmltopdf as the default
(it is what ERPNext's print formats are tuned for, and it is the only engine on the email
path); provision Chrome and flip individual formats to it where you need modern CSS.

---

## 8. Asset builds got much heavier

**Applies to: v16.**

`bench build` passes `--run-build-command`, which makes `esbuild.js` execute `yarn build`
for any app that declares one. ERPNext v16 declares one `[SOURCE-VERIFIED]`:

```json
"scripts": {
    "postinstall": "cd banking && yarn install",
    "dev": "cd banking && yarn dev",
    "build": "cd banking && yarn build"
}
```

`erpnext/version-16/banking/package.json` is a **React 19.2 + Vite 8.0 + Tailwind 4.3**
app (`vite build --base=/assets/erpnext/banking/`). **erpnext 15.97.0's `package.json` has
no `scripts` block at all** — only `dependencies: {onscan.js}` `[V15-LOCAL, verified by
reading the file]`. This build simply did not exist in v15.

It matters because esbuild proper is a **Go binary in a separate process** and is not
constrained by the Node heap, whereas **Vite/Rollup runs inside the Node heap** — so the
banking build is the actual OOM point on a small VPS `[SOURCE-VERIFIED]`.

> **Correction — the heap cap is not a v16 change.** `get_safe_max_old_space_size()`
> (`max(1024, int(total_memory * 0.75))`, physical RAM only, swap not counted) and the
> `env = dict(environ, **env)` override that makes your exported `NODE_OPTIONS` useless
> are both present in **v15** at `[V15-LOCAL frappe/build.py:290-306 and
> frappe/commands/__init__.py:66-69]`. Identical logic in v16. What changed is the amount
> of work being fed into that budget.

On the 3911 MB Kantishiva box the computed ceiling is `--max_old_space_size=2933` while
MariaDB and Redis are also resident `[SOURCE-VERIFIED arithmetic: int(3911*0.75)]`. V8 will
not GC hard until it approaches that, but the kernel OOM killer fires long before — and
may kill MariaDB rather than node.

The working recipe — swap-on and per-app builds are what Kantishiva did `[KANTISHIVA]`;
stopping MariaDB during the build is `[SOURCE-VERIFIED]` advice we did not need there:

```bash
# 1. swap FIRST, before anything memory-hungry  (4 GB on the 3.9 GB box)
swapon --show                        # must be empty before you create /swapfile
# 2. defer assets out of bench init so a failure costs minutes, not the env
bench init frappe-bench --frappe-branch version-16 --python /usr/bin/python3 --skip-assets
# 3. build per app — the erpnext/banking Vite build is the peak
systemctl stop mariadb               # optional; bench build needs no database
bench build --production --apps frappe
bench build --production --apps erpnext
systemctl start mariadb
```

Two more facts worth knowing before you tune: `sourcemap: true` is set unconditionally, so
`--production` does not turn sourcemaps off (fixed memory and disk cost), and each app
builds JS + LTR CSS + RTL CSS concurrently while apps iterate sequentially — which is why
`--apps` genuinely lowers the peak `[SOURCE-VERIFIED: esbuild.js:249,255,323,326]`.

Distinguish the two exit codes: **137 is a real OOM kill**; **143 is SIGTERM** and on our
boxes has meant a half-registered app missing from `sites/apps.txt`, not memory — see the
v15 file, Lesson 3.

---

## 9. Smaller deltas that will still surprise you

**Applies to: v16 unless noted.** All `[SOURCE-VERIFIED]`.

| Area | v15 → v16 |
|---|---|
| Python deps **added** | `pyarrow~=25.0.0`, `duckdb~=1.4.3`, `orjson~=3.11.5`, `nh3~=0.3.2`, `xlsxwriter~=3.2.9`, `pycountry~=24.6.1`, `distro~=1.9.0`, `websockets~=15.0.1` |
| Python deps **removed** | `boto3`, `dropbox`, `posthog`, `maxminddb-geolite2`, `cairocffi`, `tinycss2` — **S3/Dropbox backup integrations moved out of core**. If a client relies on those, that is a separate app now. |
| Major bumps | `rq 1.15.1 → 2.6.1`, `redis 4.5 → 7.1`, `hiredis 2.2 → 3.3`, `cryptography 44 → 50` |
| PyPika | now a frappe **git fork** pin — `git` must be reachable at install time (as `gunicorn` already was in v15) |
| Redis protocol | v16 calls `_enable_client_tracking()`, which needs **RESP3**. Against a RESP2-only server `bench new-site` aborts with `Client tracking is currently not supported for RESP2. Please use RESP3.` Ubuntu 26.04's redis 8.0.5 is fine; do not swap in valkey or a managed/proxied Redis without checking. |
| Redis instances | unchanged — two bench-managed instances (cache 13000, queue 11000); `redis_socketio` is a legacy alias force-synced to `redis_cache`, not a third server |
| New DB backend | `bench new-site --db-type` now accepts `mariadb\|postgres\|sqlite` |
| `bench new-site` flags | `--db-root-username` / `--db-root-password` (the `--mariadb-*` names still work); `--no-mariadb-socket` deprecated in favour of `--mariadb-user-host-login-scope`. Run `bench new-site --help` first — the names have moved once already. |
| `socketio_backend` | new key in `common_site_config.json`; `"python"` is reserved for a `frappe.realtime.server` that **does not exist** in v16. Do not set it. |
| `[tool.uv] exclude-newer` | v16's pyproject carries `exclude-newer = "7 days"`. Whether bench's `uv pip install` honours it is `[UNVERIFIED]` — if it does, resolution deliberately ignores anything published in the last week. Only matters when chasing a just-released fix. |
| Click | v16 pins `Click~=8.3.1` while bench 5.31.0 wants `click~=8.2.0`; expect a cosmetic pip resolver complaint. Do not train yourself to ignore resolver errors — a real one looks identical. |
| Production wiring | **unchanged.** `bench setup production <user>` still means supervisor + nginx, same program names, systemd still opt-in. |
| `bench setup production` | still aborts on `bench setup role fail2ban` (needs Ansible) — true on v15's bench 5.29.0 too. Install fail2ban with apt and run `bench setup supervisor --yes` / `bench setup nginx --yes` individually. |
| nginx | **not a v16 change.** `unknown log format "main"` fires whenever `bench setup nginx` is run standalone — fix with `/etc/nginx/conf.d/00-log-format.conf`, which sorts before `frappe-bench.conf` `[KANTISHIVA]`. **v15's bench 5.29.0 has the same defaults** (`bench setup nginx` → `--logging combined`, `--log_format main`, `bench/commands/setup.py:28-41` → `config/nginx.py:49-53` → `templates/nginx.conf:124`); only `bench setup production` escapes it, because it calls `make_nginx_conf(bench_path, yes=yes)` with no `logging` argument. |

---

## 10. Should I upgrade, or start new on v16?

**Applies to: both.** This is the section to read before quoting a client.

| Situation | Do this |
|---|---|
| New client, no custom app, no legacy data | **v16 on Ubuntu 26.04.** No reason to start on v15 in 2026. |
| New client, wants one of our existing apps (`trustbit_mandi`, `item_creator`, …) | **v15 on 22.04/24.04**, or budget a real port. The apps are not v16-ready today. |
| Existing v15 client, running fine | **Leave it.** There is no operational pressure; v15 boxes need no OS change. |
| Existing v15 client on 22.04 nearing EOL | Move to **24.04 on v15** (Python 3.12 satisfies `>=3.10,<3.14`) — that is a small job. 26.04 forces the v16 migration whether you wanted it or not. |
| Client explicitly wants v16 features + has custom apps | Scope it as a migration project with its own budget. Do not fold it into a hosting move. |

### Why v15-built apps are not drop-in

There is no shared runtime. erpnext v16 pins `frappe >=16.21.0,<17.0.0` and
`requires-python >=3.14`, while v15 frappe pins `>=3.10,<3.14` `[SOURCE-VERIFIED +
V15-LOCAL]` — an app cannot straddle the two on one bench. Concretely, before promising a
port, check the app for:

1. **Python 3.14 compatibility** of the app itself and any pip dependency it declares.
   A well-behaved app has no upper pin — `item_creator`'s `pyproject.toml` says
   `requires-python = ">=3.10"` with no ceiling `[verified in the app]`, so syntax is not
   the blocker. The blocker is API.
2. **Core deps that v16 removed.** Anything importing `boto3`, `dropbox`, `posthog`,
   `maxminddb-geolite2`, `cairocffi` or `tinycss2` "because frappe brings it in" now has
   an unmet import at runtime.
3. **Client-library bumps.** `redis 4.5 → 7.1` and `rq 1.15 → 2.6` are majors. Any app
   touching the queue or cache directly needs review.
4. **Front-end build.** Any app with a `package.json` build now runs on Node 24; a v15-era
   toolchain may not.
5. **Third-party apps you don't control.** `india_compliance` had **no confirmed
   version-16 branch** at the time of the research pass `[UNVERIFIED — not checked
   upstream]`. For an Indian client, GST compliance is a hard gate: confirm it exists
   before you agree a date. Same question for LMS, HRMS, or anything else in the bench.
6. **Fixtures, Custom Fields and DocPerm patterns** are not version-portable assumptions —
   revalidate against v16 rather than assuming the v15 `after_migrate` hooks still fire the
   same way.

### What we actually decided at Kantishiva

The v16 box was built **fresh**. The v15 apps (`trustbit_mandi`, `item_creator`) were
**not** installed on it, and the Mandi v15 database backup was **not** restored — both were
explicitly recorded as pending decisions, because moving them is "a real migration with
breaking API changes, not a reinstall" `[KANTISHIVA]`. That was the right call and it is
the default posture: **a v16 build is a new system, not an upgrade, until someone has done
the port work.**

### Honest gaps

- **We have never run an in-place v15 → v16 upgrade.** No transcript exists in any Trustbit
  source. Nothing in this repo tells you what `bench update --to-version` or a
  restore-then-migrate does to a real client database. `[UNVERIFIED]`
- **We have never restored a v15 dump into a v16 site.** Note that it is a MariaDB jump too
  (10.6 → 11.8) with a collation-default change in between (§6), so it is two migrations
  wearing one coat.
- **The fresh v16 build was not hard, but it was slow.** Almost every failure at Kantishiva
  was OS/stack-level (uv, PEP 668, node, MariaDB config, wkhtmltopdf, chromium libs,
  tzdata) rather than ERP-level. Budget the first v16 build of any new OS release
  generously and the second one cheaply.

---

## 11. Greppable error index — v16 vs v15

| Error string (literal) | Version | Cause → fix |
|---|---|---|
| `FileNotFoundError: 'uv'` | v16 (bench ≥5.28) | bench builds the env with uv and does not install it → §4 |
| `error: externally-managed-environment` | v16 (PEP 668 OS) | system Python is 3.14 and apt-managed → install bench via uv/pipx, never `--break-system-packages` → §2 |
| `Exception: pkg-config is not installed. Please install it before proceeding.` | v16 | bench's `check_pkg_config()` guard for frappe ≥16 → §5 |
| `Can not find valid pkg-config name. Specify MYSQLCLIENT_CFLAGS and MYSQLCLIENT_LDFLAGS env vars manually` | v16 | `libmariadb-dev` missing; mysqlclient compiles from sdist → §5 |
| `fatal error: Python.h: No such file or directory` | v16 | `python3-dev` missing → §5 |
| `Please install yarn using below command and try again.` | both | yarn classic missing; `--skip-assets` does not skip yarn → §3 |
| `Warning: MariaDB version ... is newer than 11.8 which is not yet tested with Frappe Framework.` | v16 | MariaDB's own apt repo installed 12.x → use Ubuntu's 11.8.6 → §6 |
| `Warning: MariaDB version ... is more than 10.8 which is not yet tested with Frappe Framework.` | v15 | v15's ceiling is 10.8 → §6 |
| `Illegal mix of collations` (error 1267) | v16 / MariaDB 11.8 | 11.8 default collation is `utf8mb4_uca1400_ai_ci`; set `collation-server` explicitly → §6 |
| `Client tracking is currently not supported for RESP2. Please use RESP3.` | v16 | Redis server too old / proxied → §9 |
| `TimeoutError: Chromium took too long to start` | v16 | chrome-headless-shell downloaded but its shared libraries are missing → §7 |
| `OSError: cannot load library 'libpangoft2-1.0-0'` | both (bites on v16) | WeasyPrint's native stack absent; nothing complains until something calls it → §7 |
| `zoneinfo._common.ZoneInfoNotFoundError: 'No time zone found with key Asia/Calcutta'` | both, Ubuntu 24.04+ | deprecated tz aliases moved to `tzdata-legacy` → §2 |
| `ModuleNotFoundError: No module named 'tzdata'` | same | pip `tzdata` missing from the bench venv → §2 |
| `unknown log format "main"` | v16 default path | bench emits `access_log ... main`; Debian/Ubuntu nginx defines no such format → §9 |
| exit code `137` on `bench build` | v16 in practice | genuine OOM kill during the erpnext Vite build → §8 |
| exit code `143` on `bench build` | both | SIGTERM, **not** OOM — usually an app missing from `sites/apps.txt` → v15 file, Lesson 3 |

---

## 12. Still open

- In-place v15 → v16 upgrade: never attempted here. `[UNVERIFIED]`
- v15 dump → v16 site restore, and what `bench migrate` does to it. `[UNVERIFIED]`
- v16 on any OS other than Ubuntu 26.04 (i.e. via `uv python install 3.14`). `[UNVERIFIED]`
- `india_compliance` version-16 availability. `[UNVERIFIED]` — check upstream before
  committing to any Indian client's v16 date.
- Whether `[tool.uv] exclude-newer = "7 days"` actually applies during `bench init`. `[UNVERIFIED]`
- Whether `bench setup-chrome`'s pinned Chromium 133.0.6943.35 will keep downloading from
  Google's CDN. Kantishiva got it on 2026-08-17; that URL is not forever.
