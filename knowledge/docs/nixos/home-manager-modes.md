# home-manager modes — standalone vs NixOS module

home-manager has two deployment modes and they feel very different in day-to-day use. Choosing the right one for a given host is worth getting right once, because changing modes later involves small-but-real migration work.

This page exists because both modes are used interchangeably across the rest of the chapter and the distinction is rarely made explicit. Short version:

- **Standalone** — home-manager is its own tool with its own CLI (`home-manager switch`). Good when the host is *not* NixOS: Kali, Debian, WSL2-Debian. Also fine as a first step on NixOS.
- **As a NixOS module** — home-manager becomes part of the NixOS system build. `nixos-rebuild switch` rebuilds the whole thing — system plus every user's home config — atomically, in one closure. The canonical choice once the host is NixOS and you are managing it as a single-user (or known-users) machine.

## In this page

- [Standalone mode](#standalone-mode)
- [NixOS module mode](#nixos-module-mode)
- [Side-by-side comparison](#side-by-side-comparison)
- [Which to use where](#which-to-use-where)
- [Migrating between modes](#migrating-between-modes)
- [The `stateVersion` gotcha](#the-stateversion-gotcha)
- [`pkgs` evaluation — shared or not?](#pkgs-evaluation-shared-or-not)

## Standalone mode

The mode that works on any Linux, NixOS or not. `home-manager` is a thin CLI that:

1. Evaluates `~/.config/home-manager/home.nix` (or your flake's `homeConfigurations.<name>`).
2. Builds a user-profile closure.
3. Swaps symlinks in your `$HOME` to point at the new closure.
4. Writes a generation record under `~/.local/state/nix/profiles/home-manager-*`.

### Install (classic)

```bash
nix-channel --add https://github.com/nix-community/home-manager/archive/master.tar.gz home-manager
nix-channel --update
nix-shell '<home-manager>' -A install
```

### Install (flakes)

```nix
# ~/.config/home-manager/flake.nix
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

### The `home.nix` is self-contained

```nix
{ config, pkgs, ... }:
{
  home.username = "alice";
  home.homeDirectory = "/home/alice";
  home.stateVersion = "24.11";

  # On non-NixOS hosts — critical for GUI integration:
  targets.genericLinux.enable = true;

  programs.home-manager.enable = true;
  programs.fish.enable = true;

  home.packages = with pkgs; [ ripgrep fd bat eza ];
}
```

### Switch

```bash
home-manager switch                            # classic
home-manager switch --flake ~/nixos-config#alice  # flakes
```

Generations:

```bash
home-manager generations
/nix/store/<hash>-home-manager-generation/activate   # jump to a prior one
```

### Pros

- **Works on any Linux.** This is the only mode available on Debian/Kali/WSL2.
- **User-owned — no root.** Nothing the user does touches `/etc` or `/run/current-system`.
- **Independent update cadence.** System and user can be updated at different times, with different input revisions.
- **Portable across hosts.** The exact same `home.nix` works on Kali, on a NixOS VM, and inside WSL2 — no host-shape assumptions.

### Cons

- **Two tools to know** — `nixos-rebuild` for the system, `home-manager` for the user.
- **Two profiles to roll back.** Rolling back the system without rolling back home-manager can leave them at inconsistent input revisions.
- **Duplicated evaluation.** Nix evaluates nixpkgs twice — once for the system build, once for home-manager. Slightly slower, and the two evaluations can import different revisions of nixpkgs if you're not careful.

## NixOS module mode

On a NixOS host, home-manager can be imported as a module and becomes part of the system build. One command (`nixos-rebuild switch`), one closure, one rollback unit.

### Classic

`/etc/nixos/configuration.nix`:

```nix
{ config, pkgs, ... }:
{
  imports = [
    ./hardware-configuration.nix
    <home-manager/nixos>
  ];

  home-manager.useGlobalPkgs = true;
  home-manager.useUserPackages = true;

  home-manager.users.alice = import ./home.nix;
}
```

The `<home-manager/nixos>` path requires a `home-manager` nix-channel, added once:

```bash
sudo nix-channel --add https://github.com/nix-community/home-manager/archive/release-24.11.tar.gz home-manager
sudo nix-channel --update
```

### Flakes

The more common modern setup. In your flake:

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
          home-manager.useGlobalPkgs = true;
          home-manager.useUserPackages = true;
          home-manager.users.alice = import ./home.nix;
        }
      ];
    };
  };
}
```

`inputs.nixpkgs.follows = "nixpkgs"` is the key line — it forces home-manager to use the *same* nixpkgs as the system, eliminating the duplicated-evaluation concern.

### Activate — no separate CLI

```bash
sudo nixos-rebuild switch --flake /etc/nixos#myhost
```

The same command that builds and activates the system also builds and activates every user's home-manager closure.

### Pros

- **Single source of truth.** One file, one rebuild, one rollback. Generation `N` of the system contains generation `N` of every user's home.
- **Shared `pkgs`.** With `useGlobalPkgs = true`, home-manager imports nixpkgs exactly once with the system's overlays and config — no duplication, no drift.
- **User packages go system-wide.** With `useUserPackages = true`, each user's `home.packages` is installed into the system closure via `users.users.<name>.packages`, so they're visible in login-shell `PATH` even before home-manager's session hooks have run.
- **Declarative everywhere.** No "remember to also run `home-manager switch`" after a reboot.

### Cons

- **NixOS-only.** Cannot be used on Kali, Debian, or WSL2-with-Debian.
- **Single bootloader transaction.** If the build fails because of a user-level mistake (a broken `home.nix`), `nixos-rebuild switch` fails as a whole — you don't get the system changes without the home ones.
- **Harder to work on a user's config without touching root.** Every `switch` requires `sudo`. Partial workaround: `nixos-rebuild build` as a normal user, inspect `./result`, then `sudo nixos-rebuild switch` when happy.

## Side-by-side comparison

| Concern                         | Standalone                                    | NixOS module                                  |
| ------------------------------- | --------------------------------------------- | --------------------------------------------- |
| Works on non-NixOS?             | ✅ (Kali, Debian, WSL2, Arch, …)              | ❌                                            |
| Needs `sudo`?                   | Never                                         | Always for switch                             |
| Rebuild command                 | `home-manager switch`                         | `sudo nixos-rebuild switch`                   |
| Atomicity with system           | Independent                                   | One atomic closure                            |
| `targets.genericLinux.enable`   | Required on non-NixOS                         | N/A (system handles XDG paths)                |
| `pkgs` evaluation               | Duplicated unless you share `inputs.nixpkgs`  | Shared via `useGlobalPkgs = true`             |
| Generation rollback             | `home-manager generations` (per-user)         | Bootloader menu (system + home as one)        |
| Packages on PATH at login       | Via `~/.profile` hook                         | Via `users.users.<name>.packages` directly    |
| First-party in nixpkgs options  | No (home-manager has its own option index)    | Yes, under `home-manager.*`                   |

## Which to use where

Mapping to the six venues from [getting-started.md](getting-started.md#the-experimentation-matrix):

| Venue                              | Host    | Mode                                                                                                                                  |
| ---------------------------------- | ------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| WSL2 + Nix on Debian/Ubuntu        | Windows | **Standalone only.** (Host is not NixOS.)                                                                                             |
| NixOS-WSL                          | Windows | Either works; **standalone** if you are iterating on `home.nix` quickly, **module** if your Windows-side NixOS is your "real" NixOS.  |
| VirtualBox NixOS VM                | Windows | **NixOS module** is natural — your flake already declares the VM as a `nixosConfigurations.*`, adding home-manager is one more module. |
| Nix / home-manager on Kali         | Kali    | **Standalone only.**                                                                                                                  |
| libvirt/QEMU NixOS VM on Kali      | Kali    | **NixOS module** — same reasoning as VirtualBox.                                                                                      |
| NixOS on an external SSD           | Either  | **NixOS module.** This is a real system; treat it like one.                                                                           |

The practical rule: **use standalone where you have to (Kali, WSL2-Debian), module where you can (any real NixOS).** Keep the same `home.nix` file across both — the module mode wraps the standalone `home.nix` without rewriting.

## Migrating between modes

The file you care about — `home.nix` — is the same in both modes. The migration work is entirely about how you invoke the build.

### Standalone → NixOS module

Assuming you already have `home.nix` working standalone on a NixOS host:

1. Add `home-manager.nixosModules.home-manager` to your flake's `modules = [ ... ]` list.
2. Add these three lines to `configuration.nix`:

   ```nix
   home-manager.useGlobalPkgs = true;
   home-manager.useUserPackages = true;
   home-manager.users.alice = import ./home.nix;
   ```

3. `sudo nixos-rebuild switch --flake /etc/nixos#myhost`. It will build and activate home-manager inside the system closure.
4. Optional: uninstall the standalone CLI to avoid confusion — `nix-env -e home-manager` or remove it from your user profile. The `home-manager` binary will still be on the system PATH because the NixOS module installs it; the difference is that you use `nixos-rebuild`, not `home-manager`, as the entry point.

### NixOS module → standalone

The rare direction. Typical reason: you want to iterate on `home.nix` faster than a full `nixos-rebuild` allows.

1. Remove the three `home-manager.*` lines from `configuration.nix` (or `.users.alice` to leave other users intact).
2. Rebuild system: `sudo nixos-rebuild switch --flake /etc/nixos#myhost`. The previous home-manager generation is still on disk but no longer the "current".
3. Install the standalone CLI and point it at your `home.nix`:

   ```bash
   nix run home-manager/master -- switch --flake ~/nixos-config#alice
   ```

4. From now on, `home-manager switch --flake ~/nixos-config#alice` is your per-user update command; `nixos-rebuild switch` handles system-only.

## The `stateVersion` gotcha

`home.stateVersion` and `system.stateVersion` are **separate options with unrelated semantics**. Both exist to pin state defaults that change between releases; do not cross-pollinate them.

- `system.stateVersion = "24.11";` — lives in `configuration.nix`. Pins NixOS state semantics. Bump intentionally, rarely.
- `home.stateVersion = "24.11";` — lives in `home.nix`. Pins home-manager state semantics. Bump intentionally, rarely.

They can be different values on the same machine. A common layout: `system.stateVersion = "23.11";` (the release you installed two years ago), `home.stateVersion = "24.11";` (the release when you last migrated home-manager state). Nothing bad happens.

## `pkgs` evaluation — shared or not?

This subtlety trips experienced users often enough to warrant a dedicated subsection.

### Standalone — two independent evaluations

By default, standalone home-manager brings its own `nixpkgs` input and evaluates it separately from whatever the system uses. Consequences:

- Two copies of nixpkgs in the store: slightly more disk, slightly more download.
- `pkgs.git` in `configuration.nix` and `pkgs.git` in `home.nix` can be built from **different commits** of nixpkgs if the inputs weren't aligned.
- Overlays applied in `configuration.nix` are not visible to `home.nix`, and vice versa, unless you explicitly share them.

Fix if this bothers you: in your flake, set `inputs.home-manager.inputs.nixpkgs.follows = "nixpkgs";` — forces home-manager to use the same nixpkgs input as the system. This brings standalone mode's behavior closer to module mode.

### Module mode — shared evaluation

With `home-manager.useGlobalPkgs = true;`, the NixOS module mode reuses the system's already-evaluated `pkgs`. Overlays apply once and are visible everywhere. Version skew is impossible by construction.

With `useGlobalPkgs = false;` (the default in *some* older templates), home-manager evaluates its own `pkgs` even in module mode. Almost always worth overriding to `true`.

## See also

- [getting-started.md](getting-started.md#level-2a-home-manager) — Level 2a; where home-manager first appears.
- [kali-to-nixos.md](kali-to-nixos.md#level-2a-home-manager-on-kali) — the standalone mode in practice on Kali.
- [configuration.md](configuration.md#home-manager-integration) — the one-line pointer in the system-config anchor page.
- [multi-host-flake.md](multi-host-flake.md) — how `homeConfigurations` vs `home-manager.users.*` compose across hosts.
- [home-manager option index](https://nix-community.github.io/home-manager/options.xhtml) — authoritative list of every `programs.*` / `home.*` option.
