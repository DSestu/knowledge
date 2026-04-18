# NixOS alongside Windows

This page is for the case where Windows is, and will remain, your daily driver — and you still want a portable, reproducible Linux configuration that you can develop and test from that machine. Nothing here asks you to replace Windows. The goal is to give Windows a working Nix/NixOS story from Level 1 (package manager) all the way to Level 3 (real hardware, non-destructively), mirroring the Kali approach in [kali-to-nixos.md](kali-to-nixos.md).

If you're new to the levels, read [the four safe levels of commitment](getting-started.md#the-four-safe-levels-of-commitment) first; the rest of this page assumes them.

## In this page

- [Your situation](#your-situation)
- [The Windows-side map of the four levels](#the-windows-side-map-of-the-four-levels)
- [Level 1 — Nix package manager on WSL2](#level-1-nix-package-manager-on-wsl2)
- [Level 2a — home-manager on WSL2](#level-2a-home-manager-on-wsl2)
- [Level 2b — NixOS-WSL](#level-2b-nixos-wsl)
- [Level 2b alt — NixOS in a VirtualBox VM](#level-2b-alt-nixos-in-a-virtualbox-vm)
- [Level 2b alt — libvirt/KVM via WSL2 or Hyper-V](#level-2b-alt-libvirtkvm-via-wsl2-or-hyper-v)
- [Level 3 — NixOS on an external SSD](#level-3-nixos-on-an-external-ssd)
- [Level 3 alt — dual-boot on the internal drive](#level-3-alt-dual-boot-on-the-internal-drive)
- [Sharing one `home.nix` across Windows-side Linuxes](#sharing-one-homenix-across-windows-side-linuxes)
- [Gaming on the NixOS side (when you choose to test it)](#gaming-on-the-nixos-side-when-you-choose-to-test-it)
- [Mental model — Windows vs NixOS, at a glance](#mental-model-windows-vs-nixos-at-a-glance)

---

## Your situation

The assumed setup:

- **Windows** on the desktop, permanent. Games, Adobe, MS Office, anti-cheat-laden titles — stays here.
- You want a **portable Linux config** (same dotfiles, same CLI toolbox, same `programs.fish`/`programs.git` story) that works on a Windows desktop, a Kali laptop, or any future machine.
- You already use **VirtualBox** on Windows and are comfortable spinning up VMs.
- You are **not** trying to migrate off Windows. Replacing Windows with NixOS on this machine is explicitly off the table.

Everything below is framed around this context. The rest of this section maps the levels to Windows; individual sections give the recipes.

---

## The Windows-side map of the four levels

| Level | Venue                                  | What it gets you                                                                       |
| ----- | -------------------------------------- | -------------------------------------------------------------------------------------- |
| L1    | Nix pkg manager inside WSL2 (Debian/Ubuntu) | `nix run`, `nix shell`, ad-hoc reproducible environments; no commitment.           |
| L2a   | home-manager standalone inside WSL2    | Declarative `$HOME` — your real shell, aliases, git config, CLI tools.                 |
| L2b   | NixOS-WSL                              | A real NixOS with `configuration.nix`, wrapped as a WSL distro. No reboot.            |
| L2b   | NixOS in a VirtualBox VM               | A real NixOS sandbox on the Windows desktop. GUI + network + rollback snapshots.      |
| L2b   | NixOS in libvirt/Hyper-V               | Alternatives to VirtualBox when you outgrow it or want nested-virt.                    |
| L3    | NixOS on an external SSD               | Real hardware, native speed, **no touching the internal Windows disk**.                |
| L3    | NixOS dual-boot on internal drive      | Last step. Requires shrinking Windows; reversible but riskier than external SSD.       |

The important idea, copied from the Kali page: **L2a is the primary prize**. A portable `home.nix` that works on WSL2 Debian/Ubuntu today and on NixOS tomorrow is the thing you actually take with you. Everything else is staging.

---

## Level 1 — Nix package manager on WSL2

Goal: get `nix run` / `nix shell` / `nix develop` working on your Windows box with zero impact on Windows itself. WSL2 gives you a real Linux kernel and filesystem; Nix installs cleanly inside it.

### 1. Install WSL2 (once)

From PowerShell as administrator:

```powershell
wsl --install
# or, to pick a specific distro:
wsl --install -d Debian
wsl --install -d Ubuntu
```

Reboot if prompted. `wsl -l -v` confirms the distro and version 2.

> If `wsl` is not recognised, you are on Windows 10 pre-2004 or the feature is disabled. Run `wsl --install` from a fresh PowerShell on current Windows 11, or enable "Virtual Machine Platform" + "Windows Subsystem for Linux" from Optional Features.

### 2. Install Nix inside the WSL2 distro

Two families, same choice as on Kali.

**Official installer (classic, single-user):**

```bash
sh <(curl -L https://nixos.org/nix/install) --no-daemon
. ~/.nix-profile/etc/profile.d/nix.sh
```

**Determinate Nix installer (multi-user, flakes on by default):**

```bash
curl --proto '=https' --tlsv1.2 -sSf -L https://install.determinate.systems/nix | sh -s -- install
```

Either works. The Determinate installer is less fiddly for beginners and enables flakes for you; the official classic installer gives you a minimal setup you can extend yourself.

### 3. Verify

```bash
nix --version
nix run nixpkgs#hello
nix shell nixpkgs#cowsay nixpkgs#lolcat -c sh -c 'echo hi | cowsay | lolcat'
```

### Windows-specific quirks

- **systemd in WSL2.** Modern WSL2 supports `systemd` via `/etc/wsl.conf`:
  ```ini
  [boot]
  systemd=true
  ```
  After editing, run `wsl --shutdown` in PowerShell, then reopen the distro. With systemd on, the Determinate (multi-user) Nix installer is straightforward; without systemd, use the `--no-daemon` classic installer instead.
- **File performance.** Keep Nix work on the WSL2 filesystem (`/home/you/...`), not `/mnt/c/...`. Crossing the 9P border is catastrophically slow for the many-small-files access pattern Nix uses.
- **Networking.** WSL2 shares the host's internet; no proxy dance needed in typical cases.

### What you can stop at here

If all you wanted is a reproducible way to spawn one-shot tool environments (`nix run nixpkgs#<whatever>`) from a shell on Windows, Level 1 is enough. Revisit later when you want persistent config.

---

## Level 2a — home-manager on WSL2

Goal: manage your shell, prompt, git, CLI tools declaratively from a `home.nix` inside WSL2. No Windows changes; no NixOS involved.

### Install home-manager (standalone)

Inside your WSL2 Debian/Ubuntu shell, follow whichever of classic or flakes your Level 1 install is on. The shape is exactly the same as on Kali — see [home-manager-modes.md](home-manager-modes.md) for both paths and the trade-offs.

Minimal `~/.config/home-manager/home.nix` tuned for WSL2:

```nix
{ config, pkgs, ... }:
{
  home.username = "alice";
  home.homeDirectory = "/home/alice";
  home.stateVersion = "24.11";

  # REQUIRED on non-NixOS — integrates with the distro's /etc, systemd, xdg
  targets.genericLinux.enable = true;

  programs.home-manager.enable = true;

  # The portable toolbox
  home.packages = with pkgs; [
    ripgrep fd bat eza jq fzf htop tmux neovim git-delta
  ];

  programs.fish = {
    enable = true;
    shellAbbrs = { g = "git"; gs = "git status"; };
  };

  programs.git = {
    enable = true;
    userName = "Alice Example";
    userEmail = "alice@example.com";
    delta.enable = true;
  };

  programs.starship.enable = true;
}
```

Then:

```bash
home-manager switch
```

This is **the** piece to focus on. If you can reproduce the above on WSL2 Debian and on the Kali laptop and get the same shell experience on both, you have the portable config you came for.

### apt and home-manager on the same WSL2 distro

The coexistence rules are the same as on Debian/Kali — see [getting-started.md](getting-started.md#how-home-manager-coexists-with-apt) for the PATH precedence, dotfile ownership, `.desktop` duplication (less relevant here since WSL2 has no desktop by default), and login-shell gotchas. One additional WSL2-specific note:

- **`windows.exe` interop.** On WSL2, your `$PATH` ends with Windows binaries appended (so you can run `code.exe`, `notepad.exe`, etc. from the Linux shell). This has **no** interaction with Nix — Nix-provided binaries come from `~/.nix-profile/bin` near the front of `$PATH`; Windows binaries still work at the end. If interop causes a measurable issue, disable it in `/etc/wsl.conf`:
  ```ini
  [interop]
  appendWindowsPath = false
  ```

### Auditing what home-manager provides

Exactly as described in [getting-started.md](getting-started.md#auditing-exactly-what-home-manager-provides): `home-manager generations`, `home-manager build` for a dry-run, the pure-shell incantation for isolating what HM contributes:

```bash
env -i HOME="$HOME" TERM="$TERM" PATH="$HOME/.nix-profile/bin" bash --noprofile --norc
```

---

## Level 2b — NixOS-WSL

`home-manager` on a regular distro gets you your `$HOME`; it does **not** get you `configuration.nix`, `systemd` services declared from Nix, system-level `programs.*`, or anything else under the NixOS module system. For that you need real NixOS. NixOS-WSL is the no-reboot, no-partition way to have it.

### Install

From the [NixOS-WSL releases page](https://github.com/nix-community/NixOS-WSL/releases), download the latest `nixos-wsl.tar.gz`. From PowerShell:

```powershell
wsl --import NixOS C:\wsl\NixOS .\nixos-wsl.tar.gz --version 2
wsl -d NixOS
```

On first boot you land in a NixOS shell. The flake lives at `/etc/nixos/` as usual.

### What's different from a full NixOS

- No bootloader (WSL handles boot).
- No desktop / display manager (WSL-g handles GUI apps if you install them).
- `hardware-configuration.nix` is auto-generated for the WSL environment.
- Networking is bridged through Windows; no `networking.networkmanager` dance needed.

Inside, everything else is vanilla NixOS. `/etc/nixos/configuration.nix` behaves as described in [configuration.md](configuration.md), and you can layer home-manager as the NixOS module (see [home-manager-modes.md](home-manager-modes.md)) if you want system + user in one rebuild.

### Rebuilds

```bash
sudo nixos-rebuild switch
```

To make it permanent and set as default distro:

```powershell
wsl --set-default NixOS
```

### When to use NixOS-WSL vs WSL2+home-manager

| Question                                                              | Answer       |
| --------------------------------------------------------------------- | ------------ |
| Want to write and test real `configuration.nix` without rebooting?    | NixOS-WSL    |
| Want only your `$HOME` declarative, keep apt for system packages?     | WSL2 Debian  |
| Need systemd services declared from Nix?                              | NixOS-WSL    |
| Want Docker/WSL interop with normal Ubuntu ergonomics, Nix added on?  | WSL2 Ubuntu  |

Many people run **both** distros side by side in WSL2 — `wsl -d NixOS` for NixOS experiments, `wsl -d Debian` for the home-manager-only flow.

---

## Level 2b alt — NixOS in a VirtualBox VM

VirtualBox is already on your Windows desktop; this is the path of least resistance if you want a NixOS that boots with a real KDE/GNOME session, loads its own kernel, and can be snapshotted before every change.

### Step-by-step

1. **Download the ISO.** From [nixos.org/download](https://nixos.org/download.html): pick the "Graphical ISO" (KDE or GNOME live image) if you want a GUI installer, or the minimal ISO if you're comfortable installing from a console.
2. **Create the VM.** In VirtualBox: New → name `nixos-sandbox`, type Linux, version "Other Linux (64-bit)". RAM at least 4 GB, ideally 8. Disk at least 30 GB, dynamic.
3. **Attach the ISO** as a CD/DVD drive; enable EFI in System → Motherboard (matches modern NixOS defaults).
4. **Before first boot**, enable:
   - Display → Video memory 128 MB, 3D acceleration on.
   - Shared Clipboard → Bidirectional.
   - USB → USB 3.0 controller.
5. **Boot**, run the graphical installer, create a user (`alice`), reboot.
6. **Install Guest Additions** for proper resize + shared folders + smooth cursor:
   ```nix
   # in configuration.nix
   virtualisation.virtualbox.guest.enable = true;
   ```
   Rebuild and reboot.

### Why VirtualBox over Hyper-V or VMware on Windows

- Free, no account required.
- Snapshots are first-class — take one before every `nixos-rebuild switch`, roll back if anything feels off.
- Shared folders work both directions out of the box with Guest Additions.
- It's what you already use.

### Limitations to know about

- **Nested virtualization** on VirtualBox is fine on modern AMD/Intel CPUs, but do not expect libvirt/KVM *inside* the NixOS VM to be fast. If you want to test libvirt, use Hyper-V or WSL2 instead.
- **3D / Vulkan** performance is basic; fine for a KDE session, not for gaming benchmarks.
- **USB passthrough** to the VM requires the Oracle Extension Pack (separate download, PUEL-licensed — free for personal use, not redistributable).

### The `nixos-rebuild build-vm` shortcut

If you already have a NixOS config you want to test (from NixOS-WSL, from a flake in a git repo), you don't need to install NixOS into a VirtualBox VM at all — you can build a throwaway QEMU VM from any NixOS config:

```bash
nixos-rebuild build-vm --flake .#yourhost
./result/bin/run-yourhost-vm
```

This is faster than VirtualBox for iterating on a config, and it runs from inside WSL2 if you enabled `systemd=true`. Use VirtualBox when you want a persistent, named, snapshot-able sandbox.

### Shared folders

In VirtualBox settings, Shared Folders → Add → point at `C:\Users\you\code\nix-config` and check "Auto-mount". Inside the VM:

```bash
sudo usermod -aG vboxsf alice
# reboot or re-login
cd /media/sf_nix-config
```

Now you can edit your flake in VS Code on Windows and `sudo nixos-rebuild switch --flake /media/sf_nix-config#nixos-sandbox` inside the VM.

---

## Level 2b alt — libvirt/KVM via WSL2 or Hyper-V

If VirtualBox starts to feel limiting (snapshot disk size, USB quirks, nested-virt performance), two fallbacks:

- **libvirt/KVM inside WSL2.** Requires `systemd=true` and enabling virtualization passthrough in `.wslconfig`. Slower-to-set-up than VirtualBox on Windows but closer to what you'd run on Kali or on a real NixOS host — so the config is portable.
- **Hyper-V with a NixOS VM.** Hyper-V coexists badly with VirtualBox historically; recent VirtualBox + Hyper-V Platform work together but with a performance hit. If you go Hyper-V, go all-in and uninstall VirtualBox from this machine.

Neither is a must-have. Mention only as "when VirtualBox is no longer enough".

---

## Level 3 — NixOS on an external SSD

Real hardware, native speed, **nothing changes on your internal Windows disk**. This is the exact equivalent of the external-SSD strategy on the Kali page — and it's the one recommended path when you want to stress-test your NixOS config against the actual Windows desktop's GPU/audio/WiFi.

### Why this is the safest Level 3

- Your internal NVMe stays exactly as it is. No partition shrink, no GRUB-on-Windows-disk question.
- You control boot selection via the BIOS/UEFI boot menu (usually F12 at POST).
- You can unplug the SSD and the machine boots Windows normally with zero trace of the experiment.

### Hardware

- A USB 3.2 / Thunderbolt / NVMe-in-enclosure SSD. SATA-over-USB 3.0 works but is slower; NVMe enclosures with USB 3.2 Gen 2 (10 Gbit/s) or USB4/TB3 are the sweet spot.
- 500 GB is generous; 250 GB is plenty for a testbed.

### Install

1. Write the NixOS ISO to a separate USB stick (Rufus on Windows, or `dd`/Etcher from a running Linux).
2. Boot the USB installer with **the external SSD plugged in and the internal Windows disk identified**.
3. **Before partitioning**: identify disks carefully. `lsblk -d -o NAME,SIZE,MODEL,TRAN` shows transport (`sata`, `nvme`, `usb`); your external SSD is `usb`. Double-check before any `parted`/installer step. An installer mishap on the wrong disk is the one class of catastrophic mistake this whole strategy is designed to prevent — so go slow here.
4. Install NixOS **onto the USB device only**. Let the installer put the bootloader on the **external SSD's EFI partition**, not on the internal Windows disk.
5. Reboot. Use the BIOS boot menu (F12 or equivalent) to pick the external SSD.

### Result

- SSD plugged in + boot menu → NixOS.
- SSD unplugged or boot menu → Windows.
- Internal Windows EFI entries untouched.

This lets you iterate on your real `configuration.nix` against real hardware, come back to Windows when you're done, and never risk your daily driver.

### `nixos-anywhere` for a faster install

Once comfortable, [nixos-anywhere](https://github.com/nix-community/nixos-anywhere) can install NixOS onto the external SSD from a flake, replacing the interactive installer. Run the live installer ISO once to get SSH + a known IP; then from your WSL2 or Kali, `nix run github:nix-community/nixos-anywhere -- --flake .#external-ssd root@<ip>`. Overkill for your first install, very useful once your flake is multi-host — see [multi-host-flake.md](multi-host-flake.md).

---

## Level 3 alt — dual-boot on the internal drive

The classic option. Works, reversible, but strictly riskier than the external SSD. Don't go here first.

### Outline

1. **Back up Windows** (File History, Macrium Reflect free, Windows's own system image).
2. **Turn off Fast Startup**: Control Panel → Power Options → "Choose what the power buttons do" → uncheck "Turn on fast startup". Fast Startup leaves NTFS in an inconsistent state that Linux refuses to mount rw.
3. **Turn off BitLocker** on the Windows partition before you touch partitions (or at least pause it). Forgetting this leaves you unable to resize.
4. **Shrink Windows** from Disk Management → right-click C: → Shrink Volume. Leave 200–300 GB for NixOS if that's plausible; 100 GB minimum.
5. Boot the NixOS installer. Install into the freshly-freed space. The installer will add a NixOS entry to the **existing Windows EFI partition**. This is the key risk — if it misdetects and reformats the EFI partition, you lose the Windows boot entries.
6. After install, the bootloader (systemd-boot or GRUB) shows both entries.

### Why external SSD is preferred

Everything on that list has a failure mode. BitLocker surprises are the most common; "I didn't realise Fast Startup was on and now my NTFS is read-only" is second. External SSD has none of these because you never touch Windows's disk.

If you still want dual-boot, treat it as a **graduation step** after you've run NixOS off the external SSD for a few weeks and know your config.

---

## Sharing one `home.nix` across Windows-side Linuxes

The whole point of Level 2a is that the same `home.nix` works on WSL2 Debian, NixOS-WSL (as a home-manager module), the VirtualBox NixOS, and the external-SSD NixOS. Two patterns make that cheap:

### Pattern 1 — one flake, many users and hosts

```nix
# flake.nix
{
  outputs = { self, nixpkgs, home-manager, ... }: {
    # standalone hm outputs for non-NixOS targets
    homeConfigurations."alice@wsl-debian" = home-manager.lib.homeManagerConfiguration {
      pkgs = import nixpkgs { system = "x86_64-linux"; config.allowUnfree = true; };
      modules = [ ./home.nix ./hosts/wsl-debian.nix ];
    };

    # NixOS hosts (NixOS-WSL, VirtualBox sandbox, external SSD)
    nixosConfigurations = {
      nixos-wsl        = nixpkgs.lib.nixosSystem { /* ... */ };
      nixos-vbox       = nixpkgs.lib.nixosSystem { /* ... */ };
      nixos-extssd     = nixpkgs.lib.nixosSystem { /* ... */ };
    };
  };
}
```

The per-host files (`hosts/wsl-debian.nix`, `hosts/nixos-vbox.nix`, etc.) carry the tiny deltas (a font set on a real desktop, `targets.genericLinux.enable` on the WSL2 Debian one, `virtualisation.virtualbox.guest.enable` on the VM). The shared `home.nix` is the constant.

See [multi-host-flake.md](multi-host-flake.md) for the full structure.

### Pattern 2 — small per-host deltas via `if`

```nix
# home.nix
{ config, pkgs, lib, ... }:
{
  home.packages = with pkgs; [
    ripgrep fd bat eza jq fzf htop tmux neovim
  ] ++ lib.optionals (!config.wsl.enable or false) [
    # packages that only make sense on a real desktop, not in WSL
    vlc firefox-wayland
  ];
}
```

Use this for one-off differences; reach for Pattern 1 when there are more than a handful.

---

## Gaming on the NixOS side (when you choose to test it)

A condensed pointer section — the Windows desktop is your gaming machine and will remain so; this section is only relevant when you want to see whether a given game runs inside your NixOS external-SSD config for a weekend experiment.

The short story:

- **Steam/Proton** runs the majority of Windows games on Linux. Check [protondb.com](https://www.protondb.com) per title.
- **Kernel-level anti-cheat** (Vanguard for LoL/Valorant, Ricochet for Call of Duty, some BattlEye modes) does **not** work on Linux, VM or not. Check [areweanticheatyet.com](https://areweanticheatyet.com) for current status.
- **Lutris / Heroic / Bottles** cover GOG / Epic / Battle.net / Ubisoft.
- **Wine + winetricks** for one-off Windows apps.
- **GPU passthrough Windows VM** is possible but involved, and kernel anti-cheat often detects it anyway.

Minimal NixOS snippet to enable the stack on the external-SSD install (all of this lives in [configuration.md](configuration.md)'s module grammar and [packages.md](packages.md)'s binary-running recipes):

```nix
hardware.graphics = { enable = true; enable32Bit = true; };
nixpkgs.config.allowUnfree = true;

programs.steam = {
  enable = true;
  remotePlay.openFirewall = true;
  gamescopeSession.enable = true;
  extraCompatPackages = with pkgs; [ proton-ge-bin ];
};

programs.gamemode.enable = true;

environment.systemPackages = with pkgs; [
  mangohud lutris protontricks heroic bottles
  wineWowPackages.stable winetricks
];
```

Nvidia GPU additions live in [snippets.md](snippets.md).

The important **reframe** for your situation: gaming is a *test case* for your NixOS config's hardware readiness, not a reason to leave Windows. If Steam+Proton+your GPU work on the external SSD, good — that proves the config is viable on the desktop's hardware. If a game doesn't run, you shrug and play it on Windows. Windows is still there.

---

## Mental model — Windows vs NixOS, at a glance

A quick orientation table for when you cross between the two. Unchanged from the previous version of this page; useful regardless of which direction you're coming from.

| Windows                                                | NixOS                                                                |
| ------------------------------------------------------ | -------------------------------------------------------------------- |
| Installer + drivers + "next-next-finish" + activation  | One `configuration.nix` file, `nixos-rebuild switch`, done           |
| Registry holds system state                            | `/etc/nixos/` + `~/.config/` hold all declared state                 |
| Program Files / Apps & Features                        | `environment.systemPackages` + `users.users.<you>.packages`          |
| `.exe` from vendor websites                            | `nix-shell -p foo`, `nix search nixpkgs foo`                         |
| Restore Point                                          | Generations (boot menu selects any past system)                      |
| Driver updates via Windows Update                      | Kernel + firmware pinned in your config                              |
| GPU drivers via NVIDIA installer                       | `hardware.nvidia.*` options — see [snippets.md](snippets.md)         |
| UAC prompts                                            | `sudo` / `polkit` — similar concept, quieter                         |
| Task Manager / Services.msc                            | `systemctl`, `journalctl`                                            |
| Scheduled Tasks                                        | `systemd.timers` declared in `configuration.nix`                     |

### What's easier on the NixOS side

- Reinstalling — minutes from a flake on any hardware.
- Updates — atomic; bad rebuild → boot previous generation.
- Automating setup — your whole machine is a git repo.
- Development — compilers, toolchains, language servers, Docker are one `systemPackages` line. See [dev-tools.md](dev-tools.md).

### What's harder (honest list)

- Some games with kernel-level anti-cheat (see the gaming section).
- Adobe Creative Cloud — mostly doesn't work; use GIMP / Darktable / Krita / Resolve instead.
- Exotic hardware (fingerprint readers, RGB keyboards, some WiFi/audio chips). Check [NixOS Hardware](https://github.com/NixOS/nixos-hardware) for your model.
- Niche Windows-only accounting/vendor tools — expect a VM or an alternative.

---

## See also

- [getting-started.md](getting-started.md) — the four commitment levels and the 6-venue experimentation matrix.
- [kali-to-nixos.md](kali-to-nixos.md) — the sister page for the Kali laptop.
- [home-manager-modes.md](home-manager-modes.md) — standalone vs NixOS module mode.
- [multi-host-flake.md](multi-host-flake.md) — one flake, many hosts including WSL + VM + external SSD.
- [configuration.md](configuration.md) — anatomy of `configuration.nix` (applies to NixOS-WSL, VM, and external SSD identically).
- [packages.md](packages.md) — running prebuilt Windows-adjacent binaries under Nix.
- [snippets.md](snippets.md) — Nvidia/AMD/Intel GPU recipes.
- [protondb.com](https://www.protondb.com) — per-game Proton compatibility.
- [areweanticheatyet.com](https://areweanticheatyet.com) — current anti-cheat status by game.
- [github.com/nix-community/NixOS-WSL](https://github.com/nix-community/NixOS-WSL) — the NixOS-on-WSL distribution.
