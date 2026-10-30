---
title: "MetaTrader 5 on macOS: Fixing an Empty Broker Search and Missing Servers"
description: "Installing MT5 on an Apple Silicon Mac, why the installer rejects your password, and how resetting servers.dat fixed an empty broker list and a missing XM server (XMGlobal-MT5 10)."
date: 2026-10-30 13:00:00 +0700
tags: [metatrader5, mt5, macos, wine, forex, xm, troubleshooting]
---

*Setup: Apple Silicon MacBook, macOS 26.5.1, MetaTrader 5 build 6275 (MetaQuotes' official Mac build, running on Wine 11.7), broker XM, server `XMGlobal-MT5 10`. Internet provider: Telkomsel, Indonesia.*

## TL;DR

| Symptom | Cause | Fix |
|---|---|---|
| Installer asks for a username + password, says it's wrong, aborts | Your macOS account **isn't an administrator** | Enter the **admin account's** username and password (not your Apple ID/iCloud) |
| Charts don't move while the market is open | Not logged in to a broker account | Log in first, because quotes come from the broker's server |
| Broker search is empty / broker not found / your server (e.g. `XMGlobal-MT5 10`) isn't in the dropdown | MT5's server-list cache (`servers.dat`, `dnsperf.dat`) is broken or stale | Quit MT5, **move those two files away**, reopen MT5 |
| One specific server still missing | MT5 hasn't looked that server up yet | Start MT5 once with a `/config` file containing `Server=...` |

---

## 1. Installing

**Official source:** MetaQuotes, `https://download.mql5.com/cdn/web/metaquotes.software.corp/mt5/MetaTrader5.dmg`. This redirects to `MetaTrader5.pkg.zip` (about 506 MB).

> The "MT5 for Mac" button on xm.com downloads **the same generic MetaQuotes installer**. XM has no Mac-specific build of MT5, so reinstalling "XM's version" changes nothing.

Before installing, verify the package is genuine:

```bash
pkgutil --check-signature "MetaTrader 5.pkg"
spctl -a -vv -t install "MetaTrader 5.pkg"
```

The output should show `Developer ID Installer: MetaQuotes Software Corp. (4LK8F7J843)` and `Notarized Developer ID`.

### Problem: "wrong password" during install

That dialog asks for a **macOS administrator account**, not your Apple ID. If your everyday account is a standard user, your own password is rejected even when it's correct.

To find out which accounts are admins:

```bash
dscl . -read /Groups/admin GroupMembership
```

Then type that admin account's short username and its password in the dialog.

---

## 2. The real problem: broker search doesn't work

### Symptoms
- The recommended broker list in **File → Open an Account** is completely empty.
- Searching "XM", or pasting `XMGlobal-MT5 10` and pressing Enter, returns nothing.
- After restarting MT5, XM appeared, but only with `XMGlobal-MT5` and `XMGlobal-MT5 2`. Server 10 was missing.
- The Server field in the login dialog is a dropdown, so the server name can't be typed in.

### Things we ruled out
- **Mac internet:** fine.
- **macOS firewall:** off. No system proxy.
- **Networking inside Wine** (MT5 on Mac runs inside Wine): `ping` and DNS from inside the Wine prefix both worked.
- **ISP blocking** (Indonesia's "Internet Positif" filter blocks many offshore brokers): turning on Cloudflare WARP didn't change anything.
- **Reaching XM at all:** the MT5 log showed it *did* reach XM's server. Login failed only because the account lives on server 10, not server 1:
  `authorization on XMGlobal-MT5 failed (Invalid account)`

So the network was fine. What was broken was **MT5's cached server list**.

### The fix

**Step A: reset the server cache**

1. Quit MT5 completely (**Cmd+Q**).
2. Move these two files somewhere else as a backup (don't delete them yet):
   ```bash
   C=~/Library/Application\ Support/net.metaquotes.wine.metatrader5/drive_c/Program\ Files/MetaTrader\ 5/config
   mkdir -p ~/mt5-backup && mv "$C/servers.dat" "$C/dnsperf.dat" ~/mt5-backup/
   ```
3. Reopen MT5. It rebuilds both files from MetaQuotes' servers.

**Step B: make MT5 look up a specific server**

1. Create `xm10.ini` inside `.../drive_c/Program Files/MetaTrader 5/` (no password needed):
   ```ini
   [Common]
   Login=12345678
   Server=XMGlobal-MT5 10
   ```
2. Quit MT5, then start it with that file:
   ```bash
   W="/Applications/MetaTrader 5.app/Contents/SharedSupport/wine/bin"
   export WINEPREFIX="$HOME/Library/Application Support/net.metaquotes.wine.metatrader5"
   "$W/wine" "C:\\Program Files\\MetaTrader 5\\terminal64.exe" "/config:C:\\Program Files\\MetaTrader 5\\xm10.ini"
   ```
3. The log should show `successfully initialized from start config`.

After steps A and B, MT5 was reopened normally. The full broker list was back, the XM servers (2, 10, 17, 19, …) all appeared, and logging in to `XMGlobal-MT5 10` worked.

> **Honest note:** steps A and B were done back to back, so we can't say for certain which one fixed it. Our strongest suspicion is **step A**, since a stale or corrupted `servers.dat` explains the empty broker search. Next time, try step A first, and use step B only if one specific server is still missing.

---

## 3. Useful paths and diagnostics

| What | Where |
|---|---|
| App | `/Applications/MetaTrader 5.app` |
| MT5 data (Wine prefix) | `~/Library/Application Support/net.metaquotes.wine.metatrader5/` |
| MT5 folder (= `C:\Program Files\MetaTrader 5`) | `.../drive_c/Program Files/MetaTrader 5/` |
| Daily log | `.../MetaTrader 5/logs/YYYYMMDD.log` (**UTF-16** encoded) |
| Server cache | `.../MetaTrader 5/config/servers.dat`, `dnsperf.dat` |

Read today's log (`iconv` is needed because the log is UTF-16):

```bash
L=~/Library/Application\ Support/net.metaquotes.wine.metatrader5/drive_c/Program\ Files/MetaTrader\ 5/logs
iconv -f UTF-16LE -t UTF-8 "$L/$(date +%Y%m%d).log" | tail -30
```

What the common log lines mean:

| Log line | Meaning |
|---|---|
| `authorization on <server> failed (Invalid account)` | Reached the broker, but the account isn't on that server: wrong server |
| `authorization ... failed (Invalid password)` | Wrong trading-account password |
| No `Network` lines at all | No login attempted yet |
| `terminal stopped due to system shutdown` | MT5 was closed normally (not a crash) |

Test networking from inside Wine:

```bash
W="/Applications/MetaTrader 5.app/Contents/SharedSupport/wine/bin"
export WINEPREFIX="$HOME/Library/Application Support/net.metaquotes.wine.metatrader5"
"$W/wine" ping -n 2 www.mql5.com
```

---

## 4. Other tips
- **Change the MT5 interface language:** **View → Languages → (your language)**, then restart MT5.
- **Need to trade right now while desktop MT5 is broken?** Use your broker's **MT5 WebTrader** in the browser. Note that it doesn't support Expert Advisors or custom indicators.
- **Broker blocked by your ISP?** Turn on **Cloudflare WARP** (or another VPN), then restart MT5.

---

*Written by Panca Walendra Putra and Claude Opus 5.5 (Anthropic), from a real troubleshooting session.*
