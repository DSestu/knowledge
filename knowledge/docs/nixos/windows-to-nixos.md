# From Windows to NixOS

If you've just switched from Windows, this is the orientation page: what changes, how to set up the gaming stack (Steam/Proton, Lutris, Heroic, Bottles, Wine), what works almost transparently, and what — most painfully League of Legends — simply does not run any more because of kernel-level anti-cheat.

## In this page

- [What actually changes vs. Windows](#what-actually-changes-vs-windows)
- [Gaming stack overview](#gaming-stack-overview)
- [Steam and Proton](#steam-and-proton)
- [Non-Steam gaming: Lutris, Heroic, Bottles](#non-steam-gaming-lutris-heroic-bottles)
- [League of Legends — the Vanguard problem](#league-of-legends-the-vanguard-problem)
- [Anti-cheat compatibility in general](#anti-cheat-compatibility-in-general)
- [Wine directly](#wine-directly)
- [Performance — Gamemode, MangoHud](#performance-gamemode-mangohud)
- [Non-gaming Windows apps](#non-gaming-windows-apps)
- [Last resort — Windows VM with GPU passthrough](#last-resort-windows-vm-with-gpu-passthrough)

---

## What actually changes vs. Windows

### Mental model

| Windows                                                | NixOS                                                                |
| ------------------------------------------------------ | -------------------------------------------------------------------- |
| Installer + drivers + "next-next-finish" + activation  | One `configuration.nix` file, `nixos-rebuild switch`, done           |
| Registry holds system state                            | `/etc/nixos/` + `~/.config/` hold all declared state                 |
| Program Files / Apps & Features                        | `environment.systemPackages` + `users.users.<you>.packages`          |
| `.exe` from vendor websites                            | `nix-shell -p foo`, `nix search nixpkgs foo`                         |
| Restore Point                                          | Generations (boot menu selects any past system)                      |
| Driver updates via Windows Update                      | Kernel + firmware pinned in your config                              |
| GPU drivers via NVIDIA installer                       | `hardware.nvidia.*` options — see [snippets.md](snippets.md)         |
| Antivirus as a layered third-party thing               | Not a thing on Linux for personal use                                |
| UAC prompts                                            | `sudo` / `polkit` — similar concept, much quieter                    |

### What gets easier

- **Reinstalling.** 15 minutes from a USB stick, bit-for-bit identical. Windows's "clean install" takes an afternoon.
- **Updates.** Atomic. You don't get a failed update leaving the machine bricked — either the new generation boots or you pick the old one from the menu.
- **Automating setup.** Your whole machine is a git repo. No "note down 40 settings" between reinstalls.
- **Development.** Everything you need (compilers, toolchains, VSCode, Docker, virt-manager) is one `environment.systemPackages` line. See [dev-tools.md](dev-tools.md).

### What gets harder (be honest)

- **Some games with kernel-level anti-cheat** — Vanguard (League of Legends, Valorant), BattlEye in hardcoded mode, Easy Anti-Cheat when the publisher hasn't opted into Linux support. Full section on this below.
- **Adobe Creative Cloud** — mostly doesn't work on Linux at all. GIMP / Darktable / Krita / DaVinci Resolve are the alternatives.
- **MS Office** — the desktop version is flaky in Bottles/Wine. Office 365 web works fine. LibreOffice handles 95% of Office docs.
- **Exotic hardware** — fingerprint readers, some audio chips, some WiFi cards, fancy RGB keyboards. Usually works but not always. Check the [NixOS Hardware](https://github.com/NixOS/nixos-hardware) repo for your model.
- **Some niche Windows-only apps** — accounting software, specific vendor tools, etc. Expect to either run them in a VM or find alternatives.

---

## Gaming stack overview

The layers, from user-facing to kernel:

```
  Steam client / Lutris / Heroic / Bottles
              │
          Proton / Wine    (translates Win32 API → POSIX)
              │
         DXVK / VKD3D       (translates DirectX 9/10/11/12 → Vulkan)
              │
           Vulkan            (your GPU driver)
              │
         Mesa / NVIDIA       (actual driver)
              │
           Kernel
```

For >90% of games the whole stack is invisible — click Play in Steam, game runs. For the rest you poke at one layer at a time.

Enable the essentials in `configuration.nix`:

```nix
# System
hardware.graphics = {
  enable = true;
  enable32Bit = true;                 # required for 32-bit game code
};
nixpkgs.config.allowUnfree = true;

# Steam
programs.steam = {
  enable = true;
  remotePlay.openFirewall = true;
  dedicatedServer.openFirewall = true;
  gamescopeSession.enable = true;
  extraCompatPackages = with pkgs; [ proton-ge-bin ];
};

# Better performance for fullscreen games
programs.gamemode.enable = true;

# GPU overlay (FPS, temps, frametime)
environment.systemPackages = with pkgs; [ mangohud lutris protontricks heroic ];
```

Nvidia users also need:

```nix
services.xserver.videoDrivers = [ "nvidia" ];
hardware.nvidia = {
  modesetting.enable = true;
  open = false;                       # proprietary module — more features for gaming
  nvidiaSettings = true;
  package = config.boot.kernelPackages.nvidiaPackages.stable;
};
```

Rebuild and reboot. Steam should appear in your application launcher.

---

## Steam and Proton

### Proton = Valve's Wine

**Proton** is Valve's fork of Wine, tuned for games and bundled into Steam. Most games run on Proton with no manual work. Check a game's Linux compatibility at [protondb.com](https://www.protondb.com) before buying — ratings "Platinum" and "Gold" mean "works perfectly" and "works with minor tweaks".

### Proton GE — the community fork

Many games need **Proton GE** (Glorious Eggroll's fork), which ships newer DXVK, VKD3D, and custom patches. Install it system-wide as above (`extraCompatPackages`), then:

1. In Steam: right-click a game → Properties → Compatibility.
2. Tick "Force the use of a specific Steam Play compatibility tool".
3. Pick the Proton GE version that appeared.

For per-user management without touching `configuration.nix`, use **protonup-qt** (GUI) or **protonup** (CLI):

```bash
nix-shell -p protonup-qt --run protonup-qt
```

### Enabling Proton for every Steam game

Steam → Settings → Compatibility → "Enable Steam Play for all other titles". Pick Proton Experimental or GE. Now every Windows-only Steam game has the "Play" button available.

### Useful environment flags (per-game)

In Steam → Properties → **Launch Options**, you can set:

```
PROTON_USE_WINED3D=0 DXVK_HUD=fps gamemoderun %command%
```

Common flags:

- `gamemoderun %command%` — CPU governor tweaks for fullscreen performance.
- `mangohud %command%` — FPS/temps overlay (`Right-Shift + F12` to toggle).
- `PROTON_LOG=1` — writes a detailed log to `~/steam-<appid>.log`.
- `PROTON_USE_WINED3D=0` — default; force `=1` only to troubleshoot.
- `PROTON_EAC_RUNTIME=1` — enable Easy Anti-Cheat runtime (Proton Experimental).
- `PROTON_BATTLEYE_RUNTIME=1` — enable BattlEye runtime.

---

## Non-Steam gaming: Lutris, Heroic, Bottles

### Lutris — install anything

[Lutris](https://lutris.net) runs installer scripts (think of them as game-specific Wine prefixes). Good for:

- GOG / Epic / Battle.net / Rockstar launcher / Ubisoft Connect (before Heroic landed).
- Older games, mods, non-Steam Windows installers.

```nix
environment.systemPackages = with pkgs; [ lutris wineWowPackages.staging winetricks ];
```

### Heroic — Epic, GOG, Amazon

[Heroic](https://heroicgameslauncher.com) is a clean, native launcher that talks to Epic Games Store, GOG, and Amazon Prime Gaming. If your Epic / GOG library is what you care about, Heroic is more convenient than Lutris.

```nix
environment.systemPackages = with pkgs; [ heroic ];
```

### Bottles — per-app Wine prefixes

[Bottles](https://usebottles.com) makes it easy to create dedicated Wine prefixes for **non-gaming** Windows apps (small utilities, installers you only want to run once, etc.). Good ergonomics, solid defaults.

```nix
environment.systemPackages = with pkgs; [ bottles ];
```

Or Flatpak if you want the sandboxed version:

```bash
flatpak install flathub com.usebottles.bottles
```

---

## League of Legends — the Vanguard problem

**Short version: League of Legends does not run on Linux as of 2024 and will not run on Linux for the foreseeable future.** Riot Games made **Vanguard** — their kernel-level, driver-based anti-cheat — mandatory on all platforms. Vanguard:

- Runs as a Windows **kernel driver** that boots before the OS.
- Actively detects and refuses to run inside Wine / Proton / any VM it recognizes.
- Works around GPU passthrough detection (though passthrough sometimes still works depending on patch).

Before Vanguard (pre-2024 on LoL), the game was playable on Linux via Lutris with Riot's older anti-cheat. That era is over.

### What you can actually do

1. **Dual-boot Windows** — keep a small Windows partition purely for LoL. Painful but supported.
2. **Cloud gaming** — check [GeForce NOW](https://www.nvidia.com/geforce-now/) for LoL availability (has changed over time). Runs in your browser; legally operates around the client restriction.
3. **GPU passthrough Windows VM** — advanced but possible. Vanguard does detect many VM markers; some users report success, many don't. Not a reliable path.
4. **Play something else** — the Dota, Deadlock, and Heroes of the Storm communities will welcome you.

### What NOT to waste your time on

- Compatibility patches in Proton / Lutris / Wine — these don't and won't fix Vanguard.
- `PROTON_VANGUARD_RUNTIME=1` — **does not exist**. Don't trust forum posts claiming otherwise.
- Kernel modules that "spoof" a non-VM environment — cat-and-mouse with an anti-cheat vendor, always ends badly.

### Related casualties

Same situation (as of writing):

- **Valorant** — also Vanguard, also Windows-only.
- **Fortnite** — BattlEye in hardcoded mode, Epic has refused to flip the Linux switch.
- **Apex Legends** — flipped on, then off, then the anti-cheat was swapped for a kernel-level one that doesn't run on Linux. Check ProtonDB for current status.
- **Destiny 2** — Bungie explicitly bans Proton.

Always check [areweanticheatyet.com](https://areweanticheatyet.com) for the current state of specific titles.

---

## Anti-cheat compatibility in general

Three tiers, publisher-dependent:

- **Green** — works transparently on Proton. Easy Anti-Cheat and BattlEye both have Linux-friendly runtimes; many publishers opt in. Examples: Elden Ring, Apex Legends (intermittent), Halo MCC, Warhammer: Vermintide 2.
- **Yellow** — works only if the publisher has flipped the Linux switch server-side. A game can look broken then suddenly work (or vice versa) after a patch. Always recheck protondb.
- **Red / kernel-level** — Vanguard (LoL, Valorant), FACEIT AC, Ricochet (Call of Duty). These don't work and aren't going to.

If a game uses kernel-level anti-cheat and you want it to run on Linux, there are only two serious options: publisher changes their policy, or you dual-boot.

---

## Wine directly

When you don't want a launcher — just install a small Windows app — use Wine directly.

### Install

```nix
environment.systemPackages = with pkgs; [
  wineWowPackages.stable         # both 64-bit and 32-bit Wine
  winetricks                     # install Windows components (msvcrun, dotnet, fonts)
  dxvk                           # DirectX → Vulkan (usually not needed; Proton has it)
];
```

### Run a `.exe` once

```bash
wine64 /path/to/setup.exe
```

### Dedicated prefix per app

A prefix is a fake Windows filesystem. Keep one per app so they don't pollute each other.

```bash
export WINEPREFIX=~/.wine-fooapp
export WINEARCH=win64
wineboot -i                          # initialize the prefix
winetricks corefonts vcrun2019       # install common deps
wine64 /path/to/Foo.exe
```

### Common winetricks incantations

```bash
winetricks dotnet48          # .NET Framework 4.8
winetricks vcrun2022         # Visual C++ 2022 runtime
winetricks corefonts         # Arial, Times New Roman, etc.
winetricks d3dcompiler_47    # shader compiler (D3D 10+)
winetricks allfonts          # everything font-related
```

### Debugging

```bash
WINEDEBUG=err+all wine64 app.exe     # verbose errors
WINEDEBUG=fixme-all wine64 app.exe   # quieter (skip fixme messages)
```

---

## Performance — Gamemode, MangoHud

### Gamemode

Switches CPU governor to performance, disables C-states, boosts GPU priority during gameplay. Zero-config after enabling:

```nix
programs.gamemode.enable = true;
```

Use via `gamemoderun %command%` in Steam launch options, or `gamemoderun ./my-binary` from shell.

### MangoHud — the FPS / temps overlay

```nix
environment.systemPackages = with pkgs; [ mangohud goverlay ];
```

Toggle in-game with `Right-Shift + F12` by default. Configure via `~/.config/MangoHud/MangoHud.conf` or the GUI app **goverlay**. A reasonable starter config:

```ini
# ~/.config/MangoHud/MangoHud.conf
cpu_stats
cpu_temp
gpu_stats
gpu_temp
vram
ram
fps
frametime
frame_timing
position=top-left
font_size=22
```

Or declaratively via home-manager:

```nix
programs.mangohud = {
  enable = true;
  settings = {
    fps = true;
    cpu_stats = true;
    gpu_stats = true;
    position = "top-left";
    font_size = 22;
  };
};
```

### Gamescope

Valve's micro-compositor for gaming. Gives you native resolution override, forced fullscreen, and consistent frame timing. Enabled via `programs.steam.gamescopeSession.enable = true;` (see the gaming stack snippet above). Then from Steam: launch into a dedicated gamescope session, or use `gamescope -W 2560 -H 1440 -- %command%` per-game.

---

## Non-gaming Windows apps

Quick guide for the common cases:

- **Microsoft Office** — use web versions (office.com); for desktop install, Bottles + "Microsoft Office 365" template often works. LibreOffice handles the vast majority of `.docx` / `.xlsx` / `.pptx` fine.
- **Adobe Photoshop / Illustrator** — don't try. Use GIMP / Krita / Inkscape / Affinity (web) or run a Windows VM.
- **Adobe Lightroom** — use Darktable or RawTherapee.
- **Notepad++** — packaged in nixpkgs (`pkgs.notepad-plus-plus`) and it runs via Wine transparently.
- **Total Commander / File Explorer replacements** — try Krusader, Dolphin (KDE), or nnn.
- **Windows-only accounting / tax software** — Bottles or a small Windows VM.
- **iTunes** — don't. Use Strawberry / Clementine; for iOS sync, use libimobiledevice-based tools.
- **OneNote / Evernote** — use Obsidian, Joplin, or the web client.
- **Outlook** — use Thunderbird or KMail; business accounts via Davmail + Thunderbird.

Most "I can't live without X" turns out to be "I haven't tried the Linux-native alternative", with exceptions for professional tools in art / accounting / engineering.

---

## Last resort — Windows VM with GPU passthrough

When nothing else works, run Windows in a VM with a dedicated GPU passed through. Full-speed gaming on the Windows guest, your Linux host unaffected.

Requirements:

- CPU with **IOMMU** support (Intel VT-d, AMD-Vi).
- Two GPUs (integrated + discrete, or two discretes). You dedicate one to the VM, keep the other for the host.
- Motherboard that exposes the GPU in its own IOMMU group (consult [pci.ids](https://linux-hardware.org) or wiki pages for your board).

Minimal NixOS config for the VFIO side:

```nix
boot.kernelParams = [ "intel_iommu=on" "iommu=pt" ];  # or "amd_iommu=on"
boot.initrd.kernelModules = [ "vfio_pci" "vfio_iommu_type1" "vfio" ];
boot.extraModprobeConfig = ''
  options vfio-pci ids=10de:1b81,10de:10f0   # replace with your GPU's PCI IDs
'';

virtualisation.libvirtd = {
  enable = true;
  qemu.ovmf.enable = true;
  qemu.swtpm.enable = true;
};
programs.virt-manager.enable = true;
users.users.alice.extraGroups = [ "libvirtd" "kvm" ];
```

Then create a Windows VM in virt-manager, attach the passed-through GPU, install Windows normally. Performance is within a few percent of native.

Caveats:

- **Kernel anti-cheat may still detect and block.** Vanguard especially. Sometimes VM detection can be masked (hidden state flags on QEMU, hiding hypervisor CPUID leaf); sometimes it cannot. This is an arms race and your mileage varies by patch and publisher.
- **Setup is involved** — expect a weekend the first time. The [Arch Wiki PCI passthrough page](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF) is the canonical reference.
- **Single-GPU passthrough** is possible but hairy (you lose your Linux display while the VM is running).

For non-gaming Windows needs, a plain (non-passthrough) VM under virt-manager is enough and much simpler — configure it in a minute, run Outlook or whatever.

---

## See also

- [packages.md](packages.md) — the `programs.steam`, `programs.nix-ld`, `buildFHSEnv` details for running prebuilt binaries in general.
- [kde.md](kde.md) — desktop setup.
- [snippets.md](snippets.md) — Nvidia/AMD/Intel GPU recipes.
- [tricks.md](tricks.md) — VM build recipes.
- [protondb.com](https://www.protondb.com) — per-game Proton compatibility.
- [areweanticheatyet.com](https://areweanticheatyet.com) — current anti-cheat status by game.
- [github.com/NixOS/nixos-hardware](https://github.com/NixOS/nixos-hardware) — per-model hardware tweaks.
