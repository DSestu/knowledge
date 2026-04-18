# Snippets

Short copy-paste recipes — split into three sections so you always know where each one goes.

## Where each snippet belongs

Three scopes. Each H2 below tells you which one applies.

- **`configuration.nix`** — system-wide. Services, users, hardware, firewall, packages for everyone. Needs `sudo nixos-rebuild switch` to apply.
- **`home.nix`** — your user. Shell plugins, dotfiles, editor, theme, git identity, personal aliases. Needs `home-manager switch` (or, if integrated with the NixOS flake, just rebuilds with it).
- **Shell commands** — run in a terminal, not pasted into a config file.

If you've split your system config into `./modules/*.nix`, treat them as `configuration.nix`-equivalent — the module system merges everything.

> This page is a **quick reference**. Fuller treatments live elsewhere:
>
> - Module system, flakes, `nixos-rebuild` variants → [configuration.md](configuration.md)
> - Home-manager modes (standalone vs NixOS module) and `targets.genericLinux.enable` → [home-manager-modes.md](home-manager-modes.md)
> - Multiple hosts in one flake, `mkHost` helper → [multi-host-flake.md](multi-host-flake.md)
> - Declarative partitioning (disko) → [disko.md](disko.md)
> - Secrets (agenix / sops-nix) → [secrets.md](secrets.md)
> - Wipe-on-boot (impermanence) → [impermanence.md](impermanence.md)
> - User-level recipes (fish, git, ssh client, fleet patterns) → [personal-setup.md](personal-setup.md)
> - Packaging non-Nix binaries (`nix-ld`, `autoPatchelfHook`, AppImages, FHS) → [packages.md](packages.md)

## In this page

**[Command snippets (shell)](#command-snippets-shell)**

- [Search for packages and options](#search-for-packages-and-options)
- [Try a package without installing](#try-a-package-without-installing)
- [Install / uninstall for your user (imperative)](#install-uninstall-for-your-user-imperative)
- [Rebuild the system](#rebuild-the-system)
- [Generations and rollback](#generations-and-rollback)
- [Update channels and flake inputs](#update-channels-and-flake-inputs)
- [Garbage collection and store optimization](#garbage-collection-and-store-optimization)
- [Per-project dev shells](#per-project-dev-shells)
- [Home-manager](#home-manager)
- [Ephemeral unfree / insecure](#ephemeral-unfree-insecure)
- [Build images from a config](#build-images-from-a-config)
- [Remote install / remote build](#remote-install-remote-build)
- [Diagnose what's going on](#diagnose-whats-going-on)

**[System config snippets — `configuration.nix`](#system-config-snippets-configuration-nix)**

- [Users and groups](#users-and-groups)
- [Locale, time, keymap](#locale-time-keymap)
- [Networking](#networking)
- [OpenSSH server](#openssh-server)
- [Tailscale](#tailscale)
- [WireGuard (client)](#wireguard-client)
- [Fish shell — system-level enable](#fish-shell-system-level-enable)
- [Direnv — system-level enable](#direnv-system-level-enable)
- [Docker (rootless)](#docker-rootless)
- [Libvirt / virt-manager](#libvirt-virt-manager)
- [PipeWire audio](#pipewire-audio)
- [Bluetooth](#bluetooth)
- [Printing (CUPS)](#printing-cups)
- [Nvidia proprietary](#nvidia-proprietary)
- [AMD GPU](#amd-gpu)
- [Intel GPU (hardware acceleration)](#intel-gpu-hardware-acceleration)
- [Power management (laptop)](#power-management-laptop)
- [Trim for SSD](#trim-for-ssd)
- [Swap file (hibernate-capable)](#swap-file-hibernate-capable)
- [Automatic garbage collection and store optimization](#automatic-garbage-collection-and-store-optimization)
- [Automatic upgrades](#automatic-upgrades)
- [Binary caches](#binary-caches)
- [Declarative disk mounts](#declarative-disk-mounts)
- [Flatpak](#flatpak)
- [Fonts](#fonts)
- [Secrets management (pointer)](#secrets-management-pointer)
- [Quick tweaks](#quick-tweaks)

**[User config snippets — `home.nix`](#user-config-snippets-home-nix-home-manager)**

- [Fish shell](#fish-shell)
- [Starship prompt](#starship-prompt)
- [Git](#git)
- [Direnv (user integration)](#direnv-user-integration)
- [Eza (listing + theme)](#eza-listing-theme)
- [Yazi (file manager + `y` wrapper)](#yazi-file-manager-y-wrapper)
- [SSH client (`~/.ssh/config` declaratively)](#ssh-client-ssh-config-declaratively)

---

# Command snippets (shell)

## Search for packages and options

```bash
# find a package
nix search nixpkgs firefox

# show the current value of any option
nixos-option services.openssh.enable

# find which package provides a binary (needs nix-index + database)
nix-locate bin/ffprobe

# explore nixpkgs interactively
nix repl '<nixpkgs>'
# nix-repl> pkgs.firefox.meta

# browse every NixOS module option interactively
nix repl '<nixpkgs/nixos>'
```

## Try a package without installing

```bash
# classic — drop into a shell with the package on PATH
nix-shell -p ripgrep

# classic — run a single command
nix-shell -p cowsay --run "cowsay hello"

# flakes — run a single command
nix run nixpkgs#cowsay -- hello

# flakes — shell with multiple packages
nix shell nixpkgs#ripgrep nixpkgs#fd nixpkgs#bat
```

## Install / uninstall for your user (imperative)

```bash
# classic
nix-env -iA nixpkgs.ripgrep
nix-env -e ripgrep
nix-env -q                   # list installed

# flakes
nix profile install nixpkgs#ripgrep
nix profile remove ripgrep
nix profile list
```

> Prefer `home.packages` in `home.nix` (declarative) over either of the above. Imperative profiles drift; declarative ones don't.

## Rebuild the system

```bash
sudo nixos-rebuild switch                             # build + activate + set default boot
sudo nixos-rebuild test                               # build + activate, no boot entry
sudo nixos-rebuild boot                               # build + set next boot, don't activate now
sudo nixos-rebuild build                              # build only, symlink ./result
sudo nixos-rebuild dry-build                          # what would be built
sudo nixos-rebuild dry-activate                       # what services would restart

# flakes variant
sudo nixos-rebuild switch --flake /etc/nixos#myhost

# also update channels (classic) / inputs (flakes)
sudo nixos-rebuild switch --upgrade
```

## Generations and rollback

```bash
# list every system generation
sudo nix-env --list-generations -p /nix/var/nix/profiles/system

# roll back to the previous generation
sudo nixos-rebuild switch --rollback

# jump to a specific generation N
sudo /nix/var/nix/profiles/system-42-link/bin/switch-to-configuration switch

# diff two generations
nix-shell -p nvd --run 'nvd diff /nix/var/nix/profiles/system-41-link /nix/var/nix/profiles/system-42-link'
nix store diff-closures /nix/var/nix/profiles/system-41-link /nix/var/nix/profiles/system-42-link
```

## Update channels and flake inputs

```bash
# classic
sudo nix-channel --update

# flakes — bump every input
sudo nix flake update --flake /etc/nixos

# flakes — bump only nixpkgs (Nix 2.19+: positional form)
sudo nix flake update --flake /etc/nixos nixpkgs

# flakes — show pinned inputs, hashes, and the outputs tree
nix flake metadata /etc/nixos
nix flake show /etc/nixos
```

## Garbage collection and store optimization

```bash
# delete unreferenced store paths older than 14 days
sudo nix-collect-garbage --delete-older-than 14d

# nuke everything except the current generation (aggressive — removes rollback options)
sudo nix-collect-garbage -d

# dedupe store paths by hardlinking identical files
sudo nix-store --optimise

# show what a generation references
nix-shell -p nix-tree --run 'nix-tree /run/current-system'

# store size
du -sh /nix/store
```

## Per-project dev shells

```bash
# classic — read ./shell.nix
nix-shell

# flakes — read ./flake.nix devShells.default
nix develop

# allow direnv (after writing .envrc with `use flake` or `use nix`)
direnv allow
```

## Home-manager

```bash
# classic
home-manager switch

# flakes
home-manager switch --flake ~/nixos-config#alice

# list home-manager generations
home-manager generations
```

## Ephemeral unfree / insecure

```bash
NIXPKGS_ALLOW_UNFREE=1   nix-shell -p spotify --run spotify
NIXPKGS_ALLOW_INSECURE=1 nix-shell -p old-pkg --run old-pkg
```

## Build images from a config

```bash
# a QEMU VM of a config
nixos-rebuild build-vm -I nixos-config=./configuration.nix
./result/bin/run-*-vm

# any image format — iso / qcow / raw / docker / amazon / ...
nix-shell -p nixos-generators --run 'nixos-generate -f iso -c ./configuration.nix -o result-iso'
```

## Remote install / remote build

```bash
# install your flake onto a Linux target over SSH (wipes target disk!)
nix run github:nix-community/nixos-anywhere -- --flake .#myhost root@target

# build locally, activate remotely
nixos-rebuild switch --flake .#myhost --target-host root@myhost --use-remote-sudo

# copy a closure between machines
nix copy --to ssh://root@target /run/current-system
```

## Diagnose what's going on

```bash
# services
systemctl status
journalctl -b                          # boot log
journalctl -u sshd -f                  # follow one service

# where does this binary come from?
which firefox
readlink -f "$(which firefox)"         # lands in /nix/store/<hash>-...

# store integrity check
sudo nix-store --verify --check-contents

# nix daemon state
systemctl status nix-daemon
```

---

# System config snippets — `configuration.nix`

## Users and groups

```nix
users.users.alice = {
  isNormalUser = true;
  description = "Alice";
  extraGroups = [ "wheel" "networkmanager" "video" "audio" "docker" "libvirtd" "kvm" ];
  shell = pkgs.fish;
  openssh.authorizedKeys.keys = [
    "ssh-ed25519 AAAAC3Nz... alice@laptop"
  ];
};

# Allow wheel users to sudo without a password (desktop only!)
security.sudo.wheelNeedsPassword = false;
```

## Locale, time, keymap

```nix
time.timeZone = "Europe/Paris";

i18n.defaultLocale = "en_US.UTF-8";
i18n.extraLocaleSettings = {
  LC_ADDRESS = "fr_FR.UTF-8";
  LC_MEASUREMENT = "fr_FR.UTF-8";
  LC_MONETARY = "fr_FR.UTF-8";
  LC_PAPER = "fr_FR.UTF-8";
  LC_TIME = "fr_FR.UTF-8";
};

console.keyMap = "fr";
services.xserver.xkb = { layout = "fr"; options = "ctrl:nocaps"; };
```

## Networking

```nix
networking.hostName = "nixos";
networking.networkmanager.enable = true;
networking.firewall.enable = true;
networking.firewall.allowedTCPPorts = [ 22 ];
networking.firewall.allowedUDPPorts = [ ];
```

## OpenSSH server

```nix
services.openssh = {
  enable = true;
  openFirewall = true;
  settings = {
    PasswordAuthentication = false;
    PermitRootLogin = "no";
    KbdInteractiveAuthentication = false;
  };
};
```

> Fuller version, including client config and per-interface restrictions, in [personal-setup.md](personal-setup.md).

## Tailscale

```nix
services.tailscale.enable = true;
networking.firewall.trustedInterfaces = [ "tailscale0" ];
# Then once: `sudo tailscale up`
```

## WireGuard (client)

```nix
networking.wg-quick.interfaces.wg0 = {
  address = [ "10.0.0.2/24" ];
  privateKeyFile = "/etc/wireguard/privatekey";
  peers = [{
    publicKey = "PEER_PUBLIC_KEY=";
    allowedIPs = [ "10.0.0.0/24" ];
    endpoint = "vpn.example.com:51820";
    persistentKeepalive = 25;
  }];
};
```

## Fish shell — system-level enable

```nix
# Required so fish is in /etc/shells and completions are installed.
# The fuller user config lives in home.nix — see personal-setup.md.
programs.fish.enable = true;
users.users.alice.shell = pkgs.fish;
```

## Direnv — system-level enable

```nix
# Also required for `use flake` / `use nix` to work reliably.
programs.direnv = {
  enable = true;
  nix-direnv.enable = true;
  loadInNixShell = true;
};
```

## Docker (rootless)

```nix
virtualisation.docker = {
  enable = true;
  rootless = {
    enable = true;
    setSocketVariable = true;
  };
  autoPrune.enable = true;
};

users.users.alice.extraGroups = [ "docker" ];
```

## Libvirt / virt-manager

```nix
virtualisation.libvirtd = {
  enable = true;
  qemu = {
    package = pkgs.qemu_kvm;
    swtpm.enable = true;
    ovmf.enable = true;
  };
};
programs.virt-manager.enable = true;
users.users.alice.extraGroups = [ "libvirtd" "kvm" ];
```

## PipeWire audio

```nix
services.pipewire = {
  enable = true;
  alsa.enable = true;
  alsa.support32Bit = true;
  pulse.enable = true;
  jack.enable = true;
};
services.pulseaudio.enable = false;
security.rtkit.enable = true;
```

## Bluetooth

```nix
hardware.bluetooth = {
  enable = true;
  powerOnBoot = true;
  settings.General.Experimental = true;  # needed for some headsets' battery reporting
};
services.blueman.enable = true;   # system tray applet
```

## Printing (CUPS)

```nix
services.printing = {
  enable = true;
  drivers = with pkgs; [ gutenprint hplip cnijfilter2 ];
};
services.avahi = {
  enable = true;
  nssmdns4 = true;
  openFirewall = true;
};
```

## Nvidia proprietary

```nix
services.xserver.videoDrivers = [ "nvidia" ];
hardware.graphics = {
  enable = true;
  enable32Bit = true;   # needed for Steam / 32-bit games
};
hardware.nvidia = {
  modesetting.enable = true;
  powerManagement.enable = true;
  open = false;                        # use proprietary kernel module (more features)
  nvidiaSettings = true;
  package = config.boot.kernelPackages.nvidiaPackages.stable;
};
```

## AMD GPU

```nix
hardware.graphics = {
  enable = true;
  enable32Bit = true;
  extraPackages = with pkgs; [ amdvlk rocmPackages.clr.icd ];
  extraPackages32 = with pkgs.pkgsi686Linux; [ amdvlk ];
};
services.xserver.videoDrivers = [ "amdgpu" ];
```

## Intel GPU (hardware acceleration)

```nix
hardware.graphics = {
  enable = true;
  enable32Bit = true;
  extraPackages = with pkgs; [ intel-media-driver vaapiIntel vaapiVdpau libvdpau-va-gl ];
};
```

## Power management (laptop)

Simpler option:

```nix
services.power-profiles-daemon.enable = true;
```

More aggressive (TLP — mutually exclusive with power-profiles-daemon):

```nix
services.power-profiles-daemon.enable = false;
services.tlp = {
  enable = true;
  settings = {
    CPU_SCALING_GOVERNOR_ON_AC = "performance";
    CPU_SCALING_GOVERNOR_ON_BAT = "powersave";
    CPU_BOOST_ON_BAT = 0;
    START_CHARGE_THRESH_BAT0 = 40;
    STOP_CHARGE_THRESH_BAT0 = 80;
  };
};
```

## Trim for SSD

```nix
services.fstrim.enable = true;
```

## Swap file (hibernate-capable)

```nix
swapDevices = [{ device = "/swapfile"; size = 16 * 1024; }];  # 16 GiB

boot.resumeDevice = "/dev/disk/by-uuid/<ROOT-UUID>";
boot.kernelParams = [ "resume_offset=<OFFSET>" ];
# resume_offset from: sudo filefrag -v /swapfile | awk 'NR==4 {gsub(/\./,""); print $4}'
```

## Automatic garbage collection and store optimization

```nix
nix = {
  gc = {
    automatic = true;
    dates = "weekly";
    options = "--delete-older-than 14d";
  };
  optimise = {
    automatic = true;
    dates = [ "weekly" ];
  };
  settings = {
    auto-optimise-store = true;
    experimental-features = [ "nix-command" "flakes" ];
  };
};

# Limit boot entries so GRUB/systemd-boot stays readable
boot.loader.systemd-boot.configurationLimit = 10;
```

## Automatic upgrades

```nix
system.autoUpgrade = {
  enable = true;
  allowReboot = false;          # set true on servers
  dates = "04:00";
  flake = "/etc/nixos";         # or omit for classic channels
  # Nix 2.19+: positional input names; --commit-lock-file keeps the lock under git.
  flags = [ "--update-input" "nixpkgs" "--commit-lock-file" ];
  # If your Nix is 2.19+ and you want only the new syntax, use:
  #   flags = [ "--commit-lock-file" ];
  # and run `nix flake update nixpkgs` from a timer or hook instead.
};
```

## Binary caches

```nix
nix.settings = {
  substituters = [
    "https://cache.nixos.org"
    "https://nix-community.cachix.org"
    "https://devenv.cachix.org"
  ];
  trusted-public-keys = [
    "cache.nixos.org-1:6NCHdD59X431o0gWypbMrAURkbJ16ZPMQFGspcDShjY="
    "nix-community.cachix.org-1:mB9FSh9qf2dCimDSUo8Zy7bkq5CX+/rkCWyvRCYg3Fs="
    "devenv.cachix.org-1:w1cLUi8dv3hnoSPGAuibQv+f9TZLr6cv/Hm9XgU50cw="
  ];
};
```

## Declarative disk mounts

For one-off extra mounts on an already-partitioned host. If you want the *root disk layout* itself declared in your flake (partitioning, LUKS, subvolumes, encrypted swap), use [disko.md](disko.md) instead.

```nix
fileSystems."/data" = {
  device = "/dev/disk/by-uuid/AAAAAAAA-BBBB-CCCC-DDDD-EEEEEEEEEEEE";
  fsType = "ext4";
  options = [ "noatime" "nofail" ];
};

fileSystems."/mnt/nas" = {
  device = "//nas.local/share";
  fsType = "cifs";
  options = let
    auto = "x-systemd.automount,noauto,x-systemd.idle-timeout=60";
  in [ "${auto},credentials=/etc/nixos/smb-secrets,uid=1000,gid=100" ];
};
```

## Flatpak

```nix
services.flatpak.enable = true;
xdg.portal = {
  enable = true;
  extraPortals = [ pkgs.kdePackages.xdg-desktop-portal-kde ];
};
# One-time on the user's side:
#   flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
```

## Fonts

```nix
fonts = {
  enableDefaultPackages = true;
  packages = with pkgs; [
    noto-fonts noto-fonts-emoji noto-fonts-cjk-sans
    liberation_ttf
    fira-code jetbrains-mono
    nerd-fonts.fira-code nerd-fonts.jetbrains-mono
    font-awesome
  ];
  fontconfig.defaultFonts = {
    serif     = [ "Noto Serif" ];
    sansSerif = [ "Noto Sans" ];
    monospace = [ "JetBrains Mono" ];
    emoji     = [ "Noto Color Emoji" ];
  };
};
```

## Secrets management (pointer)

Don't put secrets in `configuration.nix` — `/nix/store` is world-readable. The full treatment (threat model, agenix vs sops-nix walkthrough, home-manager scope, bootstrap on a fresh install, impermanence integration) lives in [secrets.md](secrets.md). Minimal sops-nix example for reference:

```nix
# flake input
sops-nix.url = "github:Mic92/sops-nix";

# module
{
  imports = [ inputs.sops-nix.nixosModules.sops ];
  sops.defaultSopsFile = ./secrets/secrets.yaml;
  sops.age.keyFile = "/var/lib/sops-nix/key.txt";
  sops.secrets."wireguard/private" = { };

  networking.wg-quick.interfaces.wg0.privateKeyFile =
    config.sops.secrets."wireguard/private".path;
}
```

## Quick tweaks

```nix
# Shorter hostname in shell prompts
networking.hostName = "nixos";

# Set timezone automatically via geoip
services.geoclue2.enable = true;
services.automatic-timezoned.enable = true;

# System-wide editor
environment.variables.EDITOR = "nvim";
environment.variables.VISUAL = "nvim";

# Mouse on tty
services.gpm.enable = true;

# NTP
services.timesyncd.enable = true;

# journald retention
services.journald.extraConfig = ''
  SystemMaxUse=500M
  MaxRetentionSec=1month
'';
```

---

# User config snippets — `home.nix` (home-manager)

These are short quick-reference versions. Fuller recipes with plugin lists, key bindings, and aliases live in [personal-setup.md](personal-setup.md).

## Fish shell

```nix
programs.fish = {
  enable = true;
  shellAbbrs = {
    g  = "git";
    gs = "git status";
    gp = "git pull --rebase";
  };
  shellAliases = {
    l = "eza --icons --git --group-directories-first";
  };
  interactiveShellInit = ''
    set -U fish_greeting ""
    bind \cH backward-kill-word
  '';
};
```

## Starship prompt

```nix
programs.starship = {
  enable = true;
  enableFishIntegration = true;
  settings = {
    add_newline = false;
    format = "$directory$git_branch$git_status$character";
  };
};
```

## Git

```nix
programs.git = {
  enable = true;
  userName  = "Alice Example";
  userEmail = "alice@example.com";
  extraConfig = {
    init.defaultBranch = "main";
    pull.rebase = true;
    push.autoSetupRemote = true;
    rerere.enabled = true;
  };
  aliases = {
    st = "status";
    co = "checkout";
    lg = "log --oneline --graph --decorate";
  };
};
```

## Direnv (user integration)

```nix
programs.direnv = {
  enable = true;
  nix-direnv.enable = true;
  enableFishIntegration = true;
};
```

## Eza (listing + theme)

```nix
programs.eza = {
  enable = true;
  enableFishIntegration = true;
  git = true;
  icons = "auto";
  extraOptions = [ "--group-directories-first" "--header" ];
};

# theme file alongside home.nix
xdg.configFile."eza/theme.yml".source = ./files/eza-theme.yml;
```

## Yazi (file manager + `y` wrapper)

```nix
programs.yazi = {
  enable = true;
  enableFishIntegration = true;
  settings = {
    manager = { show_hidden = true; show_symlink = true; };
  };
};
```

## SSH client (`~/.ssh/config` declaratively)

```nix
programs.ssh = {
  enable = true;
  matchBlocks = {
    "github.com" = {
      user = "git";
      identityFile = "~/.ssh/id_ed25519_github";
      identitiesOnly = true;
    };
    "*.internal" = {
      user = "alice";
      identityFile = "~/.ssh/id_ed25519_internal";
    };
  };
  extraConfig = ''
    AddKeysToAgent yes
    ServerAliveInterval 60
  '';
};

services.ssh-agent.enable = true;
home.sessionVariables.SSH_AUTH_SOCK = "$XDG_RUNTIME_DIR/ssh-agent";
```
