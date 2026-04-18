# Multi-host flake — one repo, many machines

[getting-started.md](getting-started.md#your-system-is-a-git-repo-of-text-files) promises "one repo, many hosts". This page is the mechanics of that promise — what `flake.nix` actually looks like when you have a VirtualBox VM, a libvirt VM on Kali, and an external-SSD NixOS all expressed in the same repo, with a `home.nix` that travels everywhere.

Prereq: read [nix-language.md](nix-language.md) if function-and-attribute-set syntax isn't fluent yet; read [home-manager-modes.md](home-manager-modes.md) to understand when we use `homeConfigurations.*` vs `home-manager.users.*` inside a NixOS config.

## In this page

- [The shape of a maturing repo](#the-shape-of-a-maturing-repo)
- [A concrete `flake.nix`](#a-concrete-flakenix)
- [Per-host directory contents](#per-host-directory-contents)
- [Shared modules — the part that pays for itself](#shared-modules-the-part-that-pays-for-itself)
- [`specialArgs` — passing things to modules](#specialargs-passing-things-to-modules)
- [Building and switching each host](#building-and-switching-each-host)
- [`inputs.follows` — avoiding duplicated nixpkgs](#inputsfollows-avoiding-duplicated-nixpkgs)
- [Adding a new host](#adding-a-new-host)
- [homeConfigurations alongside nixosConfigurations](#homeconfigurations-alongside-nixosconfigurations)

## The shape of a maturing repo

```
~/nixos-config/
├── flake.nix
├── flake.lock
├── home.nix                    ← portable user config, works on any host
├── hosts/
│   ├── desktop-vbox/
│   │   ├── configuration.nix        ← VirtualBox VM on Windows host
│   │   └── hardware-configuration.nix
│   ├── kali-vm/
│   │   ├── configuration.nix        ← libvirt VM on Kali host
│   │   └── hardware-configuration.nix
│   └── portable-ssd/
│       ├── configuration.nix        ← external-SSD NixOS, real hardware
│       └── hardware-configuration.nix
└── modules/
    ├── common.nix              ← keyboard, locale, timezone, system packages everyone wants
    ├── dev.nix                 ← git, docker, direnv, language toolchains
    ├── kde.nix                 ← Plasma 6 desktop
    ├── fish.nix                ← fish + prompt + plugins
    └── pentest.nix             ← nmap, metasploit, wireshark, ... (for the Kali-replacement future)
```

Three claims this structure makes, worth reading carefully:

1. **`home.nix` is at the root.** It is the same file for every host — the common user-level config you want identically everywhere.
2. **Each `hosts/<name>/configuration.nix` is thin.** It only declares what makes this host unique (its bootloader style, its hostname, its networking, which shared modules it imports). The bulk lives in `modules/`.
3. **`hardware-configuration.nix` is generated per-host** by `nixos-generate-config` at install time and committed to its host directory. Not shared across hosts.

Smaller repos can collapse this: put `configuration.nix` at the root, keep only one host. But as soon as there are two, the tree above pays off.

## A concrete `flake.nix`

```nix
{
  description = "My portable NixOS + home-manager setup";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";

    home-manager = {
      url = "github:nix-community/home-manager";
      inputs.nixpkgs.follows = "nixpkgs";
    };

    # Optional extras — uncomment as you adopt them.
    # disko.url = "github:nix-community/disko";
    # disko.inputs.nixpkgs.follows = "nixpkgs";

    # impermanence.url = "github:nix-community/impermanence";
  };

  outputs = inputs@{ self, nixpkgs, home-manager, ... }:
    let
      system = "x86_64-linux";

      # Factor out the common bits of a nixosSystem invocation so each host is a one-liner.
      mkHost = hostName: extraModules:
        nixpkgs.lib.nixosSystem {
          inherit system;
          specialArgs = { inherit inputs; };
          modules = [
            ./hosts/${hostName}/configuration.nix
            ./modules/common.nix
            home-manager.nixosModules.home-manager
            {
              networking.hostName = hostName;
              home-manager.useGlobalPkgs = true;
              home-manager.useUserPackages = true;
              home-manager.users.alice = import ./home.nix;
            }
          ] ++ extraModules;
        };
    in
    {
      nixosConfigurations = {
        desktop-vbox = mkHost "desktop-vbox" [
          ./modules/kde.nix
          ./modules/dev.nix
          ./modules/fish.nix
        ];

        kali-vm = mkHost "kali-vm" [
          ./modules/dev.nix
          ./modules/fish.nix
          # no KDE — this VM runs headless, accessed over SSH
        ];

        portable-ssd = mkHost "portable-ssd" [
          ./modules/kde.nix
          ./modules/dev.nix
          ./modules/fish.nix
          # Future: ./modules/pentest.nix
        ];
      };

      # Also expose a standalone home-manager config, for hosts that aren't NixOS (Kali, WSL2-Debian).
      homeConfigurations."alice" = home-manager.lib.homeManagerConfiguration {
        pkgs = import nixpkgs { inherit system; config.allowUnfree = true; };
        modules = [ ./home.nix ];
      };
    };
}
```

Worth calling out:

- **`mkHost`** is a tiny local helper that captures "the same five lines I'd otherwise repeat three times". It's plain Nix — no magic.
- **`specialArgs = { inherit inputs; };`** passes the whole `inputs` attribute set down to every module as the `inputs` argument. Modules can then do `inputs.impermanence.nixosModules.impermanence` without re-importing the flake.
- **`networking.hostName`** is set in the flake, not in the host's `configuration.nix`. This avoids the classic "I copied the VM and forgot to change the hostname" bug.
- **`homeConfigurations."alice"`** coexists with the NixOS configs. On Kali and WSL2-Debian you call `home-manager switch --flake ~/nixos-config#alice`; on the NixOS hosts the home-manager module handles it automatically as part of `nixos-rebuild switch`. Same `home.nix`, two entry points.

## Per-host directory contents

A typical host's `configuration.nix` stays small — the heavy lifting is in `modules/`. Example for `hosts/desktop-vbox/configuration.nix`:

```nix
{ config, pkgs, lib, ... }:
{
  imports = [ ./hardware-configuration.nix ];

  # Bootloader (VirtualBox uses BIOS by default unless EFI is enabled in the VM settings)
  boot.loader.grub = {
    enable = true;
    device = "/dev/sda";
    useOSProber = false;
  };

  # VirtualBox Guest Additions for seamless clipboard, resize, shared folders
  virtualisation.virtualbox.guest.enable = true;

  # Users
  users.users.alice = {
    isNormalUser = true;
    description = "Alice";
    extraGroups = [ "wheel" "networkmanager" "video" "docker" "vboxsf" ];
    shell = pkgs.fish;
  };

  system.stateVersion = "24.11";
}
```

The host file says "this machine's name, its bootloader, its virtualization-guest quirks, who its user is". Everything else — KDE, dev tools, fish plugins — comes from the shared modules imported at the flake level.

## Shared modules — the part that pays for itself

`modules/common.nix` — the lowest common denominator that every host wants:

```nix
{ pkgs, ... }:
{
  time.timeZone = "Europe/Paris";
  i18n.defaultLocale = "en_US.UTF-8";
  console.keyMap = "fr";

  networking.networkmanager.enable = true;
  nixpkgs.config.allowUnfree = true;

  environment.systemPackages = with pkgs; [
    git vim wget curl htop
  ];

  nix.settings = {
    experimental-features = [ "nix-command" "flakes" ];
    auto-optimise-store = true;
  };

  nix.gc = {
    automatic = true;
    dates = "weekly";
    options = "--delete-older-than 30d";
  };

  system.stateVersion = "24.11";
}
```

`modules/dev.nix`, `modules/kde.nix`, etc. follow the same shape. A host opts in by including the module in its `mkHost` invocation — nothing more.

Two rules of thumb that keep the modules composable:

- **A module does one thing.** If `modules/dev.nix` ends up also enabling Bluetooth, split it.
- **A module never assumes it is the only module.** Always extend with `++` and `//`, never replace. `lib.mkDefault` / `lib.mkForce` (see [nix-language.md](nix-language.md#lib-and-common-helpers)) exist for exactly this.

## `specialArgs` — passing things to modules

By default, modules are called with `{ config, pkgs, lib, options, ... }`. `specialArgs` extends that set with your own additions, available as destructurable arguments inside every module.

```nix
mkHost = hostName: extraModules:
  nixpkgs.lib.nixosSystem {
    inherit system;
    specialArgs = {
      inherit inputs;
      myFlags = {
        isLaptop = false;
        hasNvidia = false;
      };
    };
    # ...
  };
```

Inside any module:

```nix
{ pkgs, inputs, myFlags, ... }:
{
  imports = [ inputs.impermanence.nixosModules.impermanence ];

  hardware.nvidia.enable = lib.mkIf myFlags.hasNvidia true;
}
```

Avoid overloading `specialArgs` — anything more than `inputs` and a small flag set belongs in a proper option defined in a module. Keep the indirection budget low.

## Building and switching each host

### From the host itself

```bash
# Classic nixos-rebuild, flake-aware:
sudo nixos-rebuild switch --flake ~/nixos-config#desktop-vbox

# Build only, inspect result:
nixos-rebuild build --flake ~/nixos-config#desktop-vbox
ls -la ./result

# Build a VM to try a change before switching:
nixos-rebuild build-vm --flake ~/nixos-config#desktop-vbox
./result/bin/run-*-vm
```

### From another machine (remote build)

`--target-host` and `--build-host` invert who builds and who activates:

```bash
# Build locally on Kali, activate remotely on desktop-vbox over SSH:
nixos-rebuild switch \
  --flake ~/nixos-config#desktop-vbox \
  --target-host root@desktop-vbox.local \
  --use-remote-sudo

# Build remotely on a beefy server, activate on this machine:
nixos-rebuild switch \
  --flake ~/nixos-config#portable-ssd \
  --build-host user@fastbox.example.com
```

This is how you deploy from a laptop to a server without uploading the whole flake — `nixos-rebuild` copies only what's needed over SSH.

### For a non-NixOS host (Kali, WSL2-Debian)

```bash
home-manager switch --flake ~/nixos-config#alice
```

No system changes — only `home.nix` applies. Same flake, same commit, different entry point.

## `inputs.follows` — avoiding duplicated nixpkgs

Every input in your flake that itself depends on nixpkgs brings its own copy. Without intervention, a flake that uses nixpkgs + home-manager + disko + impermanence can drag in **four** different revisions of nixpkgs.

`inputs.*.inputs.nixpkgs.follows = "nixpkgs"` overrides each input's nixpkgs to match yours:

```nix
inputs = {
  nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";

  home-manager = {
    url = "github:nix-community/home-manager";
    inputs.nixpkgs.follows = "nixpkgs";
  };

  disko = {
    url = "github:nix-community/disko";
    inputs.nixpkgs.follows = "nixpkgs";
  };

  impermanence.url = "github:nix-community/impermanence";  # no nixpkgs dep, no follows needed
};
```

After these `follows`, every input resolves the same nixpkgs. `flake.lock` shows it: no `nixpkgs_2`, `nixpkgs_3`, just a single `nixpkgs` entry.

## Adding a new host

The reproducible recipe, the nth time you do this:

1. Boot the target machine (ISO, live USB, or an existing Linux host via `nixos-anywhere`).
2. Partition, mount root at `/mnt`.
3. Generate the machine-specific config:

   ```bash
   sudo nixos-generate-config --root /mnt --dir /mnt/etc/nixos/hosts/new-host
   ```

4. Clone your flake to the target:

   ```bash
   git clone git@github.com:you/nixos-config.git /mnt/etc/nixos
   # move the just-generated files into place
   mv /mnt/etc/nixos/hosts/new-host/configuration.nix /mnt/etc/nixos/hosts/new-host/configuration.generated.nix
   # write a short configuration.nix that imports hardware-configuration.nix and sets the bootloader, like the hosts/desktop-vbox example above
   ```

5. Add the host to `flake.nix`:

   ```nix
   nixosConfigurations.new-host = mkHost "new-host" [
     ./modules/kde.nix
     ./modules/dev.nix
   ];
   ```

6. Commit, then install:

   ```bash
   sudo nixos-install --flake /mnt/etc/nixos#new-host
   ```

7. Reboot. `home-manager switch --flake ~/nixos-config#alice` runs automatically as part of `nixos-rebuild` because the home-manager module is in the flake.

## homeConfigurations alongside nixosConfigurations

A common layout worth naming explicitly. Your flake exposes:

- `nixosConfigurations.<host>` — the full system, for NixOS hosts.
- `homeConfigurations."<user>"` — the user-only config, for non-NixOS hosts.

Both pull the same `./home.nix`. On a NixOS host, the NixOS module runs home-manager as part of `nixos-rebuild`; on Kali, the standalone CLI uses `homeConfigurations.alice`; both produce the same user environment.

### The `"alice@desktop-vbox"` pattern — multiple homes per user

If your `home.nix` grows enough that you want per-host variants (e.g. a more aggressive editor setup on the dev VM, a lighter one on the pentest box), expose multiple homes:

```nix
homeConfigurations = {
  "alice" = ...;                     # default, portable
  "alice@desktop-vbox" = ...;        # with KDE-specific tweaks
  "alice@kali" = ...;                # with Kali-specific PATH adjustments
};
```

Then on each host: `home-manager switch --flake ~/nixos-config#alice@<host>`.

In practice, most people keep a single `alice` and use conditionals inside `home.nix` (`lib.mkIf (builtins.getEnv "HOSTNAME" == "...") { ... }`) — but the multi-entry pattern is there when you need it.

## See also

- [getting-started.md](getting-started.md) — the big picture, the "levels" framing.
- [configuration.md](configuration.md) — the `configuration.nix` anatomy and option discovery.
- [home-manager-modes.md](home-manager-modes.md) — the standalone vs module distinction used throughout this page.
- [nix-language.md](nix-language.md) — the syntax behind `mkHost = hostName: extraModules: ...`.
- [tricks.md](tricks.md) — `nixos-anywhere` and remote-build workflows referenced above.
