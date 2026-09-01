# Publishing Frappe/ERPNext through a Cloudflare Tunnel

Running a bench behind **`cloudflared`** instead of a public IP: no port forwarding, no
static IP, no Let's Encrypt on the origin, and nothing inbound on the router.

**When this is the right answer**

- The uplink has a **dynamic IP** (every ISP reassignment would otherwise drop clients for
  the DNS TTL)
- The uplink is behind **CGNAT** — inbound is impossible, full stop
- You do not want the origin IP discoverable (it leaks via certificate transparency,
  historical DNS and outbound mail headers)
- An office/on-prem box that must not have holes punched to it

**When it is not** — if you already have a datacentre VPS with a static IP and a clean
certbot setup, this adds a dependency for little gain. See
[ssl-nginx-and-production.md](ssl-nginx-and-production.md) for that path.

First proven on the four-site bare-metal build:
[builds/2026-09-01-…](../builds/2026-09-01-bare-metal-four-site-cloudflare-tunnel.md).
Versions below are `cloudflared` **2026.8.3**, bench **5.31.0**, Frappe **15.119.1**.

---

## 0. The shape of it

```
client browser ──HTTPS──> Cloudflare edge ──┐
                                            │  outbound-only QUIC tunnel
                                            │  (the SERVER dials OUT)
   ┌────────────────────────────────────────┘
   ▼
cloudflared ──> nginx :80 (localhost) ──> gunicorn :8000
                                     └──> socket.io :9000
                                          MariaDB / Redis on 127.0.0.1
```

The server **dials out**. Nothing connects inward. That single property is why the dynamic
IP stops mattering and why `ufw` can deny all inbound while the sites stay up.

**TLS terminates at Cloudflare.** You do not run certbot on the origin, and
`bench setup lets-encrypt` is not just unnecessary but actively harmful here — it needs
inbound :80 you do not have, and it **stops nginx for the duration**, taking every site on
the bench down.

---

## 1. Naming — decide before you create any site

Cloudflare's free **Universal SSL covers the apex and exactly one label**:

```
DNS:example.com, DNS:*.example.com
```

| Hostname | Covered by free Universal SSL? |
|---|---|
| `demo1.example.com` | yes |
| `example.com` | yes |
| `demo1.erp.example.com` | **no** — certificate error in every browser |

Two labels deep needs **Advanced Certificate Manager** (~$10/mo/zone), and ACM does *not*
make wildcards recursive — you name each level explicitly, up to 50 hosts.

Because there is **no `bench rename-site` in v15**, and with `dns_multitenant on` the site
directory name *is* the nginx `server_name`, this choice is effectively permanent. Pick
one-label-deep hostnames and name each site its real FQDN from creation.

---

## 2. Install and authenticate

```bash
sudo mkdir -p --mode=0755 /usr/share/keyrings
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg \
  | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null
echo 'deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared noble main' \
  | sudo tee /etc/apt/sources.list.d/cloudflared.list
sudo apt-get update && sudo apt-get install -y cloudflared

cloudflared tunnel login        # prints a URL; approve the zone in a browser
cloudflared tunnel create <tunnel-name>
cloudflared tunnel list         # note the UUID
```

`tunnel login` writes `~/.cloudflared/cert.pem`. It is small (a few hundred bytes) — that is
normal, it is a token, not a full certificate chain.

**The zone must already be on Cloudflare** (nameservers delegated and active) before
`tunnel login` will offer it. Delegation takes hours; start it first.

---

## 3. Config — the two traps

```yaml
# ~/.cloudflared/config.yml
tunnel: <TUNNEL-UUID>
credentials-file: /home/frappe/.cloudflared/<TUNNEL-UUID>.json

originRequest:
  connectTimeout: 30s
  tlsTimeout: 10s
  tcpKeepAlive: 30s
  keepAliveTimeout: 90s

ingress:
  - hostname: demo1.example.com
    service: http://localhost:80
  - hostname: demo2.example.com
    service: http://localhost:80
  - service: http_status:404      # mandatory catch-all
```

### Trap 1 — never set `originRequest.httpHostHeader`

It is one of the most copy-pasted options from self-hosting blogs and it is **catastrophic
for a multi-tenant bench**. Frappe resolves which site you are on **purely from the `Host`
header**; overriding it collapses every tenant onto whichever site you named. Leave it unset.

### Trap 2 — prefer explicit hostnames over a wildcard

`- hostname: "*.example.com"` works, but every typo and every subdomain-bruteforce scanner
then becomes a live request traversing your tunnel and consuming a gunicorn worker. Explicit
entries let Cloudflare answer NXDOMAIN at the edge for free.

(Wildcard *ingress* with explicit *DNS* is a reasonable middle ground — adding client five
then needs no `cloudflared` restart, and restarts drop every open socket.io session.)

Validate before going live:

```bash
cloudflared tunnel ingress validate
cloudflared tunnel ingress rule https://demo1.example.com/app
```

---

## 4. DNS and service install

```bash
for n in 1 2 3 4; do
  cloudflared tunnel route dns <tunnel-name> "demo${n}.example.com"
done
```

This creates **proxied CNAMEs to `<UUID>.cfargotunnel.com`**. Do not create A records; your
origin IP should never appear in DNS.

> **The orange cloud is mandatory.** A grey-cloud "DNS only" record cannot work —
> `cfargotunnel.com` resolves only inside Cloudflare's network.

```bash
# under sudo, $HOME becomes /root — pass --config explicitly or it finds nothing
sudo cloudflared --config /home/frappe/.cloudflared/config.yml service install
sudo systemctl enable --now cloudflared
sudo journalctl -u cloudflared -n 20 --no-pager
```

Healthy output shows several **QUIC** connections to nearby PoPs:

```
INF Registered tunnel connection connIndex=0 ... location=nag01 protocol=quic
INF Registered tunnel connection connIndex=1 ... location=bom09 protocol=quic
```

---

## 5. The four origin fixes — apply before anyone logs in

Every one of these is a real defect behind a proxy. Skipping them produces bugs that look
like ERPNext problems and are not.

### 5.1 Real client IP — or one user's typo locks out everyone

`cloudflared` connects over **loopback**, so nginx sees every request as `127.0.0.1`.
Frappe's login-attempt tracker then collapses into a single shared bucket: **the default 10
failed logins by any one person lock out every user on every site.** Per-user `restrict_ip`
breaks, rate limits go global, and access logs are worthless.

```nginx
# /etc/nginx/conf.d/00-frappe-prereq.conf
# also define log_format main here — see §7
set_real_ip_from 127.0.0.1;
set_real_ip_from ::1;
real_ip_header   CF-Connecting-IP;
```

A **separate `conf.d` file sorting before `frappe-bench.conf`** — never an edit to bench's
generated config, which is regenerated by every `bench setup nginx`.

> If you instead expose the origin directly and proxy through Cloudflare DNS (no tunnel),
> trust Cloudflare's published ranges rather than loopback, and refresh them on a timer.

### 5.2 `host_name` — or password-reset emails go out as `http://`

Frappe derives `https://` **solely** from `X-Forwarded-Proto`, and bench's nginx template
overwrites that header with `$scheme` — which is `http`, because nginx listens on plain :80
for the tunnel. Background jobs have no request at all and hard-code `http://`.

**Set it per site:**

```bash
bench --site demo1.example.com set-config host_name https://demo1.example.com
```

```bash
bench --site demo1.example.com console
>>> frappe.utils.get_url()
'https://demo1.example.com'
```

### 5.3 Upload ceilings — there are three, stacked

| Layer | Limit | Where |
|---|---|---|
| Cloudflare Tunnel | **100 MB** | hard ceiling; a plan upgrade does not lift it for tunnel traffic |
| nginx | 50 MB | **hard-coded** in bench's template (`config/templates/nginx.conf`) |
| Frappe | 25 MB fallback | System Settings → Max File Size |

The effective limit is the smallest. A Cloudflare 413 leaves **nothing in your nginx logs**,
which makes it miserable to diagnose.

```bash
BENCH_PKG=$(python3 -c 'import bench,os;print(os.path.dirname(bench.__file__))')
sudo cp "$BENCH_PKG/config/templates/nginx.conf" "$BENCH_PKG/config/templates/nginx.conf.bak"
sudo sed -i 's/client_max_body_size 50m;/client_max_body_size 100m;/' \
  "$BENCH_PKG/config/templates/nginx.conf"
bench setup nginx --yes && sudo nginx -t && sudo systemctl reload nginx
```

**This is a patch to bench's own template — it reverts on every `bench update`.** Put it on
a re-apply checklist. (And note `bench setup nginx` will then trip §7 unless the
`log_format` file exists.)

### 5.4 `http_timeout` below Cloudflare's proxy timeout

Cloudflare's default **Proxy Read Timeout is 125 s**, after which the user gets an opaque
**error 524** that is invisible at your origin. bench's default `http_timeout` is 120 s —
too close.

```bash
bench config http_timeout 90
bench setup supervisor --yes && bench setup nginx --yes
sudo supervisorctl reread && sudo supervisorctl update && sudo supervisorctl restart all
```

Be accurate about what users then see: gunicorn **SIGKILLs** the worker at 90 s, so nginx
returns a bare **502** and Frappe logs nothing. The gain is that a 502 is attributable to
your stack and appears in your nginx log, where a 524 appears nowhere. The real fix for slow
reports is Prepared Reports and background jobs.

---

## 6. Cloudflare dashboard settings

| Setting | Value | Why |
|---|---|---|
| SSL/TLS mode | **Full** (or Full-strict) | **Never Flexible** — Flexible + `host_name=https://` gives `ERR_TOO_MANY_REDIRECTS` |
| Always Use HTTPS | On | |
| **Rocket Loader** | **Off** | Reliably breaks the Frappe Desk — login box disappears, form submits error |
| Email Obfuscation | Off | Corrupts email fields in rendered pages |
| **WebSockets** | **On** | socket.io depends on it |
| Argo Smart Routing | Off | Incompatible with WebSockets |
| Minimum TLS | 1.2 | |

### Cache rules — order matters, first match wins

**Rule 1 — bypass, and it must be first.** Cloudflare caches by **file extension**, not by
your application's permissions. Attachments end in `.pdf`, `.png`, `.xlsx` — all on the
default-cached list — and Frappe's `X-Accel-Redirect` responses carry no cache-suppressing
headers. **Without this, one tenant's invoice can be cached at the edge and served to
another.**

```
(starts_with(http.request.uri.path, "/api/"))
or (starts_with(http.request.uri.path, "/files/"))
or (starts_with(http.request.uri.path, "/private/"))
or (starts_with(http.request.uri.path, "/app"))
or (starts_with(http.request.uri.path, "/socket.io"))
or (starts_with(http.request.uri.path, "/method/"))
or (starts_with(http.request.uri.path, "/backups"))
→ Cache eligibility: Bypass cache
```

**Rule 2 — cache `/assets/`, but never cache an error.**

```
starts_with(http.request.uri.path, "/assets/")
→ Cache eligibility: Eligible for cache
→ Edge TTL: respect origin, with status-code scoping:
     200–299 → 2678400   (31 days)
     300–599 → 0         (never cache an error)
```

> **Do not use a blanket "Edge TTL: override origin".** It applies to error responses too. A
> transient origin fault — a permissions bug, a restart — gets cached as a 404 for the full
> TTL. On the reference build a five-minute permission error became a **month-long** cached
> 404 that looked exactly like the original bug.
> [builds/2026-09-01-… §12](../builds/2026-09-01-bare-metal-four-site-cloudflare-tunnel.md)

Frappe serves hashed asset filenames with `cache-control: max-age=31536000`, so
`respect_origin` is correct anyway.

Verify a private file is **not** cached:

```bash
curl -sD- -o /dev/null https://demo1.example.com/files/some.pdf | grep -i cf-cache-status
# want BYPASS or DYNAMIC — never HIT
```

---

## 7. `log_format main` — you will hit this

Any standalone `bench setup nginx` (which §5.3 requires) renders
`access_log … main;`, and Debian/Ubuntu define no such format:

```
[emerg] unknown log format "main" in /etc/nginx/conf.d/frappe-bench.conf:100
```

Put it in the **same prerequisite file** as the real-IP directives:

```nginx
# /etc/nginx/conf.d/00-frappe-prereq.conf
log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                '$status $body_bytes_sent "$http_referer" '
                '"$http_user_agent" "$http_x_forwarded_for"';

set_real_ip_from 127.0.0.1;
set_real_ip_from ::1;
real_ip_header   CF-Connecting-IP;
```

Full analysis: [ssl-nginx-and-production.md](ssl-nginx-and-production.md).

---

## 8. Firewall — nothing inbound is needed

Because `cloudflared` dials out and reaches nginx over loopback, **80 and 443 stay closed**:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow in on tailscale0
sudo ufw allow from 192.168.0.0/24 to any port 22 proto tcp
sudo ufw enable
```

Sites keep working with zero inbound rules — that is the whole point, and it is worth
verifying explicitly right after enabling, as proof the topology is what you think it is.

**Set up your remote access (Tailscale/WireGuard) and prove it works *before* enabling the
firewall**, especially on a box with no serial console.

---

## 9. Verification

```bash
# 1. DNS points at Cloudflare, not you
dig +short demo1.example.com          # 104.x / 172.67.x — never your origin IP

# 2. certificate covers the name
echo | openssl s_client -connect demo1.example.com:443 -servername demo1.example.com 2>/dev/null \
  | openssl x509 -noout -subject -ext subjectAltName
#   expect: DNS:example.com, DNS:*.example.com

# 3. app responds
curl -s https://demo1.example.com/api/method/ping     # {"message":"pong"}

# 4. realtime survives the tunnel
curl -s -o /dev/null -w '%{http_code}\n' \
  "https://demo1.example.com/socket.io/?EIO=4&transport=polling"    # 200

# 5. EVERY asset on the login page — catches §5/§11-class faults
while IFS= read -r a; do
  printf '%s %s\n' "$(curl -s -o /dev/null -w '%{http_code}' "https://demo1.example.com$a")" "$a"
done < <(curl -s https://demo1.example.com/login | grep -oE '/assets/[^"'"'"' ]+\.(css|js)' | sort -u) \
  | grep -v '^200' || echo "all assets 200"
```

Check 5 matters: pages can return 200 while every stylesheet 404s, which looks like a
proxy fault and usually is not.

---

## 10. Origin vs edge — the diagnostic to reach for first

| origin | edge | meaning |
|---|---|---|
| 404 | 404 | real application or permission fault, nothing to do with Cloudflare |
| 200 | 404 `cf-cache-status: HIT` | **cached error** — purge, then fix the TTL rule |
| 200 | 404 `MISS` | cache rule or ingress expression problem |
| 200 | 5xx | origin timeout vs Cloudflare's 125 s — see §5.4 |

```bash
ssh server "curl -s -o /dev/null -w '%{http_code}\n' -H 'Host: demo1.example.com' http://127.0.0.1<path>"
curl -sD- -o /dev/null "https://demo1.example.com<path>" | grep -iE '^(HTTP|cf-cache-status)'
```

---

## 11. Known limits

- **100 MB upload ceiling** on tunnel traffic. Not liftable by plan. Remedies are chunked
  upload or a VPS reverse proxy where you set `client_max_body_size` yourself.
- **Restarting `cloudflared` drops every long-lived connection**, including all socket.io
  sessions. Do config changes outside business hours.
- **CIRA / "fast call for help"** — if the box also runs Intel AMT, note AMT has its own
  outbound path that no inbound-port argument protects against. See
  [bare-metal-and-firmware.md](bare-metal-and-firmware.md).
- **IPv6.** A globally-routable IPv6 address on the origin NIC can expose services even with
  zero IPv4 forwarding. Worth checking on dual-stack ISPs.
- Cloudflare is now a **dependency**. If the tunnel is down, the sites are unreachable even
  though the server is fine. Monitor the tunnel, not just the box.
