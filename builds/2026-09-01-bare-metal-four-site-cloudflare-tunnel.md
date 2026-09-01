# Build log — four v15 demo sites on bare metal, published via Cloudflare Tunnel

**Date:** 2026-09-01 · **Duration:** one session, Windows Server still booted at the start,
four sites live on the public internet at the end.

A complete, chronological record of a **bare-metal** ERPNext build: wiping an out-of-support
HP tower, patching its firmware, installing Ubuntu 24.04, standing up four v15 tenants, and
publishing them through a **Cloudflare Tunnel with zero inbound ports**.

Every error below is reproduced **verbatim** so it is greppable. Fourteen distinct failures
are recorded, including one that is a **correction to this repo's own existing docs**
(§9) and one that is a **field confirmation** of an existing `[verified]` claim (§8).

**The target**

| | |
|---|---|
| Hardware | HP ProLiant ML10 Gen9 tower, Xeon E3-1225 v5 (4C/4T, no HT), 32 GB ECC, single 240 GB consumer SATA SSD |
| Out-of-band | **No iLO.** Intel AMT only. No serial port. DisplayPort-only video. |
| Was running | Windows Server (117 GB used, SQL Server 2022 + POS printer drivers present) |
| Became | Ubuntu 24.04.4 LTS, Frappe 15.119.1 / ERPNext 15.120.0, bench 5.31.0 |
| Sites | `demo1`–`demo4.<domain>` — four tenants, one bench, `dns_multitenant on` |
| Exposure | Cloudflare Tunnel (`cloudflared` 2026.8.3), **no port forwarding, no static IP** |
| Uplink | Indian consumer broadband, **dynamic IP**, outbound port 25 blocked |

**Confidence tags** follow this repo's convention: `[verified]` run or read in source during
this build · `[source: X]` quoted, not re-run · `[unverified]` plausible, unconfirmed.

---

## Contents

| § | Failure | New to this repo? |
|---|---|---|
| 1 | `dd: ~/Downloads/…: No such file or directory` (zsh) | new |
| 2 | `Operation not permitted` writing to `/dev/rdiskN` | new |
| 3 | Intel AMT on firmware Intel publishes as its *vulnerable example* | new |
| 4 | `Error 8193: Fail to load MEI device driver` | new |
| 5 | `Error 8716: Invalid usage` — `-info` is not a flag | new |
| 6 | apt killed by a 2-minute SSH timeout, dpkg left locked | new |
| 7 | `FileNotFoundError: [Errno 2] No such file or directory: 'uv'` | confirms + extends |
| 8 | `[emerg] unknown log format "main"` | **field confirmation** of existing `[V]` |
| 9 | socketio `BACKOFF` — **new failure mode**, supersedes existing guidance | **CORRECTION** |
| 10 | ERPNext assets never built | new |
| 11 | `stat() failed (13: Permission denied)` — `HOME_MODE 0750` | **new, highest value** |
| 12 | Cloudflare served **cached 404s** for a month-long TTL | new |
| 13 | API token pasted into a chat transcript | process |
| 14 | Site naming — no `bench rename-site` in v15 | new |

---

## 1. `dd` on macOS/zsh silently receives a literal tilde

Writing the Ubuntu installer to USB from a Mac:

```bash
sudo dd if=~/Downloads/ubuntu-24.04.4-live-server-amd64.iso of=/dev/rdisk12 bs=4m
```

```
dd: ~/Downloads/ubuntu-24.04.4-live-server-amd64.iso: No such file or directory
```

The file existed. **zsh does not perform tilde expansion after an `=` inside a word**
unless `MAGIC_EQUAL_SUBST` is set; bash does. So `dd` received the four characters `~/Do…`
as a literal path. Demonstrated side by side: **[verified]**

```console
$ zsh  -c 'echo if=~/Downloads/x.iso'
if=~/Downloads/x.iso                    # literal tilde
$ bash -c 'echo if=~/Downloads/x.iso'
if=/Users/you/Downloads/x.iso           # expanded
```

**Fix.** Use an absolute path, or `"$HOME/…"`:

```bash
sudo dd if=/Users/you/Downloads/ubuntu-24.04.4-live-server-amd64.iso of=/dev/rdisk12 bs=4m status=progress
```

Every `dd`-to-USB instruction written for bash is wrong on a default macOS shell, which has
been zsh since Catalina.

**Also:** macOS `dd` **does** support `status=progress` (tested on Darwin 25) **[verified]**,
contrary to a lot of advice. `Ctrl+T` (SIGINFO) also prints a progress line at any time.

---

## 2. `Operation not permitted` writing the raw device

```
Unmount failed for /dev/disk12
sh: /dev/rdisk12: Operation not permitted
Failed to find disk /dev/disk12
```

Two causes, and they mask each other:

1. **The device was already ejected.** The first command block ended with
   `diskutil eject`, which detaches the node entirely. Everything afterwards addressed a
   device that no longer existed.
2. If the device *is* present and this persists, **Terminal lacks Full Disk Access**
   (System Settings → Privacy & Security → Full Disk Access). Root is not sufficient on
   modern macOS. **[unverified — not needed here, the eject explained it]**

**Fix.** Physically replug, **re-check the identifier** (it can change), and do not eject
until after the write:

```bash
diskutil list external physical            # confirm by SIZE, every time
diskutil unmountDisk /dev/disk12
sudo dd if=/abs/path.iso of=/dev/rdisk12 bs=4m status=progress
diskutil eject /dev/disk12                 # only now
```

`rdisk` (raw) rather than `disk` is roughly 10× faster. A correct write on a 30 GB stick
produced GPT + an `EFI ESP` partition, which is what confirms it will boot in pure UEFI:

```
GUID_partition_scheme
  1: Microsoft Basic Data  3.4 GB   (ISO payload)
  2: EFI ESP               5.2 MB   ← required for UEFI boot with CSM disabled
```

macOS then offers **"The disk you inserted was not readable."** — click **Ignore**, never
**Initialize**, which would destroy the stick you just wrote.

---

## 3. Intel AMT running firmware that Intel publishes as its *vulnerable example*

The box reported:

```
Intel AMT firmware : 11.0.0.0005
Intel ME version   : 11.0.0.1202
AMT features       : SOL, IDE-R, KVM — all enabled
Port 16992         : listening
```

`11.0.0.1202` is not merely *in range* for **CVE-2017-5689 / INTEL-SA-00075** — it is the
exact value Intel's own SA-00075 Detection Guide prints as its worked **vulnerable**
example. Intel's test is that the fourth version field must be **≥ 3000**; this reads
`1202`. CVSS 9.8, and on CISA's Known Exploited Vulnerabilities list. **[source: Intel
SA-00075 advisory + detection guide]**

CVE-2017-5689 is an **authentication bypass** — AMT's HTTP Digest check compares the
response hash using the length of the *attacker-supplied* string, so an empty response
matches. **Password strength is irrelevant.** A successful attacker gets KVM, Serial-over-LAN,
IDE-R (mount a remote ISO and boot it), power control and boot-device override — all
**below the operating system**, working identically on Windows or Linux, working while the
OS is powered off, and invisible to any OS-level AV, EDR or audit log.

### Three things that are true regardless of your OS plans

1. **Reinstalling the OS does not touch AMT.** The Management Engine runs on the PCH
   independently of the disk. Wiping to Ubuntu leaves it exactly as-is.
2. **No host firewall can block it.** `ufw`, `iptables`, `nftables`, Windows Firewall — all
   inert for 16992–16995. The ME intercepts those packets in the NIC before the host stack
   exists. "I'll just block the port" is not available. **[source: Intel AMT
   implementation reference]**
3. **A local `netstat` proves nothing.** It shows the Windows LMS *service* bound on
   16992, not what the ME answers on the wire. Scan from a **second machine**:

```bash
nmap -p 16992-16995,623,664,5900 -sV <server-ip>
nmap -p 16992 --script http-vuln-cve2017-5689 <server-ip>
```

### The fix — and the pleasant surprise

Widely-repeated advice says HPE stopped at ME `11.6.27.3264` for this model, which clears
SA-00075 only. **That is wrong for the ML10 Gen9.** HPE's support page offers
**ME 11.8.77.3664**, which clears the whole chain: **[verified — flashed and confirmed]**

| Advisory | Fixed at | 11.8.77.3664 |
|---|---|---|
| SA-00075 (CVE-2017-5689) | 11.0.25 / 11.6.27 | closed |
| SA-00086 | 11.8.50.3425 | closed |
| SA-00112 / SA-00118 | 11.8.50 | closed |
| SA-00125 | 11.8.55 | closed |
| SA-00213 | 11.8.65 | closed |

**Order is mandatory and HPE states it explicitly: System ROM first, then ME.** Flashing ME
onto an old System ROM can half-apply, and a half-applied Management Engine on a box with
no iLO is a very bad place to be.

```
1. System Firmware Upgrade … (Windows)     → reboot     [we landed on ROM 1.13]
2. System ME Firmware Upgrade … (Windows)  → COLD power cycle
```

**Do the firmware while the old OS still boots.** HPE ships these as Windows (and RHEL)
utilities. There is no iLO, no Intelligent Provisioning, no SPP and no bootable ROMPaq for
this model. An EFI variant of the ME package exists as a fallback, but Windows is far
easier — and it is sitting right there before you wipe it.

**The downloads are entitlement-gated** behind an HPE Passport account plus the server
serial. Verify you can actually download *before* travelling to the machine; this is the
single most likely thing to waste a trip.

### After patching, keep AMT

The original plan was to unprovision and disable it. Once the firmware is current that
reverses: this machine has **no iLO and no serial port**, so AMT is its only possible
remote console. Losing it means every future boot failure or `fsck` prompt is a physical trip.

**Still mandatory:** change the MEBx password from its default `admin` (`Ctrl+P` at POST).
A default MEBx password is a documented 30-second physical takeover however current the
firmware is. Rules are strict — 8–32 chars, upper + lower + digit + one of `!@#$%^&*`;
other symbols are rejected. Record it immediately: the only recovery is clearing CMOS,
which resets it straight back to `admin`.

**Never use the BIOS "Unconfigure AMT/ME" option as a final step** — it clears CMOS and
restores the default MEBx password, undoing exactly what you just did.

---

## 4. `Error 8193: Fail to load MEI device driver`

Running HPE's `WinME.bat` did nothing visible — the window flashed and closed. The package's
own `error.log` held:

```
Error 8193: Fail to load MEI device driver (PCI access for Windows)
Above error is often caused by one of below reasons:
Administrator privilege needed for running the tool
ME is in an error state causing MEI driver fail
MEI driver is not installed
```

The MEI driver was present and healthy:

```
OK  System            Intel(R) Management Engine Interface #1
OK  SoftwareComponent Intel(R) Management Engine WMI Provider
```

The actual cause was the **first** reason listed: PowerShell was not elevated. Confirm
before blaming anything else: **[verified]**

```powershell
([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole(
  [Security.Principal.WindowsBuiltInRole]::Administrator)
# must print True
```

Double-clicking a `.bat` that fails fast gives you no chance to read the error. Run it from
an **already-open elevated shell** so the output stays on screen.

`WinME.bat` on this model is simply `FWUpdLcl64 -f ME.bin` — **no `-OEMID` needed**, despite
the tool's help listing it. **[verified]**

---

## 5. `Error 8716: Invalid usage` — the flag is `-FWVER`, not `-info`

```
Error 8716: Invalid usage
```

`FWUpdLcl64.exe` has no `-info`. To read the **currently flashed** firmware version:

```powershell
.\FWUpdLcl64.exe -FWVER
```

```
Intel (R) Firmware Update Utility Version: 11.8.70.3626   ← the TOOL's version
FW Version: 11.0.0.1202                                   ← the CHIP's version
```

Those two lines are easy to confuse, and confusing them will convince you a flash succeeded
when it hasn't. The first is the utility; the second is what's on the silicon.

**Always cold power cycle after an ME flash** — full shutdown, **pull the power cord for 30
seconds**, then boot. The ME runs on standby power; a warm reboot can leave the old firmware
resident so `-FWVER` reports the old version and a successful flash looks like a failure.

---

## 6. A long `apt` killed mid-flight leaves dpkg locked

Running `apt upgrade` (48 packages) over SSH from an automation harness with a 2-minute cap:

```
Command timed out after 2m 0s
```

then:

```
dpkg: error: dpkg frontend lock was locked by another process with pid 15846
```

**The remote process survived the SSH disconnect** and was still working — the timeout
killed the *local* client, not the remote `apt`. Deleting the lock file here would have
corrupted the package database.

**Fix.** Wait for the lock rather than breaking it, then repair and continue:

```bash
for i in $(seq 1 96); do
  fuser /var/lib/dpkg/lock-frontend >/dev/null 2>&1 || break
  sleep 5
done
DEBIAN_FRONTEND=noninteractive dpkg --configure -a
```

**Lesson:** run long `apt` operations **detached** (`nohup`/`systemd-run`/background task),
never inside a foreground command with a timeout shorter than the work.

---

## 7. `bench init` fails on `uv` — on **v15**, not just v16

```
FileNotFoundError: [Errno 2] No such file or directory: 'uv'
ERROR: There was a problem while creating frappe-bench
```

This repo already notes uv as bench 5.28+ behaviour. What this build adds: **it bites a
plain v15 build too**, because it is a property of the **bench** version, not the Frappe
branch. bench **5.31.0** requires uv to create the virtualenv even for
`--frappe-branch version-15`. **[verified]**

**Fix (what we did):**

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
export PATH="$HOME/.local/bin:$PATH"
uv --version                                     # 0.12.8
bench init frappe-bench --frappe-branch version-15 --python /usr/bin/python3.12
```

**Alternative** already recorded in `v15/install-and-gotchas.md`: `BENCH_DISABLE_UV=1`
restores the venv+pip path. Installing uv is the lower-friction option on a new box; the
env var matters if you must reproduce an older toolchain exactly.

Note the error names the missing binary but not what it is or where to get it, and it
appears *after* several minutes of cloning — so it reads like a bench bug rather than a
missing dependency.

---

## 8. `unknown log format "main"` — field confirmation on bench 5.31.0

```
2026/09/01 07:00:54 [emerg] 25102#25102: unknown log format "main" in
  /etc/nginx/conf.d/frappe-bench.conf:100
nginx: configuration file /etc/nginx/nginx.conf test failed
```

**This build reproduces `operations/ssl-nginx-and-production.md` §"log_format" exactly, on a
newer bench, including its subtle asymmetry.** That doc's `[verified]` analysis said
`bench setup production` escapes the bug and standalone `bench setup nginx` triggers it.
Observed here, one minute apart: **[verified]**

| Time | Command | `nginx -t` |
|---|---|---|
| 06:59 | `bench setup production frappe --yes` | **passed** |
| 07:00 | `bench setup nginx --yes` (to re-apply an upload-limit patch) | **failed** |

Source in bench 5.31.0, unchanged from the 5.29.0 analysis: **[verified]**

```python
# bench/commands/setup.py:28-41
@click.option("--logging", default="combined", type=click.Choice([...]))
@click.option("--log_format", ..., default="main")
```

```jinja
{# bench/config/templates/nginx.conf:124 #}
access_log  /var/log/nginx/access.log {{ logging.log_format }};
```

Debian/Ubuntu's `/etc/nginx/nginx.conf` defines no `log_format main` — that name is from the
nginx.org/RHEL stock config.

**Two useful notes this build adds:**

- It fails **safe**. nginx keeps serving the previously-loaded config, so a failed reload is
  not an outage — but a later reboot *would* be, so fix it immediately rather than deferring.
- If you are already adding a `conf.d` prerequisite file for real-IP handling (§ Cloudflare
  Tunnel), **put the `log_format` in the same file**. One file, sorting before
  `frappe-bench.conf`, that bench never regenerates:

```nginx
# /etc/nginx/conf.d/00-frappe-prereq.conf
log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                '$status $body_bytes_sent "$http_referer" '
                '"$http_user_agent" "$http_x_forwarded_for"';
```

---

## 9. CORRECTION — socketio on bench 5.31.0 fails **loudly**, not silently

**Supersedes** the guidance in `v15/install-and-gotchas.md` §2.1 and the
`Node from NodeSource, never nvm (supervisor loses socketio)` row of its summary table,
**for bench 5.31.0**.

That guidance says an nvm-installed node is invisible to bench at config-generation time, so
**"the socketio program is silently omitted from the generated config"** — producing a site
with dead realtime and *no error anywhere*.

**On bench 5.31.0 with node from nvm, we observed the opposite failure.** The socketio
program **was generated**, and failed loudly:

```
frappe-bench-web:frappe-bench-node-socketio   BACKOFF
```

```
Error: Cannot start socketio: node not found and the python backend is unavailable.
Install node or a frappe version with frappe.realtime.server.
```

### Why the behaviour differs — two independent node lookups

bench 5.31.0 resolves node **twice**, at different times, in different environments:

**(a) Generation time** — decides whether the stanza is written at all: **[verified]**

```python
# bench/config/supervisor.py:49-52
"node": which("node") or which("nodejs"),
# socketio runs if backend is python (no node needed) or node is present
"socketio_enabled": config.get("socketio_backend", "node") == "python"
or bool(which("node") or which("nodejs")),
```

**(b) Runtime** — `bench socketio` re-resolves node when supervisor starts it: **[verified]**

```python
# bench/commands/socketio.py — end of the function
node = which("node") or which("nodejs")
if not node:
    raise click.ClickException(
        "Cannot start socketio: node not found and the python backend is"
        " unavailable. Install node or a frappe version with"
        " frappe.realtime.server."
    )
os.execv(node, [node, os.path.join("apps", "frappe", "socketio.js")])
```

The generated stanza bakes in **no absolute node path** — it is just
`command={{ bench_cmd }} socketio`: **[verified]**

```ini
[program:frappe-bench-node-socketio]
command=/home/frappe/.local/bin/bench socketio
```

### The trap: the standard `setup production` idiom *causes* this

The universally-recommended invocation is:

```bash
sudo -H env "PATH=$PATH" bench setup production frappe --yes
```

That `env "PATH=$PATH"` wrapper exists to stop sudo's `secure_path` hiding `bench`. But it
**also exports the nvm node directory into the generation environment**, so lookup (a)
*succeeds* and the stanza is written. Supervisor then starts the program with its own
minimal environment, where lookup (b) *fails*.

**So the documented "silent omission" and this "loud BACKOFF" are the same root cause
reached by two paths:**

| Node visible at generation? | bench 5.29 / 5.31 outcome |
|---|---|
| No (plain `sudo bench setup production`) | stanza omitted → **silent** dead realtime |
| Yes (`sudo -H env "PATH=$PATH" …`) | stanza written, runtime lookup fails → **BACKOFF** |

The loud failure is strictly better, and the `env "PATH=$PATH"` idiom is what produces it —
so following current best practice converts a silent bug into a visible one.

### Fix

The repo's standing advice — **install node system-wide, not via nvm** — remains correct and
is the cleanest answer. If node is already in nvm, expose it system-wide rather than
rebuilding:

```bash
NODEBIN=$(dirname "$(command -v node)")     # with nvm sourced
for b in node npm npx yarn; do
  sudo ln -sf "$NODEBIN/$b" /usr/local/bin/$b
done
sudo supervisorctl restart all
```

Verify it the way supervisor sees it, not the way your login shell does:

```bash
sudo -i node --version        # must print a version
sudo supervisorctl status     # node-socketio must be RUNNING, not BACKOFF/STARTING
```

### Bonus: `socketio_backend` — a node-free option

bench 5.31.0 supports a **Python** realtime backend, removing node from the runtime path
entirely: **[verified in source]**

```python
# bench/commands/socketio.py:47
backend = (get_config(".") or {}).get("socketio_backend", "node")
...
if backend == "python":
    os.execv(python, [python, "-m", "frappe.realtime.server"])
```

Set `socketio_backend: "python"` in `common_site_config.json`. **Frappe 15.119.1 does not
ship `frappe.realtime.server`** — verified, `apps/frappe/frappe/realtime.py` exists but has
no `server` submodule — so on v15 it logs a warning and falls back to node. Useful on newer
Frappe; not an escape hatch on v15 today. **[verified]**

---

## 10. ERPNext assets were never built

Symptom: pages returned HTTP 200 and the HTML was correct, but **every stylesheet 404'd**.
Frappe's own bundles existed; ERPNext's did not:

```
sites/assets/frappe/dist/css/   → desk.bundle, email.bundle, login.bundle   ✓
sites/assets/erpnext/dist/…     → absent
```

`bench get-app erpnext` had ended with:

```
$ sudo supervisorctl restart frappe:
frappe: ERROR (no such group)
WARN: restarting supervisor group `frappe:` failed. Use `bench restart` to retry.
```

That warning is **expected** when `setup production` hasn't run yet (no supervisor config
exists), but the asset build did not complete alongside it.

**Fix:**

```bash
bench build            # rebuilds every app's bundles
bench restart          # so gunicorn reloads assets.json
```

**Diagnostic worth internalising:** the hashed filenames referenced by the served HTML come
from `sites/assets/assets.json`, which **gunicorn caches in memory**. After `bench build`,
pages kept referencing the *old* hashes until the workers restarted. If asset names in the
HTML don't match what's on disk, restart before investigating anything else.

---

## 11. `HOME_MODE 0750` — Ubuntu 24.04 breaks nginx asset serving

**The highest-value finding in this build.** Every Frappe install guide predates it.

Assets 404'd through both Cloudflare *and* the origin, while the files demonstrably existed
on disk. The nginx error log gave it away:

```
2026/09/01 07:13:33 [crit] 25268#25268: *67 stat()
  "/home/frappe/frappe-bench/sites/assets/erpnext/dist/css/erpnext_email.bundle.RDU6AXP6.css"
  failed (13: Permission denied), client: 127.0.0.1, ...
```

`namei -l` down the path:

```
drwxr-xr-x root   root   /
drwxr-xr-x root   root   home
drwxr-x--- frappe frappe frappe        ← 0750: no world execute
drwxrwxr-x frappe frappe frappe-bench
```

**Ubuntu 24.04 changed the default:** **[verified]**

```console
$ grep '^HOME_MODE' /etc/login.defs
HOME_MODE	0750
$ stat -c '%a %n' /home/sysadmin
750 /home/sysadmin
```

Historically `adduser` created homes `0755`. nginx workers run as **`www-data`**, which is
not in the `frappe` group, so they cannot **traverse** `/home/frappe` at all.

### Why the symptom is so misleading

**Pages render; only styling is missing.** HTML comes from **gunicorn** via `proxy_pass` —
nginx never touches the filesystem for it. Only `/assets` and `/files` are served directly
from disk. So the site looks alive, returns 200s, `/api/method/ping` answers `{"message":
"pong"}` — and looks like a CDN or proxy problem. It is neither.

**Fix:**

```bash
sudo chmod o+rx /home/frappe
sudo systemctl reload nginx
```

**Check it on every 24.04 build, before you go looking anywhere else:**

```bash
stat -c '%a %n' /home/frappe        # want 755, not 750
sudo -u www-data test -x /home/frappe && echo "nginx can traverse" || echo "BROKEN"
```

This also breaks **user-uploaded files** (`/files`, `/private/files`), not just assets — so
attachments would 404 the same way.

> If you prefer not to widen the home directory, `usermod -aG frappe www-data` plus group
> execute is the alternative. `chmod o+rx` is what standard Frappe deployments assume, and
> is what bench's own nginx template is written against. **[unverified — the group route
> was not tested here]**

---

## 12. Cloudflare cached the 404s — for a month

After fixing §11, some assets still 404'd **through Cloudflare only**:

```
edge   : 404   cf-cache-status: HIT      ← cached from before the fix
origin : 200
on disk: yes
```

Our own cache rule caused it. It used **Edge TTL: override origin, 1 month**, and an
override applies to **every** status code — so a transient permission failure became a
month-long cached 404.

**Fix — purge, then scope the TTL by status code:**

```jsonc
// Cache Rule: /assets/*  →  cache, but never cache an error
"edge_ttl": {
  "mode": "respect_origin",
  "status_code_ttl": [
    { "status_code_range": { "from": 200, "to": 299 }, "value": 2678400 },
    { "status_code_range": { "from": 300, "to": 599 }, "value": 0 }
  ]
}
```

`respect_origin` is the better default anyway: Frappe serves hashed asset filenames with
`cache-control: max-age=31536000`, so the origin already says the right thing.

**General rule: never set a blanket edge TTL override without status-code scoping.** A
five-minute origin fault becomes a month-long outage that looks exactly like the original bug.

### The diagnostic that isolated all of this in one step

Compare **origin and edge separately**. It distinguishes an app fault from a CDN fault
immediately:

```bash
# through the CDN
curl -s -o /dev/null -w '%{http_code}\n' https://site.example.com/assets/…/x.css
# at the origin, bypassing everything
ssh server "curl -s -o /dev/null -w '%{http_code}\n' -H 'Host: site.example.com' http://127.0.0.1/assets/…/x.css"
# and on disk
ssh server "test -e /home/frappe/frappe-bench/sites/assets/…/x.css && echo yes || echo NO"
```

| origin | edge | meaning |
|---|---|---|
| 404 | 404 | real app/permission problem — §10 or §11 |
| 200 | 404 `HIT` | **cached error** — purge, then fix the TTL rule — §12 |
| 200 | 404 `MISS` | cache rule / expression problem |

---

## 13. Process — an API token went into a chat transcript

A scoped Cloudflare API token was pasted as plain text into a working transcript. It was
revoked and the local copy deleted, but transcripts persist and it had DNS + Zone Settings
edit rights on a live domain.

**What we changed for the rest of the build, and recommend generally:**

- Secrets go to a **file**, never into a message: `printf '%s' 'TOKEN' > ~/.cf-token && chmod 600 ~/.cf-token`
- Generate credentials **on the target** and never echo them:
  `openssl rand -base64 24 | tr -d '/+=' | head -c 28 > /root/.mariadb_root_pw && chmod 600 …`
- **Scope tokens to one zone and one capability**, and set a short TTL. The second token
  issued in this build had only *Cache Purge* + *Cache Rules: Edit* — it could not have
  touched DNS.
- **Never enable Client IP filtering** on a token used from a dynamic-IP office; it breaks
  within a day.
- Revoke at the provider — deleting the local file does **not** invalidate the credential.

Also note: a mistyped password at a **login prompt** is written to the auth log as the
attempted *username*, in plaintext. `journalctl --vacuum-time=1s` clears it.

---

## 14. Site naming is a one-way door — there is no `bench rename-site` in v15

With `dns_multitenant on`, the **site directory name *is* the nginx `server_name`**. Name it
anything other than the real FQDN and every path 404s, including `/login`.

**`bench rename-site` does not exist in frappe v15 or bench 5.31.0.** Any guide that depends
on it is wrong. Real options: **[verified]**

```bash
# serve an existing site directory under a different hostname
bench setup add-domain new.example.com --site demo1.example.com   # domain is POSITIONAL
bench --site demo1.example.com set-config host_name https://new.example.com
bench setup nginx --yes && sudo systemctl reload nginx            # add-domain does NOT regenerate nginx
```

The directory and database keep the old name forever. A true rename is backup → restore into
a new site.

**Also decide the depth before you create anything.** Cloudflare's free Universal SSL covers
the apex and **one label only**:

```
DNS:example.com, DNS:*.example.com
```

`demo1.example.com` is covered. `demo1.erp.example.com` throws a certificate error in every
browser and needs Advanced Certificate Manager (~$10/mo/zone).

---

## What went right — worth copying

- **Firmware before OS wipe.** The vendor tools were Windows-only and Windows was still
  installed. Reversing that order would have meant a WinPE or UEFI-shell detour.
- **Tailscale before `ufw`.** Remote access was proven working *before* the firewall was
  enabled, on a machine with no serial port. A `ufw` mistake would otherwise have meant a
  physical trip.
- **Gates, not vibes.** After `bench setup production`, four explicit checks — supervisor
  symlink present, processes RUNNING, all sites in `server_name`, and a real HTTP request
  per site — caught problems immediately instead of at demo time.
- **utf8mb4 before the first site.** Retrofitting means dumping and reloading every tenant
  database, with Devanagari data mangled meanwhile.

---

## Final state

```
Hardware   HP ML10 Gen9 · System ROM 1.13 · ME 11.8.77.3664 (6 advisories closed)
OS         Ubuntu 24.04.4 LTS · Asia/Kolkata
Storage    LVM — / 40G · /home 80G · /var/lib/mysql 60G · ~36G free in vg0
Database   MariaDB 10.11.14 · utf8mb4 · 8G buffer pool · bind 127.0.0.1
Stack      Frappe 15.119.1 · ERPNext 15.120.0 · bench 5.31.0 · Python 3.12 · Node 20
Workers    gunicorn ×5 · RQ ×4 (2 short + 2 long) · scheduler · socketio
Exposure   Cloudflare Tunnel · 4 QUIC connections to Mumbai/Nagpur PoPs
Firewall   deny inbound · SSH from LAN + Tailscale only · 80/443 NOT open
Verified   4/4 sites HTTPS 200 · 52/52 assets each · socket.io 200
```

**Deliberately not done in this session**, and listed so nobody assumes otherwise:
**backups, SMTP/email, and monitoring.** A build is not finished until those exist — see
[operations/backup-restore-and-migration.md](../operations/backup-restore-and-migration.md).

---

## See also

- [operations/cloudflare-tunnel.md](../operations/cloudflare-tunnel.md) — the exposure topology in full
- [operations/bare-metal-and-firmware.md](../operations/bare-metal-and-firmware.md) — HP ML10 Gen9, AMT, BIOS, disks
- [v15/v15-on-ubuntu-2404.md](../v15/v15-on-ubuntu-2404.md) — the 24.04-specific traps, consolidated
- [operations/ssl-nginx-and-production.md](../operations/ssl-nginx-and-production.md) — the original `log_format` analysis this build confirms
