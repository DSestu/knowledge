# Dev tools

A developer workstation on NixOS: editors, shells, containers, VMs, per-project environments, IDEs, and the "how do I run this random binary from the internet" problem.

## In this page

- [System-wide CLI tools](#system-wide-cli-tools)
- [Shells](#shells)
- [Git / pre-commit hooks](#git-pre-commit-hooks)
- [Docker / Podman](#docker-podman)
- [Virtualization (virt-manager / libvirt / QEMU)](#virtualization-virt-manager-libvirt-qemu)
- [Per-project environments](#per-project-environments)
- [IDEs](#ides)
- [Language toolchains and the FHS caveat](#language-toolchains-and-the-fhs-caveat)
- [Secrets in dev (quick note)](#secrets-in-dev-quick-note)

## System-wide CLI tools

```nix
environment.systemPackages = with pkgs; [
  # editors
  neovim
  micro
  helix

  # shell utilities
  git
  git-lfs
  tmux
  zellij
  fzf
  ripgrep
  fd
  bat
  eza
  zoxide
  jq
  yq
  httpie
  curl
  wget

  # system
  btop
  htop
  ncdu
  dust
  duf
  lsof
  tree
  file
  unzip
  zip
  p7zip

  # nix-specific
  nix-tree         # explore store closures
  nix-output-monitor  # `nom` — prettier `nix build`
  nvd              # diff generations
  nix-index        # find which package provides a command
  nixpkgs-review
];
```

## Shells

```nix
programs.fish.enable = true;
programs.zsh.enable = true;

users.users.alice.shell = pkgs.fish;
```

Per-user fish config goes in `~/.config/fish/config.fish` — or declaratively via home-manager's `programs.fish.*` options.

> Starship prompt is system-wide via `programs.starship.enable = true;` but most users prefer declaring it per-user through home-manager.

## Git / pre-commit hooks

```nix
programs.git = {
  enable = true;
  lfs.enable = true;
  config = {
    init.defaultBranch = "main";
    pull.rebase = true;
    push.autoSetupRemote = true;
  };
};
```

Pre-commit ships as `pre-commit` in nixpkgs; use it via direnv in per-project shells, not system-wide.

## Docker / Podman

```nix
virtualisation.docker = {
  enable = true;
  rootless = {
    enable = true;
    setSocketVariable = true;
  };
};

# Add your user to the docker group for rootful socket access
users.users.alice.extraGroups = [ "docker" ];
```

Podman as a drop-in with the same CLI:

```nix
virtualisation.podman = {
  enable = true;
  dockerCompat = true;       # symlinks `docker` → `podman`
  defaultNetwork.settings.dns_enabled = true;
};
```

> Pick one. If Docker Desktop / Colima was your previous setup on macOS (see [python/setup](../python/setup/setup.md)), NixOS's native Docker daemon replaces Colima entirely — just install Docker and go.

## Virtualization (virt-manager / libvirt / QEMU)

```nix
virtualisation.libvirtd = {
  enable = true;
  qemu = {
    package = pkgs.qemu_kvm;
    swtpm.enable = true;
    ovmf = {
      enable = true;
      packages = [ pkgs.OVMFFull.fd ];
    };
  };
};

programs.virt-manager.enable = true;
users.users.alice.extraGroups = [ "libvirtd" "kvm" ];
```

Now `virt-manager` can create Windows/Linux VMs with full UEFI + TPM support.

## Per-project environments

The golden pattern on NixOS: one `shell.nix` (or `flake.nix`) per repo, auto-activated by direnv.

### Enable direnv system-wide

```nix
programs.direnv = {
  enable = true;
  nix-direnv.enable = true;   # fast, caching-aware Nix integration
  loadInNixShell = true;
};
```

Then in any repo:

```bash
echo "use flake" > .envrc        # for flakes
# or
echo "use nix"   > .envrc        # for classic shell.nix
direnv allow
```

### Minimal `shell.nix`

```nix
{ pkgs ? import <nixpkgs> {} }:
pkgs.mkShell {
  packages = with pkgs; [ python312 uv ruff pre-commit ];
  shellHook = ''
    export PYTHONDONTWRITEBYTECODE=1
  '';
}
```

### Minimal `flake.nix` devShell

```nix
{
  inputs.nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
  outputs = { nixpkgs, ... }:
    let
      system = "x86_64-linux";
      pkgs = import nixpkgs { inherit system; };
    in {
      devShells.${system}.default = pkgs.mkShell {
        packages = with pkgs; [ python312 uv ruff pre-commit ];
      };
    };
}
```

### devenv / flox / pixi on top of Nix

All three run on NixOS the same as they do on Debian. See [python/setup](../python/setup/setup.md) for the devenv + direnv workflow already documented in this repo. On NixOS the install step simplifies — `devenv` is just a package:

```nix
environment.systemPackages = with pkgs; [ devenv ];
```

## IDEs

```nix
environment.systemPackages = with pkgs; [
  # editors / IDEs
  vscode                   # unfree
  vscode-fhs               # vscode in an FHS env — fixes native extensions that `patchelf`-reject
  jetbrains.idea-community
  jetbrains.pycharm-community
  jetbrains.clion          # unfree
  zed-editor
];

nixpkgs.config.allowUnfree = true;
```

> Cursor is packaged as `code-cursor` in unstable, or install as an AppImage — see [packages.md](packages.md).

## Language toolchains and the FHS caveat

Pure nixpkgs packages work perfectly. Where you hit friction is when a language's own package manager tries to install prebuilt binaries (pip wheels with native libs, `npm`/`pnpm` native modules, rustup-installed toolchains, etc.). The dynamic linker on NixOS is **not** at `/lib64/ld-linux-x86-64.so.2`, so those binaries fail with confusing errors.

Three fixes, from cleanest to easiest:

1. **Use nixpkgs's own version:**
   ```nix
   environment.systemPackages = with pkgs; [
     python312
     nodejs_22
     rustc cargo
     go
     jdk21
   ];
   ```

2. **Use `programs.nix-ld`** to fake an FHS dynamic linker for foreign binaries:
   ```nix
   programs.nix-ld = {
     enable = true;
     libraries = with pkgs; [
       stdenv.cc.cc.lib
       zlib
       openssl
       curl
       glib
       libxkbcommon
     ];
   };
   ```
   Now prebuilt binaries (VSCode extensions, pip wheels, pnpm/npm native modules, rustup, etc.) mostly Just Work.

3. **Run inside an FHS shell** for the stubborn cases:
   ```nix
   (pkgs.buildFHSEnv {
     name = "fhs-dev";
     targetPkgs = p: with p; [ gcc zlib openssl ];
     runScript = "bash";
   })
   ```
   Launch with the command name; inside, paths behave like Ubuntu/Debian.

See [packages.md](packages.md) for the same pattern applied to games and random binaries.

## Secrets in dev (quick note)

Don't put secrets in `configuration.nix` — it lands in the world-readable `/nix/store`. Use direnv-local `.envrc` (gitignored) during dev. For production-style secrets-in-config, see the agenix/sops-nix pointer in [snippets.md](snippets.md).

## Next

- [packages.md](packages.md) — adding packages, unfree, Steam, AppImages, Flatpak.
- [snippets.md](snippets.md) — docker rootless, GPU, fonts, etc.
