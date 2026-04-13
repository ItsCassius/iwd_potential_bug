# [iwd] boot autoconnect stuck after transient RF failure: 30x blacklist multiplier causes unrecoverable state

---

## Note

I used Claude Code to help investigate this (I am not a kernel or driver developer).
I did audit the output and spent hours testing, retesting, and reading source code.

Contact: `@itscassius:matrix.org`

---

## TL;DR

After a system update (NM 1.56.0 + iwd 3.12), boot autoconnect stopped working on a marginal-signal 5GHz AP.

Root cause (confirmed from source): iwd's **blacklist backoff multiplier (30×)** causes a single transient
boot failure to escalate into a **30-minute or 24-hour blacklist** within 2-3 retry cycles. Once blacklisted,
`evaluate_bss_group_rank` returns `0` for `CONNECT_FAILED` entries, permanently excluding the BSSID from
autoconnect until daemon restart.

This is not a 3.12 regression — it has been present since the group-rank/blacklist system was introduced in ~3.9.
It was exposed in my setup by a separate NetworkManager 1.56.0 bug (psk-flags regression), which previously caused
iwd to abort before even attempting the 802.11 handshake.

---

## Summary

On boot, iwd attempts to autoconnect to a known 5GHz WPA2-PSK AP at marginal signal (~-77 dBm):

1. AP sends **deauth mid-handshake** → `connect-timeout reason: 3 (NETDEV_RESULT_FAILED)`
2. BSSID added to `CONNECT_FAILED` blacklist — initial timeout: **60 seconds**
3. iwd retries after blacklist expires → fails again (signal still marginal)
4. Second failure triggers the **30× multiplier**: timeout → **1800 seconds (30 minutes)**
5. A third failure caps at **86400 seconds (24 hours)**
6. During any active blacklist window, `evaluate_bss_group_rank` (station.c:192) returns `0`
7. "No suitable BSSes found" — no further autoconnect

Manual `iwctl station wlan0 connect` also fails during this state.
Only `systemctl restart iwd` recovers it (clears the in-memory blacklist).

---

## Environment

| Component      | Value                                |
| -------------- | ------------------------------------ |
| Distribution   | Arch Linux                           |
| iwd version    | 3.12-1                               |
| Kernel         | 6.18.21-1-lts                        |
| NetworkManager | 1.56.0 (iwd backend)                 |
| NIC            | Intel 7265 (phy0, 2.4GHz HT40, 5GHz HT40/VHT80) |

---

## Source Code Analysis

### blacklist.c — constants

```c
#define BLACKLIST_DEFAULT_MULTIPLIER    30
#define BLACKLIST_DEFAULT_TIMEOUT       60      /* seconds */
#define BLACKLIST_DEFAULT_MAX_TIMEOUT   86400   /* seconds */
```

Backoff progression for repeated `CONNECT_FAILED` on the same BSSID:

| Failure | Blacklist duration |
|---------|--------------------|
| 1st     | 60s                |
| 2nd     | 60 × 30 = 1800s (30 min) |
| 3rd     | 1800 × 30 = 54000s → capped at 86400s (24h) |

### station.c — evaluate_bss_group_rank()

```c
if (blacklist_contains_bss(addr, BLACKLIST_REASON_CONNECT_FAILED))
    return 0;
```

`CONNECT_FAILED` → rank `0` → excluded from all autoconnect selection.
Unlike `AP_BUSY` (which only sets/clears a ranking bit), `CONNECT_FAILED` is a **hard exclusion**.

### pmksa.c — PMKSA cache is in-memory only

The PMKSA cache initialises empty on every daemon start (`pmksa_init` allocates a fresh heap). The two PMKSA-related commits in 3.12 (`9df8d16f`, `c346f2b8`) only matter for reconnect-within-session scenarios and have **no effect on boot-time connections**.

---

## Steps to Reproduce

**Requirements:** iwd, known WPA2-PSK AP, signal in range ~-72 to -80 dBm at boot location

### 1. Enable debug logging

```bash
sudo tee /etc/systemd/system/iwd.service.d/debug.conf <<'EOF'
[Service]
ExecStart=
ExecStart=/usr/lib/iwd/iwd -d
EOF
sudo systemctl daemon-reload
```

### 2. Reboot — do not intervene, wait ~3 minutes

### 3. Inspect logs

```bash
journalctl -b -u iwd --no-pager | grep -E \
  'connect-timeout|connect-failed|No suitable|blacklist|autoconnect|rank'
```

### 4. Attempt manual connection

```bash
iwctl station wlan0 connect "MyRouter-5G"
```

---

## Expected vs Actual

### Expected

- iwd retries after transient boot failure with reasonable backoff
- Reconnects once signal stabilises or AP becomes reachable

### Actual

- 1st failure: 60s blacklist → autoconnect retries
- 2nd failure: **1800s blacklist** → stuck for 30 minutes minimum
- `iwctl station wlan0 connect` → `Operation failed`
- Only recovery: `systemctl restart iwd`

---

## Observed Behaviour (Debug Logs)

### Initial attempt

```text
iwd[657]: autoconnect: Trying SSID: MyRouter-5G
iwd[657]: 'AA:BB:CC:DD:EE:FF' freq: 5745, rank: 1064, strength: -7200

MLME notification Authenticate(37)
MLME notification Del Station(20)     ← AP deauths mid-handshake
MLME notification Associate(38)
MLME notification Connect(46)

event: connect-timeout, reason: 3
event: connect-failed
```

### Subsequent autoconnect scans

```text
Processing BSS:
  rank: 245
  strength: -7700

autoconnect: No suitable BSSes found
```

(repeats indefinitely until restart)

### After first blacklist expiry — retry fails, escalates

```text
Removing entry AA:BB:CC:DD:EE:FF   ← 60s expired
→ connect-timeout, reason: 2
→ connect-failed
→ blacklist re-added (now 1800s timeout)
```

---

## What Changed (3.11 → 3.12)

The only functional code changes between 3.11 and 3.12 are PMKSA-related
(`9df8d16f station: check return of handshake_state_set_pmksa`,
`c346f2b8 handshake: clear expiration of pmksa in _steal_pmksa()`).
These are irrelevant to boot-time connections (no PMKSA cache exists on fresh start).

The blacklist logic and rank threshold are **unchanged**.

What actually changed in my environment was **NetworkManager 1.56.0**, which introduced a
`psk-flags` regression causing new/migrated profiles to default to `psk-flags=1`
(agent-managed). This caused iwd to abort connections before the 802.11 handshake
(no secrets provided) — meaning the `CONNECT_FAILED` blacklist was never triggered.

After fixing psk-flags (`nmcli connection modify ... wifi-sec.psk-flags 0`), iwd now
correctly attempts the handshake on boot — but at marginal signal, the handshake fails,
triggering the aggressive blacklist escalation described above.

---

## Root Cause (Claude's view)

The **30× backoff multiplier** is too aggressive for the boot autoconnect scenario.

A transient RF failure during boot (AP still initialising, signal marginal at boot location)
should not result in a 30-minute or 24-hour blackout. The current design means that:

- 2 boot-time failures → 30 minutes without connectivity
- 3 boot-time failures → up to 24 hours without connectivity

`AP_BUSY` entries use the same multiplier but only degrade rank (soft penalty).
`CONNECT_FAILED` entries use a **hard exclusion** (`return 0`). Combining a hard exclusion
with a 30× multiplier is disproportionate for transient RF failures.

---

## Workaround

Tune blacklist behaviour in `/etc/iwd/main.conf`:

```ini
[Blacklist]
InitialTimeout = 10
Multiplier = 2
MaximumTimeout = 120
```

This gives a progression of 10s → 20s → 40s → 80s → 120s (capped), allowing recovery
within 2 minutes even after several failures. Setting `InitialTimeout = 0` disables the
`CONNECT_FAILED` blacklist entirely.

---

## Additional Notes

- 2.4GHz band (same AP, same PSK): rank 591 (~-61 dBm), above threshold — also failed to
  connect initially (DHCP failure / stale lease exhaustion from repeated failed reconnects),
  but this appears to be a separate issue
- Confirmed: close to router (~60% signal) → connects immediately; at normal location
  (~-77 dBm, 2 stars) → stuck state
- Kernel/driver unchanged between working (pre-update) and broken state
