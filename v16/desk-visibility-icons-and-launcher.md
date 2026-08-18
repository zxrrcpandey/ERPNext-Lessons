# Making an app visible on the desk — launcher icon, desktop tile, workspace

Applies to: **v16 primarily; §1–2 apply to v15 with a different answer**

Real case: `trustbit_mandi` and `item_creator` on frappe **16.31.0** (Kantishiva,
2026-08-18). Both apps installed cleanly, `bench migrate` was green, the Workspace
records existed — and the apps appeared **nowhere**: not in the app switcher, not on
the desk tile grid, not in the sidebar. Getting them visible took fixes on **three
independent surfaces**, and fixing the wrong one first cost hours.

**Evidence tags:** `[KANTISHIVA]` = run and observed on the 16.31.0 box.
`[V15-LOCAL]` = read from the frappe **15.100.0** checkout. `[V16-LOCAL]` = read from
a frappe **16.14.0** checkout (line numbers cited from it may drift by release).

---

## 0. The map — four surfaces, four separate mechanisms

Nothing you do to one of these affects the others. Diagnose against this table first:

| Surface | What feeds it | Where it breaks |
|---|---|---|
| **App switcher / launcher** (top-left grid, the `/apps` page) | the `add_to_apps_screen` hook, read by `frappe.apps.get_apps()` | hook missing (§1), wrong route prefix (§2) |
| **Desk tile grid** (`/desk` home) | **Desktop Icon** records, auto-generated per app install | records missing or migrate-swept (§3), stale per-user layout (§6) |
| **Tile artwork** | per-app SVG files scanned at boot | no files → grey letter tile (§4) |
| **Workspace sidebar** (left panel) | Workspace (+ Workspace Sidebar) records | v15-authored JSON lacks `type`/`app` → see [../apps/porting-a-large-app-to-v16.md](../apps/porting-a-large-app-to-v16.md) §7; flat icon-less items (§5) |

Both the launcher (§1–2) and the tile grid (§3–5) exist so the fix for "no icon"
depends on **which** icon the user means. Ask, or fix all of them.

---

## 1. No `add_to_apps_screen` hook, no launcher icon. Ever.

Applies to: **both versions** — same gate in v15 and v16

`frappe.apps.get_apps()` builds the app switcher list, and it skips any app without
the hook — silently, unconditionally `[KANTISHIVA 16.31.0 apps.py; same gate at
V15-LOCAL apps.py:35]`:

```python
app_details = frappe.get_hooks("add_to_apps_screen", app_name=app)
if not len(app_details):
    continue          # <- the gate. No hook, no icon, no error anywhere.
```

The `bench new-app` boilerplate our apps were generated from ships this hook
**commented out** (pointing at a non-existent `logo.png`) `[KANTISHIVA]`. So the
default state of a custom app is: invisible in the launcher, forever, with nothing
in any log. What working apps declare:

```python
add_to_apps_screen = [
    {
        "name": "trustbit_mandi",
        "logo": "/assets/trustbit_mandi/images/trustbit-mandi.svg",
        "title": "Trustbit Mandi",
        "route": "/desk/mandi",      # v16 prefix! see §2
        # optional: "has_permission": "myapp.check_app_permission",
    }
]
```

Details that matter:

- Every key is read with `.get()` — a missing `logo` or `route` yields `None`
  silently, not an error. Ship both, and `curl` the logo URL for a 200 after deploy.
- **Non-System-Manager users** only see apps whose setup wizard is *completed* or
  *not required*. An app that ships no setup wizard is automatically "not required",
  so plain custom apps show for everyone — verified on Kantishiva:
  `not_required = ['india_compliance', 'item_creator', 'trustbit_mandi']`
  `[KANTISHIVA]`. Apps that DO declare a wizard stay hidden from regular users until
  it is completed.

---

## 2. The route prefix FLIPS between versions — and one wrong route hurts every user

Applies to: **both versions, with opposite correct answers**

v16 renamed the desk URL prefix from `/app/*` to `/desk/*` (`router.js
is_app_route()` returns `path[0] === "desk"` `[V16-LOCAL router.js:110]`; v15
returns `path[0] === "app"` `[V15-LOCAL router.js:119]`). Old `/app/...` URLs
answer **301 with an empty body** — which in `curl` looks like the site is down,
and in the browser costs a full page reload instead of in-app navigation
`[KANTISHIVA]`.

The same rename is enforced against your hook's `route`:

```python
# v15 [V15-LOCAL apps.py:82]          # v16 [V16-LOCAL apps.py:16; same on 16.31.0]
pattern = r"^/app(/.*)?$"             DESK_APP_PATTERN = re.compile(r"^/desk(/.*)?$")
```

And here is the blast radius. `is_desk_apps()` returns False if **any** app's route
fails the pattern, and `get_default_path()` then sends logins to `/apps` instead of
`/desk`. That function is consumed by `login.py`, `auth.py`, `sessions.py`,
`user.py` and `oauth.py` `[KANTISHIVA, by grep of 16.31.0]` — so **one custom app
carrying the other version's prefix changes the post-login landing page for every
user on the site**. The app itself still works; everyone's login just gets worse,
with nothing pointing at the culprit.

Rules:

- On v16, hook routes are `/desk/...`. On v15, `/app/...`. An app that targets both
  versions from one branch cannot hardcode either — derive it or branch it.
- Grep any app you port: `grep -rn '"/app/' <app>` — every hardcoded `/app/` link
  (workspace shortcuts, `msgprint` HTML, JS redirects) becomes a full-reload 301 on
  v16.
- Do not trust code comments here: v16.14's `router.js` still carries comments
  saying desk paths begin with `/app` while the code below them tests `"desk"`
  `[V16-LOCAL router.js:106]`. Read the `return`, not the comment.

---

## 3. "Desktop Icon" — same doctype name, completely different animal

Applies to: **v16** (v15 has a dormant namesake)

v15 ships a `Desktop Icon` doctype too, but it is the **pre-v13 module-desk
leftover** (fields: `module_name`, `blocked`, `force_show`, `_doctype`, `_report`,
`color`…) and nothing in the v15 desk renders it `[V15-LOCAL desktop_icon.json]`.
v16 **rebuilt** the doctype for the new tile grid with a different schema:
`icon_type`, `link_type`, `link_to`, `parent_icon`, `logo_url`, `bg_color`,
`restrict_removal` `[V16-LOCAL desktop_icon.json]`.

Trap from the rename: the color column is **`bg_color`** on v16, `color` on v15 —
`SELECT ... color FROM tabDesktop Icon` throws
`OperationalError: (1054, "Unknown column 'color' in 'SELECT'")` `[KANTISHIVA]`.

**You normally never create these records by hand.** frappe auto-generates them:

```python
# frappe's own hooks.py [V16-LOCAL hooks.py:19; same on 16.31.0]
after_app_install = "frappe.utils.install.auto_generate_icons_and_sidebar"
# plus, for sites upgraded to v16:
# frappe.patches.v16_0.auto_generate_desktop_icon_and_sidebar
```

That routine creates **two kinds** of icon, plus the Workspace Sidebar records:

1. an **App icon** per installed app that declares `add_to_apps_screen`
   (`icon_type="App"`, `link` = the hook's route, `logo_url` = the hook's logo) —
   so §1's hook gates this surface too;
2. a **workspace tile** per public Workspace (`link_type="Workspace Sidebar"`),
   parented under the app icon.

It is existence-guarded and safe to re-run `[V16-LOCAL install.py:185]`:

```bash
bench --site <site> execute frappe.utils.install.auto_generate_icons_and_sidebar
```

Corner case: `label` is the Desktop Icon's primary key, so a workspace named
exactly like its app's title collides with the app icon — frappe resolves it by
hiding one, but a workspace named like a *different* app's title simply loses.
Namespace your workspace titles (same discipline as report names).

### ⚠ The next `bench migrate` DELETES database-only desk records

Learned the hard way, hours after the tiles first worked `[KANTISHIVA]`: migrate
runs `remove_orphan_entities()` (`frappe/model/sync.py`), which — for the
app-level entities **Desktop Icon, Workspace Sidebar and Sidebar Item Group** —
deletes **any record whose `app` is set but has no backing JSON file** at
`<app>/<scrubbed_doctype>/<scrub(name)>.json`. Verbatim from our migrate log:

```
Removing orphan Desktop Icons
Deleting entity Desktop Icon Trustbit Mandi
Deleting entity Desktop Icon Mandi
Deleting entity Desktop Icon Item Management
```

So records created by the auto-generator (or by hand in the console) are live
grenades: they work until the next migrate, then vanish. **ERPNext's tiles
survive every migrate because it ships `erpnext/desktop_icon/*.json` — the same
sync that runs the orphan sweep re-imports file-backed records.** Ship yours the
same way, two files per app:

```jsonc
// <app>/desktop_icon/<scrub(app_title)>.json — the App icon (hidden parent)
{"doctype": "Desktop Icon", "name": "Trustbit Mandi", "label": "Trustbit Mandi",
 "icon_type": "App", "link_type": "External", "link": "/desk/mandi",
 "logo_url": "/assets/trustbit_mandi/images/trustbit-mandi.svg",
 "hidden": 1, "standard": 1, "app": "trustbit_mandi", "idx": 100,
 "modified": "2026-08-18 23:00:00.000000", "roles": []}

// <app>/desktop_icon/<scrub(workspace_name)>.json — the visible tile
{"doctype": "Desktop Icon", "name": "Mandi", "label": "Mandi",
 "icon_type": "Link", "link_type": "Workspace Sidebar", "link_to": "Mandi",
 "parent_icon": "Trustbit Mandi", "icon": "agriculture",
 "hidden": 0, "standard": 1, "app": "trustbit_mandi", "idx": 1,
 "modified": "2026-08-18 23:00:00.000000", "roles": []}
```

Filenames come from `frappe.scrub(name)` — `"Item Management"` →
`item_management.json`. The `modified` timestamp must be **newer than the DB
row's** or the import is skipped as already-current. The same rule applies to
**Workspace Sidebar**: a sidebar customized only in the database is equally
orphan-swept; ship `<app>/workspace_sidebar/<scrub(title)>.json`.

Verified: after shipping all four files, a repeat migrate deleted nothing and
re-imported the records `[KANTISHIVA]`.

---

## 4. Branded tiles are per-app SVG **files**, not a doctype field

Applies to: **v16 only**

The tile grid has two rendering paths `[V16-LOCAL utils.js: get_desktop_icon(),
desktop_icon()]`:

1. **A per-app SVG file** at
   `<app>/public/icons/desktop_icons/{solid,subtle}/<scrub(label)>.svg` — the label
   is the Desktop Icon's label, scrubbed: `"Item Management"` →
   `item_management.svg`. `boot.get_desktop_icon_urls()` **live-scans** these
   directories on every boot build `[V16-LOCAL boot.py:611]` — no `bench build`
   needed, only `bench --site <site> clear-cache` to refresh cached boot info.
2. **Letter fallback** — no file means a colored square with the label's first
   letter. The palette is exactly two colors, `blue` and `gray`
   (`frappe.utils.desktop_pallete`), chosen by the record's `bg_color`; unset means
   blue. This is why a custom app sits as a grey/blue "M" tile in a grid of branded
   ERPNext icons `[KANTISHIVA]`.

To ship real tiles, copy ERPNext's format exactly (54×54, its rounded-square
background path; **solid** = brand-color fill + white glyph, **subtle** = same
background at `fill-opacity="0.1"` + brand-color glyph) into both variant folders.
Deployed for both our apps on 2026-08-18; the URLs appear in
`get_desktop_icon_urls()` and serve `200 image/svg+xml` `[KANTISHIVA]`.

**Do not confuse this with the Workspace `icon` field.** That field picks a
monochrome **sidebar** glyph from frappe's sprite sheets
(`frappe/public/icons/{espresso,lucide}`); check a name exists with
`grep -r 'id="icon-<name>"' frappe/public/icons/`. Setting it never changes the
desk tile — two icon systems, zero overlap.

---

## 5. The sidebar is its own doctype — grouping and per-item icons

Applies to: **v16 only**

**Symptom:** inside the workspace, every left-sidebar row shows the same generic
list glyph, in one flat run with no grouping — even though the workspace body is
neatly sectioned `[KANTISHIVA]`.

**Cause:** the auto-generated **Workspace Sidebar**
(`create_workspace_sidebar_for_workspaces`) is just `Home` plus one bare Link per
workspace *shortcut* — no icons, no structure. Icons and grouping live on the
**Workspace Sidebar Item** rows, which the generator never fills in.

The item model `[V16-LOCAL workspace_sidebar_item.json]`:

| Field | Values / meaning |
|---|---|
| `type` | `Link` / `Section Break` / `Spacer` / `Sidebar Item Group` |
| `link_type` | `DocType` / `Page` / `Report` / `Workspace` / `Dashboard` / `URL` |
| `icon` | sprite name (`wheat`, `handshake`, `truck`, `warehouse`, `home`, …) |
| `indent` + `keep_closed` | on a **Section Break**: renders it as a collapsible group header (`keep_closed: 1` starts collapsed) |
| `child` | on a Link: nests it under the preceding Section Break |

**The pattern to copy is ERPNext's own** (`erpnext/workspace_sidebar/stock.json`):
top-level Links carry icons; a group is a Section Break with an icon and
`indent: 1`, followed by `child: 1` Links — which render as indented labels
*without* icons (that is ERPNext's convention, not a bug). Condensed solution
from the Mandi sidebar:

```json
{
 "doctype": "Workspace Sidebar", "name": "Mandi", "title": "Mandi",
 "app": "trustbit_mandi", "module": "Trustbit Mandi",
 "header_icon": "agriculture", "standard": 1,
 "modified": "2026-08-18 22:30:00.000000",
 "items": [
  {"type": "Link", "label": "Home", "link_type": "Workspace", "link_to": "Mandi",
   "icon": "home", "collapsible": 1},
  {"type": "Section Break", "label": "Mandi", "icon": "wheat",
   "indent": 1, "keep_closed": 0, "link_type": "DocType", "collapsible": 1},
  {"type": "Link", "label": "Grain Purchase", "link_type": "DocType",
   "link_to": "Grain Purchase", "child": 1, "collapsible": 1},
  {"type": "Section Break", "label": "Vehicle", "icon": "truck",
   "indent": 1, "link_type": "DocType", "collapsible": 1},
  {"type": "Link", "label": "Vehicle Dispatch", "link_type": "DocType",
   "link_to": "Vehicle Dispatch", "child": 1, "collapsible": 1}
 ]
}
```

Ship it at `<app>/workspace_sidebar/<scrub(title)>.json` — `bench migrate` syncs
that app-level folder (`frappe/model/sync.py`, `app_level_folders`), which also
protects it from the orphan sweep (§3). Three gotchas:

- The file's `modified` must be **newer than the DB record's**, or the import is
  silently skipped as already-current.
- Validate every icon name against the sprite first
  (`grep -ro 'id="icon-<name>"' frappe/public/icons/`) — an unknown name renders
  as nothing.
- If one specific user still sees the old sidebar: user customization creates a
  per-user copy named `<title>-<user>` that **shadows** the standard record for
  them (`add_sidebar_items`). Check
  `frappe.get_all("Workspace Sidebar", filters={"title": ["like", "%@%"]})` and
  delete the stale copy.

Deployed for both apps on 2026-08-18 (Mandi: 4 iconed groups over 28 items;
Item Management: flat, one icon per item) and verified after a repeat migrate
`[KANTISHIVA]`.

---

## 6. Existing users may see nothing until they "Reset layout"

Applies to: **v16**

The desk grid renders from a per-user layout snapshot plus redis-cached boot info
(`frappe.cache.hdel("desktop_icons", user)` / `hdel("bootinfo", user)` is the
server-side invalidation `[V16-LOCAL desktop_icon.py: clear_desktop_icons_cache]`).
On Kantishiva, the Mandi tile did not appear for the existing user until they ran
**Reset layout** in the desk UI — while a fresh user would have seen it immediately
`[KANTISHIVA]`.

After a reset, verify the site holds no per-user workspace forks (there should be
none unless someone customized):

```sql
SELECT COUNT(*) FROM `tabWorkspace` WHERE IFNULL(for_user,'') != '';   -- 0 on Kantishiva
```

So the deploy runbook for desk-visibility changes ends with:
`bench --site <site> clear-cache` → hard refresh (`Cmd+Shift+R`) → and only if a
tile is still missing for a specific existing user, Reset layout for that user.

---

## 7. Ordered diagnosis — "my app is invisible on v16"

Run these in order; each step gates the next:

```bash
# 1. Is the hook there at all?                                    (§1)
grep -A6 add_to_apps_screen apps/<app>/<app>/hooks.py

# 2. Does the route match THIS major version?                     (§2)
#    v16: /desk/...   v15: /app/...

# 3. Workspace importable on v16? type + app set in the JSON?
#    -> porting-a-large-app-to-v16.md §7

# 4. What does the server actually serve?
bench --site <site> console
>>> from frappe.apps import get_apps; get_apps()          # launcher list
>>> frappe.get_all("Desktop Icon", filters={"app": "<app>"},
...     fields=["label","icon_type","link_type","hidden"]) # tile records

# 5. Records missing? Regenerate (safe to re-run) — BUT this only lasts until
#    the next migrate unless the records are file-backed; see §3's warning:
bench --site <site> execute frappe.utils.install.auto_generate_icons_and_sidebar

# 6. Assets reachable?
curl -so /dev/null -w "%{http_code} %{content_type}\n" https://<site>/assets/<app>/images/<logo>.svg

# 7. bench --site <site> clear-cache, hard refresh, then per-user Reset layout.  (§6)
```

---

## 8. Verification traps that produced false conclusions here

All `[KANTISHIVA]`, all self-inflicted, all worth avoiding:

- **A bare string grep proves nothing about rendering.** `grep '>Mandi<'` on a desk
  page matched the workspace's own `content` JSON blob and produced a confident
  "it renders" while the user was looking at a blank sidebar. Only element-level
  checks count (the actual tile/sidebar markup), or the user's own screenshot.
- **`curl` to `/app/...` on v16 returns 301 with an empty body** — looks exactly
  like a dead site. Curl `/desk/...` (and follow redirects with `-L` when probing).
- **`bench console` fed from a heredoc executes statement-by-statement** (IPython).
  A failing line does not stop the script: later lines `NameError` on its variables
  and the *last* error is the misleading one. Read the whole output, not the tail —
  and `tail -n` truncation on long output has produced flatly wrong conclusions
  here more than once.
- Windows into JSON blobs must be wide: a short regex window over a Workspace's
  huge `content` field lands on the wrong object and reports `"app": null` when the
  real value sits further along (same trap as porting doc §7's note).
