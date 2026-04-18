# Tricks

The big win of NixOS is that your system is just a function. Once you accept that, you can turn the same `configuration.nix` into a VM for testing, an ISO for rescue, a USB stick for cloning, a cloud image, or push it to a remote server over SSH. This page collects the recipes.

## In this page

- [Test a config as a VM without installing](#test-a-config-as-a-vm-without-installing)
- [`nixos-shell` — lighter VM iteration](#nixos-shell-lighter-vm-iteration)
- [Build any image shape with `nixos-generators`](#build-any-image-shape-with-nixos-generators)
- [Personalized live USB from your own config](#personalized-live-usb-from-your-own-config)
- [Install NixOS remotely over SSH — `nixos-anywhere`](#install-nixos-remotely-over-ssh-nixos-anywhere)
- [Generations, rollback, diff](#generations-rollback-diff)
- [Garbage collection and store optimization](#garbage-collection-and-store-optimization)
- [Explore and introspect](#explore-and-introspect)
- [Lightweight containers — `nixos-container`](#lightweight-containers-nixos-container)
- [Copy closures between machines](#copy-closures-between-machines)
- [Pin the kernel](#pin-the-kernel)
- [Impermanence — wipe `/` on every boot](#impermanence-wipe-on-every-boot)
- [Disaster recovery plan (worth writing down)](#disaster-recovery-plan-worth-writing-down)

## Test a config as a VM without installing

`nixos-rebuild build-vm` evaluates your `configuration.nix` and produces a QEMU VM runner. Perfect for iterating on a config before putting it on metal.

### Classic

```bash
nixos-rebuild build-vm -I nixos-config=./configuration.nix
./result/bin/run-*-vm
```

### Flakes

```bash
nixos-rebuild build-vm --flake .#myhost
./result/bin/run-*-vm
```

> The VM shares the binary cache with your host, so rebuilds after the first one are nearly instant. The VM disk image is `./nixos.qcow2` — delete it for a clean boot, keep it for persistence across runs.

Pass extra QEMU flags:

```bash
QEMU_OPTS="-m 8192 -smp 4 -vga virtio" ./result/bin/run-*-vm
```

Graphics-heavy (e.g. testing KDE with OpenGL):

```bash
QEMU_OPTS="-m 8192 -smp 4 -device virtio-gpu-gl -display gtk,gl=on" ./result/bin/run-*-vm
```

## `nixos-shell` — lighter VM iteration

[`nixos-shell`](https://github.com/Mic92/nixos-shell) gives you an even faster round-trip than `build-vm` — it mounts your home directory into the VM and drops you into a shell. Great for testing a single service.

```bash
nix-shell -p nixos-shell --run 'nixos-shell --flake .#myhost'
```

## Build any image shape with `nixos-generators`

[`nixos-generators`](https://github.com/nix-community/nixos-generators) turns a `configuration.nix` into any of: `iso`, `install-iso`, `qcow`, `raw`, `raw-efi`, `vmware`, `virtualbox`, `amazon`, `proxmox`, `docker`, `lxc`, `sd-aarch64` (Raspberry Pi), and more.

Install it as a shell-only tool:

```bash
nix-shell -p nixos-generators
```

Build an installer ISO with your config embedded:

```bash
nixos-generate -f install-iso -c ./configuration.nix -o result-iso
```

Build a qcow2 VM image:

```bash
nixos-generate -f qcow -c ./configuration.nix -o result-qcow
```

Build a raw disk image for `dd`-ing to a physical disk:

```bash
nixos-generate -f raw-efi -c ./configuration.nix -o result-raw
```

> The `install-iso` variant boots to an installer that will wipe a target disk and install your config non-interactively. The plain `iso` variant is a live-only environment. Pick intentionally.

## Personalized live USB from your own config

The killer recipe: a USB stick that boots any machine straight into a pre-configured environment — your KDE, your shell, your tools, your dotfiles — without touching the machine's disk.

Minimal `live.nix`:

```nix
{ pkgs, modulesPath, lib, ... }:
{
  imports = [
    "${modulesPath}/installer/cd-dvd/installation-cd-minimal.nix"
    "${modulesPath}/installer/cd-dvd/channel.nix"
  ];

  isoImage.isoName = lib.mkForce "nixos-live.iso";

  services.xserver.enable = true;
  services.desktopManager.plasma6.enable = true;
  services.displayManager.sddm.enable = true;
  services.displayManager.autoLogin = {
    enable = true;
    user = "nixos";
  };

  users.users.nixos = {
    isNormalUser = true;
    password = "nixos";
    extraGroups = [ "wheel" "networkmanager" ];
    shell = pkgs.fish;
  };

  programs.fish.enable = true;

  environment.systemPackages = with pkgs; [
    firefox kate partitionmanager
    git neovim tmux ripgrep fd bat
    gparted rsync
  ];

  networking.networkmanager.enable = true;

  # Don't bake in host-specific state
  system.stateVersion = "24.11";
}
```

Build (classic):

```bash
nix-build '<nixpkgs/nixos>' -A config.system.build.isoImage -I nixos-config=./live.nix
# output: ./result/iso/nixos-live.iso
```

Build (flakes):

```nix
# flake.nix
outputs = { nixpkgs, ... }: {
  nixosConfigurations.liveiso = nixpkgs.lib.nixosSystem {
    system = "x86_64-linux";
    modules = [ ./live.nix ];
  };
};
```

```bash
nix build .#nixosConfigurations.liveiso.config.system.build.isoImage
# output: ./result/iso/nixos-live.iso
```

Or via nixos-generators:

```bash
nixos-generate -f iso -c ./live.nix -o result-iso
```

Flash to USB:

```bash
# identify the stick (be careful!)
lsblk

# write — adjust of= to match your USB device (NOT a partition)
sudo dd if=./result/iso/nixos-live.iso of=/dev/sdX bs=4M status=progress conv=fsync
sudo sync
```

> **Be absolutely sure** `of=/dev/sdX` is the USB stick and not your internal disk. `dd` will happily overwrite anything. Double-check with `lsblk -o NAME,SIZE,MODEL,TRAN` — USB transports show `usb`.

Boot any machine with the stick: your desktop, your tools, your shell. Read-only system, writable `/tmp` and user home (in tmpfs).

## Install NixOS remotely over SSH — `nixos-anywhere`

[`nixos-anywhere`](https://github.com/nix-community/nixos-anywhere) installs NixOS onto any Linux box you can SSH into as root — cloud VM, VPS, a running Debian machine, a Raspberry Pi, anything. It wipes the target and replaces it with your flake's config. No ISO or physical access needed.

```bash
nix run github:nix-community/nixos-anywhere -- \
  --flake .#myhost \
  root@target.example.com
```

> `nixos-anywhere` will erase the target's disk. For an existing Debian machine, this means nuking it from orbit — use only when that is the intent.

It also drives `disko` for declarative partitioning, so your disk layout is part of the config too. Repo link for examples: [github.com/nix-community/nixos-anywhere](https://github.com/nix-community/nixos-anywhere).

## Generations, rollback, diff

List all system generations:

```bash
sudo nix-env --list-generations -p /nix/var/nix/profiles/system
```

Roll back one:

```bash
sudo nixos-rebuild switch --rollback
```

Activate a specific generation:

```bash
sudo /nix/var/nix/profiles/system-42-link/bin/switch-to-configuration switch
```

Boot an older generation: on the GRUB/systemd-boot menu at boot time, pick the "NixOS - Configuration N" entry. This always works, even if your current system is completely broken.

Diff two generations (what actually changed):

```bash
nix-shell -p nvd --run 'nvd diff /nix/var/nix/profiles/system-41-link /nix/var/nix/profiles/system-42-link'
```

Or the built-in:

```bash
nix store diff-closures \
  /nix/var/nix/profiles/system-41-link \
  /nix/var/nix/profiles/system-42-link
```

Preview a pending rebuild before applying:

```bash
sudo nixos-rebuild dry-build     # what would be built/fetched
sudo nixos-rebuild dry-activate  # what services would restart
sudo nixos-rebuild build         # build, don't activate
nvd diff /run/current-system ./result
```

## Garbage collection and store optimization

One-shot:

```bash
# remove old profiles
sudo nix-collect-garbage -d
# only keep roots from the last 14 days
sudo nix-collect-garbage --delete-older-than 14d
# dedupe store paths by hardlinking identical files
sudo nix-store --optimise
```

Declarative (recommended):

```nix
nix.gc = {
  automatic = true;
  dates = "weekly";
  options = "--delete-older-than 14d";
};
nix.optimise = { automatic = true; dates = [ "weekly" ]; };
```

> Aggressive GC (`-d`) deletes every generation that isn't the current one, including your rollback options. Prefer `--delete-older-than` for day-to-day.

## Explore and introspect

```bash
# current value of a specific option
nixos-option services.openssh.enable

# everything nixpkgs exposes, REPL-style
nix repl '<nixpkgs>'
# then:  nix-repl> pkgs.firefox

# every module option visible interactively
nix repl '<nixpkgs/nixos>'

# explore a closure's dependencies
nix-shell -p nix-tree --run 'nix-tree /run/current-system'

# find which package ships a specific binary
nix-locate bin/ffprobe
```

## Lightweight containers — `nixos-container`

Spin up full NixOS systems as systemd-nspawn containers on the same host. Much lighter than VMs, useful for testing services in isolation — a real NixOS inside, sharing the host's kernel.

```nix
containers.adguard = {
  autoStart = true;
  privateNetwork = true;
  hostAddress  = "10.42.0.1";
  localAddress = "10.42.0.2";

  config = { config, pkgs, ... }: {
    services.adguardhome = {
      enable = true;
      openFirewall = true;
      settings = {
        bind_host = "0.0.0.0";
        bind_port = 3000;
      };
    };
    system.stateVersion = "24.11";
  };
};
```

Real, packaged services work here: `services.adguardhome`, `services.nginx`, `services.postgresql`, `services.grafana`, `services.jellyfin` — anything with a NixOS module. Perfect for building a declarative homelab service on a real host without polluting the host's service list.

Manage them:

```bash
sudo nixos-container list
sudo nixos-container start adguard
sudo nixos-container root-login adguard
sudo nixos-container update adguard       # re-evaluate + restart after editing the host's configuration.nix
sudo nixos-container stop adguard
sudo nixos-container destroy adguard
```

> Each container is a full NixOS evaluation — it gets its own `/etc/nixos/configuration.nix` derivation, its own generations, its own journal. `systemctl status systemd-nspawn@adguard.service` on the host shows it.

## Copy closures between machines

```bash
# push your entire current system to a remote host's store
nix copy --to ssh://root@target /run/current-system

# pull a build result from a remote builder
nix copy --from ssh://builder.example.com /nix/store/<hash>-thing
```

Combined with `nixos-rebuild --target-host root@...`, you can build on a fast machine and activate on a slow one:

```bash
nixos-rebuild switch --flake .#myhost --target-host root@myhost --use-remote-sudo
```

## Pin the kernel

```nix
boot.kernelPackages = pkgs.linuxPackages_6_6;    # LTS
# or:
boot.kernelPackages = pkgs.linuxPackages_latest;
# or pin to a specific:
boot.kernelPackages = pkgs.linuxPackagesFor (pkgs.linux_6_6.override {
  # ...
});
```

Limit boot menu entries so you don't accumulate:

```nix
boot.loader.systemd-boot.configurationLimit = 10;
boot.loader.grub.configurationLimit = 10;
```

## Impermanence — wipe `/` on every boot

Advanced but transformative: tmpfs-root so every boot is clean, with explicit `environment.persistence."/persist"` bind-mounts for the things you *do* want to keep. Forces you to be honest about what state actually matters. Full walkthrough in [impermanence.md](impermanence.md), and the partitioning that goes with it in [disko.md](disko.md).

## Disaster recovery plan (worth writing down)

1. Keep `configuration.nix` (and `flake.lock` if flakes) in a private git repo.
2. Build a personalized live USB (see above) and keep one physical copy.
3. Know how to boot an older generation from the bootloader menu.
4. Test `nixos-anywhere` against a throwaway VM so you are comfortable with it.

With those four, "reinstalling" becomes: boot live USB → `nix-shell -p nixos-anywhere` → point it at the target disk → done. 15 minutes instead of a weekend.

## See also

- [getting-started.md](getting-started.md) — the VM-first learning loop.
- [configuration.md](configuration.md) — `nixos-rebuild` subcommands.
- [disko.md](disko.md) — declarative partitioning (required for `nixos-anywhere` and impermanence).
- [impermanence.md](impermanence.md) — wipe-on-boot root + `/persist` bind-mounts.
- [secrets.md](secrets.md) — agenix / sops-nix for values that shouldn't live in the store.
- [packages.md](packages.md) — for Flatpak and Steam alongside these tricks.
