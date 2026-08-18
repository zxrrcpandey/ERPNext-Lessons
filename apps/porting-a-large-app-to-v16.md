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
