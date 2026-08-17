# ERPNext Lessons — Trustbit Software

Hard-won deployment knowledge for **Frappe / ERPNext v15 and v16**, written from real
client builds rather than from documentation. Every lesson here cost us time on a live
box at least once.

**How to use this repo:** find the error string you just hit (they are reproduced
verbatim so they are greppable), or follow the install runbook top to bottom for a new
server.

```bash
grep -rn "Chromium took too long" .
grep -rn "ZoneInfoNotFoundError" .
```

---

## Start here

| I want to… | Read |
|---|---|
| Build a **new v16 server** from a bare VPS | [v16/install-ubuntu-2604.md](v16/install-ubuntu-2604.md) |
| Understand **what v16 changed** before committing | [v16/what-changed-from-v15.md](v16/what-changed-from-v15.md) |
| Keep an existing **v15 client** running | [v15/install-and-gotchas.md](v15/install-and-gotchas.md) |
| Build or port a **custom app** | [apps/custom-app-development.md](apps/custom-app-development.md) |
| **Back up, restore or move** a site | [operations/backup-restore-and-migration.md](operations/backup-restore-and-migration.md) |
| Sort out **nginx / SSL / supervisor** | [operations/ssl-nginx-and-production.md](operations/ssl-nginx-and-production.md) |
| **Secure** a deployment, or check one for compromise | [operations/security.md](operations/security.md) |

---

## Which version for a new project?

**v16 unless you have a reason not to.** It is GA and actively developed
(`[UNVERIFIED]` release date — no primary source in this repo states one; the versions we
actually built are frappe 16.31.0 / erpnext 16.32.1 on 2026-08-17). The catch is that its
Python pin dictates your operating system:

| | v15 | v16 |
|---|---|---|
| Python | 3.10 – 3.13 (`>=3.10,<3.14`) | **3.14 only** (`>=3.14,<3.15`) |
| Practical OS | Ubuntu 22.04 / 24.04 | **Ubuntu 26.04 LTS** |
| Node | 18 / 20 | **>= 24** |
| MariaDB | 10.6 | **11.8** (what v16 CI tests) |
| PDF | wkhtmltopdf only | wkhtmltopdf **+ Chromium/CDP** |

Because v16 pins Python to the 3.14 series exactly, **only Ubuntu 26.04 ships a suitable
system Python**. On 24.04 (Python 3.12) you would be maintaining a hand-built interpreter
for the life of the server. Pick the OS to match the framework, not the other way round.

> **On the PDF row:** the genuinely new thing in v16 is the **Chromium/CDP** generator —
> v15's `Print Format.pdf_generator` field exists but offers `wkhtmltopdf` alone.
> `WeasyPrint==68.0` is pinned in **both** versions (verified in frappe 15.100.0's
> `pyproject.toml`), so it is not a v16 addition — though on Ubuntu 26.04 it needs native
> libraries installed or it fails to load at all. See
> [v16/what-changed-from-v15.md](v16/what-changed-from-v15.md).

> **Custom apps do not move for free.** An app built against v15 is not drop-in on v16.
> Treat that as a migration with breaking API changes, not a reinstall.

---

## The five that bite hardest

If you read nothing else before a deploy:

1. **Add swap before you build anything.** A 1 vCPU VPS will be OOM-killed during asset
   builds. Frappe caps the Node heap at 75% of *physical* RAM and ignores swap, and
   exporting `NODE_OPTIONS` does nothing because Frappe overrides it.
2. **Never copy the classic Frappe `my.cnf`.** `innodb-file-format` and
   `innodb-large-prefix` were removed in MariaDB 10.3 — including them stops the server
   from starting.
3. **`certbot certonly --nginx`, never `certbot --nginx`.** Bench's nginx config is a
   *symlink*; certbot's installer mode rewrites it and the next `bench setup nginx`
   destroys your SSL.
4. **`chown -R frappe:frappe` after any bench command you ran as root.** Root-owned files
   inside the bench break the runtime in ways that surface much later.
5. **Custom Fields created through the UI never reach git.** Without a `fixtures` entry or
   an `install.py`, they exist in exactly one database and vanish on the next fresh site.

---

## Conventions

- Commands run as **root** unless wrapped
  `sudo -u frappe -H bash -lc 'export PATH=$HOME/.local/bin:/usr/local/bin:$PATH; …'` —
  the explicit PATH export is required because Ubuntu's `~/.bashrc` returns early for
  non-interactive shells, so a PATH line written there is never picked up. **Do not use
  `su - frappe -c`.**
- `<site>` means the site name, which for us is normally the fully-qualified domain.
- Anything we could not verify first-hand is marked **unverified** rather than quietly
  asserted. Trust the rest.

## Contributing

When a deploy teaches you something, add it here the same day — with the **symptom, the
exact error text, the root cause, and the fix**. A lesson without its cause is useless six
months later.
