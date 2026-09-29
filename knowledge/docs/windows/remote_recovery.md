# Remote recovery — self-healing network watchdog

Context: the desktop lost its ethernet link while nobody was home (2026-09). The OS was alive, only the NIC was dead, and a manual reboot fixed it. Every service on the tailnet (SMB, Jellyfin, the WSL node that carries T3 Code sessions) went down with it. This watchdog makes the machine heal itself: restart the adapter first, reboot if that isn't enough.

## Prerequisites — kill the usual causes first

Elevated PowerShell:

```powershell
# NIC power saving off (most common cause of "ethernet silently died")
Get-NetAdapter -Physical | Disable-NetAdapterPowerManagement

# Fast Startup off — a "shutdown" is otherwise a hibernate, and drivers come back in a stale state
Set-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\Power' HiberbootEnabled 0
```

Find what actually happened last time (look at the day of the outage):

```powershell
Get-WinEvent -LogName System -MaxEvents 3000 |
  ? { $_.ProviderName -match 'NDIS|Tcpip|Dhcp|Kernel-Power|e1d|e2f|rt640|Netwtw' } |
  ft TimeCreated, ProviderName, Id, Message -Wrap
```

## The watchdog

`C:\Tools\net-watchdog.ps1`. Runs every 5 min as SYSTEM. Logic: internet reachable → clear state and exit. First failure → restart every physical adapter. Still down 20 min later → reboot.

```powershell
$state = "$env:ProgramData\net-watchdog.down"
$up = (Test-Connection 1.1.1.1 -Count 2 -Quiet) -or (Test-Connection 8.8.8.8 -Count 2 -Quiet)
if ($up) { Remove-Item $state -ErrorAction SilentlyContinue; exit }
if (-not (Test-Path $state)) { Get-Date -Format o | Set-Content $state; Get-NetAdapter -Physical | Restart-NetAdapter; exit }
if ((Get-Date) - [datetime](Get-Content $state) -gt [TimeSpan]::FromMinutes(20)) { Restart-Computer -Force }
```

Register it (elevated cmd or PowerShell):

```
schtasks /Create /TN net-watchdog /SC MINUTE /MO 5 /RU SYSTEM /RL HIGHEST /TR "powershell -NoProfile -ExecutionPolicy Bypass -File C:\Tools\net-watchdog.ps1"
```

Test without waiting: `schtasks /Run /TN net-watchdog`, then check that `%ProgramData%\net-watchdog.down` does **not** exist while the network is up.

### Gotchas

* **NordVPN / Threat Protection.** Check `Test-Connection 1.1.1.1 -Quiet` returns `True` in the machine's normal running state before registering the task. If the kill switch or a DNS filter makes ICMP fail while everything else works, the watchdog will reboot the box every 25 minutes.
* **Restart-NetAdapter is disruptive.** Every physical adapter goes down for a couple of seconds on the first failure, including Wi-Fi if present. Acceptable here: at that point the network is already dead.
* **Two ICMP targets, `-or`.** One dead public resolver must not trigger a reboot.
* **State file, not a counter in memory.** Each run is a fresh process; the file's timestamp is the only memory between runs. Delete it by hand to reset.

## Make the reboot land somewhere useful

A reboot is only a recovery if the machine comes back serving. Checklist, one time:

* **BIOS:** `Restore on AC Power Loss = Power On`. Needed for the smart-plug power cycle below; harmless otherwise.
* **No BitLocker pre-boot PIN**, or the machine waits at a prompt forever.
* **Auto-logon** (Sysinternals Autologon). Windows services (Tailscale, SMB, Jellyfin as a service) do not need it, but WSL does.
* **WSL autostart** — logon-triggered task, so the `nixos-wsl` tailnet node and T3 Code sessions come back without a human:

  ```
  schtasks /Create /TN wsl-keepalive /SC ONLOGON /TR "wsl -d NixOS -- sleep infinity"
  ```

## Next layer — smart-plug watchdog (Shelly Plug S Gen3)

The PowerShell watchdog cannot help if the OS freezes. A Wi-Fi smart plug with local scripting probes the PC over HTTP and power-cycles it after 15 min unreachable; with `Restore on AC Power Loss` the PC boots straight back. Wi-Fi matters: it survives an ethernet-only failure on the PC. The Shelly cloud app doubles as a manual power-cycle button from anywhere.

The script runs **on the plug itself**, in its flash. Nothing is installed on Windows; it only depends on the plug having Wi-Fi and power.

### Setup

1. **Health URL.** Reuse Jellyfin: `http://<PC-LAN-IP>:8096/health` returns `Healthy`. Give the PC a DHCP reservation in the router first. Test the URL from a phone on Wi-Fi; if it fails, allow TCP 8096 on the Private profile in Windows Firewall.
2. **Plug web UI** (`http://<plug-ip>`, IP from the Shelly app → Device information, or the router's client list) → Settings → Input/Output settings → "Initial state after power outage": **On**.
3. **Scripts → Add script**, name `pc-watchdog`, paste the code below → **Save → Run → enable "Run on startup"**. If the Scripts menu is missing, update the firmware first (Settings → Firmware).
4. **Test** without waiting 15 min: set `failsToCycle: 1`, `interval: 30`, stop Jellyfin's service, watch the console print then cut power. Restore the values after.

```js
// pc-watchdog.js — Shelly Plug S Gen3 (mJS)
let CONFIG = {
  url: "http://192.168.1.50:8096/health", // PC LAN IP; any always-on HTTP port on the PC works
  interval: 300,      // seconds between checks
  failsToCycle: 3,    // 3 x 300 s = 15 min unreachable before a cycle
  offSeconds: 10,
  bootGrace: 600,     // leave the PC alone this long after a cycle
  maxCycles: 3        // consecutive cycles before giving up (stop/start the script to reset)
};
let fails = 0, cycles = 0, timer = null;

function start() { timer = Timer.set(CONFIG.interval * 1000, true, check); }

function cycle() {
  cycles++;
  print("PC unreachable, power cycle #" + cycles);
  Timer.clear(timer);
  Shelly.call("Switch.Set", { id: 0, on: false }, function () {
    Timer.set(CONFIG.offSeconds * 1000, false, function () {
      Shelly.call("Switch.Set", { id: 0, on: true });
      fails = 0;
      Timer.set(CONFIG.bootGrace * 1000, false, start);
    });
  });
}

function check() {
  Shelly.call("HTTP.GET", { url: CONFIG.url, timeout: 10 }, function (res, err) {
    if (err === 0 && res && res.code === 200) { fails = 0; cycles = 0; return; }
    fails++;
    print("check failed " + fails + "/" + CONFIG.failsToCycle);
    if (fails < CONFIG.failsToCycle) return;
    if (cycles >= CONFIG.maxCycles) { print("max cycles reached, giving up"); Timer.clear(timer); return; }
    cycle();
  });
}
start();
```

### Gotchas

* **Shelly has no ICMP ping**, so the probe is HTTP. Using Jellyfin means a dead Jellyfin service alone also triggers a power cycle. Accept it, or point `url` at any other HTTP port that starts at boot.
* **Hard power cut on a running PC.** That is the point, but it is why `failsToCycle` is 3 and not 1. Windows Update reboots finish well under 15 min.
* **`maxCycles` is the loop breaker.** If the PC cannot boot, three cycles then silence. Reset by stopping and starting the script in the app.
* **Manual override from anywhere:** Shelly app → toggle off, wait 10 s, toggle on. Works even while the script is stopped.
