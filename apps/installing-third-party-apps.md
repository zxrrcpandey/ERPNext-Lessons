# Installing Third-Party Apps (`bench get-app`)

Applies to: **v15 and v16** — each numbered lesson carries its own marking where the
versions differ; the reference sections (§7–9) apply to both.

Installing an app you don't control — `india_compliance`, HRMS, LMS — onto an existing
production bench. The failure modes are different from building your own app
([custom-app-development.md](custom-app-development.md)) because you cannot change the
app's pins, its dependencies, or its `after_install` hook.

**Reference implementation:** `india_compliance` **v16.8.4** installed onto
`kantishiva.trustbit.cloud` on **2026-08-17** — frappe 16.31.0 / erpnext 16.32.1,
bench 5.31.0, Ubuntu 26.04, Python 3.14.4, 1 vCPU / 3.9 GB + 4 GB swap.

Everything below marked **[V]** was read off that box or off the bench source in
`/opt/pipx/venvs/frappe-bench/lib/python3.14/site-packages/bench/` during that install.
Nothing here is from documentation.

---

## 1. Pick the branch from the app's own frappe pin, not from its README

**Applies to: v15 and v16.**

A third-party app's version line must match the bench's frappe major. The authoritative
statement is in the app's `pyproject.toml`, not its release notes:

```toml
# india_compliance version-16 branch, pyproject.toml  [V]
[tool.bench.frappe-dependencies]
frappe = ">=16.0.0,<17.0.0"
```

`[tool.bench.frappe-dependencies]` is bench's own compatibility table. Read it **before**
`get-app`, because a mismatched branch installs happily and then fails at runtime.

For `india_compliance` the two live branches are `version-15` and `version-16` **[V]**:

| Bench | Branch | Version at time of writing | Requires |
|---|---|---|---|
| frappe 15.x | `version-15` | 15.25.4 | frappe >=15 |
| frappe 16.x | `version-16` | **16.8.4** (= tag `v16.8.4`) | `frappe >=16.0.0,<17.0.0` |

Confirm the branch exists and what it points at **without cloning** — cheap, and it also
proves the upstream URL:

```bash
git ls-remote --heads https://github.com/resilient-tech/india-compliance.git \
  | grep -E 'version-1[56]'
git ls-remote --tags  https://github.com/resilient-tech/india-compliance.git \
  | grep -v '\^{}' | tail -5
```

On 2026-08-17 `refs/heads/version-16` and `refs/tags/v16.8.4` were the **same commit**
`d6666c2` **[V]** — i.e. the branch head was exactly a release, not a random dev commit.
Check that before trusting a branch in production; if the branch is ahead of the newest
tag you are shipping unreleased code.

> Upstream remote for `india_compliance` is
> **`https://github.com/resilient-tech/india-compliance.git`** — **[V]**, `git ls-remote`
> succeeded against it. (This supersedes the `[unverified URL]` note carried in
> [../v15/install-and-gotchas.md](../v15/install-and-gotchas.md) §2.3.)

Also read `required_apps` in the app's `hooks.py` — `india_compliance` declares
`required_apps = ["frappe/erpnext"]` **[V]**, so erpnext must already be on the site.

---

## 2. `bench get-app` ends in a traceback that is **not** a failure

**Applies to: v16 (bench 5.31.0); the code path exists in v15 bench too.**
This is the single most misleading thing about installing an app on a production bench.

### Symptom

`get-app` does all its work, then dies with a Python traceback and a non-zero exit:

```
  File ".../bench/app.py", line 974, in install_app
    bench.reload(_raise=False)
  File ".../bench/bench.py", line 156, in reload
    restart_supervisor_processes(bench_path=self.name, web_workers=web, _raise=_raise)
  File ".../bench/utils/bench.py", line 350, in restart_supervisor_processes
    supervisor_status = get_cmd_output("sudo supervisorctl status", cwd=bench_path)
subprocess.CalledProcessError: Command 'sudo supervisorctl status' returned non-zero exit status 1.
```

### Root cause

`install_app()` finishes the clone and the pip install, and only *then* calls
`bench.reload()` **[V]** — `bench/app.py:968-974`:

```python
	if not skip_assets:
		build_assets(bench_path=bench_path, app=app, using_cached=using_cached)

	if restart_bench:
		# Avoiding exceptions here as production might not be set-up
		# OR we might just be generating docker images.
		bench.reload(_raise=False)
```

Note the comment: bench *intends* this to be non-fatal, and passes `_raise=False`. It
still blows up, because of how `restart_supervisor_processes` computes its status
(`bench/utils/bench.py:336-346`) **[V]**:

```python
		sudo = ""
		try:
			supervisor_status = get_cmd_output("supervisorctl status", cwd=bench_path)
		except subprocess.CalledProcessError as e:
			if e.returncode == 127:
				log("restart failed: Couldn't find supervisorctl in PATH", level=3)
				return
			sudo = "sudo "
			supervisor_status = get_cmd_output("sudo supervisorctl status", cwd=bench_path)
```

The retry on the last line is **outside** any `try`, and `_raise` is never consulted on
this path. So:

1. `supervisorctl status` as `frappe` → permission denied on the supervisor socket,
   exit 1 (**not** 127, so the early `return` doesn't fire).
2. bench retries with `sudo`.
3. `sudo` needs a password non-interactively → exit 1 → uncaught `CalledProcessError`.

`_raise=False` is real but **cannot** protect this, because the exception happens while
*computing* `supervisor_status`, before any `_raise`-aware restart call is reached.

On Kantishiva `frappe` **is** in the `sudo` group, and it still fails **[V]**:

```
$ sudo -u frappe sudo -n true
sudo: interactive authentication is required
```

Group membership is not the issue — the absence of a **NOPASSWD** rule is.

### Why `bench setup sudoers` does not fix it

The obvious remedy is wrong, and this is worth knowing before you reach for it. bench's
own template (`bench/config/templates/frappe_sudoers`) **[V]** grants:

```jinja
{{ user }} ALL = (root) NOPASSWD: {{ service }} nginx *
{{ user }} ALL = (root) NOPASSWD: {{ systemctl }} * nginx
{{ user }} ALL = (root) NOPASSWD: {{ nginx }}
{{ user }} ALL = (root) NOPASSWD: {{ certbot }}
Defaults:{{ user }} !requiretty
```

**nginx and certbot only — `supervisorctl` is not in the template at all.** Running
`bench setup sudoers frappe` will not stop this traceback. (On Kantishiva
`/etc/sudoers.d/frappe` did not exist anyway: the build had aborted `bench setup
production` at its fail2ban/Ansible step, per the v16 runbook.)

### Fixes, in order of preference

**(a) Do nothing; verify by hand and restart as root.** What we did. The traceback is
cosmetic; the app is installed.

```bash
supervisorctl restart all          # as root, after the get-app
```

**(b) Set `supervisor_restart_cmd`** in `sites/common_site_config.json`. When present,
bench takes a different branch entirely (`bench/utils/bench.py:331-332`) **[V]**:

```python
	if cmd:
		bench.run(cmd, _raise=_raise)
```

That call *does* honour `_raise`, which `get-app` sets to `False` — so a failure becomes
a silent no-op instead of a traceback.

**(c) Grant the one missing rule** if you want `bench restart` to work as `frappe`:

```
frappe ALL = (root) NOPASSWD: /usr/bin/supervisorctl
```

in a `visudo -f /etc/sudoers.d/frappe-supervisor` file. Widens the trust boundary — read
[../operations/security.md](../operations/security.md) before doing this on a client box.

### The rule this implies

**Never judge `bench get-app` by its exit code.** Verify the three things it actually
had to do:

```bash
ls apps/                                                    # folder present
grep -x india_compliance sites/apps.txt                     # registered with bench
env/bin/python -c "import india_compliance; print(india_compliance.__version__)"
sudo -u frappe git -C apps/india_compliance describe --tags  # right commit
```

On Kantishiva all four passed — `16.8.4`, tag `v16.8.4`, branch `version-16` — despite
the non-zero exit **[V]**.

> Contrast this with the v15 lesson "[a failed `bench get-app` leaves a half-registered
> app](../v15/install-and-gotchas.md)". Both exist: `get-app` can fail *early* and leave
> real breakage, or fail *late* and leave nothing wrong. The exit code does not
> distinguish them — the four checks above do.

---

## 3. Split the fetch from the asset build

**Applies to: v16, and any box under ~4 GB RAM.**

```bash
sudo -u frappe bench get-app --branch version-16 --skip-assets \
    https://github.com/resilient-tech/india-compliance.git
sudo -u frappe bench --site <site> install-app india_compliance
sudo -u frappe bench build --app india_compliance
chown -R frappe:frappe /home/frappe/frappe-bench
supervisorctl restart all
```

`--skip-assets` makes `get-app` skip `build_assets()` (see the `if not skip_assets:` guard
in §2). The point is **blast radius**: an OOM kill during the build cannot take the clone
and pip install with it, so a retry is just the build, not the whole fetch.

For `india_compliance` the build turned out to be cheap — **4.79 s**, 5 JS + 4 CSS
bundles, largest 140.77 Kb **[V]**:

```
india_compliance/dist/js/
├─ gstr1.bundle.*.js                                8.61 Kb
├─ ims.bundle.*.js                                 14.22 Kb
├─ india_compliance.bundle.*.js                    51.81 Kb
├─ purchase_reconciliation_tool.bundle.*.js        12.14 Kb
└─ india_compliance_account.bundle.*.js           140.77 Kb
 DONE  Total Build Time: 4.790s
```

Nothing like erpnext v16's React/Vite build, which is the one that OOM-kills a 1 vCPU box
(exit 137 — see [../v16/install-ubuntu-2604.md](../v16/install-ubuntu-2604.md)). Splitting
the steps still costs nothing, so keep doing it: you don't know an app's build weight
until after you've run it.

App footprint on disk: **16 MB** **[V]**.

---

## 4. Pinned C-extension deps compile from source on Python 3.14

**Applies to: v16 (Python 3.14).** A v15-era app pins its dependencies to versions that
predate 3.14, so no `cp314` wheel exists and pip/uv falls back to a source build.

`india_compliance` declares **[V]**:

```toml
dependencies = [
    "python-barcode~=0.15.1",
    "titlecase~=2.3",
    "pycryptodome~=3.19.0",
    "pypng~=0.20220715.0",
]
```

`pycryptodome~=3.19.0` means `>=3.19.0,<3.20.0` — a release line from 2023, two years
before Python 3.14. It is a **C extension**. The other three are pure Python and are
never a problem.

It built without complaint on Kantishiva, producing **pycryptodome 3.19.1** under gcc
**15.2.0** **[V]** — but only because the toolchain was already there from the v16 build:

```bash
# check BEFORE get-app, not after it fails
gcc --version
ls /usr/include/python3.14/Python.h
dpkg -l | grep -E 'build-essential|python3.14-dev'
```

If `Python.h` is missing you get the familiar `fatal error: Python.h: No such file or
directory` (already in the v16 error index). Install `build-essential python3.14-dev`
first. This is the same class of problem as `mysqlclient` needing `libmariadb-dev` — on
3.14, assume a source build for anything with a C extension and an old pin.

> **Do not "fix" this by relaxing the app's pin.** You do not control the app; the next
> `bench update` reverts it. Install the build toolchain instead.

---

## 5. `install-app` runs the app's `after_install`, and it needs a real Company

**Applies to: v15 and v16.** For compliance apps this is the difference between an app
that is installed and an app that works.

`india_compliance` hooks **[V]**:

```python
after_install = "india_compliance.install.after_install"
after_migrate = "india_compliance.audit_trail.setup.after_migrate"
```

A clean `install-app` prints its progress **[V]**:

```
Updating DocTypes for india_compliance: [========================================] 100%
Setting up Income Tax...
Setting up GST...
Patching Existing Data...
Thank you for installing India Compliance!
```

`Setting up GST` and `Patching Existing Data` provision **against the Companies that
already exist**. Install onto a site whose setup wizard never completed and those steps
have nothing to attach to: you get the doctypes and none of the accounting scaffolding.

**So: finish the setup wizard before installing a compliance app, not after.**

Verified provisioning on Kantishiva, against `Kantishiva Roller Flour Mill Pvt. Ltd.`
(abbr `KRFMPL`, country India, INR) **[V]**:

| What | Result |
|---|---|
| GST HSN Codes | **18,687** |
| Custom Fields (site total) | 473 — **13 on Company**, incl. `gstin`, `gst_category`, `pan` |
| GST accounts | 16 (`Output/Input Tax CGST/SGST/IGST [RCM] - KRFMPL`, `GST Expense`) |
| Sales tax templates | 4 (`Output GST In-state`, `Out-state`, `RCM In-state`, `RCM Out-state`) |
| Purchase tax templates | 4 (`Input GST …` mirror set) |
| GST Settings `gst_accounts` | 4 rows |
| Doctypes registered | 26 across modules `GST India`, `Income Tax India`, `Audit Trail`, `VAT India` |

Check it the same way — through the ORM, not the desk:

```bash
cd /home/frappe/frappe-bench/sites && sudo -u frappe ../env/bin/python -c "
import frappe; frappe.init(site='<site>'); frappe.connect()
print('HSN:', frappe.db.count('GST HSN Code'))
c = frappe.get_doc('Company', '<company>')
print('gstin:', repr(c.get('gstin')), 'category:', repr(c.get('gst_category')))
print(frappe.get_all('Sales Taxes and Charges Template', pluck='name'))
"
```

> **`bench console` swallows output when fed on stdin.** Piping a heredoc into
> `bench --site <site> console` printed only the first few lines of a 12-line script and
> silently dropped the rest **[V]**. Use `env/bin/python` with `frappe.init()/connect()`
> from the `sites/` directory for any scripted verification. `bench execute` also works
> but only calls one function.

### Installed ≠ configured

Immediately after a clean install, Kantishiva showed **[V]**:

```
gstin: None      gst_category: 'Unregistered'
enable_e_waybill: 1   enable_api: 1   enable_e_invoice: 0
```

Those `enable_*` values are **post-install defaults**, not a working configuration. With
`Company.gstin` unset, e-Waybill/e-Invoice and GSTR filing are inert. Entering a real
GSTIN needs the client, so treat "app installed" and "compliance live" as two separate
milestones on the plan — and never invent a GSTIN to make a screen look finished.

---

## 6. Root-run commands re-poison the bench every time

**Applies to: v15 and v16.** Already rule #4 in the repo README; third-party installs are
where it recurs, because you keep reaching for verification one-liners as root.

A **single** root-run `python -c "import india_compliance"` left **1403 root-owned files**
in the bench **[V]** — the import wrote `__pycache__/*.pyc` throughout `env/site-packages`
as root. Nothing fails at that moment; it surfaces later as a `PermissionError` in a
worker.

```bash
find /home/frappe/frappe-bench -not -user frappe | wc -l   # expect 0
chown -R frappe:frappe /home/frappe/frappe-bench
```

**Exception — do not chase these:** `config/pids/*.pid` and `config/pids/*.rdb` are
root-owned and get recreated on every `supervisorctl restart all`, because supervisor
starts the bench's redis as root **[V]**. That is normal. Filter them out:

```bash
find /home/frappe/frappe-bench -not -user frappe -not -path '*/config/pids/*' | wc -l
```

---

## 7. Post-install checklist

Run all of it; each line caught something real at least once.

```bash
sudo -u frappe bench --site <site> list-apps      # app + version + branch
supervisorctl status                              # all 7 RUNNING
sudo -u frappe bench --site <site> scheduler status
curl -s https://<site>/api/method/ping            # {"message":"pong"}
curl -so /dev/null -w '%{http_code}\n' https://<site>/assets/<app>/dist/js/<bundle>.js
find /home/frappe/frappe-bench -not -user frappe -not -path '*/config/pids/*' | wc -l
tail -100 logs/worker.error.log                   # compare timestamps to your install
```

Kantishiva result **[V]**: `india_compliance 16.8.4 version-16`, 7/7 RUNNING, scheduler
enabled, `pong` at HTTP 200, asset bundles 200 over HTTPS, 0 stray-owned files, and no new
`worker.error.log` entries during the install window.

> **Read log timestamps before panicking.** The `worker.error.log` tail showed
> `ZoneInfoNotFoundError: 'Asia/Calcutta'` and Redis `Error 111` lines that looked like
> fresh damage. They were from 10:44 and 11:35; the install ran at 13:48 **[V]**. Grep
> the log for your actual install window rather than reacting to `tail`.

**Take a database backup immediately before `install-app`** — that is the step that
mutates the site (doctypes, custom fields, accounts, tax templates):

```bash
sudo -u frappe bench --site <site> backup
```

`install-app` has no clean `uninstall` that reverses `after_install` side effects, so the
dump is the rollback. See
[../operations/backup-restore-and-migration.md](../operations/backup-restore-and-migration.md).

---

## 8. Greppable error index

| Error string (literal) | When | Cause → fix |
|---|---|---|
| `subprocess.CalledProcessError: Command 'sudo supervisorctl status' returned non-zero exit status 1.` | end of `bench get-app` | `frappe` can't sudo non-interactively; `_raise=False` doesn't cover this path. **Work already done** → §2 |
| `sudo: interactive authentication is required` | `sudo -u frappe sudo …` | in the `sudo` group but no NOPASSWD rule; `bench setup sudoers` doesn't add supervisorctl → §2 |
| `fatal error: Python.h: No such file or directory` | `get-app` pip step | `python3.14-dev` missing; a pinned C ext (e.g. `pycryptodome~=3.19.0`) has no cp314 wheel → §4 |
| `fatal: detected dubious ownership in repository at '…/apps/<app>'` | `git -C apps/<app> …` as root | git refusing another user's repo — run git as `frappe`, don't add a `safe.directory` exception → §6 |
| exit code `137` on `bench build` | asset build | genuine OOM → build per-app, keep swap on → §3 |
| `TypeError: paths[0] must be of type string` | `bench build --app <app>` | app absent from `sites/apps.txt` after an *early* `get-app` failure → [../v15/install-and-gotchas.md](../v15/install-and-gotchas.md) |

---

## 9. Still open

- **Uninstalling.** `bench --site <site> uninstall-app india_compliance` was **not** run.
  Whether it reverses the 473 custom fields / 16 accounts / tax templates, or leaves
  orphans, is `[UNVERIFIED]`. Assume restore-from-backup is the only clean rollback.
- **`bench update` with a third-party app present.** Not exercised on v16 here.
  `[UNVERIFIED]`
- **India Compliance API features.** `enable_api = 1` by default, but no GSTIN and no API
  credentials were configured, so e-Waybill / e-Invoice / GSTR sync are entirely
  unexercised. `[UNVERIFIED]`
- **Option (b) in §2** (`supervisor_restart_cmd` to silence the reload traceback) is read
  from the bench source and is sound, but was **not run** on a live box. `[UNVERIFIED]`
