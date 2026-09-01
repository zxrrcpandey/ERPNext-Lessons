# Bare-metal ERPNext — firmware, BIOS, disks, and out-of-band access

Everything this repo previously assumed away by starting from a VPS. Written from an
**HP ProLiant ML10 Gen9** build, but the *reasoning* generalises to any repurposed
out-of-support tower — which is what most on-prem ERPNext boxes actually are.

Reference build:
[builds/2026-09-01-…](../builds/2026-09-01-bare-metal-four-site-cloudflare-tunnel.md).

**The order matters more than any individual step:**

```
1. audit what the old OS is doing        ← before anything is destroyed
2. firmware, while the old OS still boots ← vendor tools are Windows/RHEL-only
3. BIOS settings                          ← after flashing; a ROM update resets them
4. wipe + install
5. remote access proven working           ← before the firewall
6. firewall
```

Getting 2 and 3 the wrong way round costs a second trip. Getting 5 and 6 the wrong way
round on a box with no serial port costs a *physical* trip.

---

## 1. Audit before you destroy

The machine in the reference build presented as "just Windows Server, 117 GB used". A
two-minute look found **SQL Server 2022**, **two POS receipt-printer driver stacks**, an
**IIS install** and a **backup folder** — the fingerprint of a billing workstation, not a
bare server.

```powershell
# top-level folders by size (a few minutes)
Get-ChildItem C:\ -Directory -Force | ForEach-Object {
  $s = (Get-ChildItem $_.FullName -Recurse -File -Force -EA SilentlyContinue |
        Measure-Object Length -Sum).Sum
  [PSCustomObject]@{ Folder=$_.Name; GB=[math]::Round($s/1GB,1) }
} | Sort-Object GB -Descending | Select-Object -First 15

# the big system files (NOT folders — these explain most of "used" space)
Get-ChildItem C:\ -Force -File | Where-Object Extension -eq '.sys' |
  Select-Object Name, @{n='GB';e={[math]::Round($_.Length/1GB,1)}}

# is it serving anyone?
Get-SmbShare
Get-Service | Where-Object { $_.Name -like 'MSSQL*' } | Select Name, Status, StartType

# databases and backups anywhere on the disk — the one that decides it
Get-ChildItem C:\ -Recurse -Include *.mdf,*.ldf,*.bak -File -Force -EA SilentlyContinue |
  Select FullName, @{n='MB';e={[math]::Round($_.Length/1MB,1)}}, LastWriteTime |
  Sort-Object MB -Descending
```

**Reading it:** `pagefile.sys` and `hiberfil.sys` are *files*, not folders, so a
folders-only scan appears to lose tens of GB. On a 32 GB-RAM box the page file alone can be
32 GB — which is usually the whole mystery. If `Users`, `inetpub` or a company-named folder
is large, or a `.mdf` has a recent `LastWriteTime`, stop and copy it off.

**Take a full disk image first** (Clonezilla, an hour). It is the difference between "we can
get that back" and "it's gone."

---

## 2. Intel AMT — check it, because you may already be exposed

Many entry-tier towers ship **Intel AMT instead of a BMC**. It is a genuine asset — a real
remote console on a machine that otherwise has none — and a genuine liability on old firmware.

### Is it enabled, and on what version?

From **Windows**, using Intel's SA-00075 Detection and Mitigation Tool:

```powershell
Intel-SA-00075-console.exe -Discover      # ME version, SKU, provisioning state, control mode
```

From **Linux** afterwards: `mei-amt-check` (github.com/mjg59/mei-amt-check), or check the
`mei_me` module and `/dev/mei0`.

**A local `netstat` proves nothing.** It shows the Windows LMS *service* bound on 16992,
not what the Management Engine answers on the wire. Scan **from a second machine**:

```bash
nmap -p 16992-16995,623,664,5900 -sV <server-ip>
nmap -p 16992 --script http-vuln-cve2017-5689 <server-ip>
```

An Intel AMT web interface returns a distinctive `Server:` header.

### CVE-2017-5689 (INTEL-SA-00075)

Intel's own test: the **fourth version field must be ≥ 3000**. The reference build reported
ME `11.0.0.1202` — build `1202`, and *literally the value Intel prints as its worked
vulnerable example*. CVSS 9.8, on CISA's KEV list. **[source: Intel SA-00075]**

It is an **authentication bypass** — the HTTP Digest check compares the response hash using
the length of the *attacker-supplied* string, so an empty response matches. **AMT password
strength is irrelevant.** A successful attacker gets KVM, Serial-over-LAN, IDE-R (mount a
remote ISO and boot it), power and boot-device control — **below the OS**, identical on
Windows or Linux, working while the OS is powered off, invisible to any OS-level AV, EDR or
audit log.

### Three facts that change how you plan

1. **Reinstalling the OS does not touch AMT.** The ME runs on the PCH independently of the
   disk. Wiping to Ubuntu leaves it exactly as it was.
2. **No host firewall can block it.** `ufw`, `iptables`, `nftables`, Windows Firewall — all
   inert on 16992–16995, because the ME takes those packets in the NIC before the host stack
   exists. **[source: Intel AMT implementation reference]**
3. **CIRA ("fast call for help")** is an *outbound* AMT path, so "we have no inbound ports"
   does not cover it. Check Remote Access settings; unprovisioning disables it.

### Fix it — and check the vendor page rather than the folklore

Widely-repeated advice for this model stops at ME `11.6.27.3264` (SA-00075 only). **HPE
actually shipped `11.8.77.3664`**, which closes the whole chain: **[verified — flashed]**

| Advisory | Fixed at | 11.8.77.3664 |
|---|---|---|
| SA-00075 (CVE-2017-5689) | 11.0.25 / 11.6.27 | closed |
| SA-00086 | 11.8.50.3425 | closed |
| SA-00112 / SA-00118 | 11.8.50 | closed |
| SA-00125 | 11.8.55 | closed |
| SA-00213 | 11.8.65 | closed |

**Always read the vendor's actual download page for your exact model before concluding a
machine is unpatchable.**

### After patching, keep AMT — but fix the password

On a box with no iLO and no serial port, AMT is the only remote console. Disabling it means
every future boot failure or `fsck` prompt is a physical trip.

**Mandatory regardless of firmware version:** change the MEBx password from its default
`admin` (`Ctrl+P` at POST). A default MEBx password is a documented **30-second physical
takeover** whatever the firmware level. Rules are strict — 8–32 chars, upper + lower + digit
+ one of `!@#$%^&*`; other symbols are rejected. Record it immediately; the only recovery is
clearing CMOS, which resets it to `admin`.

**Never use the BIOS "Unconfigure AMT/ME" option as a final step** — it clears CMOS and
restores the default password, undoing exactly what you just did.

> **If no fixed firmware exists for your model**, the cheapest real mitigation is physical:
> AMT only rides the **ME-paired onboard NIC**. Fit an add-in PCIe NIC, cable only that, and
> leave the onboard port unplugged — AMT then has no network path. You lose the remote
> console. **[unverified — not needed on this build]**

---

## 3. Flashing — sequence and gotchas

**Do the firmware while the old OS still boots.** Vendor utilities are Windows (and often
RHEL) only. Entry towers typically have no iLO, no Intelligent Provisioning, no service-pack
ISO and no bootable ROMPaq, so afterwards you are driving a WinPE stick or a UEFI shell.

```
1. System ROM / BIOS   → reboot, let the OS come fully back
2. ME firmware         → COLD power cycle (see below)
```

**Order is vendor-mandated.** Flashing ME onto an old System ROM can half-apply, and a
half-applied Management Engine on a box with no out-of-band access is a very bad place to be.

**Downloads are entitlement-gated** behind a vendor account plus the server serial number.
Verify the download works *before* travelling — it is the single most likely thing to waste a
trip.

```powershell
Get-CimInstance Win32_BIOS | Select SMBIOSBIOSVersion, Manufacturer, ReleaseDate, SerialNumber
```

### Run the flasher elevated, from an already-open shell

```
Error 8193: Fail to load MEI device driver (PCI access for Windows)
```

reads like a missing driver but is usually **missing privileges**. The MEI driver was
present and healthy on the reference build. Confirm elevation explicitly:

```powershell
([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole(
  [Security.Principal.WindowsBuiltInRole]::Administrator)   # must be True
```

Double-clicking a `.bat` that fails fast closes before you can read the error — run it from
an open elevated shell.

### Read the version with `-FWVER`, not `-info`

```
Error 8716: Invalid usage          ← -info does not exist
```

```powershell
.\FWUpdLcl64.exe -FWVER
```

```
Intel (R) Firmware Update Utility Version: 11.8.70.3626   ← the TOOL
FW Version: 11.0.0.1202                                   ← the CHIP
```

Confusing those two lines will convince you a flash succeeded when it did not.

### Cold power cycle after an ME flash

Full shutdown, **pull the power cord for 30 seconds**, then boot. The ME runs on standby
power; a warm reboot can leave the old firmware resident, so `-FWVER` reports the old version
and a successful flash looks like a failure.

### Re-check BIOS settings after a ROM update

A major System ROM change can load Optimized Defaults and silently reset SATA mode, CSM
state and boot order. **Set BIOS options after flashing, never before.**

---

## 4. BIOS — do not assume the vendor's usual keys

Entry-tier towers often use a stock **AMI Aptio** BIOS rather than the vendor's enterprise
firmware, so the familiar function keys do not apply. On the ML10 Gen9 the enterprise
ProLiant keys (F9/F10/F11) do nothing; it is: **[verified]**

| Key | Does |
|---|---|
| `DEL` or `ESC` | BIOS setup ("Press DEL to run Setup") |
| `F7` | boot menu |
| `F4` | save and exit |
| `F12` | PXE boot |
| `Ctrl+P` | Intel MEBx |

Confirm which family you have before travelling:

```powershell
Get-CimInstance Win32_BIOS | Select Manufacturer
# "American Megatrends Inc." → AMI keys, not the vendor's enterprise ones
```

### Settings that matter for Linux

| Setting | Value | Why |
|---|---|---|
| **SATA Mode** | **AHCI** | RAID mode is Intel RST/RSTe **fakeraid** with IMSM metadata. Ubuntu's installer offers plain MD RAID only and cannot manage IMSM — you get an array no rescue ISO reassembles cleanly. |
| **CSM** | Disabled | = pure UEFI. With CSM on you get duplicate/legacy boot entries and inconsistent behaviour. |
| Secure Boot | Off (or shim) | Ubuntu boots with signed shim, but off is one less variable. |

**Change SATA mode *after* you finish using the old OS.** Flipping it under an existing
Windows install bugchecks `0x7B INACCESSIBLE_BOOT_DEVICE` on the next boot — irrelevant if
you are wiping, fatal if you still needed Windows to run the flasher.

### Video and cabling

Modern entry servers are frequently **DisplayPort-only** — no VGA, no HDMI. If your monitor
is VGA you need an **active** DP→VGA adapter; passive ones do not work. Turning up with the
wrong cable is a wasted trip. Bring a DP dummy plug too if the box will run headless.

---

## 5. Writing the installer USB

Full detail in
[builds/2026-09-01-… §1–2](../builds/2026-09-01-bare-metal-four-site-cloudflare-tunnel.md).
The two that catch people:

**On macOS/zsh, `~` is not expanded after `=`.** `dd if=~/path.iso` passes a literal tilde
and fails with `No such file or directory` while the file plainly exists. Use an absolute
path.

```bash
diskutil list external physical                  # identify by SIZE, every time
diskutil unmountDisk /dev/disk12
sudo dd if=/abs/path.iso of=/dev/rdisk12 bs=4m status=progress
diskutil eject /dev/disk12                       # only AFTER the write
```

A correct write shows GPT plus an **`EFI ESP`** partition — that is what confirms it will
boot with CSM disabled. macOS then says *"The disk you inserted was not readable"*: click
**Ignore**, never **Initialize**.

---

## 6. Storage layout

**Always choose "Custom storage layout"** in the Ubuntu Server installer. Guided LVM caps
root at **100 GiB** on any volume group over 200 GiB and leaves the rest unallocated — the
install completes and looks entirely normal. **[verified]**

A layout that has proven right for a multi-tenant bench, on a 240 GB disk:

| Mount | Size | Notes |
|---|---|---|
| `/boot/efi` | 1 GiB | created by "Use As Boot Device" |
| `/boot` | 2 GiB | ext4 |
| `/` | 40 GiB | LV |
| `/var/lib/mysql` | 60 GiB | LV — a runaway import cannot fill root |
| `/home` | 80 GiB | LV — **the big one**: bench, all sites, attachments, local backups |
| *(unallocated)* | ~36 GiB | headroom + LVM snapshot before a risky `bench update` |
| swap | 4 GiB | LV |

`/home` is the one people undersize. With classic bench **everything** lives in
`/home/frappe/frappe-bench` — app code, every site, every attachment, and local backups.

Leaving free space in the VG is deliberate: it lets you grow whichever fills first, and take
an LVM snapshot before an upgrade.

### If you are mirroring two disks

**The EFI partition cannot live on an mdadm mirror**, but subiquity handles it properly if
you use the right order: **[unverified — the reference build ran a single disk]**

1. First disk → **Use As Boot Device**
2. Second disk → **Add As Another Boot Device** (creates an ESP on each, installs GRUB to both)
3. Add one max-size partition per disk, **Format: Leave unformatted** (a formatted partition
   is not offered as a RAID candidate)
4. **Create software RAID (MD)** level 1 from those two *partitions*, never from whole disks
5. Build the VG on `md0`

Then verify — and test the round people skip:

```bash
debconf-show grub-efi-amd64 | grep install_devices   # both -part1 IDs must be listed
```

**Pull the disk holding `/boot/efi`**, not the other one. Pulling the second disk always
works and proves nothing. Expect a 30-second-plus stall on a degraded boot; do not
power-cycle through it.

Also: `mdadm` sizes the array to the **smallest member**, so do not pair a leftover small
disk with new large ones.

### `/etc/fstab` is a lockout risk

On a box with no serial port and DisplayPort-only video, a typo in `fstab` drops the boot to
an emergency shell reachable only from a physical monitor. Run this after **every** fstab
edit, before **every** reboot:

```bash
sudo findmnt --verify --verbose
```

---

## 7. Remote access before the firewall

Non-negotiable on a machine with no out-of-band console. Install and **prove** your remote
path works *before* enabling `ufw`:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --ssh
# then actually connect over it from another machine BEFORE touching the firewall
```

Only then:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow in on tailscale0
sudo ufw allow from 192.168.0.0/24 to any port 22 proto tcp
sudo ufw enable
```

With [Cloudflare Tunnel](cloudflare-tunnel.md), **80 and 443 stay closed** — `cloudflared`
dials out and reaches nginx over loopback. Verify the sites still work immediately after
enabling the firewall; that is the proof the topology is what you think it is.

Confirm what is actually listening, and on which address:

```bash
sudo ss -lntp
# MariaDB, Redis and gunicorn must be 127.0.0.1 only
```

---

## 8. Honest assessment — should client production live on this box?

Worth writing down before anyone depends on it.

**Defensible:** demos, pilots, UAT, internal use, staging, CI, a backup landing zone. Nobody's
month-end close depends on it, so office power and a consumer uplink stop being contractual
problems.

**Hard to defend for paying-client production on a single repurposed tower:** one machine,
often one disk, one office, one broadband line, no failover, and — typically — hardware that
is years past vendor support. Availability is bounded by the worse of the ISP and the
building's power.

Two specifics that bite on this class of hardware:

- **Consumer SSDs are the wrong drive for a database.** A DRAM-less consumer SSD rated
  ~80 TBW will wear under constant InnoDB redo, RQ queues and nightly backups. Prefer a
  DRAM-equipped or NAS/enterprise-rated drive, mirrored.
- **RAID is not a backup.** It survives one disk dying. It does not survive `DROP DATABASE`,
  a bad `bench migrate`, ransomware, or the building. See
  [backup-restore-and-migration.md](backup-restore-and-migration.md).

If the answer is "demos here, client production elsewhere", that is a *better* use of the
machine, not a consolation prize — and it removes the static IP, TLS, UPS and uptime
problems in one move. Decide the trigger for moving a converted demo **before** it converts.

---

## See also

- [builds/2026-09-01-…](../builds/2026-09-01-bare-metal-four-site-cloudflare-tunnel.md) — the full chronological build log
- [cloudflare-tunnel.md](cloudflare-tunnel.md) — publishing with no inbound ports
- [../v15/v15-on-ubuntu-2404.md](../v15/v15-on-ubuntu-2404.md) — OS-level traps once Ubuntu is installed
- [security.md](security.md) — application and host hardening after the build
