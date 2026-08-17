# Custom Frappe App Development

Building, packaging, shipping and reusing custom Frappe apps across client projects.

**Reference implementation:** `item_creator` — a real standalone app extracted out of
`trustbit_ethanol` so several client projects could share it.
Repo `https://github.com/zxrrcpandey/Item-Creator`, local checkout
`~/frappe-bench-v15/apps/item_creator`, installed on `mandi.trustbit.in` (prod) and
`mandi.local` (dev). Every path, filename and error string below was read off that app
or off the framework source in the same bench.

**Verification environment for everything marked "verified":**

| Component | Version | Where |
|---|---|---|
| frappe | `15.100.0` (`apps/frappe/frappe/__init__.py:54`) | `~/frappe-bench-v15` |
| bench | `5.29.0` (`bench --version`) | same |
| node | `v18` (`/opt/homebrew/opt/node@18/bin/node`) | same |

v16 deltas are in the last section and come from the researched v16 runbook, not from a
running v16 desk — they are labelled accordingly.

---

## 1. App skeleton — the layout that actually works

**Applies to: v15 and v16.**

This is the on-disk tree of `item_creator`. `__pycache__/`, `public/dist/` and `.git/`
are elided, and only one of the five doctype folders is shown — the other four
(`ts_item_creator_variant`, `ts_item_code_settings`, `ts_code_counter`, `ts_variant`)
have the identical shape. Everything else is verbatim.

The doubled app name is not a typo: the outer `item_creator/` is the Python package, the
inner `item_creator/` is the **module directory**, and doctypes/pages/workspaces live
under the module, never under the package root.

```
item_creator/                                  # repo root (git)
├── pyproject.toml                             # flit_core backend, name = "item_creator"
├── README.md
├── license.txt
├── .gitignore                                 # must contain dist/ and __pycache__/
└── item_creator/                              # <app> — the importable python package
    ├── __init__.py                            # __version__ = "0.1.3"   ← REQUIRED
    ├── hooks.py
    ├── install.py
    ├── modules.txt                            # single line: "Item Creator"
    ├── patches.txt
    ├── config/__init__.py                     # empty
    ├── templates/__init__.py                  # empty
    ├── public/
    │   └── js/item_creator.bundle.js          # source; esbuild consumes this
    └── item_creator/                          # <module_dir> = scrub("Item Creator")
        ├── __init__.py                        # empty  ← REQUIRED
        ├── doctype/
        │   ├── __init__.py                    # empty  ← REQUIRED
        │   └── ts_item_creator/
        │       ├── __init__.py                # empty  ← REQUIRED
        │       ├── ts_item_creator.json
        │       ├── ts_item_creator.py
        │       ├── ts_item_creator.js
        │       └── ts_item_creator_list.js
        ├── page/
        │   ├── __init__.py                    # empty  ← REQUIRED
        │   └── item_creator/
        │       ├── __init__.py                # empty  ← REQUIRED
        │       ├── item_creator.json
        │       ├── item_creator.js
        │       ├── item_creator.html
        │       └── item_creator.css
        └── workspace/
            ├── __init__.py                    # empty  ← REQUIRED
            └── item_management/
                ├── __init__.py                # empty  ← REQUIRED
                └── item_management.json
```

Verified: every one of those `__init__.py` files exists and is **0 bytes** except the
app-root one, which holds only `__version__ = "0.1.3"`.

```bash
# Prove the __init__.py set is complete before you ship.
cd ~/frappe-bench-v15/apps/item_creator
find item_creator -type d -not -path '*/__pycache__*' -not -path '*/public*' \
  -exec test ! -e '{}/__init__.py' \; -print
# any output = a directory that will fail to import at runtime
```

Verified: produces **no output** on `item_creator` at v0.1.3.

The `-not -path '*/public*'` exclusion is deliberate — `public/`, `public/js/` and
`public/dist/js/` have **no** `__init__.py` and must not have one. They are asset
directories, not Python packages. Drop the exclusion and the check reports four false
positives.

> `bench new-app <app>` is the standard generator for this skeleton. Its exact flag set
> on bench 5.29.0 is **unverified** — `bench new-app --help` did not return within the
> tool timeout in this session. Do not quote flags you have not run.

### 1.1 `modules.txt` must match the doctype JSON `"module"` key EXACTLY

**Applies to: v15 and v16.** This is the single most common "why doesn't my app load"
cause, and the failure is at *runtime*, not at install time.

`modules.txt` in `item_creator` is one line:

```
Item Creator
```

Every shipped JSON carries the identical string. Verified across all five doctypes plus
the page and the workspace:

| File | `"module"` |
|---|---|
| `doctype/ts_item_creator/ts_item_creator.json` | `Item Creator` |
| `doctype/ts_item_creator_variant/ts_item_creator_variant.json` | `Item Creator` |
| `doctype/ts_item_code_settings/ts_item_code_settings.json` | `Item Creator` |
| `doctype/ts_code_counter/ts_code_counter.json` | `Item Creator` |
| `doctype/ts_variant/ts_variant.json` | `Item Creator` |
| `page/item_creator/item_creator.json` | `Item Creator` |
| `workspace/item_management/item_management.json` | `Item Creator` |

**Why it matters.** Frappe builds a reverse map at boot from every app's `modules.txt`
(`frappe/__init__.py:1679-1688`):

```python
	# Init module_app (reverse mapping)
	local.module_app = {}
	for app, modules in local.app_modules.items():
		for module in modules:
			if module in local.module_app:
				warnings.warn(
					f"WARNING: module `{module}` found in apps `{local.module_app[module]}` and `{app}`",
					stacklevel=1,
				)
			local.module_app[module] = app
```

The lookup that consumes it (`frappe/modules/utils.py:298-302`):

```python
def get_module_app(module: str) -> str:
	app = frappe.local.module_app.get(scrub(module))
	if app is None:
		frappe.throw(_("Module {} not found").format(module), exc=frappe.DoesNotExistError)
	return app
```

**Symptom / greppable error string:**

```
frappe.exceptions.DoesNotExistError: Module Item Creator not found
```

If you see that, your JSON `"module"` and your `modules.txt` disagree — usually because
you copied doctype folders from another app and forgot step 3 of the port checklist
(§7). Note the map is keyed on `scrub(module)` (lowercase, spaces → underscores), so
`"Item Creator"` → `item_creator`, which is also why the module directory must be named
`item_creator`.

**Module-name collision across apps** is only a `warnings.warn`, never an error, and the
**last app loaded wins** (`local.module_app[module] = app`). Two installed apps both
shipping a module called e.g. `Stock` silently re-point every path lookup for that module
at whichever app sorted last. Namespace your module names per client.

### 1.2 `pyproject.toml`

Verified content of the license line and its comment in `item_creator/pyproject.toml`:

```toml
[project]
name = "item_creator"
requires-python = ">=3.10"
# Table form (PEP 621) rather than the SPDX string — the string form is only
# understood by newer build backends and would break older bench environments.
license = { file = "license.txt" }
dynamic = ["version"]

[build-system]
requires = ["flit_core >=3.4,<4"]
build-backend = "flit_core.buildapi"
```

`dynamic = ["version"]` is what makes flit read `__version__` out of
`item_creator/__init__.py`. If you delete or blank that variable the app will not build.

### 1.3 Declare hard app dependencies in `hooks.py`

From `item_creator/hooks.py`, verbatim:

```python
# The app creates Items, Item Attributes and Stock Entries and reads Company /
# Item Group / Brand, so ERPNext is a hard dependency. Declaring it here makes
# `install-app` refuse up front on a frappe-only site instead of failing later
# with an unrelated-looking error.
required_apps = ["erpnext"]
```

Cheap, and it converts a confusing mid-migrate traceback into a refusal at
`bench install-app`.

---

## 2. Custom Fields must be installed by CODE, not the UI

**Applies to: v15 and v16.** This is non-negotiable.

If you create a Custom Field through the desk UI and your `hooks.py` has **no `fixtures`
entry**, that field exists in exactly one database. It is not in git, it does not travel
to prod, and no amount of `bench migrate` will produce it on the next site. The
`item_creator` port plan flagged this explicitly:

> **Do not create these in the UI** — no fixtures configured, so they'd live in one
> database only.
> — `PLAN_item_creator_port.md`, step 14

The failure mode is delayed and ugly. From a real incident on a *different* client
(`rdps.trustbit.cloud`, 2026-07-21), when install-time custom fields are missing:

```
pymysql.err.OperationalError: (1054, "Unknown column 'tabContact.is_billing_contact' in 'WHERE'")
```

Weeks after the deploy, on the first form that touches the column.

**And `bench migrate` does NOT fix it.** Migrate syncs schema for fields whose
DocField/Custom Field *record* exists. If the record itself is missing, migrate has
nothing to sync. Diagnostic rule from the RD School lessons: if a user reports
`Unknown column 'tabX.y'`, first check whether the Custom Field record exists (Custom
Field list filtered by `dt`). **Record missing → re-run the app's install-time creator.
Record present → `bench migrate`.**

### The pattern — `install.py` wired to BOTH `after_install` and `after_migrate`

`~/frappe-bench-v15/apps/item_creator/item_creator/install.py`, verbatim and complete
except for three of the five field dicts:

```python
"""Install-time setup for the Item Creator app.

The app builds item codes out of short codes stored on master data (Company,
Item Group, Brand). Those code fields are not part of stock ERPNext, so the app
ships them as Custom Fields and creates them here.

Everything in this module must be safe to run repeatedly: `after_migrate` fires
on every `bench migrate`, and a partially-configured site must be able to heal
itself by re-running it.
"""

from frappe.custom.doctype.custom_field.custom_field import create_custom_fields

CUSTOM_FIELDS = {
	"Company": [
		{
			"fieldname": "company_code",
			"label": "Company Code (ABC)",
			"fieldtype": "Data",
			"insert_after": "company_name",
			"description": "3-letter character code for item coding (e.g., ACM, TBT)",
		},
		# ... company_num_code ...
	],
	"Item Group": [
		# ... category_code, category_num_code ...
	],
	"Brand": [
		{
			"fieldname": "brand_code",
			"label": "Brand Code",
			"fieldtype": "Data",
			"insert_after": "brand",
			"description": "3-letter code for item coding (e.g., CAR, ADM, MCL)",
		},
	],
}


def install_custom_fields():
	"""Create the master-data code fields, or refresh them if they exist.

	`update=True` makes this idempotent: missing fields are inserted, and
	fields that already exist have their label/description/position brought
	back in line with the definitions above instead of raising a duplicate
	error. Safe on a fresh install and on every migrate.
	"""
	create_custom_fields(CUSTOM_FIELDS, update=True)


def after_install():
	install_custom_fields()


def after_migrate():
	install_custom_fields()
```

And the two hook lines from `item_creator/hooks.py`, verbatim:

```python
# ── Install / migrate ───────────────────────────────────────────────────────
# Both entry points call the same idempotent routine, so a site that installed
# an older version picks up newly added fields on the next `bench migrate`.
after_install = "item_creator.install.after_install"
after_migrate = "item_creator.install.after_migrate"
```

**Why both hooks, not just `after_install`.** `after_install` fires exactly once, at
`bench install-app`. A site installed at v0.1.0 that upgrades to v0.1.3 never runs it
again — so any field you add later never lands. `after_migrate` fires on every
`bench migrate`, so the existing fleet heals itself on the next deploy. This is only safe
because `create_custom_fields(..., update=True)` is idempotent: it inserts what is
missing and refreshes label/description/`insert_after` on what already exists, instead of
throwing a duplicate error.

Run it by hand at any time:

```bash
bench --site <site> execute item_creator.install.install_custom_fields
bench --site <site> clear-cache      # dev/demo site only — on production use the targeted form, v15 §11
```

On production, replace the `clear-cache` with the targeted form from `bench console`
([v15/install-and-gotchas.md §11](../v15/install-and-gotchas.md)) — a blanket
`clear-cache` deletes every redis key for the site:

```python
frappe.clear_cache(doctype="Item")     # the doctype whose custom fields you just changed
from frappe.cache_manager import clear_global_cache; clear_global_cache()
```

---

## 3. Fixtures pitfalls (v15)

**Applies to: v15.** Both of these are recorded incidents from `rdschool`, 2026-06-15.

### 3.1 Exactly ONE fixtures entry per doctype

Two `custom_field` entries in `hooks.py` **silently overwrite each other**. We shipped
1 of 15 fields that way and did not notice.

Root cause, verified in `frappe/utils/fixtures.py:81-87`:

```python
			export_json(
				fixture,
				frappe.get_app_path(app, "fixtures", frappe.scrub(fixture) + ".json"),
				filters=filters,
				or_filters=or_filters,
				order_by="idx asc, creation asc",
			)
```

The output filename is derived **only** from `frappe.scrub(fixture)` — the doctype name.
The loop at `fixtures.py:70` iterates every hook entry and writes to that same path, so
with two `custom_field` entries the second `export_json` truncates the first entry's file.
The second entry wins; the rows from the first are gone with no warning.

**Rule:** exactly one fixtures entry per doctype. After any UI field change:

```bash
bench --site <site> export-fixtures --app <app>
git diff --stat <app>/fixtures/          # eyeball for LOST rows, not just added ones
```

`--site` is not optional in practice: the command loops `context.sites` and raises
`SiteNotSpecifiedError` when empty (`frappe/commands/utils.py:370-385`). It only works
bare if `sites/currentsite.txt` is set — and then it silently exports from whichever site
that happens to be, which is how §3.2 happens.

### 3.2 Fixtures bake the exporting site's company into `link_filters`

`bench migrate` clobbers company-specific `link_filter`s with the **dev** company name.
Real case: fixtures exported from a site whose company is `"RD School Betul"`, applied to
a prod site whose company is `"RDPS Betul"` → every filtered Link field renders an
**empty dropdown**, with no error anywhere.

Fixed on that project with an `after_migrate` hook (`relocalize_company_link_filters`)
that rewrites the filters to the local company on every migrate.

**Rule: never hardcode a company name anywhere — code, fixtures, or JS.** Prod and dev
companies differ on every single client. The same rule is why `item_creator` reads its
company codes out of a Custom Field on Company rather than from a constant.

### 3.3 When to prefer `install.py` over fixtures

`item_creator` ships **no fixtures at all** — the five Custom Fields come from
`install.py` (§2). Trade-off:

| | `create_custom_fields` in `install.py` | `fixtures` in `hooks.py` |
|---|---|---|
| Field definitions | Python dict, reviewable in a diff | JSON blob, regenerated by export |
| Idempotent | Yes, with `update=True` | Yes, but overwrite semantics per §3.1 |
| Carries site-specific values | No | **Yes** — company names, link_filters |
| Round-trips through a UI edit | No (code is the source of truth) | Yes, and that is the danger |
| Right for | Field *definitions* an app owns | Master data / config rows you genuinely want to seed |

Default to `install.py` for anything the app itself owns.

---

## 4. Assets: reference a BUNDLE, never a raw `/assets` path

**Applies to: v15 and v16.**

### The bug we shipped and then fixed

`item_creator` v0.1.1 had this in `hooks.py`:

```python
app_include_js = "/assets/item_creator/js/item_list.js"
```

Fixed in commit `4bac8a4` — the commit message is the lesson, verbatim:

> `app_include_js` pointed at the raw path `/assets/item_creator/js/item_list.js`.
> That works (`bundled_asset` passes `/assets` paths through untouched), but the URL
> carries no content hash, so a browser that cached the previous version keeps
> serving it after an upgrade — **the button fix would appear not to have landed.**

**Symptom:** you deploy a JS fix, `bench build` succeeds, you hard-reload and it works on
your machine, and the client swears nothing changed. Because for them nothing did — their
browser is still serving the byte-identical URL out of cache.

### Root cause, verified in framework source

`frappe/utils/jinja_globals.py:128-138`:

```python
def bundled_asset(path, rtl=None):
	from frappe.utils import get_assets_json
	from frappe.website.utils import abs_url

	if ".bundle." in path and not path.startswith("/assets"):
		bundled_assets = get_assets_json()
		if path.endswith(".css") and is_rtl(rtl):
			path = f"rtl_{path}"
		path = bundled_assets.get(path) or path

	return abs_url(path)
```

Two conditions must both hold for the hash lookup to happen: the path must contain
`.bundle.` **and** must not start with `/assets`. A raw `/assets/...` path short-circuits
the `assets.json` lookup entirely and is emitted unchanged, forever.

### The fix

Rename the source file to `<name>.bundle.js`, leave it under `<app>/public/`, and
reference it **by bundle name only**:

```python
# item_creator/hooks.py — current, verified
app_include_js = "item_creator.bundle.js"
```

esbuild discovers bundles by glob (`apps/frappe/esbuild/esbuild.js:192`):

```js
			path.resolve(public_path, "**", "*.bundle.{js,ts,css,sass,scss,less,styl,jsx}")
```

so the file must live somewhere under `<app>/public/` and must literally be named
`*.bundle.js`. Then:

```bash
bench build --app item_creator
```

Verify the mapping landed:

```bash
cd ~/frappe-bench-v15 && python3 -c "
import json
d = json.load(open('sites/assets/assets.json'))
print({k: v for k, v in d.items() if 'item_creator' in k})
"
```

Verified output from this bench:

```
{'item_creator.bundle.js': '/assets/item_creator/dist/js/item_creator.bundle.Z2JOFACL.js'}
```

`Z2JOFACL` is the content hash. Change one byte of the source and it changes, the URL
changes, and every browser refetches. That is the entire point.

**Corollary:** `public/dist/` is build output. It must be in `.gitignore` — the
`item_creator` `.gitignore` contains `dist/`. Never commit hashed bundles; they go stale
against source and confuse the next `bench build`.

---

## 5. Workspace/Page ROUTE COLLISIONS

**Applies to: v15.** (v16 reroutes `/app` → `/desk`; see §10. The router's resolution
order is the mechanism and is unlikely to have changed, but is unverified there.)

### Symptom

Clicking the workspace's own shortcut **appears to do nothing.** No error, no console
warning, no 404. The URL bar changes and the page looks identical, because you are being
re-rendered the same workspace you were already looking at.

### What happened

`item_creator` originally shipped a Workspace named **"Item Creator"** alongside a Page
named **`item-creator`**. From commit `eeb7eab`, verbatim:

> The workspace shipped as "Item Creator", which `frappe.router.slug()`s to
> "item-creator" — the same route as the Page. `router.js` resolves workspaces
> **BEFORE** pages (`if (frappe.workspaces[route[0]])`), so `/app/item-creator` always
> rendered the workspace and the page was unreachable. Clicking the workspace's
> own "Create New Item" shortcut therefore appeared to do nothing: it re-rendered
> the workspace the user was already looking at.
>
> Verified with headless Chrome against a real desk session: before the change the DOM
> rendered `id="page-Workspaces"` with no `itc-` markup; after it renders
> `id="page-item-creator"` with the full form.

### Root cause, verified in framework source

`frappe/public/js/frappe/router.js:180-183` — the workspace branch is the **first** test
in `convert_to_standard_route`, before doctype/page resolution:

```js
		if (frappe.workspaces[route[0]]) {
			// public workspace
			route = ["Workspaces", frappe.workspaces[route[0]].title];
		} else if (route[0] == "private") {
```

The key is the **Workspace record's `name`**, slugged (`frappe/public/js/frappe/desk.js:277-283`):

```js
	setup_workspaces() {
		frappe.modules = {};
		frappe.workspaces = {};
		for (let page of frappe.boot.allowed_workspaces || []) {
			frappe.modules[page.module] = page;
			frappe.workspaces[frappe.router.slug(page.name)] = page;
		}
	}
```

And `slug` is exactly what you'd fear (`router.js:580-582`):

```js
	slug(name) {
		return name.toLowerCase().replace(/ /g, "-");
	},
```

So a Workspace named `Item Creator` occupies the route `item-creator`, and any Page with
`"name": "item-creator"` is permanently shadowed.

### Why nothing warned you

Frappe *has* a guard — and **it is disabled during migrate**
(`frappe/desk/utils.py:9-25`):

```python
def validate_route_conflict(doctype, name):
	"""
	Raises exception if name clashes with routes from other documents for /app routing
	"""

	if frappe.flags.in_migrate:
		return

	all_names = []
	for _doctype in ["Page", "Workspace", "DocType"]:
		all_names.extend(
			[slug(d) for d in frappe.get_all(_doctype, pluck="name") if (doctype != _doctype and d != name)]
		)

	if slug(name) in all_names:
		frappe.msgprint(frappe._("Name already taken, please set a new name"))
		raise frappe.NameError
```

`if frappe.flags.in_migrate: return` is the whole story. Records created by hand in the
desk get checked; records installed by your app during `bench migrate` do **not**. The
guard would have caught this collision instantly if you had typed the workspace name into
the UI — which is exactly why the bug survived to production.

Greppable error string, for when the guard *does* fire (manual creation only):

```
Name already taken, please set a new name
```
followed by `frappe.NameError`.

**Note the guard covers `["Page", "Workspace", "DocType"]`** — a DocType named
`Item Creator` would collide with the same route too.

### The fix, and the upgrade trap

Rename the Workspace so the slugs differ. Current shipped values in
`workspace/item_management/item_management.json`:

```json
 "label": "Item Management",
 "module": "Item Creator",
 "name": "Item Management",
 "title": "Item Management",
```

Page keeps `/app/item-creator`; workspace moves to `/app/item-management`.

**`bench migrate` ADDS the renamed workspace but does NOT delete the old record.** While
the stale `Item Creator` Workspace exists, it keeps shadowing the route and the rename
appears to have done nothing. On every site installed before v0.1.3 you must delete it
explicitly. Done on `mandi.trustbit.in` and `mandi.local`.

```bash
# Confirm which workspaces exist and whether the stale one is still there
bench --site <site> console
>>> frappe.get_all("Workspace", pluck="name")
>>> frappe.delete_doc("Workspace", "Item Creator", force=1)
>>> frappe.db.commit()
```

> **Unverified in this session.** The cleanup was performed on `mandi.trustbit.in` and
> `mandi.local` per `ITEM_CREATOR_CONFIG.md`, but the exact commands used were not
> recorded; the console snippet above is the standard frappe API, not a transcript.
> Deleting the record through the desk UI (Workspace list → delete) is equally valid and
> is what you should prefer if you are not certain. Hard-reload the desk afterwards —
> `frappe.workspaces` is built once from `frappe.boot.allowed_workspaces` at page load.

**Design rule going forward:** never name a Workspace the same as a Page or DocType in
the same app. Pick the noun for one and a different noun for the other — here,
Page = "Item Creator" (the verb-ish thing you do), Workspace = "Item Management" (the
area you browse).

---

## 6. Overriding list view buttons robustly

**Applies to: v15.**

### Two approaches that look right and do not hold

**(a) `setTimeout` hijack of `cur_list`.** From commit `257007d`, verbatim:

> The button was hijacked with a one-shot `setTimeout` against `cur_list`, which
> never held: ListView re-applies its own primary action after render and on
> every bulk-actions toggle (`list_view.js set_primary_action`, called again from
> `toggle_actions_menu_button`), so the override was immediately undone.

Verified in `frappe/public/js/frappe/list/list_view.js`: `set_primary_action()` is defined
at line 285, called at line 159 (initial render) and again at line 593 from inside
`toggle_actions_menu_button()` (line 587), which is itself called at lines 582 and 1631.
Ticking a single checkbox in the list re-applies the stock button and wipes your override.

**(b) `frappe.listview_settings["Item"].primary_action`.** Also unreliable, because
ERPNext assigns that object **wholesale**. Verified — line 1 of
`apps/erpnext/erpnext/stock/doctype/item/item_list.js`:

```js
frappe.listview_settings["Item"] = {
	add_fields: ["item_name", "stock_uom", "item_group", "image", "has_variants", "end_of_life", "disabled"],
	filters: [["disabled", "=", "0"]],
```

That is a plain `=` assignment, not a merge. Any property you set on
`frappe.listview_settings["Item"]` survives **only if your script happens to load after
ERPNext's bundle** — which is load-order luck, not engineering.

> For a doctype **your own app owns**, `frappe.listview_settings` in a
> `<doctype>_list.js` is fine, because nobody else assigns it. `item_creator` uses
> exactly that for its own doctype
> (`doctype/ts_item_creator/ts_item_creator_list.js`, 8 lines):
> ```js
> frappe.listview_settings["TS Item Creator"] = {
> 	refresh(listview) {
> 		// Override the primary action button to open the Item Creator wizard page
> 		listview.page.set_primary_action(__("Item Creator"), function () {
> 			frappe.set_route("item-creator");
> 		}, "add");
> 	},
> };
> ```
> The robust approach below is for overriding a button on a doctype **another app owns**.

### The robust approach: override `ListView.prototype.set_primary_action`

Complete, verified content of
`~/frappe-bench-v15/apps/item_creator/item_creator/public/js/item_creator.bundle.js`:

```js
// Route the Item list's primary "create" button into the Item Creator page.
//
// This is implemented as a ListView.prototype override rather than the more
// obvious frappe.listview_settings["Item"].primary_action, because two things
// fight a simpler approach:
//   1. ListView re-applies its own primary action after render and whenever the
//      bulk-actions menu is toggled, so a one-shot setTimeout hijack is undone.
//   2. ERPNext assigns frappe.listview_settings["Item"] wholesale, so any
//      property we set there survives only if our script happens to load last.
// Overriding the method itself is immune to both: every call routes through us.
frappe.provide("frappe.views");

(function () {
	const TARGET_DOCTYPE = "Item";
	const TARGET_ROUTE = "item-creator";
	let patched = false;

	function patch() {
		if (patched) return true;

		const ListView = frappe.views && frappe.views.ListView;
		if (!ListView || !ListView.prototype || !ListView.prototype.set_primary_action) {
			return false;
		}

		const original = ListView.prototype.set_primary_action;

		ListView.prototype.set_primary_action = function () {
			const is_target = this.doctype === TARGET_DOCTYPE;
			const may_create = this.can_create && !(frappe.boot && frappe.boot.read_only);

			if (is_target && may_create) {
				this.page.set_primary_action(
					__("Create New Item"),
					() => frappe.set_route(TARGET_ROUTE),
					"add"
				);
				return;
			}

			return original.apply(this, arguments);
		};

		patched = true;
		return true;
	}

	// frappe.views.ListView may not exist yet when a global desk include runs,
	// so fall back to retrying on the events that fire once the desk is ready.
	if (!patch()) {
		$(document).on("startup", patch);
		$(document).on("page-change", patch);
	}
})();
```

Five properties that make this hold, each of which you should copy:

1. **`patched` guard.** The retry handlers fire repeatedly; without the flag you would
   wrap your own wrapper on every page change and build a chain.
2. **`return original.apply(this, arguments)` for every other doctype.** You are patching
   a prototype used by the entire desk — an early `return` on the non-target path would
   silently kill primary buttons everywhere.
3. **Mirrors the stock permission gate exactly.** The original at `list_view.js:286` is
   `if (this.can_create && !frappe.boot.read_only)`. Copy the condition; do not invent a
   looser one, or read-only users get a create button.
4. **Deferred patch via `startup` / `page-change`.** `app_include_js` runs before
   `frappe.views.ListView` is necessarily defined. `patch()` returning `false` is the
   normal first-run path, not an error.
5. **Ships as a hashed bundle** (§4), so the fix cannot be defeated by a cached browser —
   which is precisely what happened the first time.

---

## 7. Porting a feature between apps — the checklist

**Applies to: v15 and v16.**

Source of truth: `PLAN_item_creator_port.md` (porting `ts_item_creator` out of
`trustbit_ethanol` / module `ts_gate_entry`). Scope was 5 doctypes + controller + desk
form JS + page ≈ **4,100 lines**.

| # | Item | Why it bites |
|---|---|---|
| 1 | **`"module"` key in EVERY doctype/page/workspace JSON** | Must equal the target `modules.txt` line verbatim or the doctype will not resolve (§1.1). Original plan: `"module": "TS Gate Entry"` → `"module": "Trustbit Mandi"`. |
| 2 | **`autoname` prefixes** | `format:BIC-{####}` → `format:MIC-{####}`. Ported names collide with the source app's series if you leave them. Shipped `item_creator` uses `format:ITC-{####}` with `naming_rule: "Expression (old style)"`. |
| 3 | **Python namespace inside `frappe.call` strings** | These are strings; no linter, no importer, nothing catches them. In this port: **9** occurrences in `item_creator.js` and **2** in `ts_item_creator.js`. Shipped app has **8** occurrences of `item_creator.item_creator.doctype.ts_item_creator.ts_item_creator` in the page JS. |
| 4 | **Cross-app imports** | Exactly two in this case, both in the notification path, both inside try/except. `_get_role_users` was pasted in verbatim (now at `ts_item_creator.py:44`, a 9-line `frappe.db.sql_list`); the WhatsApp notifier was dropped and email/bell kept. |
| 5 | **Roles that do not exist on the target site** | See below — this is the silent one. |
| 6 | Sibling literals a rename would break | Renaming the `TS` prefix means chasing raw SQL against `` `tabTS Code Counter` ``, the literal parent string `TS Item Code Settings`, and an Item Attribute literally named `TS Variant Code`. The port deliberately kept `TS`. |
| 7 | Things that only exist on the source site | The Fixed Asset lane force-set Item Group `Fixed Assets`, which did not exist on the target → insert fails. Neutralise before shipping. |

### 7.1 Verify the namespace rewrite is complete

```bash
cd ~/frappe-bench-v15/apps/item_creator
# 1. No reference to the SOURCE app may survive.
grep -rn "trustbit_ethanol\|ts_gate_entry" --include='*.js' --include='*.py' --include='*.json' item_creator
# expect: no output

# 2. Every frappe.call method string must resolve to a real python dotted path.
grep -rhno 'method: *"[a-z_][a-z0-9_.]*"' --include='*.js' item_creator \
  | sed 's/.*"\(.*\)"/\1/' | sort -u
```

Verified output on `item_creator` v0.1.3 — this is what a *clean* port looks like:

```
create_item
frappe.client.insert
frappe.client.set_value
item_creator.item_creator.doctype.ts_item_creator.ts_item_creator.get_current_fiscal_year
item_creator.item_creator.doctype.ts_item_creator.ts_item_creator.get_my_recent_requests
item_creator.item_creator.doctype.ts_item_creator.ts_item_creator.get_next_asset_serial_preview
item_creator.item_creator.doctype.ts_item_creator.ts_item_creator.get_next_serial_preview
item_creator.item_creator.doctype.ts_item_creator.ts_item_creator.get_variant_code
item_creator.item_creator.doctype.ts_item_creator.ts_item_creator.is_current_user_item_approver
item_creator.item_creator.doctype.ts_item_creator.ts_item_creator.search_existing_items
item_creator.item_creator.doctype.ts_item_creator.ts_item_creator.submit_item_request
run_doc_method
```

Eyeball it. Anything starting with your app name must correspond to an actual
`@frappe.whitelist()` function. `frappe.client.*` and `run_doc_method` are framework.
`create_item` and `run_doc_method` appear because the grep also matches the `method:`
key inside `args: { dt: ..., dn: ..., method: "create_item" }` of a `run_doc_method`
call — that is a controller method name, not a dotted path, so it is expected noise.
Anything still carrying the **source** app's namespace is a bug you are about to ship.

The controller carries **11** `@frappe.whitelist()` endpoints
(`grep -c "@frappe.whitelist" item_creator/item_creator/doctype/ts_item_creator/ts_item_creator.py`
→ `11`). Five of them ship with a bare `@frappe.whitelist()` and **no permission check**,
so any authenticated desk user can enumerate company codes, category codes and serial
counters. Low impact at four users, real nonetheless — audit whitelisted endpoints as
part of any port.

### 7.2 SILENT ROLE POLLUTION — the one nobody sees

**Frappe auto-creates missing Roles from a Page's `roles` array and from a DocType's
permissions, with no warning.**

Verified. `frappe/core/doctype/page/page.py`, in `on_update`:

```python
		from frappe.core.doctype.doctype.doctype import make_module_and_roles

		make_module_and_roles(self, "roles")
```

and `frappe/core/doctype/doctype/doctype.py:1876-1885`:

```python
		roles = [p.role for p in doc.get("permissions") or []] + list(AUTOMATIC_ROLES)

		for role in list(set(roles)):
			if frappe.db.table_exists("Role", cached=False) and not frappe.db.exists("Role", role):
				r = frappe.new_doc("Role")
				r.role_name = role
				r.desk_access = 1
				r.flags.ignore_mandatory = r.flags.ignore_permissions = True
				r.insert()
```

Note `r.desk_access = 1` — the phantom roles are desk roles, and they show up in the
Role Permissions Manager and in every role picker on the site forever.

The port plan measured the damage on the real site:

> Copying untrimmed silently adds **15 empty roles** (IT Head, Stores User/Manager, CEO,
> MD, AVP, CTO, Grain Manager…) that nobody holds.

The source page carried **30** roles; the target site had **15** that actually existed.

**Rule: trim the `roles` array in every Page JSON and the `permissions` block in every
DocType JSON to roles that exist on the target site, before the first migrate.** Once
they are created, removing them from the JSON does not delete them.

The shipped `item_creator` page JSON carries **13** roles, all stock ERPNext:

```json
 "roles": [
  {"role": "System Manager"}, {"role": "Item Manager"}, {"role": "Stock Manager"},
  {"role": "Stock User"}, {"role": "Purchase Manager"}, {"role": "Purchase User"},
  {"role": "Accounts Manager"}, {"role": "Accounts User"}, {"role": "Sales User"},
  {"role": "Manufacturing User"}, {"role": "Maintenance User"},
  {"role": "Projects User"}, {"role": "Auditor"}
 ],
```

and its three non-child doctypes all use the same trio:
`["System Manager", "Item Manager", "Stock Manager"]`.

**Audit before shipping to a new client site:**

```bash
# Roles the app is about to demand
cd ~/frappe-bench-v15/apps/item_creator
python3 - <<'PY'
import json, glob, os
want = set()
for p in glob.glob('item_creator/**/*.json', recursive=True):
    d = json.load(open(p))
    for r in d.get("roles") or []:
        want.add(r["role"] if isinstance(r, dict) else r)
    for p_ in d.get("permissions") or []:
        want.add(p_.get("role"))
print("\n".join(sorted(x for x in want if x)))
PY
```

Then compare against the target site:

```bash
bench --site <site> console
>>> sorted(frappe.get_all("Role", pluck="name"))
```

Anything in the first list and not the second **will be silently created**. Decide
deliberately: trim it, or accept it.

### 7.3 Roles that exist but nobody meaningfully holds

A softer version of the same problem, and a real finding on `mandi.local`: the app's
approval lane is **inert** because all four users (Administrator, ra.pandey008@gmail.com,
saransh@gmail.com, user36@gmail.com) hold `System Manager`, which is an approver role.
Everyone self-approves in one step and nothing ever queues.

> You have installed a switch that only turns on when you add a non-System-Manager user.

Ship the lane if you want, but tell the client it does nothing today.

---

## 8. Reusable standalone app vs copy-paste

**Applies to: v15 and v16.**

`PLAN_item_creator_port.md` originally planned a copy-paste into `trustbit_mandi`, and
was superseded:

> **STATUS 2026-08-16 — SUPERSEDED BY A STANDALONE APP.**
> Rather than copying the feature into `trustbit_mandi`, it was extracted into its own
> reusable Frappe app so several projects can share it.

| | Standalone app | Copy-paste into the client app |
|---|---|---|
| Fix once, deploy everywhere | Yes — `git pull` + `bench migrate` per site | No — N divergent copies |
| Doctype-name collisions | **Real risk** — see below | None (names live in one app) |
| Client can be given/sold the app alone | Yes | No |
| Client-specific tweaks | Need config (Custom Fields / a Settings Single), not code edits | Just edit the code |
| Extra ops surface | One more repo, one more `bench get-app` per server | None |
| Right when | The feature is genuinely generic (an item-coding engine) | The feature encodes one client's business rules |

`item_creator` kept itself client-agnostic by pushing every client-specific value into
**data**, not code: company codes and category codes are Custom Fields on Company and
Item Group, and separator/serial-digits live in the `TS Item Code Settings` Single. Two
sites (`mandi.trustbit.in`, `mandi.local`) run the identical code with different
configuration.

### 8.1 The doctype-name collision risk

DocType `name` is the primary key of `tabDocType` — **it is global across the whole
site, not namespaced by app.** Two installed apps that both ship a doctype called e.g.
`Vehicle Master` cannot coexist cleanly.

`frappe/modules/import_file.py:230` is `if frappe.db.exists(doc.doctype, doc.name):` —
existing records are **updated**, not rejected. So at migrate time the app that syncs
later overwrites the record wholesale, including its `"module"`, its fields and its
permissions. Symptoms are downstream and confusing: fields vanish from a form, a
controller stops being called (because `module` now points at the other app's directory),
permissions reset.

**Rules:**

1. **Prefix every doctype your apps ship.** `item_creator` prefixes everything with `TS`
   (Trustbit Software): `TS Item Creator`, `TS Item Creator Variant`,
   `TS Item Code Settings`, `TS Code Counter`, `TS Variant`. Keep the prefix even when
   porting — the plan explicitly refused to rename it because of the raw-SQL and
   literal-string tail (§7 row 6).
2. **Prefix module names too** (§1.1) — module collisions only produce a `warnings.warn`
   and the last app silently wins.
3. **Check before installing a second app on a live site:**

```bash
# What doctype names does the incoming app claim?
find ~/frappe-bench-v15/apps/item_creator -path '*/doctype/*' -name '*.json' \
  -exec grep -ho '"name": *"[^"]*"' {} \; | sort -u
```

Verified output:

```
"name": "TS Code Counter"
"name": "TS Item Code Settings"
"name": "TS Item Creator Variant"
"name": "TS Item Creator"
"name": "TS Variant"
```

Then check the target site for any of those names:

```bash
bench --site <site> console
>>> frappe.get_all("DocType", filters={"name": ["in", ["TS Item Creator", "TS Variant"]]}, fields=["name", "module"])
```

4. **Never install a whole app just to get one feature.** Option C in the port plan was
   "install `trustbit_ethanol` alongside" and was rejected as a *non-starter*: ~80
   doctypes, gate entry/tankers/weighbridge/WhatsApp/cascade-delete, and ~15 new roles
   onto a 5-user site — half a day of work plus a permanent tax.

---

## 9. Pre-ship verification

**Applies to: v15 and v16.**

Three checks, run from the app repo root. The third is the one everybody skips, and it is
the one that matters most.

```bash
cd ~/frappe-bench-v15/apps/item_creator

# 1. Every .py compiles
python3 -m compileall -q item_creator && echo "PY OK"

# 2. Every .json parses
find item_creator -name '*.json' -print0 \
  | xargs -0 -I{} python3 -c "import json; json.load(open('{}'))" && echo "JSON OK"

# 3. Every .js parses (EXCLUDING build output)
find item_creator -name '*.js' -not -path '*/dist/*' -print0 \
  | xargs -0 -n1 node --check && echo "JS OK"
```

Verified: all three pass on `item_creator` at v0.1.3, on node v18.

**Why check 3 exists.** A JavaScript syntax error is caught by **neither** the Python
compile nor the JSON parse. `bench build` will happily bundle a file that a browser then
refuses to execute, and the failure is entirely client-side: the desk page renders as a
blank white area, `frappe.pages["item-creator"].on_page_load` never registers, and the
only evidence is a message in the browser console that nobody on the client side is
looking at. A page controller of 1,585 lines (`page/item_creator/item_creator.js`) is far
too big to eyeball. `node --check` costs ~50 ms per file.

**`-not -path '*/dist/*'` is mandatory.** Without it you lint esbuild's own minified
output and the sourcemap, which is meaningless noise.

Additional checks worth running before a client deploy:

```bash
# 4. Version bumped? (flit reads this; forget it and the "upgrade" is a no-op)
grep '__version__' item_creator/__init__.py
#   → __version__ = "0.1.3"

# 5. modules.txt agrees with EVERY JSON "module" key (§1.1)
cat item_creator/modules.txt
#   → Item Creator
grep -rho '"module": *"[^"]*"' item_creator | sort -u
#   → "module": "Item Creator"          ← exactly ONE line, and it must match above

# 6. No source-app references survived a port (§7.1)
grep -rn "trustbit_ethanol\|ts_gate_entry" item_creator ; echo "exit=$?"   # want exit=1

# 7. No build output staged
git status --short | grep dist/ ; echo "exit=$?"                          # want exit=1
```

All four verified on `item_creator` v0.1.3, with the outputs shown. Check 5 is the
cheapest possible guard against §1.1: if `sort -u` returns more than one line, or the
line does not appear in `modules.txt`, you have a doctype that will not resolve.

Then on the site:

```bash
bench --site <site> migrate
bench build --app item_creator
bench --site <site> clear-cache   # dev/demo site only — on production use the targeted form, v15 §11
bench restart                     # production only
```

On a live production site drop the `clear-cache` line and clear narrowly from
`bench console` instead — see [v15/install-and-gotchas.md §11](../v15/install-and-gotchas.md);
`bench clear-cache` and `bench --site <site> clear-cache` are the same command and it
deletes every redis key for the site.

> The `migrate` → `build` pair is step 17 of the port checklist, verbatim:
> "`bench --site mandi.local migrate` then `bench build --app trustbit_mandi`".

And confirm the bundle hash actually changed (§4) — if `assets.json` still points at the
old hash, `bench build` did not pick your file up and the client will not see the fix.

---

## 10. v16 deltas that break custom apps

**Applies to: v16 only.** Source: the researched v16 runbook, verified against
`frappe/frappe@version-16` and frappe CI. **Not** yet exercised against a running v16
desk by us — treat as "read the code before you rely on it".

| Change | What it breaks in a custom app |
|---|---|
| **Default sort order changed from `modified` to `creation`** across `frappe.get_all` / `get_list` / `db.get_value` / `db.get_values` | Any query that relied on the implicit "most recently touched first" now returns creation order. Add an explicit `order_by="modified desc"` wherever the old behaviour mattered. |
| **`has_permission` hook must explicitly `return True`** | Previously any non-`False` return granted access; a hook that fell off the end returning `None` used to **allow** and now **denies**. Audit every `has_permission` hook for a missing return. |
| **`hooks.py` document hooks can no longer commit a transaction** | Any `frappe.db.commit()` inside a doc event hook must go. |
| **`/app` rerouted to `/desk`; `/apps` deprecated** | Hardcoded `/app/...` URLs in your JS, print formats, emails and workspace shortcuts. `frappe.set_route("item-creator")` is route-relative and is fine; a literal `"/app/item-creator"` string is not. |
| **Energy Points, Newsletter, Backup Integrations and Blog split into separate apps** | An app importing from those modules raises `ModuleNotFoundError` during migrate/backup. Install them as separate apps or drop the dependency. |

Runbook guidance, verbatim:

> Install frappe + erpnext only and confirm the site is healthy before adding anything
> else. Audit any custom code for `order_by` assumptions (add an explicit
> `order_by='modified desc'` where the old behaviour was relied on) and for
> `has_permission` hooks that fall off the end returning `None`.

Third-party apps that are not v16-ready produce `ModuleNotFoundError` during
migrate/backup — install your custom app **after** the site is confirmed healthy, never
as part of `bench new-site --install-app`.

**Not verified on v16 and worth checking yourself before you port an app:**
`bundled_asset` (§4), `validate_route_conflict`'s `in_migrate` bypass (§5),
`ListView.prototype.set_primary_action` (§6), and `make_module_and_roles` (§7.2). All four
lessons are structural rather than version-specific, but the line numbers above are v15.

---

## Appendix — error strings, greppable

| Error / symptom | Section | Cause |
|---|---|---|
| `frappe.exceptions.DoesNotExistError: Module <X> not found` | §1.1 | JSON `"module"` ≠ `modules.txt` line |
| ``WARNING: module `<x>` found in apps `<a>` and `<b>` `` | §1.1, §8.1 | two apps ship the same module name; last wins |
| `pymysql.err.OperationalError: (1054, "Unknown column 'tabX.y' in 'WHERE'")` | §2 | Custom Field record missing (record missing → re-run installer; record present → `bench migrate`) |
| `Name already taken, please set a new name` + `frappe.NameError` | §5 | Page/Workspace/DocType route slug collision — **only raised outside migrate** |
| Shortcut click does nothing, DOM shows `id="page-Workspaces"` | §5 | Workspace shadows a Page on the same route |
| JS fix deploys but client still sees old behaviour | §4 | `app_include_js` points at a raw `/assets` path — no content hash |
| List-view primary button reverts after ticking a checkbox | §6 | `toggle_actions_menu_button` re-calls `set_primary_action` |
| Desk page renders blank, no server error | §9 | JS syntax error — caught only by `node --check` |
| Filtered Link dropdowns empty on prod, fine on dev | §3.2 | fixtures baked the dev company into `link_filters` |
| Only 1 of N custom fields arrived from fixtures | §3.1 | two fixtures entries for the same doctype; second overwrote the first |
