# Getting started

NixOS is a Linux distribution where the **entire system** (kernel, services, packages, users, desktop environment, dotfiles) is defined by a single evaluated expression. Every rebuild produces a new generation that you can boot into or roll back from. The learning curve is real but you get atomic upgrades, true rollbacks, reproducible machines, and a declarative way to answer "what did I install six months ago?".

This page is the safe on-ramp: try Nix without reinstalling, then test a full NixOS config in a VM, then — only when you are comfortable — install on metal.

## Chapter map

- **getting-started.md** (you are here) — how to try NixOS without risking your Debian, VM-first workflow, install on metal.
- [configuration.md](configuration.md) — anatomy of `configuration.nix`, `nixos-rebuild` subcommands, discovering module options, modules system, unfree packages.
- [kde.md](kde.md) — Plasma 6 setup, Wayland/X11, fonts, HiDPI, and plasma-manager for declarative shortcuts, themes, panels.
- [dev-tools.md](dev-tools.md) — editors, shells, Docker, libvirt, direnv, IDEs, language toolchains, the FHS caveat.
- [packages.md](packages.md) — scopes, search, pinning, unfree, and the Steam / prebuilt-binary problem (`nix-ld`, `steam-run`, FHS envs, autoPatchelfHook).
- [personal-setup.md](personal-setup.md) — SSH server & client, firewall/open ports, Fish translated from the Debian recipe, eza/yazi/git/byobu.
- [snippets.md](snippets.md) — reference grab-bag: shell-command snippets plus short `configuration.nix` and `home.nix` recipes.
- [tricks.md](tricks.md) — build VMs and ISOs from a config, live USB, `nixos-anywhere` remote install, generations/rollback/diff/GC.
- [impermanence.md](impermanence.md) — wipe root on every boot, declare exactly what persists; the cleanest expression of "my system is a git repo".
- [windows-to-nixos.md](windows-to-nixos.md) — orientation for Windows switchers, the full gaming stack (Steam/Proton, Lutris, Heroic, Bottles, Wine), anti-cheat realities including League of Legends.

## In this page

- [Your system is a git repo of text files](#your-system-is-a-git-repo-of-text-files)
- [The three safe levels of commitment](#the-three-safe-levels-of-commitment)
- [Level 1 — Nix package manager on Debian](#level-1-nix-package-manager-on-debian)
- [Level 2a — home-manager on Debian](#level-2a-home-manager-on-debian)
- [Level 2b — NixOS config in a VM](#level-2b-nixos-config-in-a-vm)
- [Level 3 — NixOS on metal](#level-3-nixos-on-metal)
- [Classic vs flakes — which to learn first](#classic-vs-flakes-which-to-learn-first)

## Your system is a git repo of text files

A fresh NixOS install drops two files in `/etc/nixos/` and nothing else:

```
/etc/nixos/
├── configuration.nix            ← the whole system, declaratively
└── hardware-configuration.nix   ← machine-specific, auto-generated
```

Every package, service, user, kernel parameter, desktop environment, font, keyboard layout, and shell alias is expressed in `configuration.nix` (or in files it `imports`). There is no hidden state scattered across `/etc/*.d/` directories the way there is on Debian — `/etc/nixos/configuration.nix` is the single source of truth for the whole machine. Edit it, run one command, reboot into the result.

### The core files

- **`configuration.nix`** — the root of your system definition. It is a Nix expression that returns an attribute set (Nix's name for "object" / "dict") of options: things like `services.openssh.enable = true;` and `environment.systemPackages = [ pkgs.git ];`. When you run `nixos-rebuild switch`, NixOS reads this file, evaluates it, builds a new system closure, and activates it.

- **`hardware-configuration.nix`** — generated once by `nixos-generate-config` at install time. Contains the bits that are specific to *this particular machine*: detected disks and their UUIDs, kernel modules for your hardware, firmware paths. Usually kept **per-machine** and not shared between machines.

- **Your own modules (optional)** — once `configuration.nix` gets long, split it: `./modules/fish.nix`, `./modules/kde.nix`, `./modules/dev.nix`. Each module is a file that returns the same shape of attribute set. You list them in an `imports = [ ./modules/kde.nix ./modules/dev.nix ];` line at the top of `configuration.nix` and the module system merges them into one big config.

- **`flake.nix` + `flake.lock`** (optional, flakes only) — `flake.nix` declares which external inputs your config depends on (`nixpkgs`, `home-manager`, `disko`, …) and `flake.lock` pins every one of them to an exact commit hash. Together they make your config byte-identical to reproduce anywhere, months or years later.

- **`home.nix`** (optional, home-manager) — your *user-level* config: dotfiles, shell plugins, editor config, GUI apps, themes. Same language, different scope. See the home-manager section below.

### A typical layout after a few weeks

```
~/nixos-config/
├── flake.nix                    ← inputs + outputs (if using flakes)
├── flake.lock                   ← pinned versions, auto-generated
├── configuration.nix            ← the root module
├── hardware-configuration.nix   ← per-machine, usually kept out of shared repo
├── home.nix                     ← home-manager user config
└── modules/
    ├── kde.nix
    ├── dev.nix
    ├── fish.nix
    └── gaming.nix
```

A symlink `/etc/nixos/configuration.nix → ~/nixos-config/configuration.nix` lets `nixos-rebuild` find it without flags. With flakes you don't even need the symlink — `nixos-rebuild switch --flake ~/nixos-config#myhost` points at the directory directly.

### Why this is powerful: git

Because everything is plain text, the whole system fits in a git repo:

- **`git init`** the config directory and you have the full history of every system change. `git log` tells you when you enabled Bluetooth, added Steam, bumped the kernel, changed your keyboard layout.
- **Push to GitHub/GitLab** (private repo, if you prefer) and you have off-site backup of your entire system definition. The repo is small — just a few hundred lines of config text.
- **Rollback via git** is just as effective as rolling back via the NixOS generation menu. `git revert <bad-commit>` + `nixos-rebuild switch` and the bad change is undone, with a commit message explaining why.
- **Branch per host** — if you have a laptop, a desktop, and a server, either keep them as git branches or (cleaner) as separate `nixosConfigurations.laptop` / `nixosConfigurations.desktop` entries in one flake, sharing modules.
- **Clone to install a new machine.** This is the big one. From a fresh NixOS install ISO:

  ```bash
  # boot the ISO, partition, mount root at /mnt, then:
  nix-shell -p git
  git clone https://github.com/you/nixos-config.git /mnt/etc/nixos

  # regenerate hardware-configuration.nix for THIS machine
  nixos-generate-config --root /mnt --dir /mnt/etc/nixos

  # install — classic:
  nixos-install
  # or with flakes:
  nixos-install --flake /mnt/etc/nixos#myhost
  ```

  One reboot later, you are running your exact setup — same KDE theme, same shell, same packages, same fonts, same services — on brand-new hardware.

- **Clone onto a running Linux box over SSH.** [`nixos-anywhere`](tricks.md) takes your flake and a target hostname and turns any Linux machine into your NixOS setup remotely, no ISO needed. Your git repo effectively becomes a zero-click installer.

> On Debian, cloning your setup between machines means syncing `/etc` by hand, running Ansible, or living with drift. On NixOS, your dotfiles and your operating system are the same kind of artifact: a git repo you clone.

---

## The three safe levels of commitment

1. **Nix package manager on Debian** — install packages declaratively for your user, no system risk.
2. **NixOS config in a VM** — build a full NixOS system from `configuration.nix` and boot it as a QEMU VM. Zero impact on the host.
3. **NixOS on metal** — only once the VM workflow feels natural.

You can live on level 1 + 2 for weeks. That is the recommended approach.

---

## Level 1 — Nix package manager on Debian

### Install Nix (multi-user)

```bash
sh <(curl -L https://nixos.org/nix/install) --daemon
```

> The multi-user install uses a systemd daemon and a `/nix` store shared by all users. Prefer it unless you really cannot get root. Reboot or `. /etc/profile` afterwards.

### Try a package without installing

```bash
nix-shell -p cowsay --run "cowsay hello"
```

The `cowsay` binary is fetched from the binary cache, used once, and stays garbage-collectable.

### Install a package for your user (classic)

```bash
nix-env -iA nixpkgs.ripgrep
```

### Install a package for your user (flakes)

```bash
nix profile install nixpkgs#ripgrep
```

> `nix-env` and `nix profile` share the same `~/.nix-profile` but use different metadata. Pick one style and stick to it.

### Enable flakes (optional, recommended long term)

Add to `~/.config/nix/nix.conf`:

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

## Level 2a — home-manager on Debian

`home-manager` manages your user environment (packages, dotfiles, fish, neovim, git config, etc.) from a single `home.nix`. It runs happily on top of Debian — no NixOS required.

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

---

## Level 2b — NixOS config in a VM

This is the killer feature for learning. You write a `configuration.nix`, build a QEMU VM from it, boot the VM, poke around. Edit. Rebuild. Repeat. The host is untouched.

### Prereqs on Debian

```bash
sudo apt install qemu-system-x86 qemu-utils
sudo adduser "$USER" kvm
# log out / log in to pick up the group
```

### Minimal `configuration.nix`

Make a folder, drop this in:

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

  users.users.test = {
    isNormalUser = true;
    password = "test";
    extraGroups = [ "wheel" ];
  };

  environment.systemPackages = with pkgs; [ firefox kate ripgrep ];

  system.stateVersion = "24.11";
}
```

### Build and run (classic)

```bash
nix-channel --add https://nixos.org/channels/nixos-unstable nixos
nix-channel --update
nixos-rebuild build-vm -I nixos-config=./configuration.nix
./result/bin/run-*-vm
```

### Build and run (flakes)

`flake.nix` alongside `configuration.nix`:

```nix
{
  inputs.nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
  outputs = { nixpkgs, ... }: {
    nixosConfigurations.testvm = nixpkgs.lib.nixosSystem {
      system = "x86_64-linux";
      modules = [ ./configuration.nix ];
    };
  };
}
```

Build and run:

```bash
nixos-rebuild build-vm --flake .#testvm
./result/bin/run-*-vm
```

### Iteration loop

1. Edit `configuration.nix`.
2. Re-run the same `nixos-rebuild build-vm ...` command.
3. Relaunch `./result/bin/run-*-vm`.

Subsequent rebuilds only fetch what changed — the binary cache is shared across VM builds and your eventual on-metal install. You are building real NixOS evaluation skills, not a throwaway sandbox.

> The VM's disk is at `./nixos.qcow2` next to `result/`. Delete it when you want a clean boot; keep it to preserve state across runs.

---

## Level 3 — NixOS on metal

When you are ready:

1. Download the NixOS installer ISO from [nixos.org/download](https://nixos.org/download).
2. Boot the ISO, partition, mount root at `/mnt`.
3. `nixos-generate-config --root /mnt` — writes a seed `configuration.nix` and `hardware-configuration.nix`.
4. Edit `/mnt/etc/nixos/configuration.nix` (paste in what you validated in the VM).
5. `nixos-install` — installs the bootloader and first generation.
6. Reboot.

> Keep `configuration.nix` in a git repo from day one. The seed `hardware-configuration.nix` is machine-specific and should stay alongside but is often gitignored.

See [configuration.md](configuration.md) for the `nixos-rebuild` workflow once installed, and [tricks.md](tricks.md) for `nixos-anywhere` if you want to skip the ISO and install from another machine over SSH.

---

## Classic vs flakes — which to learn first

- **Classic (channels + `configuration.nix`)**: imperative channels (`nix-channel --update`) pull the latest nixpkgs revision when you update. Simpler conceptually, no extra syntax. Nothing is pinned by default — reproducibility comes from locking the channel revision yourself.
- **Flakes (`flake.nix` + `flake.lock`)**: inputs are explicit, pinning is automatic via `flake.lock`, composition across machines is easier. Adds the `flake.nix` file and the `nix <subcommand>` CLI to learn.

Throughout this chapter both styles are shown side by side under `### Classic` and `### Flakes` subsections. If you are brand new, read the Classic side first and come back for Flakes once the module system feels comfortable.

## Next

- [configuration.md](configuration.md) — the `configuration.nix` mental model and the `nixos-rebuild` workflow.
- [kde.md](kde.md) — Plasma 6 options.
- [dev-tools.md](dev-tools.md) — git, docker, direnv, IDEs, language toolchains.
- [packages.md](packages.md) — adding packages, unfree, and the Steam/prebuilt-binary problem.
- [personal-setup.md](personal-setup.md) — SSH, firewall, fish, eza/yazi translated from the Debian recipes.
- [snippets.md](snippets.md) — copy-paste recipes.
- [tricks.md](tricks.md) — live USB from your config, VM tricks, `nixos-anywhere`, rollback.
