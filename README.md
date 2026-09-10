# Pi-hole on a Raspberry Pi B+ (ARMv6) — Setup & Troubleshooting Log

A real deployment log for running Pi-hole on old, low-RAM ARMv6 Raspberry Pi boards (Pi B+, Pi Zero/Zero W, and similar) behind a **TP-Link Deco mesh network**. Written up from an actual install so the gotchas are documented alongside the happy path.

## Hardware & Environment

| Item | Value |
|---|---|
| Device | Raspberry Pi Model B+ (single-core ARMv6, BCM2835) |
| OS | Raspbian GNU/Linux 13 (Trixie), `armhf` |
| Kernel | `6.18.x-rpt-rpi-v6` |
| RAM | ~426 MB usable, 425 MB swap |
| Storage | 8 GB microSD (~4.5 GB free after OS) |
| Router | TP-Link Deco mesh system (app-managed only — no browser admin page) |
| Pi-hole versions at time of writing | Core v6.4.3 · Web v6.6 · FTL v6.7 |

> Applies to any similarly low-spec ARMv6 board (Pi B+, Pi Zero, Pi Zero W). FTL ships an ARMv6 build and Pi-hole's official docs list Raspberry Pi OS as supported down to this spec, so none of this is a workaround — it's the intended path, just under-documented for hardware this old.

Replace the example IPs below (`192.168.68.x`) with whatever your own network hands out.

---

## 1. Initial Access

### 1.1 `.local` mDNS discovery didn't work
`ssh nav@pihole.local` and `ping pihole.local` both failed to resolve from macOS, even with Bonjour/mDNS in theory supported on both ends. Never fully chased down why — just worked around it (see §1.2). If you hit this too, don't burn time on it; go straight to IP-based access.

### 1.2 Find the Pi's IP directly
```bash
arp -a
```
Cross-reference the MAC/IP list against your router's client list, or just try SSH-ing to candidates.

### 1.3 SSH in and confirm the environment
```bash
ssh nav@192.168.68.xx
cat /etc/os-release
dpkg --print-architecture      # should print: armhf
free -h
df -h
```
Confirms OS version, architecture, and that you have enough headroom on the SD card before installing anything.

---

## 2. System Update (before touching Pi-hole)

Don't install Pi-hole on a stale OS image — update first:

```bash
sudo apt update
sudo apt full-upgrade -y
sudo reboot
```
Reconnect after ~30–60 seconds with the same `ssh nav@192.168.68.xx`.

---

## 3. Reserve a Static IP

Pi-hole needs a stable IP — if DHCP ever reassigns it, every device pointed at it for DNS breaks.

### 3.1 Get the MAC address
```bash
ip link show eth0
# note the value after "link/ether"
```

### 3.2 TP-Link Deco: Address Reservation
Deco has no browser admin page — it's entirely app-managed:

**Deco app → More → Advanced → Address Reservation → (+) → Select from Client** (easiest, since it just locks in whatever IP the Pi currently has) **or Custom** (manually enter MAC + desired IP).

### 3.3 ⚠️ Gotcha: the reservation may not hold across reinstalls/reboots
In this deployment, the Pi came up on a **different IP** (`.67` instead of the originally reserved `.56`) after a later reboot. The reservation had either not been applied correctly or was tied to a lease that changed. **Don't assume a reservation stuck — always re-check the Pi's actual IP (`hostname -I`) against what's in the Deco app after a reboot**, before pointing anything else at it.

---

## 4. Pre-Install Network Sanity Check

Before running the installer, confirm the Pi can actually reach the internet and resolve names:

```bash
ip route                              # expect: default via <gateway> dev eth0
ping -c 3 1.1.1.1
getent hosts install.pi-hole.net
```

---

## 5. Installing Pi-hole

### 5.1 Use the manual-download method, not curl-piped-to-bash
Piping an installer straight into `bash` means you never see what you're running as root. Download it first, optionally skim it, then run it:

```bash
wget -O basic-install.sh https://install.pi-hole.net
less basic-install.sh          # optional but recommended — it runs as root
sudo bash basic-install.sh
```

### 5.2 Choosing an upstream DNS provider
The installer prompts you to pick one. Good defaults:

- **Cloudflare (1.1.1.1 / 1.0.0.1)** — fastest of the major public resolvers, solid no-logging policy. Good all-around default.
- **Quad9 (9.9.9.9 / 149.112.112.112)** — adds malicious/phishing domain blocking *at the DNS level*, stacking on top of Pi-hole's ad-blocking. Slightly slower, extra layer of protection.
- Avoid your ISP's DNS or Google (8.8.8.8) if privacy is part of the motivation — both log more.

Enable **DNSSEC** if your chosen provider supports it (Cloudflare and Quad9 both do).

### 5.3 What a clean install looks like
The installer will report the architecture it detected, the upstream DNS chosen, and a domain count from gravity, e.g.:
```
[✓] Detected ARMv6 architecture
[i] Using upstream DNS: Cloudflare (DNSSEC) (1.1.1.1, 1.0.0.1)
[i] Number of gravity domains: 79541 (79541 unique domains)
[✓] Installation complete!
[i] Web Interface password: <random string>
```
**Save that password**, or set your own immediately:
```bash
sudo pihole setpassword
```

---

## 6. Verifying Blocking Actually Works

### 6.1 ⚠️ Gotcha: don't test with an apex domain
`nslookup doubleclick.net <pi-ip>` will resolve **normally** — that's expected, not a bug. Blocklists target specific ad-serving *subdomains*, not the bare parent domain (which DoubleClick also uses for non-ad purposes).

### 6.2 Use a domain that's actually on the blocklist
```bash
nslookup ad.doubleclick.net 192.168.68.xx
```
Should return `0.0.0.0` (and `::` for the AAAA record). If it does, blocking works.

### 6.3 ⚠️ Gotcha: `pihole -q` can give false negatives
On this install, `pihole -q ad.doubleclick.net` reported **"Found 0 domains exactly matching"** even though the domain was correctly loaded and actively being blocked. Don't trust it alone — verify against gravity.db directly (ground truth, bypasses whatever bug is in the query tool):

```bash
sudo pihole-FTL sqlite3 /etc/pihole/gravity.db "SELECT COUNT(*) FROM gravity;"
sudo pihole-FTL sqlite3 /etc/pihole/gravity.db "SELECT * FROM gravity WHERE domain='ad.doubleclick.net';"
```

Note: the standalone `sqlite3` binary usually isn't installed — use the one bundled with FTL (`pihole-FTL sqlite3`) as shown above.

Also watch for a **stale cached answer** masking a real fix — `dig` will flag it:
```bash
dig ad.doubleclick.net @192.168.68.xx
# look for: EDE: 3 (Stale Answer)
```
A fresh query (or a `pihole-FTL` restart) clears this.

---

## 7. Deep Troubleshooting Checklist

If blocking genuinely looks broken, work through these roughly in order — cheapest checks first:

```bash
# 1. Is blocking enabled at all?
pihole status

# 2. Cross-check gravity directly (don't trust pihole -q alone — see §6.3)
sudo pihole-FTL sqlite3 /etc/pihole/gravity.db "SELECT COUNT(*) FROM gravity;"

# 3. Is the service actually healthy, or crash-looping?
sudo systemctl status pihole-FTL --no-pager

# 4. Any crashes / OOM kills? (relevant on <512MB boards)
sudo journalctl -u pihole-FTL --no-pager -n 100
dmesg | grep -iE "oom|killed process"

# 5. Check file ownership if you suspect a permissions issue
# (FTL runs as the 'pihole' user, not root — gravity.db should be pihole:pihole)
ls -la /etc/pihole/gravity.db

# 6. Force a clean rebuild + restart
sudo pihole -g
sudo systemctl restart pihole-FTL
```

In this deployment, none of the above turned out to be the actual problem — gravity, ownership, and FTL health were all fine. The "failure" was two red herrings stacked together: a stale cached DNS answer (§6.3) plus a buggy `pihole -q` giving a false negative. A raw `sqlite3` query against gravity.db was what actually confirmed the system was working the whole time. **Lesson: when a diagnostic tool and the underlying data disagree, trust the data.**

---

## 8. Network-Wide Rollout (TP-Link Deco)

Once blocking is confirmed on the Pi itself, point the whole network at it:

1. **Deco app → Internet → Advanced → DHCP Server → Primary DNS** → set to the Pi's (confirmed, locked) IP.
2. **Secondary DNS**: leave blank, or repeat the same IP. Don't fill it with Cloudflare/Google directly — devices will silently fall back to it (unfiltered, no blocking) any time Pi-hole is briefly slow or restarting, quietly defeating the whole point.
3. Save/apply. This only changes what DNS clients receive via DHCP — no downtime.
4. **Existing connected devices won't pick this up until their lease renews** (can be hours). Force it: toggle Wi-Fi off/on on phones/laptops, or reboot the Deco units.
5. Confirm from a client **without specifying a server** (so it's using whatever your OS was handed by DHCP):
   ```bash
   nslookup ad.doubleclick.net
   ```
   If this returns `0.0.0.0` with no `@<ip>` specified, the whole chain (Deco → DHCP → Pi-hole → gravity) is confirmed working end-to-end.

---

## 9. Ongoing Maintenance

- Check the dashboard periodically: `http://<pi-ip>/admin/` — the "Queries Blocked" stat climbing from *other* devices (not just the machine you tested from) is the real proof it's protecting the whole household.
- If something breaks unexpectedly, it's almost always an over-eager blocklist entry — whitelist it from the dashboard's Query Log rather than editing gravity by hand.
- Gravity has a default weekly cron refresh already installed — you generally don't need to run `pihole -g` manually unless troubleshooting.
- On a board this constrained (~426 MB RAM), keep an eye on `free -h` and `systemctl status pihole-FTL` occasionally, especially after Pi-hole version upgrades that might bump baseline memory use.

---

## 10. Quick-Reference: Gotchas Specific to This Hardware/Router Combo

- `pihole.local` mDNS resolution may not work from macOS — use the IP directly.
- A Deco Address Reservation isn't guaranteed to stick across reboots/reinstalls — re-verify the actual IP afterward, don't assume.
- Testing with an apex domain (`doubleclick.net`) gives a misleading "not blocked" result — test with a real listed subdomain instead.
- `pihole -q` can report false negatives — cross-check against `gravity.db` directly via `pihole-FTL sqlite3` before assuming blocking is broken.
- The standalone `sqlite3` CLI often isn't installed — use `pihole-FTL sqlite3` instead.
- A stale cached DNS answer can look identical to "blocking isn't working" — check `dig` for the `EDE: 3 (Stale Answer)` flag before troubleshooting further.
- On Deco (and probably most mesh systems), never put a public DNS provider in the *secondary* DHCP DNS slot — it becomes an unfiltered escape hatch.

---

## Appendix: Full Command Reference

```bash
# --- Access & environment ---
arp -a
ssh nav@192.168.68.xx
cat /etc/os-release
dpkg --print-architecture
free -h
df -h

# --- Update before installing ---
sudo apt update
sudo apt full-upgrade -y
sudo reboot

# --- Static IP prep ---
ip link show eth0
hostname -I

# --- Pre-install network check ---
ip route
ping -c 3 1.1.1.1
getent hosts install.pi-hole.net

# --- Install ---
wget -O basic-install.sh https://install.pi-hole.net
sudo bash basic-install.sh
sudo pihole setpassword

# --- Verify blocking ---
nslookup ad.doubleclick.net 192.168.68.xx
dig ad.doubleclick.net @192.168.68.xx
sudo pihole-FTL sqlite3 /etc/pihole/gravity.db "SELECT COUNT(*) FROM gravity;"
sudo pihole-FTL sqlite3 /etc/pihole/gravity.db "SELECT * FROM gravity WHERE domain='<domain>';"

# --- Troubleshooting ---
pihole status
sudo systemctl status pihole-FTL --no-pager
sudo journalctl -u pihole-FTL --no-pager -n 100
dmesg | grep -iE "oom|killed process"
ls -la /etc/pihole/gravity.db
sudo pihole -g
sudo systemctl restart pihole-FTL
```

---

*Log written from a live deployment on a Raspberry Pi B+ / Raspbian 13 (Trixie) / TP-Link Deco setup, September 2026.*
