# The worker-timeout death spiral (and why users call it "being banned")

How a single slow whitelisted endpoint turns into a self-sustaining, site-wide outage
that recurs every morning; how to diagnose it from the logs in minutes instead of days;
and the five-part fix. Written from a four-day production incident on a live retail POS.

**Applies to:** v15 (mechanics verified on frappe 15.118.0); the caching and request-side
behaviour is unchanged in v16's source as of this writing.

**Evidence confidence:**

| Tag | Meaning |
|---|---|
| **[V]** | Verified: read verbatim from frappe/gunicorn source on disk, or executed on the live box |
| **[B]** | From the real incident record (KGS / splashbox.in, 2026-08-24 → 27: ERPNext 15.119.3, POS Awesome fork, 1 vCPU / 3.9 GB, ~4–5k req/h) |

Live result this document describes: 584 gateway timeouts over four consecutive mornings
(105 / 46 / 277 / 156), all three gunicorn workers pinned at 99% CPU for ~2 h/day, fixed
in one deploy — the same call went from *killed at 120 s, every time* to **66.5 s once,
then 0.04 s from cache** [B].

---

## 0. The shape of the failure

Five independent facts, each harmless alone, compose into a perpetual-motion outage:

1. **An endpoint slower than the worker timeout.** gunicorn runs `-t 120` in bench's
   default supervisor config; a request that takes longer is SIGABRT-killed mid-query.
   You'll see this in `logs/web.error.log` **[V]**:

   ```
   [CRITICAL] WORKER TIMEOUT (pid:245623)
   ...
   SystemExit: 1
   ```

   and in nginx's error log (its `proxy_read_timeout` is also 120 in bench's template):

   ```
   upstream timed out (110: Unknown error) while reading response header from upstream
   ```

2. **`@redis_cache` writes only on success.** `frappe/utils/caching.py` calls the
   function first and `frappe.cache.set_value(...)` *after* it returns **[V]**. A worker
   killed at 120 s therefore leaves the cache key unwritten — in our incident the key had
   **never existed** in production. "Slow but cached after the first hit" is a design that
   only works if at least one call ever completes.

3. **A cache TTL shorter than the client's refresh cadence.** The endpoint was cached for
   60 s while the client re-requested every ~2 min — so even a *successful* build could
   never be reused **[B]**. A cache that always expires before its next reader is pure cost.

4. **A client retry loop with no error handler.** The Vue background loader's
   `frappe.call` had a `callback` but no `error`; on failure its "loaded" flag stayed
   false and every keystroke re-armed another doomed 120 s call **[B]**. Three terminals
   ⇒ three permanently-occupied workers ⇒ with `-w 3`, the whole site queues.

5. **A framework upgrade as the trigger.** The N+1 code (one `frappe.get_all` per item,
   inside a loop over a 20k-item catalog) had run for years. frappe ≥ 15.101 runs
   `validate_generated_query` — a **sqlparse parse of every generated SQL statement** —
   which multiplied the pure-Python cost of those 20k query builds and pushed the same
   old code past the 120 s cliff on a 1-core box **[V]** (the kill traceback lands inside
   `sqlparse/engine/grouping.py`). Nothing in the app changed; the floor moved.

**What users say while this is happening:** *"we are banned", "still blocked",
"loading and loading"*. Not "an endpoint is slow" — because for them everything is slow:
login pages, list views, the public website, all queued behind the pinned workers.

---

## 1. Diagnose it in minutes: the "banned" triage ladder

A "we're banned" report has at least four different real causes. Walk the ladder in
order and demand evidence at each rung — never fix a layer you haven't proven guilty [B]:

```bash
# Rung 1 — is anything actually banning at the firewall?
fail2ban-client status sshd
iptables -S | grep -Ei "f2b|DROP.*-s |REJECT.*-s "
ipset list -n
cat /proc/net/xt_recent/DEFAULT          # ufw 'limit' rule's block list

# Rung 2 — does the client's traffic even ARRIVE?
grep -cF "<client_ip>" /var/log/nginx/access.log
# ZERO hits = their packets never reach you: dead DNS/bookmark/ISP, not a server block.
# UFW logs what it drops — silence in ufw.log/kern.log means "not arriving", not "blocked".

# Rung 3 — application layer
#   Activity Log for the IP: "Incorrect Verification code" spam is humans mistyping
#   OTPs, NOT a lockout. Check the real lockout state:
redis-cli -p 13000 --scan --pattern '*login_failed*'

# Rung 4 — capacity (this incident lived here)
awk '$9>=500 {split($4,a,":"); print a[2]":00", $9}' /var/log/nginx/access.log | sort | uniq -c
grep -c "WORKER TIMEOUT" logs/web.error.log
uptime; mysql -e "SHOW FULL PROCESSLIST"
```

Two rung-1/2 traps worth naming, both from the same week [B]:

- **A decommissioned server's redirect dies with it.** Our old box was parked with a
  nginx 301 to the new domain — then died ahead of schedule, taking the redirect down.
  Stale bookmarks now fail outright, which staff report as being banned. Copy anything
  irreplaceable off a doomed box **the day you park it**, not "before deletion" — our
  parked box also held the only copy of a set of rollback dumps, now gone.
- **You can ban yourself.** `ufw limit 22/tcp` blocks an IP after ~6 connections/30 s,
  and a whitelist goes stale the day your ISP rotates your IP. Release with
  `echo / > /proc/net/xt_recent/DEFAULT`; prevent with SSH `ControlMaster auto` +
  `ControlPersist` and batching many checks into one invocation.

---

## 2. Find the exact line that's burning the CPU

The gunicorn kill traceback is a free profiler: SIGABRT lands wherever the worker was,
so the frame above `handle_abort` **is the hot spot** [V]. Ours read:

```
File ".../posawesome/api/posapp.py", line 266, in _get_items
    item_barcode = frappe.get_all(
...
File ".../gunicorn/workers/base.py", line 204, in handle_abort
    sys.exit(1)
```

— one `frappe.get_all("Item Barcode", filters={"parent": item_code})` **per item**,
inside `for item in items_data:` over the whole catalog. Grep your custom apps for the
pattern; it is everywhere in the third-party app ecosystem:

```bash
grep -rn "frappe.get_all\|frappe.db.get_value" apps/<custom_app> | grep -v test
# then eyeball which hits sit inside a loop over query results
```

MariaDB's slow log rounds out the picture (`long_query_time=2`). Ours also surfaced a
**missing index on a hot custom field**: a filter on `posa_pos_opening_shift` scanned all
52,989 Sales Invoices (27.6 s) on every POS submit, and a concurrent
`SELECT ... FOR UPDATE` then waited 19.6 s — the POS "hanging on save" was lock waits,
not the save itself [B].

---

## 3. The fix, in five parts

### 3.1 Batch the N+1 (the actual cure)

One bulk query + a dict, built once before the loop:

```python
item_barcodes_map = {}
for b in frappe.get_all(
    "Item Barcode",
    filters={"parent": ["in", items]},
    fields=["parent", "barcode", "posa_uom"],
):
    item_barcodes_map.setdefault(b.parent, []).append(
        {"barcode": b.barcode, "posa_uom": b.posa_uom}
    )
# in the loop:  item_barcode = item_barcodes_map.get(item_code, [])
```

Same treatment for per-item stock and serials. ~20,000 queries became 4; the call went
from >120 s (killed) to 66.5 s cold on a loaded 1-core box, 0.04 s cached [B].

**Stock-source note:** we replaced a per-item "latest Stock Ledger Entry
`qty_after_transaction`" lookup with one bulk read of `Bin.actual_qty`. That is not a
compromise: `Bin.get_actual_qty` is *defined* as the latest non-cancelled SLE's
`qty_after_transaction`, Bin re-derives on backdated entries, and ERPNext's own POS
reads Bin **[V]**. If your code has both patterns, converge on Bin.

### 3.2 Cache TTL: longer than the client cadence, capped absolutely

```python
ttl = pos_profile.get("posa_server_cache_duration")
ttl = min(int(ttl) * 60, 300) if ttl else 300
@redis_cache(ttl=ttl)
```

Two constraints pull against each other: the TTL must **exceed the client's refresh
interval** (or the cache never serves anyone) and must stay **short** when the payload
embeds prices/stock and nothing invalidates on `Item`/`Item Price`/`Bin` change — a
30-minute TTL here means selling at a stale price for 30 minutes. 5 minutes satisfied
both. Also know: `redis_cache`'s per-call key hashes **all args** — if the client sends
a whole profile JSON, every byte-level variation is a separate multi-MB Redis entry.

### 3.3 Client retries: five mandatory properties

An in-flight guard, an attempt cap, exponential backoff, an error handler, and a
**user-visible give-up**. And one frappe-specific trap that makes naive handlers dead
code **[V]**: `frappe.call` invokes `error()` **with no argument** for 500/502/504
(only the generic fall-through passes the xhr), so:

```js
error: function (err) {
    if (err && err.name === "AbortError") return;   // guard BEFORE err.name
    vm.retry_count += 1;
    if (vm.retry_count <= 3)
        setTimeout(retry, 1000 * Math.pow(2, vm.retry_count - 1));
    else
        frappe.show_alert({ message: __("…failed — search still works"), indicator: "orange" }, 8);
}
```

Reset the counter on success *and* on any natural restart point (our profile
re-registration), or three transient failures disable the feature for the life of the
tab. Related traps we verified while here: jQuery's `$.ajax` **ignores AbortController
signals** (passing `signal:` to `frappe.call` cancels nothing), and `frappe.call`'s
`freeze: true` on a background poll paints a blocking overlay on every tick.

### 3.4 Index the custom fields you filter on

Both halves, or the next migrate disagrees with the database:

```bash
bench --site <site> execute frappe.db.add_index --args '["Sales Invoice", ["posa_pos_opening_shift"]]'
# AND set "search_index": 1 on the same Custom Field in the app's fixture JSON
```

EXPLAIN before/after: `ALL, rows=52989` → `ref, rows=1` [B]. On MariaDB 10.6 the ADD
INDEX on 53k rows took seconds; still an off-hours operation on a busy table.

### 3.5 Deduplicate background job storms

If a hot path enqueues work per event, the same document can get concurrent jobs that
serialize on `FOR UPDATE`:

```python
enqueue(method=..., job_id="submit_{0}".format(doc.name), deduplicate=True, ...)
```

Caveat **[V]**: with `deduplicate=True`, frappe silently returns `None` when a job with
that id is QUEUED/STARTED — a job stuck in STARTED (worker OOM-killed) blocks re-enqueue
until RQ's registry cleanup. Acceptable when something else sweeps stragglers (our shift
close does); know the trade before copying.

### What we deliberately did NOT do

Suppress 504 dialogs via the global `ajaxError`/`report_error` monkey-patch route.
Verified pointless and harmful **[V]**: frappe's `statusCode[504]` handler fires its own
`msgprint` *before* either hook, and anything loaded via `app_include_js` runs on
**every desk page** — plus a 504 usually means the request is *still executing*
server-side, so hiding it invites double-submits.

---

## 4. Prove the fix the same day

1. Time the exact call that was dying (bench console, real profile, the flag
   combination the client uses). Must complete well under the worker timeout.
2. Confirm the cache key now **exists**: `redis-cli -p 13000 --scan --pattern '*<func>*'`
   — its absence was the spiral's signature, its presence is the proof.
3. Call it again: cache hit should be milliseconds.
4. Next trading morning, tally nginx: `awk '$9==504' access.log | wc -l` in the window
   that used to burn. Target zero.

## 5. The transferable morals

1. **After any frappe upgrade, re-profile your hottest endpoints.** The framework's
   per-query overhead only ever grows; N+1 code that "worked for years" is a time bomb
   with a moving fuse.
2. **A cache that requires a successful call to fill cannot rescue an endpoint that
   never succeeds.** Check worst-case runtime against the worker timeout, not typical.
3. **Every client retry loop is a DoS tool pointed at yourself** until it has a cap,
   backoff, and an error handler that actually executes (mind frappe's zero-arg
   `error()`).
4. **"Banned" is a symptom with ≥4 distinct causes.** Triage bottom-up with evidence:
   firewall → does traffic arrive → app auth → capacity. The fix belongs on the rung
   you proved, not the rung that's easiest to toggle.
5. **Parked servers are already dead.** Treat "we'll copy it off before deletion" as
   "we lost it" — the box chooses its own deletion date.
