# [iwd 3.12] connect-timeout on boot blacklists BSSID; rank threshold prevents autoconnect recovery without daemon restart

---

## ⚠️ Potential Bug Report — iwd 3.12-1

> **Note**  
> I used Claude Code to help investigate this (I am not a kernel or driver developer).  
> I *did* audit the output and spent hours testing and retesting.  
> This is not just vibe-coded output — it's a somewhat polished hot mess.  
>  
> Contact: `@itscassius:matrix.org`

---

## TL;DR

After updating from iwd **3.11-2 → 3.12-1** (through a -Syu), my system stopped connecting to a known 5GHz AP.

It appears that:
- A transient failure during boot
- Combined with signal strength
- Leads to **blacklisting + rank filtering**
- Which **prevents any recovery without restarting iwd**

---

## Summary

On boot, `iwd` attempts to connect to a known 5GHz network.

- The AP sends a **deauth mid-handshake** (potential boot-time race condition)
- This results in:
```

connect-timeout reason: 3 (NETDEV_RESULT_FAILED)

```
- The BSSID is then blacklisted (`src/blacklist.c`)

After blacklist expiry:
- Signal stabilizes (~ -77 dBm)
- BSS rank drops below autoconnect threshold (**245**)
- `iwd` filters it permanently:

```

No suitable BSSes found

````

### Result

- No further autoconnect attempts
- Manual `iwctl` fails
- Only recovery:  
  ```bash
  systemctl restart iwd
````

This setup worked **reliably on every boot with iwd 3.11-2**.

---

## Environment

| Component      | Value                               |
| -------------- | ----------------------------------- |
| Distribution   | Arch Linux                          |
| iwd version    | 3.12-1 (regression from 3.11-2)     |
| Kernel         | 6.18.21-1-lts                       |
| NetworkManager | 1.56.0 (iwd backend)                |
| NIC            | phy0 (2.4GHz HT40, 5GHz HT40/VHT80) |
| Adapter        | Intel 7265                          |

---

## Steps to Reproduce

### Requirements

* iwd 3.12
* Known 5GHz WPA2-PSK network
* Signal: **-72 to -77 dBm**
* AP may still be initializing at boot

---

### 1. Ensure PSK exists

Make sure a valid `.psk` file exists for the network.

---

### 2. Enable debug logging

```bash
sudo tee /etc/systemd/system/iwd.service.d/debug.conf <<'EOF'
[Service]
ExecStart=
ExecStart=/usr/lib/iwd/iwd -d
EOF

sudo systemctl daemon-reload
```

---

### 3. Reboot

* Do **not** intervene
* Wait ~3 minutes

---

### 4. Inspect logs

```bash
journalctl -b -u iwd --no-pager | grep -E \
  'connect-timeout|connect-failed|No suitable|blacklist|autoconnect'
```

---

### 5. Attempt manual connection

```bash
iwctl station wlan0 connect "MyRouter-5G"
```

---

## Expected vs Actual

### ✅ Expected

* iwd retries after transient failure
* Successfully connects (as in 3.11-2)

---

### ❌ Actual

* BSSID is blacklisted after failure (`reason: 3`)
* After expiry:

  * Rank = 245 (below threshold)
  * Permanently filtered
* Manual connect:

  ```
  Operation failed
  ```
* Requires:

  ```bash
  systemctl restart iwd
  ```
* Even then, requires stronger signal to reconnect

---

## Observed Behaviour (Debug Logs)

### Initial attempt (stronger signal, -72 dBm)

```text
iwd[657]: autoconnect: Trying SSID: MyRouter-5G
iwd[657]: 'AA:BB:CC:DD:EE:FF' freq: 5745, rank: 1064, strength: -7200
iwd[657]: event: state → connecting (auto)

MLME notification Authenticate(37)
MLME notification Del Station(20)
MLME notification Associate(38)
MLME notification Connect(46)

event: connect-timeout, reason: 3
event: connect-failed
```

> **Key detail:**
> `Del Station(20)` occurs between Authenticate and Associate
> → AP likely sends **deauth mid-handshake**

---

### Subsequent scans (weaker signal, -77 dBm)

```text
Processing BSS:
  rank: 245
  strength: -7700

autoconnect: No suitable BSSes found
```

(repeats indefinitely)

---

### After blacklist expiry

```text
Removing entry AA:BB:CC:DD:EE:FF

Retry:
  rank: 245
  → connect-timeout, reason: 2
  → connect-failed
```

* BSSID is **re-blacklisted**
* Loop resumes

---

### Manual attempt

```bash
iwctl station wlan0 connect "MyRouter-5G"
```

```
Operation failed
```

---

### NetworkManager failure

```text
device (wlan0): config → failed
reason: supplicant-disconnect
Activation failed
```

---

## Recovery Behaviour

```bash
sudo systemctl restart iwd
```

Clears:

1. BSSID blacklist
2. PMKSA cache

### Observations

* Manual connect may still fail
* Autoconnect only works if:

  * Rank > 245
  * (~ stronger than -74 dBm)

---

## What Changed (3.11 → 3.12)

iwd 3.12 changelog includes:

* PMKSA expiration fixes
* PMKID/buffer initialization fix

---

## Suspected Root Cause

The **PMKID change** likely triggers the initial failure.

Observed sequence:

```text
Authenticate(37)   ← OK
Del Station(20)    ← AP deauths client
Associate(38)
Connect(46) aborting
connect-timeout (reason: 3)
```

### Hypothesis

* iwd 3.12 sends a **different PMKID** during association
* Possibly from a **stale PMKSA cache**
* AP rejects it → sends deauth

---

### Important clarification

* `src/blacklist.c` is unchanged between versions
* Rank threshold is also unchanged

➡️ These are **not regressions**

The regression is:

> The initial AP deauth (reason: 3), which triggers the existing mechanisms and creates a non-recoverable state

---

## Additional Notes

* 2.4GHz network:

  * Rank: 591 (-61 dBm)
  * Above threshold
  * Still failed to connect

* Issue is **reproducible on boot**

* AP load:

  ```
  3–5 / 255 (not congested)
  ```

* Kernel/driver unlikely:

  * Worked on iwd 3.11-2
  * Same kernel + hardware tested
  * Downgrading kernel did not help
