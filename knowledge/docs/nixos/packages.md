# Packages

Where packages live, how to search, how to pin them, and how to install software that wasn't built for NixOS in the first place — Steam, vendor binaries, AppImages, Flatpak.

## In this page

- [The three scopes](#the-three-scopes)
- [Searching](#searching)
- [Pinning](#pinning)
- [Unfree packages](#unfree-packages)
- [The prebuilt-binary problem](#the-prebuilt-binary-problem)
- [Flatpak as escape hatch](#flatpak-as-escape-hatch)
- [Overlays (bumping or patching a single package)](#overlays-bumping-or-patching-a-single-package)

## The three scopes

| Scope                                    | File                                          | Persisted in         |
| ---------------------------------------- | --------------------------------------------- | -------------------- |
| **System-wide**                          | `environment.systemPackages`                  | Every user's `PATH`. |
| **Per-user (declarative)**               | `users.users.<name>.packages`                 | Only that user.      |
| **Per-user (home-manager)**              | `home.packages` in `home.nix`                 | Only that user, plus dotfile management. |
| **Per-user (imperative)**                | `nix profile install` / `nix-env -iA`         | Only that user, but not declarative. Avoid on systems you care about reproducing. |

Pick one per concern. Typical split: system packages for CLI essentials + system services; home-manager for everything user-facing (editor config, shell, GUI apps, themes).

## Searching

Web:

- [search.nixos.org/packages](https://search.nixos.org/packages) — packages.
- [search.nixos.org/options](https://search.nixos.org/options) — NixOS and home-manager options.

CLI:

### Classic

```bash
nix-env -qa firefox
nix-env -qa '.*firefox.*' --description
```

### Flakes

```bash
nix search nixpkgs firefox
nix search nixpkgs 'steam-run'
```

Find which package provides a command:

```bash
# once
nix-channel --add https://github.com/nix-community/nix-index-database/archive/main.tar.gz nix-index-database
nix-channel --update

# then
nix-locate bin/gcc
```

## Pinning

### Classic — channels

Channels are named pointers to a tarball URL. Pin by adding a tarball URL with an explicit commit:

```bash
sudo nix-channel --add https://github.com/NixOS/nixpkgs/archive/<sha>.tar.gz nixos
sudo nix-channel --update
```

### Flakes — `flake.lock`

Inputs pin automatically:

```bash
nix flake update                       # bump all inputs, update flake.lock
nix flake update --update-input nixpkgs
nix flake lock --override-input nixpkgs github:NixOS/nixpkgs/<sha>
```

Commit `flake.lock` alongside `flake.nix`. Together they give you bit-identical rebuilds on any machine.

## Unfree packages

Global opt-in:

```nix
nixpkgs.config.allowUnfree = true;
```

Per-package opt-in (preferred — auditable):

```nix
nixpkgs.config.allowUnfreePredicate = pkg:
  builtins.elem (lib.getName pkg) [
    "steam" "steam-original" "steam-run" "steam-unwrapped"
    "discord" "spotify" "vscode" "obsidian"
    "nvidia-x11" "nvidia-settings" "nvidia-persistenced"
  ];
```

One-off for a CLI invocation:

```bash
NIXPKGS_ALLOW_UNFREE=1 nix-shell -p spotify --run spotify
NIXPKGS_ALLOW_INSECURE=1 nix-shell -p oldtool --run oldtool
```

## The prebuilt-binary problem

NixOS has no `/usr/lib`, no `/lib64/ld-linux-x86-64.so.2`, nothing at the FHS paths a vendor binary expects. Five tools fix this, in roughly increasing effort:

### 1. `programs.steam` — the Steam escape hatch

Valve's Steam client ships a 32-bit ELF that dynamically links to a huge pile of libraries. nixpkgs wraps it in its own Steam runtime + FHS layer via a module:

```nix
programs.steam = {
  enable = true;
  remotePlay.openFirewall = true;
  dedicatedServer.openFirewall = true;
  gamescopeSession.enable = true;
  extraCompatPackages = with pkgs; [ proton-ge-bin ];
};

# Needed for games
hardware.graphics = {
  enable = true;
  enable32Bit = true;
};

# Gamemode: better performance for fullscreen games
programs.gamemode.enable = true;
```

After a rebuild, launch `steam` from the app menu. Proton / native games / Workshop all work.

> `hardware.graphics.enable32Bit = true;` is required for 32-bit GL — many games still ship 32-bit binaries.

### 2. `programs.nix-ld` — fake FHS dynamic linker

For arbitrary prebuilt binaries (VSCode extensions with native code, pip wheels with native deps, pnpm/npm native modules, rustup-installed toolchains, vendor binaries):

```nix
programs.nix-ld = {
  enable = true;
  libraries = with pkgs; [
    stdenv.cc.cc.lib
    zlib zstd
    openssl
    curl
    glib
    libxkbcommon
    icu
    fuse3
    xorg.libX11 xorg.libXcursor xorg.libXi xorg.libXrandr xorg.libXxf86vm
    libGL
  ];
};
```

This plants a shim at `/lib64/ld-linux-x86-64.so.2` that points to a curated library set. Most random binaries now start without you touching them.

### 3. `appimage-run` — one-command AppImage launcher

```nix
programs.appimage = {
  enable = true;
  binfmt = true;     # run AppImages just by double-clicking / ./foo.AppImage
};
```

With `binfmt = true`, AppImages behave like on Debian. Without it:

```bash
appimage-run ./Cursor.AppImage
```

### 3b. Declaratively install a remote AppImage

To turn an AppImage URL into a real, installed, PATH-available binary that ships with your config — use `appimageTools.wrapType2` together with `fetchurl`. The result is pinned by hash, reproducible on any machine, and survives reinstalls.

```nix
{ pkgs, ... }:
let
  cursor = pkgs.appimageTools.wrapType2 {
    pname   = "cursor";
    version = "0.42.0";
    src = pkgs.fetchurl {
      url  = "https://downloader.cursor.sh/linux/appImage/x64";
      hash = "sha256-AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=";
    };
  };
in {
  environment.systemPackages = [ cursor ];
}
```

**Getting the hash the first time.** Set `hash = lib.fakeSha256;` (or any obviously-wrong value), then `nixos-rebuild build`. The build fails with a message like:

```
got:    sha256-rq4GOVm3...=
```

Copy that hash into `hash = ...` and rebuild — it succeeds.

**Adding a `.desktop` entry, icon, and proper app menu integration.** The single-line `wrapType2` call gets you a working binary but no application-menu entry. For that, extract the AppImage first and hand-install the metadata:

```nix
let
  pname = "obsidian";
  version = "1.7.7";

  src = pkgs.fetchurl {
    url = "https://github.com/obsidianmd/obsidian-releases/releases/download/v${version}/Obsidian-${version}.AppImage";
    hash = "sha256-REPLACE_ME=";
  };

  appimageContents = pkgs.appimageTools.extractType2 {
    inherit pname version src;
  };
in
pkgs.appimageTools.wrapType2 {
  inherit pname version src;

  extraInstallCommands = ''
    install -m 444 -D ${appimageContents}/${pname}.desktop \
      $out/share/applications/${pname}.desktop
    install -m 444 -D ${appimageContents}/${pname}.png \
      $out/share/icons/hicolor/512x512/apps/${pname}.png
    substituteInPlace $out/share/applications/${pname}.desktop \
      --replace 'Exec=AppRun' 'Exec=${pname}'
  '';
}
```

That gives you:
- A binary on `PATH` (e.g. `obsidian`).
- A menu entry so it shows up in KRunner / the app launcher.
- An icon rendered everywhere apps are rendered.

**Keeping a private collection.** Put each AppImage in its own file under `./packages/<name>.nix`, import them from `configuration.nix`:

```nix
environment.systemPackages = [
  (pkgs.callPackage ./packages/cursor.nix {})
  (pkgs.callPackage ./packages/obsidian.nix {})
  (pkgs.callPackage ./packages/balena-etcher.nix {})
];
```

**Auto-updating the version.** AppImage URLs from GitHub releases can be bumped by running `nix-update`:

```bash
nix-shell -p nix-update --run 'nix-update --file ./packages/obsidian.nix obsidian'
```

…or, for less-scripted workflows, bump `version` manually, set `hash = lib.fakeSha256;`, rebuild to learn the new hash, paste.

**wrapType1 vs wrapType2.** Modern AppImages are "type 2" (the overwhelmingly common format since 2017). Use `wrapType2`. `wrapType1` is only needed for pre-2017 AppImages (rare).

> On NixOS, a `.AppImage` file by itself cannot run — the dynamic linker paths are wrong. `appimage-run` or `appimageTools.wrapType2` both fix that; the difference is that `wrapType2` gives you a permanent, config-managed package, while `appimage-run` is for one-off launches.

### 4. `steam-run` — run any binary inside Steam's FHS

Steam's runtime includes hundreds of common libs, so `steam-run` is a swiss-army knife for running third-party binaries:

```bash
steam-run ./some-vendor-binary
steam-run bash               # drop into an FHS shell
```

Install it alongside Steam or standalone:

```nix
environment.systemPackages = with pkgs; [ steam-run ];
```

### 5. `buildFHSEnv` — bespoke FHS environment

When the above aren't enough, wrap your own FHS shell with exactly the libs you need:

```nix
(pkgs.buildFHSEnv {
  name = "my-fhs";
  targetPkgs = pkgs: with pkgs; [
    gcc glibc zlib openssl curl libxml2
    python312 nodejs
  ];
  runScript = "bash";
})
```

Drop that into `environment.systemPackages`; `my-fhs` becomes a command that opens a Debian-like shell.

### 6. Packaging a binary yourself — `autoPatchelfHook`

For something you want as a proper system package:

```nix
pkgs.stdenv.mkDerivation {
  pname = "vendor-tool";
  version = "1.2.3";
  src = pkgs.fetchurl {
    url = "https://vendor.example/vendor-tool-1.2.3-linux-x64.tar.gz";
    sha256 = lib.fakeSha256;   # replace after first build
  };
  nativeBuildInputs = [ pkgs.autoPatchelfHook ];
  buildInputs = with pkgs; [ stdenv.cc.cc.lib zlib openssl ];
  installPhase = ''
    mkdir -p $out/bin
    cp -r . $out/
    ln -s $out/bin/vendor-tool $out/bin/vendor-tool
  '';
}
```

`autoPatchelfHook` rewrites every ELF's rpath to the nix store, so the binary finds its libs at runtime without FHS.

## Flatpak as escape hatch

Flatpak coexists with Nix and is the right answer for a few categories: proprietary apps with complex sandboxing needs, desktop apps only packaged for Flathub, things that break nix-ld.

```nix
services.flatpak.enable = true;
xdg.portal = {
  enable = true;
  extraPortals = [ pkgs.kdePackages.xdg-desktop-portal-kde ];
};
```

Add Flathub (one-time, user-level):

```bash
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
flatpak install flathub com.spotify.Client
```

## Overlays (bumping or patching a single package)

Overlays let you override one package without forking nixpkgs. Common case: you want a newer version of a tool than the current channel ships.

```nix
nixpkgs.overlays = [
  (final: prev: {
    # use a specific GitHub commit
    my-tool = prev.my-tool.overrideAttrs (old: {
      version = "1.2.3-custom";
      src = prev.fetchFromGitHub {
        owner = "example";
        repo = "my-tool";
        rev = "v1.2.3";
        sha256 = "sha256-AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=";
      };
    });

    # pull from unstable for one package
    firefox = (import (builtins.fetchTarball {
      url = "https://github.com/NixOS/nixpkgs/archive/nixos-unstable.tar.gz";
    }) { system = prev.system; config.allowUnfree = true; }).firefox;
  })
];
```

> For flakes, add `inputs.nixpkgs-unstable.url = "github:NixOS/nixpkgs/nixos-unstable";` and reference `inputs.nixpkgs-unstable.legacyPackages.${system}.firefox` directly — cleaner than the tarball trick.

## Next

- [snippets.md](snippets.md) — copy-paste recipes for common services.
- [tricks.md](tricks.md) — VM, live USB, `nixos-anywhere`, rollback.
