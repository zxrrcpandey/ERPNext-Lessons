# Backup, Restore & Migration

Operational runbook for moving a Frappe/ERPNext site between machines: taking backups,
restoring them (including the no-root-password trick), aligning app versions before a
restore, and the failure modes that eat an afternoon.

**Applies to:** v15 and v16 unless a lesson is tagged otherwise.

**Evidence confidence** — every claim below carries one of these:

| Tag | Meaning |
|---|---|
| **[V]** | Verified in this session: command executed, or read verbatim from frappe/bench source, or read from git reflog / file listing on disk |
| **[B]** | From a real build record (`SERVER_KANTISHIVA.md`, `PROJECT_STATE.md`, `DEPLOYMENT_LESSONS.md`) |
| **[U]** | Unverified pattern — sound, but not executed on a Trustbit box. Test before trusting |

Source material: the v15 `mandi.trustbit.in` → local `mandi.local` restore (2026-08-16),
the v16 `kantishiva.trustbit.cloud` build (2026-08-17), and frappe `15.100.0` /
bench `5.29.0` source read directly off `~/frappe-bench-v15`.

---

## 1. Taking a backup

### 1.1 The command

```bash
cd /home/frappe/frappe-bench
bench --site <site> backup --with-files
```

Without `--with-files` you get the database only. **[V]** — verified from
`apps/frappe/frappe/commands/site.py`, `@click.command("backup")`, option
`--with-files ... help="Take backup with files"`.

Useful siblings on the same command **[V]**:

| Flag | Effect |
|---|---|
| `--with-files` | also tar public + private files |
| `--compress` | gzip the two file tars (`.tgz` instead of `.tar`) |
| `--backup-path <dir>` | write all four artefacts somewhere other than the site's backup dir |
| `--include`/`--only` , `--exclude` | partial backup by doctype — **cannot be used with `bench restore`**, see §2.5 |
| `--verbose` | what the cron job uses |

### 1.2 The four files

They land in `sites/<site>/private/backups/`. Naming is
`<YYYYmmdd_HHMMSS>-<site slug>-<kind>` where the slug is the site name with dots
replaced by underscores (`site.replace(".", "_")`). **[V]** —
`frappe/utils/backups.py:86` and `:215-220`.

Real example, the Kantishiva/Mandi production backup taken 2026-08-16 17:36 **[V]**
(file listing on disk):

| File | Size | What it is |
|---|---|---|
| `20260816_173653-mandi_trustbit_in-database.sql.gz` | 1.8 M | gzipped `mariadb-dump` of the site database |
| `20260816_173653-mandi_trustbit_in-files.tar` | 90 K | `./<site>/public/files/…` |
| `20260816_173653-mandi_trustbit_in-private-files.tar` | 10 K | `./<site>/private/files/…` |
| `20260816_173653-mandi_trustbit_in-site_config_backup.json` | 116 B | a **byte-for-byte copy of `site_config.json`** |

The dump carries a frappe metadata header. Read it before you restore anything **[V]**:

```bash
$ gzcat 20260816_173653-mandi_trustbit_in-database.sql.gz | head -12
-- begin frappe metadata
-- [frappe]
-- version = 15.100.0
-- branch = version-15
-- end frappe metadata
--
/*M!999999\- enable the sandbox mode */
-- MariaDB dump 10.19  Distrib 10.6.23-MariaDB, for debian-linux-gnu (x86_64)
--
-- Host: 127.0.0.1    Database: _b7f182df37f7099e
```

That header is the source of truth for §3 (version alignment) — you do not have to
guess what the source server was running.

### 1.3 `site_config_backup.json` is a secrets file

`copy_site_config()` does a straight `n.write(c.read())` of the live
`site_config.json` **[V]** (`frappe/utils/backups.py:375-380`). Whatever is in the site
config is in the backup: `db_password`, `encryption_key`, SMTP creds, API keys.

On the Mandi backup the keys were `db_name`, `db_password`, `db_type`,
`developer_mode` **[V]**.

**Rule:** backup directories go in `.gitignore` and never into a client repo. This was
handled explicitly on Kantishiva — `Backups/` and `_server_snapshot/` are excluded, only
the inner app repo is pushed **[B]**.

### 1.4 The 6-hourly cron

`bench setup backups` writes exactly one crontab entry for the `frappe_user`
**[V]** — reproduced from `bench/bench.py:470-491` and rendered through
python-crontab to confirm the literal line:

```cron
0 */6 * * * cd /home/frappe/frappe-bench && /usr/local/bin/bench --verbose --site all backup >> /home/frappe/frappe-bench/logs/backup.log 2>&1 # bench auto backups set for every 6 hours
```

Three things to know:

1. **The generated job has no `--with-files`.** **[V]** — the source builds
   `backup_command = f"cd {bench_dir} && {sys.argv[0]} --verbose --site all backup"`
   with nothing appended. If you want files (you do), edit the crontab and add the
   flag. Kantishiva runs "every 6 hours with files" **[B]**, i.e. the entry was edited
   after generation.
2. **`bench init` already installs this cron** unless you passed `--no-backups`
   **[V]** (`bench/utils/system.py:114-115` — it is `bench init`, not
   `bench setup production`, that calls `bench.setup.backups()`). Check with
   `crontab -u frappe -l` before adding a second one; `bench setup backups` is
   idempotent and skips if the identical `job_command` is already present **[V]**.
3. Backups land on the **same disk as the site**. Copy them off the box separately —
   a dead VPS takes its backups with it.

```bash
# inspect / edit
crontab -u frappe -l
crontab -u frappe -e     # add --with-files after `backup`
```

Verify by taking one by hand and checking four files appear, which is what was done on
Kantishiva **[B]**.

---

## 2. Restoring onto another machine WITHOUT the MariaDB root password

This is the single most reusable trick in this document. It is how the production
`mandi.trustbit.in` database got into the local `~/frappe-bench-v15` bench on 2026-08-16,
where the local MariaDB root password was unknown and `sudo` also wanted a password
**[B]**.

### 2.1 Why it works

**A frappe database dump contains no `CREATE DATABASE` and no `USE` statement.** It is a
bare list of `DROP TABLE` / `CREATE TABLE` / `INSERT` against whatever schema the client
is currently connected to. So it imports into *any* database, whatever its name.

Verified directly **[V]**:

```bash
$ gzcat 20260816_173653-mandi_trustbit_in-database.sql.gz | grep -ciE "^(CREATE DATABASE|USE )"
0
```

The source database was `_b7f182df37f7099e` (see the header in §1.2); it was imported
into the local site's database `_62a6c5b61dc5570b` with zero errors **[B]**.

And the site's own DB user has everything it needs **[V]** — run on the local bench:

```
$ mysql -h 127.0.0.1 -u _62a6c5b61dc5570b -p<db_password> -N -B -e "SHOW GRANTS FOR CURRENT_USER();"
GRANT USAGE ON *.* TO `_62a6c5b61dc5570b`@`localhost` IDENTIFIED BY PASSWORD '*E58...'
GRANT ALL PRIVILEGES ON `_62a6c5b61dc5570b`.* TO `_62a6c5b61dc5570b`@`localhost`
```

`ALL PRIVILEGES` on its own schema → it can `DROP`, `CREATE`, `INSERT`.
`USAGE` (i.e. nothing) globally → it **cannot** `CREATE DATABASE`. That single line
explains both halves of this section: manual import needs no root, site *creation* does.

Frappe itself agrees: `DbManager.restore_database()` builds
`gzip -cd … | sed … | mariadb --user=<site user> --password=<site password> <db>` —
**the site user, not root** **[V]** (`frappe/database/db_manager.py`). Root is only used
by `setup_database()` to create the database and user in the first place.

### 2.2 The recipe

Run from the bench root on the **target** machine. Substitute your site.

```bash
cd ~/frappe-bench-v15
SITE=mandi.local
DUMP=/path/to/20260816_173653-mandi_trustbit_in-database.sql.gz
MYSQL=/opt/homebrew/bin/mysql          # macOS/brew. On Ubuntu: mariadb (or mysql)
MYSQLDUMP=/opt/homebrew/bin/mysqldump  # on Ubuntu: mariadb-dump (or mysqldump)

DB=$(python3   -c "import json;print(json.load(open('sites/$SITE/site_config.json'))['db_name'])")
DBPW=$(python3 -c "import json;print(json.load(open('sites/$SITE/site_config.json'))['db_password'])")
echo "target db = $DB"
```

**Step 1 — safety dump of what is about to be destroyed.** Same flags frappe uses
(`--single-transaction --quick --lock-tables=false`, **[V]** from
`frappe/database/__init__.py::get_command`). This was executed in this session against
the live local site DB and completed with exit 0, 1.88 MB out, no stderr **[V]**:

```bash
"$MYSQLDUMP" --single-transaction --quick --lock-tables=false \
  -h 127.0.0.1 -u "$DB" -p"$DBPW" "$DB" | gzip > /tmp/pre-restore-$DB.sql.gz
```

**Step 2 — generate a drop script for every object in the schema.** Executed in this
session against the local site DB; produced 749 lines for 742 base tables + 5 sequences
**[V]**:

```bash
{
  echo "SET FOREIGN_KEY_CHECKS=0;"
  "$MYSQL" -h 127.0.0.1 -u "$DB" -p"$DBPW" "$DB" -N -B -e \
    "SELECT CONCAT('DROP ', IF(table_type='SEQUENCE','SEQUENCE','TABLE'), ' IF EXISTS \`', table_name, '\`;')
     FROM information_schema.tables WHERE table_schema = DATABASE();"
  echo "SET FOREIGN_KEY_CHECKS=1;"
} > /tmp/drop_all.sql

wc -l /tmp/drop_all.sql   # sanity: should be ~ (number of tables + 2)
head -3 /tmp/drop_all.sql
```

Frappe sites use `BASE TABLE` and `SEQUENCE` only. If a site somehow has a `VIEW`, the
`IF()` above emits `DROP TABLE` for it and that statement fails — extend the `IF()` or
drop the view by hand **[U]**.

**Step 3 — drop, then import.** The `sed` filters are copied verbatim from frappe's own
restore pipeline **[V]** (`db_manager.py`); the first strips MariaDB 11's sandbox-mode
line (present in the Mandi dump header, see §1.2) which older clients choke on, the
second strips `SQL SECURITY DEFINER` from views:

```bash
set -o pipefail

"$MYSQL" -h 127.0.0.1 -u "$DB" -p"$DBPW" "$DB" < /tmp/drop_all.sql

gzip -cd "$DUMP" \
  | sed '/\/\*M\{0,1\}!999999\\- enable the sandbox mode \*\//d' \
  | sed '/\/\*![0-9]* DEFINER=[^ ]* SQL SECURITY DEFINER \*\//d' \
  | "$MYSQL" -h 127.0.0.1 -u "$DB" -p"$DBPW" "$DB"
echo "import exit = $?"
```

`set -o pipefail` is not optional — without it, `gzip … | mysql` reports success even
when the import dies. Frappe's own pipeline opens with the same line **[V]**.

**Step 4 — sanity count before you celebrate:**

```bash
"$MYSQL" -h 127.0.0.1 -u "$DB" -p"$DBPW" "$DB" -N -B -e \
  "SELECT COUNT(*) FROM information_schema.tables WHERE table_schema=DATABASE();"
```

Then go straight to §4 (migrate / build / verify).

### 2.3 When root IS required

Root is needed for anything that creates or destroys a database **or a database user**:

* `bench new-site`
* `bench restore`
* `bench drop-site`

`bench restore` calls `_new_site(..., force=True)` → `setup_database(force=True)`, which
does, in order **[V]** (`frappe/database/mariadb/setup_db.py:21-52`):

```
dbman.delete_user(db_name)      # drops the site's DB user
dbman.drop_database(db_name)    # DROPS THE DATABASE
dbman.create_user(db_name, ...)
dbman.create_database(db_name)
dbman.grant_all_privileges(db_name, db_name)
```

So `bench restore` is **destructive by design**, not just "restore-y". If you have the
root password and a site you are happy to blow away, it is the right tool:

```bash
bench --site <site> restore /path/to/<ts>-<slug>-database.sql.gz \
  --db-root-password '<mariadb root pw>' \
  --with-public-files  /path/to/<ts>-<slug>-files.tar \
  --with-private-files /path/to/<ts>-<slug>-private-files.tar
```

**[V]** — those four option names are verbatim from `frappe/commands/site.py`
(`--db-root-username` / `--db-root-password` are aliased as
`--mariadb-root-username` / `--mariadb-root-password`).

If you omit `--db-root-password` on a non-TTY SSH session the command hangs on
`getpass.getpass("MySQL root password: ")` **[V]** (`setup_db.py:130`) with no visible
prompt. Same class of bug as the missing `--yes` in
[ssl-nginx-and-production.md](ssl-nginx-and-production.md).

**Two `new-site` options worth knowing** **[V]** (`frappe/commands/site.py`):

* `--source-sql <file>` — bootstrap the new site straight from a dump instead of the
  blank framework schema, in one command.
* `--setup-db` / `--no-setup-db` — "Create user and database in mariadb/postgres; only
  bootstrap if false". With `--no-setup-db` you create the database and user yourself
  (with whatever credentials you do have) and frappe skips the root-only step. **[U]** as
  a Trustbit procedure — not used on any recorded build; the §2.2 path is the tested one.
* `--force` on `new-site` means "Force restore if site/database already exists", i.e.
  **it drops the existing database**. Never pass it after the first successful creation.

### 2.4 The decision table

| Situation | Use |
|---|---|
| Target site already exists, you don't have DB root | §2.2 manual drop + import |
| Target site already exists, you *do* have root, don't mind recreating it | `bench restore` |
| No site on the target yet | `bench new-site` (needs root) then §2.2 or `bench restore` |
| Bringing data into a **different** site name than the source | §2.2 works unchanged — the dump has no `USE` |

Local reality check from the Mandi restore **[B]**: "Local MariaDB root password is
unknown; `sudo` needs a password too. Any operation that must create a *new*
site/database (`bench new-site`, `bench restore`) will need it. Importing into an
existing site DB does not."

### 2.5 Partial backups cannot be restored with `bench restore`

If the dump was taken with `--include`/`--exclude`, `bench restore` refuses **[V]**:

```
Partial Backup file detected. You cannot use a partial file to restore a Frappe site.
Use `bench partial-restore` to restore a partial backup to an existing site.
```

---

## 3. Align app versions BEFORE you restore

**Restoring a newer site database onto older app code is a backwards migration and is
not supported.** Patches that already ran on the source will not un-run; schema the new
code expects will be missing; `bench migrate` will not save you.

### 3.1 What frappe checks (and what it does not)

`bench restore` calls `is_downgrade(sql_file_path)` and, if the backup is newer than the
installed frappe, prints **[V]** (`frappe/installer.py`; the two version numbers are
interpolated at runtime):

```
Your site is currently on Frappe <installed> and your backup is <backup>.
This is not recommended and may lead to unexpected behaviour. Do you want to continue anyway?
```

Two limits you must know **[V]** — read the function, it is 20 lines:

* it returns `False` immediately unless `db_type == "mariadb"`;
* **it only compares the `frappe` version.** ERPNext, india_compliance and your custom
  app are not checked at all. A matching frappe with a stale erpnext will sail straight
  through this gate and break later.

The manual import in §2.2 bypasses the check entirely — which is exactly why you must do
the alignment yourself.

Other validation strings from the same path, worth grepping for **[V]**:

```
Table `__Auth` not found in file.
<path> is an empty file!
```

### 3.2 Read the required versions off the source

Two sources, both authoritative:

```bash
# 1. the dump header (see §1.2) — gives frappe version + branch
gzcat <ts>-<slug>-database.sql.gz | head -5

# 2. on the source server, before you wipe it
bench version
bench --site <site> list-apps
```

For Mandi this produced **[B]**:

| App | Version | Branch |
|---|---|---|
| frappe | 15.100.0 | version-15 |
| erpnext | 15.97.0 | version-15 |
| india_compliance | 15.25.4 | version-15 |
| trustbit_mandi | 0.0.1 | main @ `55cd6a7` |

### 3.3 Pinning — bench clones are SHALLOW and the remote is `upstream`

Both facts bite. Verified on `~/frappe-bench-v15` **[V]**:

```bash
$ cd apps/frappe && git remote -v
upstream	https://github.com/frappe/frappe.git (fetch)
upstream	https://github.com/frappe/frappe.git (push)

$ git rev-parse --is-shallow-repository
true

$ git config --get remote.upstream.fetch
+refs/heads/version-15:refs/remotes/upstream/version-15
```

* The remote is **`upstream`**, not `origin` — `git fetch origin` fails with
  `fatal: 'origin' does not appear to be a git repository`. This comes from
  `shallow_clone: true` in `sites/common_site_config.json` **[V]**.
* The clone is shallow (`.git/shallow` present, 222 grafts) **and** the fetch refspec is
  single-branch, so the tag you want is very likely not in the local object store.
* **Apps you added yourself may use `origin`.** On this bench `trustbit_mandi` and
  `item_creator` are on `origin`, frappe/erpnext/india_compliance on `upstream` **[V]**.
  Always run `git remote -v` first.

The exact sequence that pinned this bench is recorded in the git reflog **[V]** —
`git reflog show upstream/version-15` reads
`fetch --tags --depth=100 upstream: fast-forward`, and `git reflog` reads
`checkout: moving from version-15 to v15.100.0`:

```bash
cd ~/frappe-bench-v15/apps/frappe
git remote -v                                   # confirm the remote name first
git fetch --tags --depth=100 upstream
git checkout v15.100.0

cd ../erpnext
git fetch --tags --depth=100 upstream
git checkout v15.97.0

cd ../india_compliance
git fetch --tags --depth=100 upstream
git checkout v15.25.4

cd ~/frappe-bench-v15
bench setup requirements        # re-resolve python + node deps for the pinned code
```

`--depth=100` was enough to reach one-patch-back tags. If `git checkout <tag>` fails with
`error: pathspec '<tag>' did not match any file(s) known to git` after the fetch, deepen
further (`--depth=500`) or `git fetch --unshallow upstream` **[U]** — the deeper fetches
were not needed on this bench.

You will land on a **detached HEAD**. That is correct and expected; `git status -sb`
shows `## HEAD (no branch)` **[V]**. Do not "fix" it by checking the branch back out —
that undoes the pin.

### 3.4 Trade-off: pin, or upgrade the source first?

Pinning the target to the source's versions is always the safe move for a *move*. If you
actually want the newer code, do it as a **separate, later step**: restore at the pinned
versions, confirm the site is healthy, then upgrade forwards and `bench migrate` again.
Never combine "move the data" and "change the version" in one operation — when it breaks
you will not know which half did it.

**v15 → v16 is not this procedure.** From the Kantishiva build record **[B]**: the v15
apps "are **not** installed here. Both target Frappe v15; moving them to v16 is a real
migration with breaking API changes, not a reinstall. Same for the Mandi v15 database
backup." Treat a major-version jump as an app-porting project, not a restore.

---

## 4. After the restore

Redis must already be running — see §8.1. Then:

```bash
cd ~/frappe-bench-v15
bench --site <site> migrate
bench build
```

`bench migrate` runs patches, syncs schema and rebuilds files/translations **[V]**
(`frappe/commands/site.py`). Two flags exist if it stalls **[V]**: `--skip-failing`
(skip patches that error) and `--skip-search-index`. Reach for `--skip-failing` only
after reading the traceback — a skipped patch is a schema landmine.

`bench build` compiles JS/CSS. Options **[V]**: `--app <app>`, `--apps <a,b>`,
`--production`, `--force`. On a 1 vCPU box, **build per-app** — v16 runs a React 19 +
Vite 8 + Tailwind 4 production build that did not exist in v15, frappe caps the Node
heap at 75% of *physical* RAM and ignores swap, and exporting `NODE_OPTIONS` has no
effect because frappe overrides it **[B]**.

### Verification checklist

This is the set used to sign off the Mandi restore **[B]**:

```bash
# process / API level
bench --site <site> execute frappe.ping                 # -> "pong"
curl -s http://127.0.0.1:8004/api/method/frappe.ping    # -> {"message":"pong"}

# versions
bench version
bench --site <site> list-apps
```

`bench --site mandi.local execute frappe.ping` was run in this session against the
restored site and printed `"pong"` **[V]**. The whitelisted method is
`frappe.ping` — `@whitelist(allow_guest=True)`, `frappe/__init__.py:2437-2439` **[V]** —
so over HTTP the path is `/api/method/frappe.ping` and the response is wrapped as
`{"message":"pong"}`. (`PROJECT_STATE.md` records this as `/api/method/ping` **[B]**;
`frappe.ping` is the form that matches the source.)

Then check the data actually arrived, through the ORM rather than the UI:

```bash
bench --site <site> console
```
```python
frappe.db.count("Sales Invoice")
frappe.db.count("Customer")
frappe.get_doc("Deal", "DEAL-18-05-2026-0001")     # a real document must load
frappe.get_all("Company", pluck="name")
```

The Mandi sign-off was: 17 Deals, 27 Deal Deliveries, 13 Vehicle Dispatches, 6 Grain
Purchases, 12 Sales Invoices, 13 Customers, 28 Items, 5 Users, 2 Companies; all 9 custom
reports registered; the `Mandi` workspace present; login page rendering at the same
347 KB as prod **[B]**.

**Passwords survive the restore unchanged** — the restored `__Auth` table is production's,
so production logins work on the copy **[B]**. Treat a restored dev copy as carrying
production credentials.

---

## 5. Restoring files

### 5.1 The tar layout

Verified by listing the real archives **[V]**:

```
$ tar -tf 20260816_173653-mandi_trustbit_in-files.tar
./mandi.trustbit.in/public/files/
./mandi.trustbit.in/public/files/finicone.png
./mandi.trustbit.in/public/files/TrustbitSplash.png
./mandi.trustbit.in/public/files/trustbit.png

$ tar -tf 20260816_173653-mandi_trustbit_in-private-files.tar
./mandi.trustbit.in/private/files/
```

Note the leading `./` **and** the source site name. Two path components must be stripped.

### 5.2 How frappe does it, and the one-liner equivalent

`extract_files()` copies the tar into `sites/<site>/` and runs
`tar xvf <tar> --strip 2` from there **[V]** (`frappe/installer.py`). `--strip 2` removes
`.` and `<source site name>`, leaving `public/files/…` relative to the *target* site
directory — which is why a backup from `mandi.trustbit.in` extracts correctly into
`sites/mandi.local/`.

```bash
cd ~/frappe-bench-v15
tar xvf /path/to/<ts>-<slug>-files.tar         --strip 2 -C sites/<site>/
tar xvf /path/to/<ts>-<slug>-private-files.tar --strip 2 -C sites/<site>/

# on a server, fix ownership afterwards
sudo chown -R frappe:frappe sites/<site>/public/files sites/<site>/private/files
```

Or, via bench when you are already running `bench restore` **[V]**:
`--with-public-files <tar>` / `--with-private-files <tar>`.

Compressed backups (`--compress`) produce `.tgz`; frappe handles both extensions
(`tar zxvf … --strip 2`) **[V]**.

---

## 6. Moving a site to a new server, end to end

The sequence that got Mandi from a Hostinger VPS to a local bench **[B]**, generalised.
Steps 1–3 happen on the **source**, 4 onwards on the **target**.

```bash
# --- SOURCE ---
# 1. record the versions (§3.2) — do this BEFORE anything else
bench version > /tmp/versions.txt
bench --site <site> list-apps >> /tmp/versions.txt
cd apps/<custom_app> && git log --oneline -1 && git remote -v

# 2. take the backup
bench --site <site> backup --with-files

# 3. pull the four files + the live site_config off the box
#    (site_config.json, not just the backup copy — you want encryption_key, see §7)
```

```bash
# --- TARGET ---
# 4. build a bench at the SAME major version, pin the apps (§3.3)
# 5. install the same app set in the same order (frappe, erpnext, then the rest)
# 6. create the site  (needs DB root)
bench new-site <site> --db-root-password '<root pw>' --admin-password '<admin pw>'
bench --site <site> install-app erpnext
bench --site <site> install-app <custom_app>

# 7. import the data — §2.2 manual, or bench restore if you have root
# 8. restore the files — §5.2
# 9. carry encryption_key across if the source had one — §7
# 10. migrate, build, verify — §4
```

Things to also copy off the old box for reference (this is what
`_server_snapshot/meta/` holds on Kantishiva) **[B]**:
`common_site_config.json`, `sites/apps.txt`, the live `site_config.json`,
`bench version` output, `bench --site <site> list-apps` output.

Order matters at step 5: install apps **before** importing, so `apps.txt` and the app
python packages exist when `bench migrate` runs their patches.

A shortcut that is worth checking for: on the Mandi move, **no new bench was needed** —
`~/frappe-bench-v15` already had the custom app at the same commit as prod and the same
india_compliance version, with an empty `mandi.local` site sitting there **[B]**. Look
for an existing bench at the right versions before building a new one. But check *which*
bench: the shared `~/frappe-bench` was on frappe 15.81 and hosted four other clients, so
it was explicitly ruled out **[B]**.

---

## 7. `encryption_key` — carry it across or lose your secrets

If the source `site_config.json` has an `encryption_key`, **you must copy that exact
value into the target `site_config.json`**. Frappe encrypts stored secrets (email account
passwords, connected-app tokens, any Password-fieldtype value) with Fernet, keyed on it.

What happens if you don't **[V]** (`frappe/utils/password.py:194-210`, verbatim):

```
Failed to decrypt key <Doctype>.<name>.<fieldname>

Encryption key is invalid! Please check site_config.json

If you have recently restored the site you may need to copy the site config contaning original Encryption Key.
```

(Yes, "contaning" — the typo is in frappe. It makes the string a reliable grep target.)

Related string, thrown when the key is present but malformed **[V]**:

```
Encryption key is in invalid format!
```

The trap: **`get_encryption_key()` silently generates a new key and writes it into
`site_config.json` if none is present** **[V]** (`password.py:216-224`). So a restored
site with no key looks perfectly healthy right up until something calls
`get_decrypted_password()` — usually the scheduler trying to send an email, weeks later.

```bash
# on the source
grep encryption_key sites/<site>/site_config.json

# on the target, if the source had one
bench --site <site> set-config encryption_key '<exact value from source>'
bench --site <site> clear-cache      # fine on a freshly restored, not-yet-live site; on a
                                     # live one use the targeted form (v15 §11)
```

The Mandi production site had **no** `encryption_key` **[B]** — verified again here, the
backed-up config carries only `db_name`, `db_password`, `db_type`, `developer_mode`
**[V]**. That is why that restore was clean. Do not assume the next one will be.

### 7.1 Why a site has no key — it is generated *lazily*, and that is preventable

This is not random. **Frappe does not create `encryption_key` at `bench new-site`; it
creates it the first time something is actually encrypted.** Four freshly-built v15 sites,
checked on 2026-09-01 — every one of them had a config containing only
`db_name`, `db_password`, `db_type`, `developer_mode`, `host_name`, and **no key**
**[verified]**:

```console
$ python3 -c "import json;print(sorted(json.load(open('sites/<site>/site_config.json')).keys()))"
['db_name', 'db_password', 'db_type', 'developer_mode', 'host_name']
```

Two consequences that matter more than they first appear:

1. **A backup taken before first encryption captures no key**, so a
   `site_config_backup.json` from a young site protects nothing — there is nothing yet to
   protect. Fine today; useless the moment someone configures an Email Account, because the
   key is then generated on a live site and every backup *after* that point depends on a
   value your escrow does not have.
2. **An escrow built early goes stale silently.** You copy the keys off, feel covered, and
   the real key appears a week later.

**Force it at build time**, so every backup from the first one onward carries a stable key:

```bash
# generates and persists the key immediately — safe and idempotent on an existing site
bench --site <site> execute frappe.utils.password.get_encryption_key

# then escrow, and re-escrow on every backup run in case a site is reinstalled
grep encryption_key sites/<site>/site_config.json
```

Do this for every site as part of provisioning, not as part of incident response.

### 7.2 A restore drill can pass while proving nothing

The drill is the point of the whole exercise, so it is worth being paranoid about **what
you actually verified**. On this build the first drill reported success and had restored
nothing:

```
Error: Got unexpected extra argument (…-private-files.tar)
```

`bench restore` aborted on a malformed argument, the script continued, and the checks that
followed — "site loads", "key matches", "decryption works" — all passed **against the
freshly-created empty site**. Every assertion was true and every one was meaningless.

**Verify with numbers that can only come from real data**, never with "did it error":

```bash
bench --site <scratch> list-apps        # erpnext present, not just frappe
bench --site <scratch> mariadb -e "SELECT COUNT(*) FROM tabDocType;"   # ~776 on ERPNext v15
bench --site <scratch> mariadb -e "SELECT COUNT(*) FROM information_schema.tables
                                   WHERE table_schema=DATABASE();"     # ~707
```

A fresh site has ~2 users, no `erpnext` in `list-apps`, and far fewer tables. Those three
numbers separate a real restore from an empty one instantly.

Also note the file arguments are **options taking a path**, and a wildcard like
`*-files.tar` matches `-private-files.tar` too — which is exactly how the malformed
invocation arose:

```bash
B=sites/<site>/private/backups
SQL=$(ls -t $B/*-database.sql.gz | head -1)
PUB=$(ls -t $B/*[0-9]-*-files.tar | grep -v private | head -1)
PRIV=$(ls -t $B/*-private-files.tar | head -1)
[ -f "$SQL" ] && [ -f "$PUB" ] && [ -f "$PRIV" ] || { echo "path resolution failed"; exit 1; }

bench --site <scratch> restore "$SQL" \
  --db-root-username root --db-root-password "$DBPW" \
  --with-public-files "$PUB" --with-private-files "$PRIV"
```

Resolve the paths into variables and assert they are real files **before** invoking restore.

### 7.3 The stock backup cron omits `--with-files`

`bench setup production` installs a 6-hourly cron that runs `bench --site all backup`
**without** `--with-files` **[verified — observed in `crontab -l` on a fresh 24.04 build]**:

```
0 */6 * * * cd /home/frappe/frappe-bench && .../bench --verbose --site all backup >> ... # bench auto backups set for every 6 hours
```

Every attachment, item image and uploaded PDF is therefore missing from the automated
backups, and you find out during a restore. Replace it with your own job rather than
supplementing it — two schedules churn `backup_limit` against each other.

Also raise `backup_limit` (System Settings, default **3**). Pruning runs on a scheduler
hook, so with a 6-hourly job the default leaves roughly a 12–18 hour window — a Friday
incident found on Monday is already unrecoverable locally.

```bash
bench --site <site> execute frappe.db.set_single_value \
  --args "['System Settings','backup_limit',8]"
```

Frappe's own docs link, emitted in the error above:
`https://frappecloud.com/docs/sites/migrate-an-existing-site#encryption-key`.

---

## 8. Gotchas

### 8.1 Redis must be running before `migrate` — v15 + v16

**Symptom** (verbatim f-string from `frappe/utils/background_jobs.py:512` **[V]**,
combined with redis-py's connection error text from
`redis/connection.py:1022-1025` **[V]**):

```
Please make sure that Redis Queue runs @ redis://127.0.0.1:11000. Redis reported error: Error 111 connecting to 127.0.0.1:11000. Connection refused.
```

Error 111 is `ECONNREFUSED` — nothing is listening, not a password problem. (If it were
a password problem you would instead get
`Wrong credentials used for default user. You can reset credentials using` `bench create-rq-users` `CLI and restart the server` **[V]**.)

**Root cause:** bench runs its *own* redis instances, separate from any system redis.
Default ports are 11000 (queue) and 13000 (cache); dev benches offset them —
`~/frappe-bench-v15` uses 11003/13003 **[V]** (`sites/common_site_config.json`).
Under supervisor they are managed programs; on a dev box or a fresh server *nothing
starts them for you*.

**Fix — dev bench** (exactly what the Mandi runbook does) **[B]**:

```bash
cd ~/frappe-bench-v15
redis-server config/redis_cache.conf --daemonize yes    # port 13003
redis-server config/redis_queue.conf --daemonize yes    # port 11003
bench serve --port 8004        # or `bench start` for the full dev stack
```

**Fix — server:** `supervisorctl status` and make sure the `-redis-cache` /
`-redis-queue` programs are RUNNING before you migrate.

Related **[B]**: "Bench's redis must be running before `install-app` on a new box, or
the install fails silently." And: restart bench after every app install; a bench started
before the install hits a scheduler `ModuleNotFoundError`.

### 8.2 Root-run bench commands leave root-owned files — v15 + v16

**Symptom:** after a restore or migrate that was run as root, the site works for a while
and then throws `PermissionError` — on Kantishiva it was on
`logs/render-template.log` during a PDF render **[B]**. 125 root-owned files were found
in that bench.

**Root cause:** bench's own cleanup, `fix_prod_setup_perms()`, only chowns `logs/*` and
`config/*` **[V]** (`bench/utils/system.py`) — and only when it is called at all, i.e.
during `bench setup production`. Everything a root-run `bench migrate` / `bench build`
touched under `sites/` and `apps/` keeps root ownership. The `frappe` user then cannot
write to it.

**Fix — run it after *any* root-run bench command, unconditionally:**

```bash
chown -R frappe:frappe /home/frappe/frappe-bench
```

Better: don't run bench as root. Wrap it:

```bash
sudo -u frappe -H bash -lc 'export PATH=$HOME/.local/bin:$PATH; cd $HOME/frappe-bench && bench --site <site> migrate'
```

The explicit `export PATH` is needed because Ubuntu's `~/.bashrc` returns early for
non-interactive shells, so the bench PATH line written there is never picked up.

**Find the damage:**

```bash
find /home/frappe/frappe-bench ! -user frappe -print | head -50
```

### 8.3 `developer_mode = 0` means UI doctype edits never reach the app folder — v15 + v16

**Symptom:** an engineer edits a DocType in the UI on a production site, it works, and
the change is completely absent from git. Next `bench migrate` on another site, or the
next deploy, and it is gone.

**Root cause** **[V]** — `frappe/core/doctype/doctype/doctype.py:537-543`:

```python
allow_doctype_export = (
    not self.custom
    and not frappe.flags.in_import
    and (frappe.conf.developer_mode or frappe.flags.allow_doctype_export)
)
if allow_doctype_export:
    self.export_doc()
```

With `developer_mode` falsy, `export_doc()` is never called: the change lives only in the
database. Same for `export_module_json()` (print formats, reports, etc.) — gated on
`frappe.conf.developer_mode` **[V]** (`frappe/modules/utils.py:35`).

You also cannot create a standard DocType at all **[V]**:

```
Not in Developer Mode! Set in site_config.json or make 'Custom' DocType.
```

and customisation export is blocked **[V]**:

```
Only allowed to export customizations in developer mode
```

**Rules:**

* Production sites run `developer_mode = 0`. Mandi production did **[B]**; so does BBPL
  production **[B]**.
* Do schema work on a dev site with `developer_mode = 1`, commit the JSON, deploy.
* If someone already made a UI change on prod: you cannot recover it by flipping the
  flag on prod — flip it on a **restored copy**, re-save the doctype there to force the
  export, and commit that.

```bash
bench --site <dev site> set-config developer_mode 1
bench --site <dev site> clear-cache
```

Note the Mandi *backup's* `site_config_backup.json` carries `developer_mode` **[V]** —
so a naive "copy the backed-up site_config over the target's" can silently import a
production `developer_mode: 0` onto your dev box, or the reverse. Merge keys
deliberately; never overwrite the file wholesale.

### 8.4 MariaDB version skew across the restore

The Mandi dump came from MariaDB **10.6.23** (Ubuntu 22.04) and imported into a local
Homebrew MariaDB **12.1** with 0 errors **[B]/[V]** (versions read from the dump header
and the local client). It works, but:

* newer-server → older-client is the risky direction, not the one above;
* the `/*M!999999\- enable the sandbox mode */` line that MariaDB 11+ emits breaks older
  clients, which is exactly why frappe `sed`s it out (§2.2) **[V]**;
* on macOS Homebrew may have both `mariadb` (serving :3306) and `mariadb@10.6`
  (client-only) installed — use the full path, `/opt/homebrew/bin/mysql` **[B]**.

v16 targets MariaDB **11.8** specifically — that is the version frappe v16 CI tests
against **[B]**.

---

## Quick reference

```bash
# backup, with files
bench --site <site> backup --with-files

# where they land
ls -la sites/<site>/private/backups/

# what version made this dump
gzcat <ts>-<slug>-database.sql.gz | head -5

# import into an existing site DB, no root  (see §2.2 for the full, safe version)
gzip -cd <dump>.sql.gz | sed '/sandbox mode/d' | mysql -u "$DB" -p"$DBPW" "$DB"

# files
tar xvf <ts>-<slug>-files.tar --strip 2 -C sites/<site>/

# finish
bench --site <site> migrate && bench build
bench --site <site> execute frappe.ping        # -> "pong"

# after any root-run bench command
chown -R frappe:frappe /home/frappe/frappe-bench
```
