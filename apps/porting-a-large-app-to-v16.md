# Porting a large v15 app to v16 — what actually broke

Applies to: **v16** (several lessons apply to v15 equally — marked)

Real case: `trustbit_mandi` — 24 doctypes, 9 reports, ~8,700 lines of py+js, built and run
in production on Frappe 15.100.0 / ERPNext 15.97.0 / india_compliance 15.25.4, installed
onto **16.31.0 / 16.32.1 / 16.8.4** on 2026-08-18.

**Headline: the two worst problems had nothing to do with v16.** They were single-tenant
assumptions and a name collision that were equally wrong on v15 — they only became visible
when the app moved to a second site.

---

## 1. The app was hardcoded to one company

Applies to: **both versions** — a portability bug, not a version bug

```python
company = "Trustbit Mandi"          # in THREE places
```

Sales Invoice creation, the default-warehouse lookup, and the stock-integration patch each
named the original client's company literally. On any other site this failed **two
different ways, neither of them loud**:

```
LinkValidationError: Could not find Company: Trustbit Mandi     # invoice creation
get_default_warehouse()  ->  None                               # silent
```

…and the setup patch did this:

```python
if not frappe.db.exists("Company", company):
    frappe.log_error(title="Patch: Company not found", message=company)
    return          # <- install and migrate both report SUCCESS
```

So `bench install-app` and `bench migrate` printed nothing but success while creating no
warehouses, setting no stock defaults, and leaving the core flows dead. **An install that
looks green and is functionally inert is worse than one that fails.**

**Fix — resolve at runtime, no new settings doctype:**

```python
def get_mandi_company():
    company = frappe.defaults.get_global_default("company")     # 1. Global Defaults
    if company and frappe.db.exists("Company", company):
        return company
    companies = frappe.get_all("Company", pluck="name", limit=2) # 2. the only company
    if len(companies) == 1:
        return companies[0]
    frappe.throw(...)                                            # 3. say what to set
```

ERPNext's **Settings → Global Defaults → Default Company** is already the configuration
point. Don't invent an app-specific settings doctype for this.

**Grep every app you inherit** before deploying it anywhere new:

```bash
grep -rn '"<original client company>"' <app> --include=*.py
grep -rnE '"(Stores|Finished Goods|Main) - [A-Z]{2,6}"' <app>   # hardcoded warehouses
```

---

## 2. A report named "Stock Ledger" silently replaced ERPNext's

Applies to: **both versions** — and this one is destructive

`tabReport.name` is the primary key and report names are **globally unique across all
installed apps**. The app shipped a Script Report named literally `Stock Ledger`. Installing
it **overwrote ERPNext's core Stock Ledger report**:

| | `Report "Stock Ledger"` |
|---|---|
| before install | module=**Stock**, ref_doctype=**Stock Ledger Entry** |
| after install | module=**Trustbit Mandi**, ref_doctype=**Mandi Stock Entry** |

Anyone opening *Stock → Stock Ledger* got the app's pack-level report instead of ERPNext's
stock ledger. No warning at install time.

**Fix:** namespace app reports (`Mandi Stock Ledger`). Rename all four together or the
report breaks: the folder, the files, `name`/`report_name` in the JSON, **the
`frappe.query_reports["..."]` registration key in the JS** (must match the report name
exactly), and every workspace `label`/`link_to`/`shortcut_name`.

Removing the stale record needs raw SQL — Frappe refuses ORM deletion of standard reports
(`ValidationError: You are not allowed to delete Standard Report`):

```sql
DELETE FROM `tabReport` WHERE name='Stock Ledger';
-- then `bench migrate` re-imports ERPNext's own definition
```

**Check before installing any app alongside ERPNext:**

```bash
# report names your app ships vs names ERPNext already owns
ls apps/<your_app>/**/report/
ls apps/erpnext/erpnext/*/report/
```

---

## 3. india_compliance v16 rewrites the taxes table on new Sales Invoices

Applies to: **v16 only**

IC v16 added `before_validate` to **Sales Invoice** (`hooks.py:230`). v15 had that hook only
on Purchase Invoice, Purchase Order, Purchase Receipt and Supplier Quotation — the sales
side was untouched.

The chain, all inside one `si.insert()`:

1. `before_validate_transaction` sees `customer_address` empty and sets
   `doc._party_address_not_set = True`.
2. ERPNext's `validate()` populates the address and runs `calculate_taxes_and_totals()`.
3. IC's `validate` hook then calls the **new** `_update_place_of_supply_and_taxes()`, which
   re-fetches GST details and **replaces `doc.taxes`** — *after* totals were computed, so
   the injected rows carry no `tax_amount`.
4. IC's **new** `validate_item_tax_template()` sees taxable items with no applied tax and
   throws.

**Precondition — this is why a naive test passes:** it only fires when a GST tax template
actually resolves for that customer and company. A throwaway site with no GSTIN and no
templates never triggers it. Our first test "passed" for exactly that reason and was **not
representative** — production had a company GSTIN, four Output GST templates and five Tax
Categories.

**Fix — set the address before insert so the flag is never set:**

```python
from frappe.contacts.doctype.address.address import get_default_address
billing_address = get_default_address("Customer", customer)
if billing_address:
    si.customer_address = billing_address
```

This is the address ERPNext would resolve in `set_missing_values()` anyway, so it is a
no-op on v15 and on sites without india_compliance.

---

## 4. A Select of payment modes is not a Mode of Payment

Applies to: **both versions**

The app's `payment_mode` field was a **Select** with `Cash / NEFT / Cheque`, assigned to
`Payment Entry.mode_of_payment`, which is a **Link to Mode of Payment**. The site shipped
`['Cheque', 'Cash', 'Credit Card', 'Wire Transfer', 'Bank Draft']` — so **NEFT did not
exist** and choosing it fails link validation.

Either create the master, or make the field a Link so the options can't drift from reality.

---

## 5. Swallowed exceptions hide real failures

Applies to: **both versions**

```python
except Exception as e:
    frappe.log_error(title="ERPNext Stock Entry creation failed for {0}".format(self.name), ...)
    frappe.msgprint(_("Warning: Could not create ERPNext Stock Entry..."), indicator="orange")
    # no re-raise
```

The Mandi Stock Entry submitted "successfully" with **no stock movement** and only an Error
Log entry. When a document's whole purpose is to move stock, that failure should be loud.

If you inherit code like this, **read the Error Log after every test** — a green submit
means nothing:

```python
frappe.get_all("Error Log", fields=["method","error"], order_by="creation desc", limit=3)
```

---

## 6. Test harness: a bare site is not a realistic site

Applies to: **both versions** — a harness gap, not a version bug (see the
`set_stock_entry_type` note below).

The single biggest time sink. `bench new-site --install-app erpnext` gives you **none** of
the master data the **setup wizard** creates. We hit these one at a time, four rounds:

| Missing | Symptom |
|---|---|
| `Warehouse Type: Transit` | `LinkValidationError` creating a Company |
| UOM, Item Group | `Could not find Stock UOM: Nos`, `MandatoryError: item_group` |
| Fiscal Year | `Posting Date ... is not in any active Fiscal Year` |
| Price List | `MandatoryError: selling_price_list, price_list_currency` |
| **Stock Entry Type** | `MandatoryError: [Stock Entry]: stock_entry_type` |

That last one is instructive: it looked like a v16 regression, but
`set_stock_entry_type()` is **byte-identical** in v15 and v16 — the bare site simply had
**0** Stock Entry Types where production had **12**.

**Run the setup wizard on your test site**, or seed the masters up front. And distinguish
harness gaps from real findings in the test itself:

```python
HARNESS = ("FiscalYear", "LinkValidation", "Mandatory")
kind = "HARNESS GAP" if any(h in type(e).__name__ for h in HARNESS) else "REAL FAILURE"
```

Without that, a catch-all prints a confident **"BLOCKER CONFIRMED"** for a missing Fiscal
Year — which is exactly what happened to us once, and had to be thrown away.

---

## 7. A v15 workspace is invisible on v16 (required `type` field)

Applies to: **v16 only** — affects *every* v15-authored workspace

**Symptom:** the app installs, `bench migrate` is clean, the Workspace record exists with
all its shortcuts and links — and **nothing appears in the desk sidebar**.

**Cause:** v16 added two fields to Workspace that v15 does not have:

| Field | v16 definition |
|---|---|
| `type` | Select `Workspace / Link / URL`, **reqd=1**, default `"Workspace"` |
| `app` | Data — v16 groups the sidebar by it |

A v15-authored `workspace.json` carries neither. The record imports with both **NULL**
(import bypasses mandatory validation) and the sidebar skips it. It also cannot be re-saved
through the ORM:

```
frappe.exceptions.MandatoryError: [Workspace, <name>]: type
```

Compare a working one — every ERPNext workspace has `type="Workspace"`, `app="erpnext"`.

**Fix — set both in the shipped JSON:**

```json
{
 "doctype": "Workspace",
 "name": "Mandi",
 "module": "Trustbit Mandi",
 "type": "Workspace",
 "app": "trustbit_mandi",
 ...
}
```

`app` is normally derived in `Workspace.validate()`
(`workspace.py:102`, `self.app = get_module_app(self.module)`), but that only runs on
**save** — a fresh JSON import never triggers it, so set it explicitly.

**Repair an already-installed site** (the ORM will refuse until `type` is set):

```python
frappe.db.set_value("Workspace", "<name>", "type", "Workspace", update_modified=False)
frappe.db.commit()
frappe.get_doc("Workspace", "<name>").save(ignore_permissions=True)   # now populates app
```

**Verify what the browser actually receives** rather than trusting the DB:

```bash
curl -s "https://<site>/desk/<workspace>?sid=$SID" | grep -o '"name":"<Name>".\{0,30000\}' \
  | grep -o '"app":[^,]*'
```

> Two traps while checking this. The desk prefix is **`/desk/`** on v16 — `curl` to
> `/app/...` gets a 301 with an **empty body**, which looks like the site is down. And the
> workspace `content` blob is enormous, so a short regex window lands on the wrong object
> and reports `app=null` when the value is actually set further along.

**Related:** v16 also builds the sidebar from a new **`Workspace Sidebar`** doctype
(`frappe/boot.py:get_sidebar_items`), auto-generating one per Module Def that lacks a
record. That part worked correctly for a v15 app — the Module Def and Workspace Sidebar
were both created on install. Only the `type`/`app` fields were missing.

---

## 8. Workspace fixed — and still no icon in the launcher or on the desk

Applies to: **both versions** (the gate), **v16** (the route value and the tiles)

Fixing §7 makes the workspace render **when you navigate to it**. It puts nothing in
the app switcher and nothing on the desk tile grid — those are separate surfaces with
separate mechanisms, and chasing Workspace fields to fix them wastes hours (it did
here). The short version:

1. **`frappe.apps.get_apps()` skips any app without an `add_to_apps_screen` hook** —
   silently, on v15 and v16 alike. The `bench new-app` boilerplate ships the hook
   commented out, so a custom app's default state is: no launcher icon, ever, no error
   anywhere.
2. **The hook's `route` prefix flips between majors**: v15 validates `^/app(/.*)?$`,
   v16 validates `^/desk(/.*)?$`. Worse than the icon: one app with the wrong prefix
   makes `is_desk_apps()` false, and `get_default_path()` then degrades the
   **post-login landing page for every user on the site**.
3. **Desk tiles come from the v16 `Desktop Icon` doctype** (rebuilt — v15's namesake
   is a dead pre-v13 leftover), auto-generated per app install from the hook and from
   public workspaces. Branded tile artwork is per-app SVG files at
   `public/icons/desktop_icons/{solid,subtle}/<scrub(label)>.svg`; without them the
   app sits as a grey letter tile. Existing users may need **Reset layout** to see new
   tiles.
4. **Every `bench migrate` deletes Desktop Icon / Workspace Sidebar records that have
   no backing JSON file in their app** (`remove_orphan_entities()`), so auto-generated
   or hand-created records vanish on the next migrate. Ship them as
   `<app>/desktop_icon/*.json` and `<app>/workspace_sidebar/*.json`, exactly as
   ERPNext does.

Full mechanics, code refs, and an ordered diagnosis checklist:
[../v16/desk-visibility-icons-and-launcher.md](../v16/desk-visibility-icons-and-launcher.md).

---

## What did NOT break

Verified by diffing 15.100.0/15.97.0 against 16.31.0/16.32.1, so nobody re-audits it:

- **Install and migrate**: 24 doctypes (all with tables), 9 reports, workspace, **no
  phantom roles**.
- **Stock movement end to end**: Mandi Stock Entry → ERPNext Stock Entry (Material Receipt,
  submitted) → **Bin `actual_qty` updated**. `make_stock_entry` and `set_stock_entry_type`
  unchanged.
- **Sales Invoice schema**: every field the app writes still exists with the same fieldtype.
- **`get_payment_entry`** still exported with a compatible signature.
- v16's new `check_overdue_billing_threshold()` on Sales Invoice submit is gated on
  `Accounts Settings.enable_overdue_billing_threshold`, which is **0** by default — it
  returns immediately.
