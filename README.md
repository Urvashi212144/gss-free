# Get2Short → Direct Link Generator

You paste a `get2short.com/xxxx` link and get the destination link back (e.g. the `mega.nz` folder).
The whole chain shown in your screenshots runs automatically in a **hidden real Chrome window**:

| # | Page (from screenshots) | What the generator does |
|---|---|---|
| 1 | `get2short.com/4mEa` "Secure Link… 3" | waits the 3 s (the server rejects faster requests), then the browser check passes |
| 2/3 | `sayphotobooth.com/safe.php` / blog "Verifying…" | Cloudflare check passes on its own in real Chrome. Turnstile widgets are sent to **your solver** if they don't pass within 4 s |
| 4/5 | "Click On ads wait 10 sec and back…" | **fakes** leaving the page (blur/visibility events, no real ad click), then clicks "Click here to next" |
| 6/7/8 | floating ad + Verify → "Please Wait 18" → Continue | closes the overlay, clicks Verify, runs the timer 10× faster, scrolls, clicks Continue |
| 9/10 | "Chek Any Sponser… wait 20 sec" | fakes the ad visit again (timer 10× faster), clicks the button |
| 11/12 | `link.get2short.com/4mEa?wp=…` + Turnstile | Cloudflare check passes; Turnstile solved via solver, token injected |
| 13 | "Get Link" | the destination is read straight from the `/links/go` API response; the mega.nz page is never even loaded |

**Shortcut:** if any blog page already contains the final `?wp=` link in its HTML or JS, the generator jumps straight there and skips the rest of the blog steps.

## What the real chain looks like (verified live, 2026-10-11)
```
get2short.com/4mEa  -- browser check (3 s, server-enforced, rejects datacenter IPs)
  -> random intermediate: sayphotobooth.com/safe.php?link=...  or  odiadance.com/HQsafe.php?HQlink=...
     -> (Cloudflare check) -> Google search detour   <- skipped: blog opened directly with Google referrer
        -> blog home: Turnstile (solved by your solver) + safelink form
           -> post 1 -> post 2 -> admin-post.php: each a <form> with ak_safelink_timer_token
              (base64 "issued|20|..." = server wants 20 s; the generator waits exactly that and submits
               the form directly, skipping the ads, Verify, "click ads" and countdowns)
              -> link.get2short.com/4mEa?wp=<IP-bound token>  (Cloudflare check + Turnstile)
                 -> /links/go -> destination (mega.nz ...)
```
Typical total time is about 2-2.5 min. Most of that is the 20 s per blog step that the server enforces, which can't be skipped. Doing the chain by hand takes about 7 min.

## Why it runs on your PC and not on a server

I tested this from a cloud server:

* `get2short.com/links/protection-browser-check` answers **`ip_risk_blocked`** for datacenter IPs.
* `sub.odiadance.com` shows "Access Temporarily Restricted" to datacenter IPs.
* The final token is IP-bound and signed:
  `wp = base64("4mEa|1791660326|117.235.171.161|<hash>|<hmac>")`.
  A link generated on one IP can't be used from another, and it can't be forged.
* Cloudflare's "Just a moment" clearance cookies are tied to IP + user agent. Your Railway solver also timed out on `/solve-challenge` for these sites.

So the browser part has to use **your home or mobile IP**. Your solver handles every **Turnstile token**: `/solve` works well (about 4 s per token).

## Running it on a server (or in the Arena sandbox) with your home IP
Datacenter IPs are blocked, so a server copy needs a **residential exit IP**. With a server copy you get:
- a "Connection" panel to paste a proxy (Save + Test exit IP),
- a live view of the hidden browser while a job runs.

1. On your PC (Python installed, no extra packages): double-click **`tools\home_proxy.bat`**.
   It starts a password-protected proxy and a free Pinggy tunnel, then prints
   `http://g2s:xxxx@xxxx.run.pinggy-free.link:PORT`.
2. Paste that into the **Proxy** field → **Save** → **Test exit IP**. It should show your own ISP (e.g. Jio, Rajasthan).
3. Generate links. Every request now goes out through your connection, so the final `wp=` token is bound to the IP that's actually using it.

Free Pinggy tunnels last 60 min. `home_proxy.py` reconnects automatically, but the URL changes, so paste the new one.
A paid residential proxy (`http://user:pass@host:port`) also works, either in the field or in `config.json` → `"proxy"`.
Intermediate domains (sayphotobooth.com, odiadance.com, …) are rotated randomly by the shortener. The engine doesn't hardcode any of them: it learns every host it passes through.

## Setup (Windows)

1. Install **Python 3.10+** (tick "Add to PATH") and have **Google Chrome** installed.
2. Double-click **`setup.bat`** once.
3. Double-click **`run.bat`**. The UI opens at **http://127.0.0.1:8765**.
4. Paste link(s), one per line, and click **Generate direct link**.

Turn **VPN off** first: the shortener blocks VPNs (`vpn_blocked`/`ip_risk_blocked`).

### Other ways to use it
```
python cli.py https://get2short.com/4mEa            # prints the destination URL
python cli.py URL1 URL2 --json                      # batch, JSON output
GET http://127.0.0.1:8765/api/resolve?url=https://get2short.com/4mEa
GET http://127.0.0.1:8765/api/resolve?url=...&force=1   # skip the 24 h cache
```

## config.json
| key | meaning |
|---|---|
| `solver_url` / `solver_api_key` | your turnstile-solver instance (sent as `X-API-Key` if set) |
| `accel_factor` | `0.1` = client-side timers run 10× faster. If a step fails, the job retries once with real-time timers |
| `no_accel_hosts` | hosts whose timers are checked by the server (default `get2short.com`) |
| `show_browser` | `true` shows the Chrome window (the default parks it off-screen) |
| `browser_channel` | `chrome`, `msedge`, or `null` for the bundled Chromium |
| `native_turnstile_grace` | seconds to let a widget auto-pass before calling the solver |
| `cache_hours` | repeat requests for the same short link are answered instantly from `results.json` |
| `debug` | saves HTML + a screenshot of every step into `debug/` |

The solver bans IPs that send more than 10 requests/min, so solver calls are spaced at least 6.5 s apart. A typical run uses 1–3 calls.

## If something breaks
The blog step pages couldn't be fetched from the cloud server (they block datacenter IPs). Those steps are therefore handled by general rules: button-text patterns, ad-visit faking, timer acceleration and Turnstile detection. If the site changes or a link uses different blog pages:

1. Run **`run_visible_debug.bat`**. You can watch the browser, and every step is saved to `debug/<time>/`.
2. When a job fails, a `FAIL` snapshot (`.html` + `.png`) is saved automatically.
3. Send me those files and I can add a rule for that exact page.

## Testing
`tests/mock_site.py` is a local copy of the whole chain. It has the same steps and server-side checks: the 3 s browser check, blur-detected ad gates, the floating overlay, an HMAC/IP-bound `wp` token, Turnstile, and `/links/go`. Run it with `python tests/run_mock_test.py`. It resolves in about 33 s (27 s when the shortcut applies).
