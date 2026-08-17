# Installing a v15-built custom app on v16

Applies to: **v16** (with notes on what v15 did differently)

Real case: `item_creator` v0.1.3, built and tested against Frappe 15.100.0 / ERPNext
15.97.0, installed onto Frappe 16.31.0 / ERPNext 16.32.1 with `india_compliance` 16.8.4
on 2026-08-17. It installed cleanly and then **failed to create a single item** — for a
reason that had nothing to do with v16 itself.

---

## The method that found the bug

**Install onto a throwaway site on the same bench first.** Not a local VM, not a code
review — a real site on the real bench with the *same app set* as production:

```bash
bench new-site itc-test.localhost --db-root-password "$DB_ROOT_PW" \
  --admin-password 'TestOnly#2026' --install-app erpnext --install-app india_compliance
bench --site itc-test.localhost install-app <your_app>
bench --site itc-test.localhost migrate
# …exercise the app's core flow…
bench drop-site itc-test.localhost --db-root-password "$DB_ROOT_PW" --force --no-backup
```

Static analysis said the app was fine on the Item-field dimension — and it was right,
35 APIs verified unchanged. The blocker was a **third-party app's validation hook**, which
no amount of frappe/erpnext diffing would have surfaced.

> **Gotcha in the test harness itself:** a site created with
> `bench new-site --install-app erpnext` has **no ERPNext master data** — zero UOMs, zero
> Item Groups, zero Fiscal Years, and no `Warehouse Type: Transit`. Those come from the
> **setup wizard**, not from installing the app. Symptoms:
> `LinkValidationError: Could not find Warehouse Type: Transit` when creating a Company,
> and `LinkValidationError: Could not find Stock UOM: Nos` when creating an Item (the
> doctype default is `Nos`, which does not exist). Either run the wizard or create the
> masters by hand before testing.

---

## Lesson 1 — india_compliance makes HSN/SAC mandatory, and it is a Link

**Symptom.** Item creation fails at the very last step, after a code has already been
minted:

```
frappe.exceptions.MandatoryError: HSN/SAC Code is required. Please enter a valid HSN/SAC code.
```

**Cause.** `india_compliance` hooks `Item.validate`:

```
india_compliance/gst_india/overrides/item.py:10   validate()  ->  validate_hsn_code(doc)
india_compliance/gst_india/overrides/item.py:27   # only for sales items
gst_india/doctype/gst_hsn_code/gst_hsn_code.py:120 validate_hsn_code(hsn_code)
```

It fires when **all three** are true, which is the default state of an Indian site:

| Condition | Default |
|---|---|
| `GST Settings → validate_hsn_code` | **1 (on)** |
| `Item.is_sales_item` | **1** — and most apps never set it |
| HSN value empty *or* wrong length | — |

`get_hsn_settings()` also enforces **length** against `min_hsn_digits`, so a made-up code
of the wrong length is rejected too.

**The second half of the trap:** `Item.gst_hsn_code` is a **Link to `GST HSN Code`**, not
free text. india_compliance ships ~18,687 of those records. If your app stores HSN in a
`Data` field and copies the string across, any value not already in that master fails link
validation later with a vaguer error.

**Fix — enforce it in your app, portably.** Do not hard-require HSN in the doctype schema:
the same app may install on non-Indian sites. Detect the condition at runtime and fail
early with a message that says what to do:

```python
def _hsn_validation_active():
    """True when this site will REJECT an Item that has no HSN/SAC code."""
    try:
        if "india_compliance" not in frappe.get_installed_apps():
            return False
        from india_compliance.gst_india.utils import get_hsn_settings
        validate_hsn, _lengths = get_hsn_settings()
        return bool(validate_hsn)
    except Exception:
        return False          # fail OPEN — non-Indian sites keep working
```

Then in `validate()`, before anything expensive: throw if the code is missing, and throw a
distinct message if `frappe.db.exists("GST HSN Code", code)` is False.

**Why early matters.** The app minted the item code on *save* and only created the Item
later. Failing at the end meant a consumed serial and an error pointing at ERPNext
internals rather than at the empty field the operator needed to fill.

**Check any site before you trust it:**

```bash
bench --site <site> console
>>> frappe.get_meta("Item").get_field("gst_hsn_code").fieldtype   # 'Link'
>>> frappe.db.get_single_value("GST Settings", "validate_hsn_code")  # 1
>>> frappe.db.count("GST HSN Code")                                  # 18687
```

---

## Lesson 2 — v16 added `Item.validate_variant()`

Applies to: **v16 only**

v16 runs a new `Item.validate_variant()` on **every save of an Item that has `variant_of`
set**, enforcing conditions with no v15 equivalent. An app that creates variants by
reusing or matching Item Attribute rows can trip it — in particular a membership test like
`row.attribute == attr_name` is no longer sufficient.

v16 also changed variant lookup in `erpnext/controllers/item_variant.py` to **translate the
attribute name before matching** (`row.attribute == _(attribute)`), so on a site running a
non-English language, attribute matching can behave differently from v15.

**Status in our case: UNVERIFIED.** The variant path was not exercised on v16 — the
standalone-item path was. If your app creates variants, test that path explicitly on the
throwaway site before going live.

---

## Lesson 3 — what did NOT break (so nobody re-audits it)

Verified by diffing 15.100.0/15.97.0 against 16.31.0/16.32.1:

- **Item fields:** all 20 fieldnames the app writes exist in v16 with identical fieldtype
  and options. The only Item fields v16 *removed* are `customer`, `column_break2` and
  `section_break_avcp`. Mandatory set is unchanged (`item_code`, `item_group`, `stock_uom`),
  and `autoname` is still `field:item_code`.
- **`opening_stock` still exists** on v16 Item; `opening_warehouse` never existed in either
  version. If your app bypasses core opening-stock handling for that reason, keep the bypass.
- **Item Group** is field-for-field identical, so custom fields anchored with
  `insert_after: item_group_name` are safe.
- **Custom Field installer**, `Document` hooks (`before_validate`/`before_save`/`validate`/
  `on_trash`), `make_stock_entry`, and the whitelisted-method mechanism all worked unchanged
  — 7/7 of the app's endpoints returned correctly on v16.
- **The desk page mechanism** is unchanged: `frappe.pages[...].on_page_load` plus
  `frappe.render_template`, served from the Page doc (69,863 bytes of script on v16).

## Post-install checklist for a v15 app on v16

```bash
# 1. every artifact actually landed (install-app can create Module Def but skip doctypes)
bench --site <site> console
>>> frappe.get_all("DocType", filters={"module": "<Your Module>"}, pluck="name")
>>> frappe.db.exists("Page", "<your-page>")
>>> frappe.get_all("Workspace", filters={"module": "<Your Module>"}, pluck="name")

# 2. no phantom roles were auto-created from your JSONs
>>> frappe.get_all("Role", filters={"name": ["in", ["IT Head", "Stores User"]]}, pluck="name")  # []

# 3. build the asset bundle, or the desk include 404s
bench build --app <your_app>

# 4. root-owned files break the frappe user at runtime
chown -R frappe:frappe /home/frappe/frappe-bench

# 5. prove the page renders, do not assume
#    (headless Chrome against a real session — this is how a route-shadowing
#     bug was caught that every server-side check had passed)
```
