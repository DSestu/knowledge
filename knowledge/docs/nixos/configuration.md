# Configuration

The whole system is one Nix expression, usually rooted at `/etc/nixos/configuration.nix`. Every time you run `nixos-rebuild switch`, that expression is evaluated, a new system closure is built, and the bootloader gets a new entry. Rollback is free.

This page covers the mental model of `configuration.nix` itself: what it looks like, how to rebuild, how to find the option you need, and how to split the file into modules when it grows.

## In this page

- [Anatomy of `configuration.nix`](#anatomy-of-configuration-nix)
- [`nixos-rebuild` subcommands](#nixos-rebuild-subcommands)
- [Generations and rollback](#generations-and-rollback)
- [Discovering what options a module exposes](#discovering-what-options-a-module-exposes)
- [Channels vs flake inputs](#channels-vs-flake-inputs)
- [Modularizing your config](#modularizing-your-config)
- [Unfree packages](#unfree-packages)
- [Home-manager integration](#home-manager-integration)

## Anatomy of `configuration.nix`

```nix
{ config, pkgs, lib, ... }:
{
  imports = [
    ./hardware-configuration.nix
    ./modules/fish.nix
    ./modules/kde.nix
  ];

  # Bootloader
  boot.loader.systemd-boot.enable = true;
  boot.loader.efi.canTouchEfiVariables = true;

  # Networking
  networking.hostName = "nixos";
  networking.networkmanager.enable = true;

  # Locale / time
  time.timeZone = "Europe/Paris";
  i18n.defaultLocale = "en_US.UTF-8";
  console.keyMap = "fr";

  # Users
  users.users.alice = {
    isNormalUser = true;
    description = "Alice";
    extraGroups = [ "wheel" "networkmanager" "video" "docker" ];
    shell = pkgs.fish;
  };

  # System-wide packages
  environment.systemPackages = with pkgs; [
    git vim micro wget curl htop
  ];

  # Allow unfree packages (Steam, vscode, nvidia, etc.)
  nixpkgs.config.allowUnfree = true;

  # Keep this pinned; bump it intentionally (see nixos-rebuild --upgrade).
  system.stateVersion = "24.11";
}
```

Keys you will touch most often: `imports`, `environment.systemPackages`, `services.*`, `programs.*`, `users.users.<name>`, `networking.*`, `hardware.*`, `boot.*`.

> `system.stateVersion` does **not** mean "which nixpkgs release" — it pins state semantics (default values of services that changed behavior across releases). Set it to the release you first installed and leave it alone unless you explicitly migrate.

## `nixos-rebuild` subcommands

| Command                             | What it does                                                           |
| ----------------------------------- | ---------------------------------------------------------------------- |
| `nixos-rebuild switch`              | Build, activate now, set as default boot entry.                         |
| `nixos-rebuild test`                | Build, activate now, **no** boot entry. Reboot reverts.                 |
| `nixos-rebuild boot`                | Build, set as next boot entry, do **not** activate now.                 |
| `nixos-rebuild build`               | Build only. Symlinks result under `./result`.                           |
| `nixos-rebuild build-vm`            | Build a QEMU VM of the config. Perfect for trying a change safely.     |
| `nixos-rebuild dry-build`           | Evaluate + show what would be built/fetched. No actual build.          |
| `nixos-rebuild dry-activate`        | Show what would change on activation (which services restart, etc.).   |
| `nixos-rebuild switch --rollback`   | Activate the previous generation as the new current.                   |
| `nixos-rebuild switch --upgrade`    | `nix-channel --update` first (classic only).                           |

### Classic

```bash
sudo nixos-rebuild switch
sudo nixos-rebuild switch --upgrade   # also update channels
```

### Flakes

```bash
sudo nixos-rebuild switch --flake /etc/nixos#<hostname>
sudo nix flake update /etc/nixos       # bump all inputs
sudo nix flake update /etc/nixos --update-input nixpkgs  # only nixpkgs
```

> `sudo` is needed because activation writes to `/run/current-system` and the bootloader. The build itself runs as the invoking user.

## Generations and rollback

List:

```bash
sudo nix-env --list-generations -p /nix/var/nix/profiles/system
```

Roll back to the previous:

```bash
sudo nixos-rebuild switch --rollback
```

Jump to a specific generation:

```bash
sudo /nix/var/nix/profiles/system-<N>-link/bin/switch-to-configuration switch
```

> The bootloader menu lists every generation. If a rebuild soft-bricks your graphical session, reboot and pick the previous entry from GRUB/systemd-boot — zero command-line needed.

## Discovering what options a module exposes

"I know `programs.fish.enable = true;` works — but how would I have known that `shellAbbrs`, `interactiveShellInit`, and `plugins` are also things?" This is the most frequent question when writing your first `configuration.nix`. The toolbox below, in roughly decreasing order of usefulness.

### Package vs module — not the same thing

- A **package** (like `pkgs.fish`) is a binary/library. Install it with `environment.systemPackages = [ pkgs.fish ];`. Packages have no "options" — you configure them through their own config files.
- A **module** (like `programs.fish`) wraps a package with declarative **options**. When a module exists, prefer it: it handles enabling, setting up completions, registering the shell in `/etc/shells`, starting the related service, etc.

Common packages almost always have a NixOS module at `nixos/modules/programs/<name>.nix` or `nixos/modules/services/<kind>/<name>.nix` in nixpkgs. Many also have a separate home-manager module at `modules/programs/<name>.nix` with a **different** set of per-user options.

### 1. [search.nixos.org/options](https://search.nixos.org/options) — start here

The canonical searchable index for NixOS module options. Search `programs.fish` and you get every suboption on one page, each with:

- **Description** — human text explaining what it does.
- **Type** — `attribute set of string`, `boolean`, `list of package`, `lines`, etc.
- **Default** — what you get if you set nothing.
- **Example** — a working value you can copy.
- **Declared in** — link to the exact `.nix` file in nixpkgs that defines the option (e.g. `nixos/modules/programs/fish.nix`). Click through to read surrounding options in context.

For `programs.fish` you'll find:

```
programs.fish.enable
programs.fish.interactiveShellInit
programs.fish.loginShellInit
programs.fish.shellAbbrs
programs.fish.shellAliases
programs.fish.shellInit
programs.fish.vendor.completions.enable
programs.fish.vendor.config.enable
programs.fish.vendor.functions.enable
```

That page is the authoritative answer to "what can I configure?".

### 2. home-manager option search

home-manager has **its own** option index — with a different (often larger) set of suboptions for the same `programs.<foo>` module.

[nix-community.github.io/home-manager/options.xhtml](https://nix-community.github.io/home-manager/options.xhtml)

Searching `programs.fish` here reveals:

- `programs.fish.plugins` — list of `{ name; src; }` attribute sets.
- `programs.fish.functions` — declaratively defined fish functions.
- `programs.fish.shellAbbrs`, `shellAliases`, `interactiveShellInit` — similar to the NixOS versions but scoped to one user.

> **Important:** `programs.fish.enable` exists in *both* NixOS and home-manager with overlapping but non-identical suboptions. Rough rule: system-level concerns (register shell in `/etc/shells`, vendor completions) live NixOS-side; per-user plugins / abbreviations / prompt theme live home-manager-side. If unsure, search both indices.

### 3. `man configuration.nix` / `man home-configuration.nix`

Every option also ships as a manpage on the installed system:

```bash
man configuration.nix         # NixOS options
man home-configuration.nix    # home-manager options (if home-manager is installed)
```

Inside `less`, `/programs.fish` jumps to the section. Offline, always matches your installed version — useful on a fresh target with no internet.

### 4. `nixos-option <path>` — live values

Query the current evaluated value of any option on the running system:

```bash
nixos-option services.openssh.enable
# -> true

nixos-option programs.fish.shellAbbrs
# -> { g = "git"; gs = "git status"; ... }

nixos-option programs.fish
# -> the full attribute set of every programs.fish.* value in your config
```

Use this to confirm your config is applied the way you think. Doubles as a discovery tool — if `nixos-option <path>` errors out with "does not exist", the option doesn't exist.

### 5. `nix repl` — interactive exploration

```bash
nix repl '<nixpkgs/nixos>'

# then:
nix-repl> options.programs.fish
# shows every option under programs.fish with its type

nix-repl> options.programs.fish.shellAbbrs.description
nix-repl> options.programs.fish.shellAbbrs.example
nix-repl> options.programs.fish.shellAbbrs.type
```

Flake users:

```bash
nix repl
nix-repl> :lf /etc/nixos
nix-repl> nixosConfigurations.myhost.options.programs.fish
```

Great for exploring siblings of an option without leaving the terminal.

### 6. Read the source — the ultimate truth

Every option is defined by `mkOption { ... }` in a module file. When documentation is thin, the source is always complete.

- NixOS modules: [github.com/NixOS/nixpkgs/tree/master/nixos/modules](https://github.com/NixOS/nixpkgs/tree/master/nixos/modules)
- home-manager modules: [github.com/nix-community/home-manager/tree/master/modules](https://github.com/nix-community/home-manager/tree/master/modules)

The `programs.fish` options live in `nixos/modules/programs/fish.nix`. Skimming it reveals declarations like:

```nix
shellAbbrs = mkOption {
  type = with types; attrsOf str;
  default = {};
  example = { gco = "git checkout"; gp = "git pull --rebase"; };
  description = "Set of fish abbreviations.";
};
```

Reading the module source is often faster than web-searching for rare options or understanding how two options interact.

### The workflow, end to end

1. Package exists? `nix search nixpkgs foo`.
2. Module exists? Search `programs.foo` and `services.foo` on [search.nixos.org/options](https://search.nixos.org/options).
3. Scan the siblings of `enable` on the results page for what's configurable.
4. Unsure how to shape a value? Copy the option's **Example**.
5. Per-user concerns? Cross-check the [home-manager options index](https://nix-community.github.io/home-manager/options.xhtml).
6. Still stuck? Click **Declared in** → read the module source.

> When a package has no module, install it via `environment.systemPackages` and configure it the old-fashioned way — through its own config file, typically managed as a home-manager `home.file."~/.config/foo/config".text = '' ... '';` entry.

## Channels vs flake inputs

### Classic — channels

```bash
sudo nix-channel --add https://nixos.org/channels/nixos-24.11 nixos
sudo nix-channel --update
sudo nixos-rebuild switch --upgrade
```

Pin to a specific commit by passing the tarball URL with `?rev=<sha>`:

```bash
sudo nix-channel --add https://github.com/NixOS/nixpkgs/archive/<sha>.tar.gz nixos-pinned
```

### Flakes — `flake.nix`

`/etc/nixos/flake.nix`:

```nix
{
  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-24.11";
    home-manager = {
      url = "github:nix-community/home-manager/release-24.11";
      inputs.nixpkgs.follows = "nixpkgs";
    };
  };

  outputs = { self, nixpkgs, home-manager, ... }: {
    nixosConfigurations.myhost = nixpkgs.lib.nixosSystem {
      system = "x86_64-linux";
      modules = [
        ./configuration.nix
        home-manager.nixosModules.home-manager
        {
          home-manager.users.alice = import ./home.nix;
        }
      ];
    };
  };
}
```

`flake.lock` is committed; it pins the exact revision of every input. `nix flake update` bumps it.

## Modularizing your config

Split `configuration.nix` once it crosses ~200 lines. One module per concern.

`./modules/fish.nix`:

```nix
{ pkgs, ... }:
{
  programs.fish.enable = true;
  environment.systemPackages = with pkgs; [ fishPlugins.tide fishPlugins.fzf-fish ];
}
```

Reference it from `configuration.nix`:

```nix
imports = [
  ./hardware-configuration.nix
  ./modules/fish.nix
  ./modules/kde.nix
  ./modules/dev.nix
  ./modules/gaming.nix
];
```

Any file that returns `{ pkgs, ... }: { ... }` can be imported; the module system merges everything.

## Unfree packages

Global opt-in:

```nix
nixpkgs.config.allowUnfree = true;
```

Per-package opt-in (preferred — you document exactly what is unfree):

```nix
nixpkgs.config.allowUnfreePredicate = pkg:
  builtins.elem (lib.getName pkg) [
    "steam" "steam-original" "steam-run"
    "vscode" "discord" "spotify"
    "nvidia-x11" "nvidia-settings"
  ];
```

For a one-off CLI command:

```bash
NIXPKGS_ALLOW_UNFREE=1 nix-shell -p spotify --run spotify
```

## Home-manager integration

If you want home-manager rebuilt alongside the system (instead of separately per user), import its NixOS module (see the flakes example above for the flake path; for classic, the README shows the `<home-manager/nixos>` import). Then `nixos-rebuild switch` also rebuilds your user environment. That is usually what you want on a single-user machine.

## Next

- [kde.md](kde.md) — Plasma 6 options layered on top of the skeleton above.
- [packages.md](packages.md) — adding packages and dealing with prebuilt binaries.
