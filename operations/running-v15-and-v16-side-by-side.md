# Running v15 and v16 on the same machine

You have a working v15 box and you want v16 without losing it. This is the decision and the
build, from doing it on 2026-09-01: a KVM guest carved out of unallocated LVM space, both
majors serving the public internet through **one** Cloudflare Tunnel.

**Confidence tags:** `[verified]` run during this build · `[unverified]` plausible, untested.

---

## 1. What actually blocks you — it is not Python

The obvious blocker is the Python pin: v15 wants `>=3.10,<3.14`, v16 wants
`>=3.14,<3.15`, and Ubuntu ships one system Python per release. That is what drives the
"pick the OS to match the framework" advice in this repo, and for a *single-version* box it
is still the right call.

**But Python is solvable in seconds.** `uv` ships pre-built CPython distributions, so a
24.04 box can have 3.14 without compiling anything and without touching the system
interpreter: **[verified]**

```console
$ uv python install 3.14
Downloaded cpython-3.14.7-linux-x86_64-gnu
Installed Python 3.14.7 in 3.67s

$ /usr/bin/python3 --version        # system, untouched
Python 3.12.3
$ uv python find 3.14
/home/frappe/.local/share/uv/python/cpython-3.14-linux-x86_64-gnu/bin/python3.14
```

That is Astral's `python-build-standalone` — versioned, reproducible, updated with
`uv python install`. It is **not** the "hand-built interpreter you maintain forever" that
the OS-matching argument warns about, and the distinction matters when weighing options.

**The real blocker is MariaDB.**

| | v15 | v16 |
|---|---|---|
| Python | 3.10–3.13 | **3.14 only** |
| MariaDB | 10.6+ | **11.8** (what v16 CI tests) |
| Node | 18/20 | **>= 24** |

One host runs **one `mariadbd`**. Upgrading it to 11.8 for v16 drags your working v15 sites
along; leaving it at 10.11 runs v16 on a version Frappe does not test. Node is trivially
solved (NodeSource, per-version). Python is now trivially solved (uv). **The database is
what forces isolation.**

---

## 2. The three options

| | Keeps existing v15 sites | Version match | Notes |
|---|---|---|---|
| **A** — wipe, rebuild as v16 | no | perfect | cleanest if v15 is disposable |
| **B** — second bench on the same host | yes | **MariaDB mismatch** | Python fine via uv; DB is the problem |
| **C** — VM (or container) on the same host | yes | perfect | own MariaDB, own Python, own kernel |

**Option B is the trap.** It looks cheapest and the Python objection has evaporated, so it
is tempting — but you end up running v16 against MariaDB 10.11 and discovering the
consequences later, on a client's data. Reject it for that reason, not for the Python one.

**Option C is what this document builds.** A guest gets its own MariaDB 11.8, its own
Python 3.14 and its own kernel, so nothing about v16 can perturb v15.

> **Not dual-boot.** Only one OS runs at a time, so booting into v16 takes every v15 site
> offline. It answers a different question than the one you are asking.

---

## 3. Sizing, and where the disk comes from

The reference host: 4 cores / 31 GB / 223 GB SSD, already running four v15 demo sites.

```console
$ free -g | awk '/Mem:/{print $2" total, "$7" available"}'
31 total, 29 available          # the v15 stack idles at ~2 GB
$ vgs --units g -o vg_name,vg_free --noheadings
  vg0  36.52g
```

**Check virtualisation support first** — this is the one thing that can stop you dead:

```bash
grep -oE 'vmx|svm' /proc/cpuinfo | sort -u    # vmx = Intel VT-x, svm = AMD-V
[ -e /dev/kvm ] && echo present
```

Allocation used: **8 GB RAM, 2 vCPU, 32 GB disk.** That left the host 23 GB and ~4.5 GB
unallocated in the VG.

**On leaving VG headroom.** Do not allocate the last of it. Free extents are what let you
grow whichever volume fills first, and what LVM snapshots consume. If you need more, look at
how over-provisioned the existing volumes are before shrinking anything — on the reference
box `/var/lib/mysql` was **59 GB holding 581 MB**, so ~40 GB was reclaimable with a short
maintenance window.

**CPU is the ceiling, not RAM.** Four cores split across two benches means they contend.
Fine for demos and development; not for two busy production stacks.

---

## 4. Build it — cloud image + cloud-init, no console

Do **not** install from an ISO. A cloud image plus cloud-init is fully unattended, which
matters because a VM install over SSH has no console to click through.

```bash
sudo apt-get install -y qemu-kvm libvirt-daemon-system libvirt-clients \
                        virtinst cloud-image-utils qemu-utils
sudo systemctl enable --now libvirtd

# disk straight from the volume group — better I/O than a qcow2 on a filesystem
sudo lvcreate -L 32G -n v16vm vg0

# Ubuntu 26.04 = "resolute"
cd /var/lib/libvirt/images
sudo curl -fsSLO https://cloud-images.ubuntu.com/resolute/current/resolute-server-cloudimg-amd64.img
sudo qemu-img convert -O raw resolute-server-cloudimg-amd64.img /dev/vg0/v16vm
```

cloud-init seed — inject **the SSH public key you will actually connect with** (see §5):

```bash
cat > /tmp/user-data <<YAML
#cloud-config
hostname: trustbit-v16
users:
  - name: frappe
    groups: [sudo]
    shell: /bin/bash
    sudo: ['ALL=(ALL) NOPASSWD:ALL']
    ssh_authorized_keys:
      - ssh-ed25519 AAAA...   # your workstation's PUBLIC key
ssh_pwauth: false
timezone: Asia/Kolkata
growpart: { mode: auto, devices: ['/'] }
YAML
printf 'instance-id: v16-01\nlocal-hostname: trustbit-v16\n' > /tmp/meta-data
sudo cloud-localds /var/lib/libvirt/images/v16-seed.iso /tmp/user-data /tmp/meta-data
```

```bash
sudo virt-install \
  --name trustbit-v16 --memory 8192 --vcpus 2 \
  --disk path=/dev/vg0/v16vm,bus=virtio,format=raw \
  --disk path=/var/lib/libvirt/images/v16-seed.iso,device=cdrom \
  --os-variant ubuntu24.04 \
  --network network=default,model=virtio \
  --graphics none --noautoconsole --import

sudo virsh autostart trustbit-v16          # survives a host reboot — do not skip
sudo virsh domifaddr trustbit-v16          # its NAT address, e.g. 192.168.122.197
```

`growpart` expands the cloud image's root to fill the 32 GB volume on first boot — the image
ships around 3.5 GB and would otherwise stay that size. **[verified]**

### Use the default NAT network, not a bridge

A bridge gives the guest its own LAN address, which sounds tidier — but building one means
reconfiguring the **host's** networking, and a mistake there drops your SSH session on a
machine you may not be sitting next to.

libvirt's default NAT (`virbr0`, `192.168.122.0/24`) needs no host network changes at all,
and the tunnel does not care: `cloudflared` runs on the host and can reach the guest
directly.

---

## 5. The SSH detail that will confuse you

The guest trusts **your workstation's** public key. The *host's* `frappe` user has that key
in `authorized_keys` but owns no private key, so host → guest fails:

```
frappe@192.168.122.197: Permission denied (publickey).
```

The guest is on a NAT network only the host can reach, so connect **through** the host with
ProxyJump — your workstation authenticates to both hops with the same key: **[verified]**

```bash
ssh -J frappe@<host-lan-ip> frappe@192.168.122.197
```

Worth putting in `~/.ssh/config` on day one:

```
Host v16vm
  HostName 192.168.122.197
  User frappe
  ProxyJump frappe@192.168.0.145
```

---

## 6. One tunnel, two backends

The tunnel already running on the host can serve the guest — no second tunnel, no extra
ports, and no new certificate if the hostname is one label deep (free Universal SSL covers
`*.example.com`).

**Open the NAT bridge on the host firewall first.** `ufw` denies inbound by default, and
that includes traffic arriving on `virbr0`. Miss this and the ingress rule looks perfect
while silently failing:

```bash
sudo ufw allow in on virbr0 comment 'libvirt NAT - cloudflared to guest'
# prove it before editing the tunnel:
curl -s -o /dev/null -w '%{http_code}\n' http://192.168.122.197/
```

Then add one rule **above the catch-all**:

```yaml
ingress:
  - hostname: demo1.example.com
    service: http://localhost:80          # host, v15
  - hostname: v16.example.com
    service: http://192.168.122.197:80    # guest, v16
  - service: http_status:404
```

```bash
cloudflared tunnel route dns <tunnel-name> v16.example.com
sudo cp ~/.cloudflared/config.yml /etc/cloudflared/config.yml   # the service reads THIS copy
sudo systemctl restart cloudflared
```

Two things that bite here:

- **`service install` copies the config to `/etc/cloudflared/`.** Editing only
  `~/.cloudflared/config.yml` changes nothing for the running service. Copy it across.
- **The `--config` flag is global, not a subcommand flag.** It goes *before* `ingress`:
  `cloudflared --config FILE tunnel ingress validate`. Putting it after gives
  `flag provided but not defined: -config`, and the validation you thought you ran did not
  run. **[verified]**

A restart drops every long-lived connection, including all socket.io sessions on the
**existing** sites. Do it outside business hours.

### Real-IP inside the guest

`cloudflared` reaches the guest from the host's bridge address, so the guest's nginx should
trust that, not loopback:

```nginx
# /etc/nginx/conf.d/00-frappe-prereq.conf  — inside the guest
set_real_ip_from 192.168.122.1;      # the host end of virbr0
real_ip_header   CF-Connecting-IP;
```

Without it every user appears as `192.168.122.1` and the login-attempt tracker collapses
into one shared bucket — ten failures by anyone locks out everybody.

---

## 7. Installing v16 in the guest

Follow [v16/install-ubuntu-2604.md](../v16/install-ubuntu-2604.md) unchanged — it was
written for a real 26.04 machine, which is exactly what the guest is. The guest gives you
what the host could not:

```
Ubuntu 26.04 LTS · Python 3.14.4 (system) · MariaDB 11.8.6 (Ubuntu main)
Node 24.20.0 (NodeSource) · uv 0.12.8 · bench 5.31.0
Frappe 16.32.0 · ERPNext 16.33.0
```

Four things confirmed on this build worth carrying over:

- **Node from NodeSource, not nvm.** `node-socketio` came up `RUNNING` first attempt — no
  `BACKOFF`, no `/usr/local/bin` symlink detour. The repo's standing advice, validated.
- **`HOME_MODE 0750` applies to 26.04 too**, inherited from 24.04. `chmod o+rx /home/frappe`
  before nginx can serve `/assets`. See [../v15/v15-on-ubuntu-2404.md](../v15/v15-on-ubuntu-2404.md) §1.
- **`tzdata-legacy` is required**, and the setup wizard is where it surfaces. See §9 there.
- **`log_format main` prerequisite file** — same as every other Ubuntu bench.

Size the guest's workers down. On 2 vCPU / 7 GB: `gunicorn_workers 3`,
`background_workers 1` (which still renders **2** RQ processes on the short/long split).
The host's 5 and 2 would just contend.

---

## 8. Do not run apt on both machines at once

Two concurrent `apt` operations — one on the host, one in the guest over SSH — produced:

```
E: The package lists or status file could not be parsed or opened.
```

on a system with no broken packages, no held locks and a clean `apt-get check` moments
later. Both recovered on retry. When automating across host and guest, **serialise package
operations**; the failure looks like corruption and is not. **[verified]**

---

## 9. Verify

```bash
# every hostname, from outside
for h in demo1 demo2 demo3 demo4 v16; do
  printf '%-28s %s\n' "$h" \
    "$(curl -s -o /dev/null -w '%{http_code}' --max-time 25 https://$h.example.com/api/method/ping)"
done

# the certificate already covers the new name
echo | openssl s_client -connect v16.example.com:443 -servername v16.example.com 2>/dev/null \
  | openssl x509 -noout -ext subjectAltName

# the guest survives a host reboot
sudo virsh list --all      # trustbit-v16 should be 'running', autostart on
```

A neat tell that you really are hitting two different majors: **v15 and v16 return different
error envelopes.** A permission error from v15 is `{"exception": ...}`; v16 returns
`{"exc_type": ..., "_server_messages": ...}`. **[verified]**

---

## 10. What this costs

**Good:** both majors live simultaneously, complete database and interpreter isolation, one
tunnel and one certificate, guest snapshots via `virsh`, and the guest is disposable — break
it and rebuild in twenty minutes without touching production.

**Real costs:**

- **Two machines to patch, back up and monitor.** The guest starts with **no backup job at
  all** — the host's cron does not see it. This is the single most likely thing to be
  forgotten.
- **Shared CPU.** Four cores across two stacks.
- **A second MariaDB** with its own buffer pool (2 GB in the guest) eating host RAM.
- **The guest is invisible to host tooling.** `df`, `supervisorctl` and your backup script on
  the host say nothing about it.

Set up the guest's backups the same day you build it, or you have quietly created a second
unprotected server.

---

## See also

- [../builds/2026-09-01-…](../builds/2026-09-01-bare-metal-four-site-cloudflare-tunnel.md) — the host build and the failures behind it
- [cloudflare-tunnel.md](cloudflare-tunnel.md) — the exposure topology in full
- [../v16/install-ubuntu-2604.md](../v16/install-ubuntu-2604.md) — the v16 runbook to follow inside the guest
- [backup-restore-and-migration.md](backup-restore-and-migration.md) — including the drill that can pass while proving nothing
