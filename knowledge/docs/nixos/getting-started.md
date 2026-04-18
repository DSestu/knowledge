# Getting started

NixOS is a Linux distribution where the **entire system** (kernel, services, packages, users, desktop environment, dotfiles) is defined by a single evaluated expression. Every rebuild produces a new generation that you can boot into or roll back from. The learning curve is real but you get atomic upgrades, true rollbacks, reproducible machines, and a declarative way to answer "what did I install six months ago?".

## Your setup, and why this guide is shaped the way it is

This chapter assumes your real situation:

- **Windows Desktop** — your daily driver. It stays. You are **not** migrating to NixOS here; you want to experiment with NixOS *from* Windows so you can develop and test a config.
- **Kali laptop** — your professional machine. It stays on Kali for now. Since Kali is Debian-based, anything described for Debian elsewhere on the internet applies to Kali verbatim.
- **Goal** — a portable, reproducible Linux configuration (a git repo) that you can drop onto any machine and feel at home. Eventually: real NixOS on real hardware (an external SSD, a spare machine, or an eventual Kali replacement). Never: sacrificing Windows on your desktop.

This shape means two things for the doc:

1. **Two starting points, one config.** You will run Nix on both Windows (via a Linux environment inside it) and Kali, iterating on the same repo.
2. **All solutions discussed, none prescribed.** You pick the venue that matches what you are testing. WSL2, VirtualBox, QEMU/KVM on Kali, and an external-SSD NixOS are all legitimate and each exercises a different slice of the config.

## Chapter map

- **getting-started.md** (you are here) — the big picture, the "levels" of commitment, and the experimentation matrix.
- [configuration.md](configuration.md) — anatomy of `configuration.nix`, `nixos-rebuild` subcommands, discovering module options, the module system, unfree packages.
- [kde.md](kde.md) — Plasma 6 setup, Wayland/X11, fonts, HiDPI, and plasma-manager for declarative shortcuts, themes, panels.
- [dev-tools.md](dev-tools.md) — editors, shells, Docker, libvirt, direnv, IDEs, language toolchains, the FHS caveat.
- [packages.md](packages.md) — scopes, search, pinning, unfree, and the Steam / prebuilt-binary problem (`nix-ld`, `steam-run`, FHS envs, autoPatchelfHook).
- [personal-setup.md](personal-setup.md) — SSH server & client, firewall/open ports, Fish, eza/yazi/git/byobu.
- [snippets.md](snippets.md) — reference grab-bag: shell-command snippets plus short `configuration.nix` and `home.nix` recipes.
- [tricks.md](tricks.md) — build VMs and ISOs from a config, live USB, `nixos-anywhere` remote install, generations/rollback/diff/GC.
- [impermanence.md](impermanence.md) — wipe root on every boot, declare exactly what persists; the cleanest expression of "my system is a git repo".
- [windows-to-nixos.md](windows-to-nixos.md) — Windows-side experimentation in depth (WSL2 variants, VirtualBox walkthrough, external-SSD NixOS), plus the full gaming stack (Steam/Proton, Lutris, Heroic, Bottles, Wine, anti-cheat realities) for the day you *do* boot NixOS on real hardware.
- [kali-to-nixos.md](kali-to-nixos.md) — Kali-side experimentation: Nix on top of Kali, home-manager on Kali, NixOS VMs on Kali via libvirt, and the eventual Kali→NixOS replacement path with pentest tooling.

## In this page

- [Your system is a git repo of text files](#your-system-is-a-git-repo-of-text-files)
- [Portable means two different things in Nix](#portable-means-two-different-things-in-nix)
- [The experimentation matrix](#the-experimentation-matrix)
- [The four safe levels of commitment](#the-four-safe-levels-of-commitment)
- [Level 1 — Nix package manager](#level-1-nix-package-manager)
- [Level 2a — home-manager](#level-2a-home-manager)
- [Level 2b — Full NixOS in a sandbox](#level-2b-full-nixos-in-a-sandbox)
- [Level 3 — NixOS on real hardware](#level-3-nixos-on-real-hardware)
- [Classic vs flakes — which to learn first](#classic-vs-flakes-which-to-learn-first)

## Your system is a git repo of text files

A fresh NixOS install drops two files in `/etc/nixos/` and nothing else:

```
/etc/nixos/
├── configuration.nix            ← the whole system, declaratively
└── hardware-configuration.nix   ← machine-specific, auto-generated
```

Every package, service, user, kernel parameter, desktop environment, font, keyboard layout, and shell alias is expressed in `configuration.nix` (or in files it `imports`). There is no hidden state scattered across `/etc/*.d/` directories the way there is on Debian/Kali — `/etc/nixos/configuration.nix` is the single source of truth for the whole machine. Edit it, run one command, reboot into the result.

### The core files

- **`configuration.nix`** — the root of your system definition. A Nix expression that returns an attribute set (Nix's name for "object" / "dict") of options: things like `services.openssh.enable = true;` and `environment.systemPackages = [ pkgs.git ];`. When you run `nixos-rebuild switch`, NixOS reads this file, evaluates it, builds a new system closure, and activates it.

- **`hardware-configuration.nix`** — generated once by `nixos-generate-config` at install time. Contains the bits that are specific to *this particular machine*: detected disks and their UUIDs, kernel modules for your hardware, firmware paths. Usually kept **per-machine** and not shared between machines.

- **Your own modules (optional)** — once `configuration.nix` gets long, split it: `./modules/fish.nix`, `./modules/kde.nix`, `./modules/dev.nix`. Each module is a file that returns the same shape of attribute set. You list them in an `imports = [ ./modules/kde.nix ./modules/dev.nix ];` line at the top of `configuration.nix` and the module system merges them into one big config.

- **`flake.nix` + `flake.lock`** (optional, flakes only) — `flake.nix` declares which external inputs your config depends on (`nixpkgs`, `home-manager`, `disko`, …) and `flake.lock` pins every one of them to an exact commit hash. Together they make your config byte-identical to reproduce anywhere, months or years later.

- **`home.nix`** (optional, home-manager) — your *user-level* config: dotfiles, shell plugins, editor config, GUI apps, themes. Same language, different scope. See the portability section below.

### A typical layout after a few weeks

```
~/nixos-config/
├── flake.nix                    ← inputs + outputs (if using flakes)
├── flake.lock                   ← pinned versions, auto-generated
├── home.nix                     ← home-manager user config (portable, any Linux)
├── hosts/
│   ├── desktop-vbox/            ← VirtualBox VM on the Windows desktop
│   │   ├── configuration.nix
│   │   └── hardware-configuration.nix
│   ├── kali-vm/                 ← libvirt VM on the Kali laptop
│   │   ├── configuration.nix
│   │   └── hardware-configuration.nix
│   └── portable-ssd/            ← external-SSD NixOS that boots on any machine
│       ├── configuration.nix
│       └── hardware-configuration.nix
└── modules/
    ├── kde.nix
    ├── dev.nix
    ├── fish.nix
    └── pentest.nix              ← optional, for the day the Kali replacement happens
```

With flakes, `nixos-rebuild switch --flake ~/nixos-config#desktop-vbox` points straight at the directory — no symlink from `/etc/nixos` needed. One repo, many hosts.

### Why this is powerful: git

Because everything is plain text, the whole system fits in a git repo:

- **`git init`** the config directory and you have the full history of every system change. `git log` tells you when you enabled Bluetooth, added Steam, bumped the kernel, changed your keyboard layout.
- **Push to GitHub/GitLab** (private repo, if you prefer) and you have off-site backup of your entire system definition. The repo is small — just a few hundred lines of config text.
- **Rollback via git** is just as effective as rolling back via the NixOS generation menu. `git revert <bad-commit>` + `nixos-rebuild switch` and the bad change is undone, with a commit message explaining why.
- **Branch per host, or host per directory.** Either keep `desktop-vbox`, `kali-vm`, `portable-ssd` as separate `nixosConfigurations.*` entries in one flake (cleaner), or as git branches.
- **Clone to install a new machine.** From a fresh NixOS install ISO:

  ```bash
  # boot the ISO, partition, mount root at /mnt, then:
  nix-shell -p git
  git clone https://github.com/you/nixos-config.git /mnt/etc/nixos

  # regenerate hardware-configuration.nix for THIS machine
  nixos-generate-config --root /mnt --dir /mnt/etc/nixos/hosts/<name>

  # install — classic:
  nixos-install
  # or with flakes:
  nixos-install --flake /mnt/etc/nixos#<hostname>
  ```

  One reboot later, you are running your exact setup — same KDE theme, same shell, same packages, same fonts, same services — on brand-new hardware. This is the mechanism that makes "my config is portable" literally true.

- **Clone onto a running Linux box over SSH.** [`nixos-anywhere`](tricks.md) takes your flake and a target hostname and turns any Linux machine into your NixOS setup remotely, no ISO needed. Your git repo effectively becomes a zero-click installer.

> On Debian/Kali, cloning your setup between machines means syncing `/etc` by hand, running Ansible, or living with drift. On NixOS, your dotfiles and your operating system are the same kind of artifact: a git repo you clone.

---

## Portable means two different things in Nix

Before the levels, one mental split that will save you confusion later:

- **home-manager = user-level portability.** Manages packages, dotfiles, shell, editor, git config, `~/.config/*`. Runs on **any Linux with Nix installed** — Debian, Ubuntu, Kali, Arch, Fedora, a NixOS box, WSL2 — all behave the same. This is the layer that makes your shell feel like home on every machine you touch.

- **NixOS = system-level portability.** Manages services, kernel, boot, display manager, firewall, users, filesystems. Requires the host to actually be NixOS. This is the layer that reproduces the whole machine, not just your user.

You can (and should) develop both in parallel in the same repo. `home.nix` travels everywhere; `configuration.nix` travels to NixOS hosts. The common case once your repo matures: `home.nix` is your live, day-to-day config on both Kali and inside any NixOS VM, while `configuration.nix` exists to rehearse full-system changes and to provision new hardware.

---

## The experimentation matrix

Six legitimate venues for running Nix/NixOS against your config, across your two machines. Each tests a different slice. Pick based on *what you are trying to exercise* at the moment, not which is "best".

| # | Venue                          | Host      | Tests `home.nix`? | Tests `configuration.nix`? | Tests boot/kernel/real hardware? | Effort |
|---|--------------------------------|-----------|:-----------------:|:--------------------------:|:--------------------------------:|--------|
| 1 | WSL2 + Nix on Debian/Ubuntu    | Windows   | ✅                | ❌ (no NixOS)              | ❌                               | Low    |
| 2 | NixOS-WSL                      | Windows   | ✅                | ⚠️ partial (no bootloader/DM) | ❌                            | Low    |
| 3 | VirtualBox NixOS VM            | Windows   | ✅                | ✅                         | ⚠️ virtual hardware              | Medium |
| 4 | Nix / home-manager on Kali     | Kali      | ✅                | ❌ (no NixOS)              | ❌                               | Low    |
| 5 | libvirt/QEMU NixOS VM on Kali  | Kali      | ✅                | ✅                         | ⚠️ virtual hardware              | Medium |
| 6 | NixOS on an external SSD/USB   | Either    | ✅                | ✅                         | ✅                               | Medium |

Which to reach for:

- **"I just want to try Nix."** → 1 or 4.
- **"I want my shell and dotfiles portable today."** → 1 → 2 or 4. Get `home.nix` solid before anything else.
- **"I want to iterate on `configuration.nix` fast, from Windows."** → 2 (NixOS-WSL) for quick loops on 90% of options; 3 (VirtualBox) when you need a real display manager / boot.
- **"I want to rehearse an install on real hardware without touching Windows."** → 6 (external SSD). BIOS boot menu picks it; Windows disk is never touched.
- **"I want real hardware, but I'm on my Kali machine."** → 5 for the rehearsal, 6 for the real thing.

Details and install commands for each venue live in [windows-to-nixos.md](windows-to-nixos.md) (venues 1–3, 6-from-Windows) and [kali-to-nixos.md](kali-to-nixos.md) (venues 4–5, 6-from-Kali). This page only shows the minimal "prove it works" recipe per level.

---

## The four safe levels of commitment

1. **Nix package manager** — install packages declaratively for your user, zero system impact. Works in WSL2 on Windows and natively on Kali.
2. **home-manager** — your whole user environment (shell, editor, dotfiles, CLI tools) in one `home.nix`. Still no system changes. This is where "portable config" becomes real.
3. **Full NixOS in a sandbox** — a real `configuration.nix` boots in a VM or a WSL distro. You iterate on services, users, desktop environment, without risking either host.
4. **NixOS on real hardware** — external SSD first (Windows-side, non-destructive), then possibly the Kali machine once your config is mature enough to replace the pentest workflow.

You can live on levels 1 + 2 for weeks before touching a VM. That is the recommended approach.

---

## Level 1 — Nix package manager

Install the Nix package manager once per host. Then you can `nix-shell -p <anything>` and packages are fetched from the binary cache, used, and stay garbage-collectable.

### On Windows (via WSL2)

First time? Open PowerShell as admin:

```powershell
wsl --install -d Debian
```

Reboot, finish the Debian first-run wizard, then inside the Debian shell:

```bash
sh <(curl -L https://nixos.org/nix/install) --daemon
```

> The multi-user install uses a systemd daemon and a `/nix` store shared by all users. WSL2 supports systemd since WSL 0.67; if you hit a systemd error, add `[boot]\nsystemd=true` to `/etc/wsl.conf` and run `wsl --shutdown` from Windows. Restart the WSL shell and Nix loads. See [windows-to-nixos.md](windows-to-nixos.md) for the full walkthrough and alternatives (Ubuntu, single-user install, nixos-wsl).

### On Kali

Kali is Debian-based, so the standard installer works unchanged:

```bash
sh <(curl -L https://nixos.org/nix/install) --daemon
```

Log out / log in (or `. /etc/profile`) to pick up the `/etc/profile.d/nix-daemon.sh` sourcing.

### Same recipes both hosts

```bash
# Try a package without installing it
nix-shell -p cowsay --run "cowsay hello"

# Install a package for your user (classic)
nix-env -iA nixpkgs.ripgrep

# Install a package for your user (flakes)
nix profile install nixpkgs#ripgrep
```

> `nix-env` and `nix profile` share the same `~/.nix-profile` but use different metadata. Pick one style and stick to it.

### Enable flakes (optional, recommended long term)

Add to `~/.config/nix/nix.conf` on both hosts:

```ini
experimental-features = nix-command flakes
```

### Uninstall Nix entirely

```bash
/nix/nix-installer uninstall
```

If you installed before the new installer:

```bash
sudo rm -rf /nix /etc/nix
sudo systemctl disable --now nix-daemon.socket nix-daemon.service
```

---

## Level 2a — home-manager

`home-manager` manages your user environment (packages, dotfiles, fish, neovim, git config, etc.) from a single `home.nix`. It runs on any Linux with Nix — so the exact same `home.nix` works in WSL2 on Windows, on Kali, and later inside any NixOS VM or real hardware install. **This is the single highest-leverage file in your repo.**

### Install (classic, channel-based)

```bash
nix-channel --add https://github.com/nix-community/home-manager/archive/master.tar.gz home-manager
nix-channel --update
nix-shell '<home-manager>' -A install
```

### Minimal `~/.config/home-manager/home.nix`

```nix
{ config, pkgs, ... }:
{
  home.username = "alice";
  home.homeDirectory = "/home/alice";
  home.stateVersion = "24.11";

  home.packages = with pkgs; [
    ripgrep
    fd
    bat
    eza
    fzf
    jq
    httpie
    btop
  ];

  programs.fish.enable = true;
  programs.starship.enable = true;

  programs.git = {
    enable = true;
    userName = "Alice Example";
    userEmail = "alice@example.com";
    extraConfig = {
      init.defaultBranch = "main";
      pull.rebase = true;
    };
  };

  programs.direnv = {
    enable = true;
    nix-direnv.enable = true;
  };

  home.sessionVariables.EDITOR = "nvim";

  programs.home-manager.enable = true;
}
```

Apply:

```bash
home-manager switch
```

> Rolling back is `home-manager generations` then `/nix/store/<hash>-home-manager-generation/activate`. Same atomic guarantees as NixOS.

### Install (flakes)

Create `~/.config/home-manager/flake.nix`:

```nix
{
  inputs.nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
  inputs.home-manager.url = "github:nix-community/home-manager";
  inputs.home-manager.inputs.nixpkgs.follows = "nixpkgs";

  outputs = { nixpkgs, home-manager, ... }:
    let system = "x86_64-linux"; in {
      homeConfigurations."alice" = home-manager.lib.homeManagerConfiguration {
        pkgs = import nixpkgs { inherit system; config.allowUnfree = true; };
        modules = [ ./home.nix ];
      };
    };
}
```

Apply:

```bash
home-manager switch --flake ~/.config/home-manager#alice
```

### How home-manager coexists with apt

home-manager on top of Debian/Kali is deliberately non-invasive: apt and Nix cannot see each other. apt installs binaries to `/usr/bin` with root-owned configs under `/etc`; Nix installs to the read-only `/nix/store` and exposes your user's selection via symlinks under `~/.nix-profile`. Neither package manager ever writes into the other's territory, so installing Nix on a working Kali box does not break anything you had before. The interactions are all at the edges:

- **PATH precedence.** home-manager prepends `~/.nix-profile/bin` to your PATH. If both apt and home-manager provide a `ripgrep`, the home-manager one wins in your shell. The apt binary is still on disk, just shadowed. `which ripgrep` and `command -v ripgrep` show which is active. This is usually what you want — and it's why migrating a tool to home-manager is reversible: remove the option, `home-manager switch`, and apt's copy is back in the driver's seat instantly.

- **Dotfiles are shared, but home-manager expects to own them.** Both versions of git read `~/.gitconfig`; both shells read `~/.config/fish/`. The first `home-manager switch` after enabling `programs.git.enable = true;` will *refuse* to run if a hand-written `~/.gitconfig` already exists — the error is `existing file ... is in the way`. Rename or delete the old file, retry, and home-manager now owns it. Disable the option later and the managed file vanishes on the next switch — that is the declarative part working as designed.

- **`.desktop` files coexist.** If you install Firefox via both apt and home-manager, two Firefox entries will appear in your KDE/GNOME launcher. Pick one source of truth per GUI app — either `sudo apt remove firefox-esr` or drop it from `home.packages` — otherwise the duplicate is forever.

- **`targets.genericLinux.enable = true;`** — **this is the option that matters most on Debian/Kali.** On NixOS, desktop integration (launcher entries, icons, `XDG_DATA_DIRS`, fontconfig paths) is wired up automatically. On a non-NixOS host, home-manager does not assume it can touch system paths, so without this flag GUI apps installed via `home.packages` often fail to appear in the menu, icons go missing, and fonts aren't picked up. Turn it on in every `home.nix` that runs on Kali or WSL2-Debian:

  ```nix
  { pkgs, ... }: {
    targets.genericLinux.enable = true;
    # ... rest of your config
  }
  ```

- **systemd services don't collide.** apt installs *system* services to `/etc/systemd/system/` (`systemctl ...`); home-manager installs *user* services to `~/.config/systemd/user/` (`systemctl --user ...`). Different scopes, no conflict. A home-manager service can even depend on an apt-provided system service — the unit files just reference each other normally.

- **Login-shell vs interactive-shell gotcha.** home-manager writes its environment setup (PATH, `NIX_PATH`, XDG dirs) into `~/.profile` or a file it sources. On Debian, `~/.profile` is sourced for **login shells only**. If your terminal emulator launches a non-login shell, `nix` and your home-manager binaries will appear "not found" despite being installed — a classic first-day confusion. Three fixes, pick one:

  1. Configure your terminal (Konsole/Alacritty/etc.) to run the shell as a login shell.
  2. Source the hooks from `~/.bashrc`: `. ~/.nix-profile/etc/profile.d/hm-session-vars.sh`.
  3. Let home-manager do it — set `programs.bash.enable = true;` (or `programs.fish.enable`, `programs.zsh.enable`) and it injects the right hooks into its managed shell rc.

- **Binaries are fully self-contained.** Nix-built binaries don't link against apt's libraries — they bundle their own libc, OpenSSL, etc. via store-relative rpaths. Upside: no version skew, no "works on NixOS, breaks on Kali". Downside: doesn't apply to random prebuilt `.tar.gz` binaries you download yourself (those need `nix-ld` / `steam-run` / an FHS env — see [packages.md](packages.md)).

**Practical rule of thumb on Kali/WSL2-Debian.** Let home-manager own your *user tools* — shell, editor, CLI utilities, dotfiles, per-user GUI apps — and remove the apt equivalents as you migrate, so you don't end up running two different versions decided by PATH. Leave apt in charge of *system-level things you're not migrating* — display manager, GPU drivers, Bluetooth daemon, Kali's pentest metapackages, the kernel itself. home-manager isn't designed to manage those; that is the job `configuration.nix` does once you run NixOS proper.

A reasonable starter `home.nix` for a Kali host, reflecting the above:

```nix
{ pkgs, ... }:
{
  targets.genericLinux.enable = true;      # CRITICAL on non-NixOS
  programs.home-manager.enable = true;
  programs.bash.enable = true;             # picks up hm-session-vars in ~/.bashrc

  home.packages = with pkgs; [ ripgrep fd bat eza fzf jq httpie btop ];
  programs.fish.enable = true;
  programs.starship.enable = true;
  # ...
}
```

See [kali-to-nixos.md](kali-to-nixos.md) for the migration recipes — which `apt remove` pairs with which `programs.* = true;`, how to make fish your login shell without confusing `/etc/shells`, and the full "move my daily tools to home-manager without losing my Kali workflow" walkthrough.

### Auditing exactly what home-manager provides

A legitimate worry when home-manager sits on top of apt: *which of the tools I run came from which package manager?* Three complementary techniques — the first two show you, the third gives you a sandbox where only home-manager is on the table.

**1. Inventory what the current generation ships.**

```bash
# Every package in the current home-manager generation:
home-manager packages

# Every generation, with timestamps and store paths:
home-manager generations

# Every binary home-manager currently exposes — these are symlinks into /nix/store:
ls -l ~/.nix-profile/bin/

# Every dotfile home-manager owns, anywhere under your home dir:
find ~ -lname '/nix/store/*' 2>/dev/null
```

The `find` one-liner is the single clearest answer to "what is home-manager writing into my home?" — anything that isn't a symlink into `/nix/store/*` is either yours or apt's, not home-manager's.

**2. Build a preview without activating it.**

```bash
home-manager build                      # produces ./result/
ls result/home-path/bin/                # every binary the next generation would expose
ls result/home-files/                   # every dotfile it would place
readlink result/home-files/.config/fish/config.fish
```

`home-manager build` runs the full evaluation and build but stops short of activation — no symlinks are swapped into your home, nothing changes in your environment. You can diff `result/home-path/bin/` against the current `~/.nix-profile/bin/` to see precisely what a pending change would add or remove.

**3. Enter a pure shell where only home-manager is on PATH.**

```bash
env -i \
  HOME="$HOME" \
  TERM="$TERM" \
  PATH="$HOME/.nix-profile/bin" \
  bash --noprofile --norc
```

`env -i` strips every inherited environment variable (so no `PATH` bleed-through, no LD-library shenanigans); `bash --noprofile --norc` tells bash to skip `/etc/profile`, `~/.profile`, `/etc/bash.bashrc`, and `~/.bashrc`. Inside that shell, `/usr/bin` is not on your `PATH` and the apt toolchain is unreachable. `command -v ripgrep` resolves to `~/.nix-profile/bin/ripgrep` if home-manager provides it; `command -v apt` returns nothing at all. If basic utilities like `ls` or `cat` fail — that's the audit answering the question honestly: your generation doesn't include `coreutils`, and on a NixOS host it would be provided by the system side, not by home-manager.

To make the pure shell self-sufficient, add what you need to `home.packages`:

```nix
home.packages = with pkgs; [
  coreutils findutils gnugrep gnused gawk   # the usual GNU suspects
  ripgrep fd bat eza fzf jq
];
```

Rebuild, drop back into the pure shell, and the missing commands are now there — and *only* there, because there's still nothing else on `PATH`.

**Bonus — sandbox a single package without installing it.**

```bash
nix-shell --pure -p fish --run fish
```

A throwaway shell with exactly fish (plus Nix's minimal base) and nothing else. No home-manager, no apt, nothing from your dotfiles. Perfect for "does this package do what I think it does before I commit to adding it to `home.nix`?".

### Working the same file on both machines

The practical workflow is to keep `home.nix` (and eventually the whole config) in a git repo that both hosts clone:

```bash
# On Windows (inside WSL2) AND on Kali
git clone git@github.com:you/nixos-config.git ~/nixos-config
ln -sf ~/nixos-config/home.nix ~/.config/home-manager/home.nix
home-manager switch
```

Edit on either host, commit, pull on the other, `home-manager switch`, you are done. Your shell feels identical on both machines.

---

## Level 2b — Full NixOS in a sandbox

Once `home.nix` feels solid, start writing a real `configuration.nix`. You have three places to boot it, depending on what you are testing.

### Option A — NixOS-WSL (fastest iteration on Windows)

[NixOS-WSL](https://github.com/nix-community/NixOS-WSL) ships NixOS as a WSL distribution. You get a real `/etc/nixos/configuration.nix` and `nixos-rebuild switch`, with the important caveat that WSL supplies its own kernel and doesn't run a bootloader or a display manager — so options like `boot.loader.*`, `services.displayManager.*`, and the full Plasma desktop won't have real-world effects here. Everything else (packages, services, users, fonts, shells, networking, docker, virt-manager, language toolchains) is faithful.

Quick start, inside Windows PowerShell:

```powershell
wsl --import NixOS $env:USERPROFILE\NixOS\ .\nixos-wsl.tar.gz --version 2
wsl -d NixOS
```

Full install walkthrough and the "what works / doesn't" list in [windows-to-nixos.md](windows-to-nixos.md).

### Option B — VirtualBox VM on Windows (realistic, full desktop)

This is your primary "test it for real" venue on the Windows desktop. Two workflows:

**B.1 — Boot the installer ISO, install normally.** Download the NixOS installer from [nixos.org/download](https://nixos.org/download), create a new VM in VirtualBox (4 GB RAM, 2+ cores, 40 GB dynamic disk, EFI enabled), attach the ISO, boot, install as you would on real hardware. Pull your config repo inside the VM with `git clone`. Iterate by editing and running `sudo nixos-rebuild switch` in a terminal.

**B.2 — Build a VirtualBox image from your flake.** Inside WSL2 (or on Kali), have Nix produce a `.ova` you import into VirtualBox:

```bash
nix build .#nixosConfigurations.desktop-vbox.config.system.build.virtualBoxOVA
# import result/*.ova into VirtualBox
```

Workflow B.2 is faster once the flake is set up; B.1 is easier the very first time. Step-by-step for both, with the VirtualBox guest-additions and shared-folder settings, in [windows-to-nixos.md](windows-to-nixos.md).

### Option C — libvirt/QEMU VM on Kali (the native Linux venue)

Kali has KVM available out of the box. The Debian/Nix community's `nixos-rebuild build-vm` flow works directly — no ISO, no install, just a script that spins up QEMU with your config baked in.

Prereqs on Kali:

```bash
sudo apt install qemu-system-x86 qemu-utils libvirt-daemon-system virt-manager
sudo adduser "$USER" kvm libvirt
# log out / log in
```

Minimal `configuration.nix`:

```nix
{ pkgs, lib, modulesPath, ... }:
{
  imports = [ "${modulesPath}/virtualisation/qemu-vm.nix" ];

  virtualisation.memorySize = 4096;
  virtualisation.cores = 4;
  virtualisation.diskSize = 20480;

  services.xserver.enable = true;
  services.desktopManager.plasma6.enable = true;
  services.displayManager.sddm.enable = true;

  users.users.alice = {
    isNormalUser = true;
    password = "test";
    extraGroups = [ "wheel" ];
  };

  environment.systemPackages = with pkgs; [ firefox kate ripgrep ];

  system.stateVersion = "24.11";
}
```

Build and run (classic):

```bash
nix-channel --add https://nixos.org/channels/nixos-unstable nixos
nix-channel --update
nixos-rebuild build-vm -I nixos-config=./configuration.nix
./result/bin/run-*-vm
```

Build and run (flakes). `flake.nix` alongside `configuration.nix`:

```nix
{
  inputs.nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
  outputs = { nixpkgs, ... }: {
    nixosConfigurations.kali-vm = nixpkgs.lib.nixosSystem {
      system = "x86_64-linux";
      modules = [ ./configuration.nix ];
    };
  };
}
```

```bash
nixos-rebuild build-vm --flake .#kali-vm
./result/bin/run-*-vm
```

### Iteration loop (any option above)

1. Edit `configuration.nix`.
2. Re-run the rebuild (`sudo nixos-rebuild switch` inside the VM/WSL, or the `build-vm` command outside).
3. Observe, repeat.

Subsequent rebuilds only fetch what changed — the binary cache is shared across every venue and your eventual on-metal install. Skills you build here transfer directly to level 3.

---

## Level 3 — NixOS on real hardware

When the sandbox loop stops surprising you, move to real hardware. Two realistic endpoints given your setup:

### 3a — External SSD / USB NixOS (Windows-friendly, non-destructive)

Install NixOS to an external SSD or a fast USB stick. Your Windows disk is never touched. At boot, the BIOS/UEFI boot menu (usually F12 or F8) lets you pick the external drive when you want NixOS, or leave it alone and Windows boots normally.

Flow:

1. On Windows, flash the NixOS ISO to a USB stick with [Rufus](https://rufus.ie) or [balenaEtcher](https://www.balena.io/etcher/).
2. Plug *both* that installer stick and your target external SSD into the desktop.
3. Boot the installer stick from the BIOS boot menu.
4. During install, partition *the external SSD only* — double-check by UUID / serial. Install the bootloader to the external SSD's own ESP, not the internal disk.
5. Clone your flake: `git clone ... /mnt/etc/nixos`, then `nixos-install --flake /mnt/etc/nixos#portable-ssd`.
6. Reboot, unplug the installer, select the external SSD from the BIOS boot menu whenever you want NixOS.

Full walkthrough, including the "don't clobber Windows's ESP" safety checks, in [windows-to-nixos.md](windows-to-nixos.md).

### 3b — Replacing Kali (only when you're ready)

This is the late move. Your Kali machine becomes a NixOS machine; its pentest toolchain becomes a NixOS module (`modules/pentest.nix`) reproducing the specific tools you actually use from Kali. All covered in [kali-to-nixos.md](kali-to-nixos.md). Rule of thumb: do 3a first, use it as your daily NixOS endpoint for a few weeks, and only commit to 3b once you are sure the config reproduces your real Kali workflow.

### 3c — `nixos-anywhere` (worth knowing)

From either host, once you have a reachable target (a spare laptop, a VPS, a colleague's NixOS box), [`nixos-anywhere`](tricks.md) turns any Linux into your flake over SSH — no ISO, no console access. Overkill for the first install, invaluable later.

---

## Classic vs flakes — which to learn first

- **Classic (channels + `configuration.nix`)**: imperative channels (`nix-channel --update`) pull the latest nixpkgs revision when you update. Simpler conceptually, no extra syntax. Nothing is pinned by default — reproducibility comes from locking the channel revision yourself.
- **Flakes (`flake.nix` + `flake.lock`)**: inputs are explicit, pinning is automatic via `flake.lock`, composition across multiple hosts (which is your exact situation) is dramatically easier. Adds the `flake.nix` file and the `nix <subcommand>` CLI to learn.

Given you will run the same config across Windows/WSL, Kali, and at least one VM or SSD, **flakes are worth the upfront cost** — the "one repo, many hosts" pattern is their home turf. Both styles are shown side by side under `### Classic` and `### Flakes` subsections throughout this chapter, so you can start with Classic and migrate later if you prefer.

## Next

- [configuration.md](configuration.md) — the `configuration.nix` mental model and the `nixos-rebuild` workflow.
- [windows-to-nixos.md](windows-to-nixos.md) — WSL2 install routes in depth, VirtualBox walkthrough, external-SSD install, plus the gaming stack.
- [kali-to-nixos.md](kali-to-nixos.md) — Nix on Kali, home-manager on Kali, libvirt NixOS VMs, Kali→NixOS replacement path.
- [kde.md](kde.md) — Plasma 6 options.
- [dev-tools.md](dev-tools.md) — git, docker, direnv, IDEs, language toolchains.
- [packages.md](packages.md) — adding packages, unfree, and the Steam/prebuilt-binary problem.
- [personal-setup.md](personal-setup.md) — SSH, firewall, fish, eza/yazi recipes.
- [snippets.md](snippets.md) — copy-paste recipes.
- [tricks.md](tricks.md) — live USB from your config, VM tricks, `nixos-anywhere`, rollback.
