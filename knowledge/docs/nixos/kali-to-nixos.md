# From Kali to a NixOS-managed config

This chapter is the Kali-side deep dive. The mental framing from [getting-started.md](getting-started.md) still applies: **home-manager** gives you a portable user environment that runs on Kali unchanged, while **NixOS** (the whole-system distribution) is something you first rehearse in a VM on this machine and only install on metal when you are certain your config reproduces your real workflow. Kali's particularity is its pentest toolchain — this page explains how to coexist with it, when to leave it alone, and how to reproduce it in Nix once you commit.

## What Kali is, for Nix purposes

Kali Linux is a Debian-derivative. Kali 2024.x and 2025.x are built on **Debian bookworm / trixie**, adding the Kali APT repos, a curated pentest metapackage (`kali-linux-large`, `kali-linux-headless`, etc.) and some desktop polish. Everything underneath — `apt`, `dpkg`, systemd, `/etc/shells`, `~/.profile` semantics, fontconfig paths, the way `.desktop` files work — is Debian, and behaves exactly like Debian.

**Practical consequence:** every "Nix on Debian" / "home-manager on Debian" recipe you find on the internet applies to Kali verbatim. The only wrinkle is Kali's curated toolchain, which this page treats as a first-class concern rather than a footnote.

## In this page

- [Level 1 — Nix on Kali](#level-1-nix-on-kali)
- [Level 2a — home-manager on Kali](#level-2a-home-manager-on-kali)
- [Migrating user tools from apt to home-manager](#migrating-user-tools-from-apt-to-home-manager)
- [Making fish or zsh your login shell on Kali](#making-fish-or-zsh-your-login-shell-on-kali)
- [The Kali pentest toolchain — a strategy](#the-kali-pentest-toolchain-a-strategy)
- [Level 2b — NixOS VM on Kali via libvirt/KVM](#level-2b-nixos-vm-on-kali-via-libvirtkvm)
- [Level 3 — replacing Kali with NixOS](#level-3-replacing-kali-with-nixos)
- [`nixos-anywhere` from Kali to a target machine](#nixos-anywhere-from-kali-to-a-target-machine)
- [Reverse direction — Kali in a VM on NixOS](#reverse-direction-kali-in-a-vm-on-nixos)

---

## Level 1 — Nix on Kali

Quick reference; the full walkthrough lives in [getting-started.md — Level 1](getting-started.md#level-1-nix-package-manager). On Kali, the multi-user installer works without special handling:

```bash
sh <(curl -L https://nixos.org/nix/install) --daemon
```

Log out / log in (or `. /etc/profile`) so `/etc/profile.d/nix-daemon.sh` is picked up. Then:

```bash
nix-shell -p cowsay --run "cowsay hello"   # one-shot, no install
nix-env -iA nixpkgs.ripgrep                # install (classic)
nix profile install nixpkgs#ripgrep        # install (flakes)
```

Enable flakes by writing to `~/.config/nix/nix.conf`:

```ini
experimental-features = nix-command flakes
```

**Kali-specific notes:**

- Kali's default login shell has been **zsh** since Kali 2020.4. `$SHELL` is `/usr/bin/zsh`, the prompt comes from Kali's custom `zshrc` at `/etc/zsh/zshrc` + `~/.zshrc`. The Nix installer appends to `~/.zshrc` in recent versions, but confirm by searching for `NIX_PROFILE` afterwards.
- If you're running Kali inside a VM or WSL, verify `/nix` actually lives on the right disk — Nix's store can grow several GB once you've built a few system closures. The default is `/nix`, and it's safe to symlink `/nix` to a larger partition **before** installing Nix.
- If Kali has `AppArmor` enabled (not default, but some pentest configs turn it on), Nix's multi-user daemon can hit policy errors. `journalctl -u nix-daemon` is the first place to look.

---

## Level 2a — home-manager on Kali

Quick reference; the full walkthrough, the coexistence explanation and the pure-shell audit technique all live in [getting-started.md — Level 2a](getting-started.md#level-2a-home-manager). The **three options you must not skip on Kali**, distilled from there:

1. `targets.genericLinux.enable = true;` — wires up `XDG_DATA_DIRS`, icons, fontconfig so GUI apps from home-manager appear in the launcher.
2. `programs.<your-shell>.enable = true;` — guarantees home-manager's PATH/env setup is sourced by your shell of choice.
3. Use a git-tracked `home.nix` shared with the other machine — that is the entire payoff.

Minimal Kali-flavored `home.nix`:

```nix
{ pkgs, ... }:
{
  home.username = "alice";
  home.homeDirectory = "/home/alice";
  home.stateVersion = "24.11";

  targets.genericLinux.enable = true;
  programs.home-manager.enable = true;
  programs.zsh.enable = true;                 # or bash / fish — match Kali's login shell

  home.packages = with pkgs; [
    coreutils findutils gnugrep gnused gawk   # useful when you enter a pure shell
    ripgrep fd bat eza fzf jq httpie btop
    git tmux neovim
  ];

  programs.git = {
    enable = true;
    userName = "Alice Example";
    userEmail = "alice@example.com";
  };

  programs.starship.enable = true;
  programs.direnv = {
    enable = true;
    nix-direnv.enable = true;
  };
}
```

Apply:

```bash
home-manager switch
```

Then run the audit techniques from [getting-started.md — Auditing exactly what home-manager provides](getting-started.md#auditing-exactly-what-home-manager-provides) — `home-manager packages`, `find ~ -lname '/nix/store/*'`, and the pure-shell incantation — to confirm what you just installed.

---

## Migrating user tools from apt to home-manager

Kali ships with a lot of user tools installed via apt that home-manager can own just as well. The general recipe for each one is always the same:

1. **Declare it in `home.nix`** — either in `home.packages` (for packages that don't need config management) or via `programs.<name>.enable = true;` (for tools with dedicated home-manager modules: git, fish, neovim, tmux, starship, direnv, alacritty, kitty, bat, eza, fzf, zoxide, ssh, gpg, ...).
2. **`home-manager switch`** — home-manager installs it under `~/.nix-profile/bin/` and starts shadowing the apt binary via PATH.
3. **Verify** — `command -v <tool>` should return the `~/.nix-profile/bin/...` path. Run the tool, confirm version and config look right.
4. **`sudo apt remove <pkg>`** — only after step 3 passes. This avoids a broken gap where you've uninstalled the apt version but home-manager isn't providing the replacement yet.
5. **`sudo apt autoremove`** from time to time — Kali's metapackages will have pulled in lots of transitive dependencies you can reclaim once the top-level tool is gone.

### A migration cheatsheet for the most common tools

| Kali (apt) package(s)                  | home-manager option(s)                           | Dotfiles home-manager will own                         |
| -------------------------------------- | ------------------------------------------------ | ------------------------------------------------------ |
| `git`                                  | `programs.git.enable = true;`                    | `~/.config/git/config`, `~/.gitignore_global`          |
| `neovim` / `vim`                       | `programs.neovim.enable = true;`                 | `~/.config/nvim/init.lua`                              |
| `tmux`                                 | `programs.tmux.enable = true;`                   | `~/.config/tmux/tmux.conf`                             |
| `zsh` + oh-my-zsh                      | `programs.zsh.enable = true;` (+ plugins inline) | `~/.zshrc`, `~/.zshenv`                                |
| `fish`                                 | `programs.fish.enable = true;`                   | `~/.config/fish/config.fish`                           |
| `starship`                             | `programs.starship.enable = true;`               | `~/.config/starship.toml`                              |
| `direnv`                               | `programs.direnv.enable = true;`                 | `~/.config/direnv/direnvrc`                            |
| `ripgrep`, `fd-find`, `bat`, `exa`/`eza`, `fzf`, `jq`, `htop`, `btop`, `httpie` | `home.packages = [ pkgs.ripgrep pkgs.fd pkgs.bat ... ];` | none (no dedicated modules; just binaries) |
| `alacritty` / `kitty`                  | `programs.alacritty.enable = true;` / `programs.kitty.enable = true;` | `~/.config/alacritty/alacritty.toml` etc. |
| `gpg`                                  | `programs.gpg.enable = true;`                    | `~/.gnupg/gpg.conf`                                    |
| `openssh-client`                       | `programs.ssh.enable = true;`                    | `~/.ssh/config`                                        |

### Walked example — migrating git

```bash
# 1. Confirm what apt has right now
dpkg -s git | grep -E '^(Package|Version):'
# Package: git
# Version: 1:2.45.2-1

# 2. Edit ~/.config/home-manager/home.nix, add:
#    programs.git = {
#      enable = true;
#      userName = "Alice Example";
#      userEmail = "alice@example.com";
#      extraConfig = { init.defaultBranch = "main"; pull.rebase = true; };
#    };

home-manager switch

# 3. Verify home-manager's git is now winning
command -v git
# /home/alice/.nix-profile/bin/git
git --version
# git version 2.46.0

# 4. Move your hand-written ~/.gitconfig out of the way the first time;
#    home-manager refuses to overwrite unmanaged files.
mv ~/.gitconfig ~/.gitconfig.pre-hm    # if one existed
home-manager switch                     # now succeeds
readlink ~/.config/git/config           # → a /nix/store path (home-manager owns it)

# 5. Finally remove the apt copy
sudo apt remove git
sudo apt autoremove
```

### Rollback in one command

```bash
home-manager generations              # list
/nix/store/<hash>-home-manager-generation/activate   # roll back to any past generation
```

If everything goes wrong, this is your "undo" button. Your `home.nix` commit history is the long-term memory; home-manager generations are the short-term undo stack.

---

## Making fish or zsh your login shell on Kali

A specific gotcha worth its own section. If you enable `programs.fish.enable = true;` in home-manager, the fish binary lives at `~/.nix-profile/bin/fish`, not `/usr/bin/fish`. On Linux, the login shell for a user has to be listed in `/etc/shells` — otherwise `chsh` refuses, and some display managers refuse to start your session. You can't edit `/etc/shells` from home-manager (it's system state owned by root).

Three workable strategies, in increasing order of "properly declarative":

### 1. Keep bash/zsh as login shell, hand off to fish in rc

The simplest. Leave your login shell as whatever Kali set up; tell bash/zsh to `exec` fish when the shell is interactive. This way login sequence, `~/.profile`, PAM, display manager all see a shell in `/etc/shells`.

In `~/.bashrc` or `~/.zshrc` (or via home-manager's `programs.bash.initExtra`):

```bash
if [[ $- == *i* ]] && [[ -z "$FISH_EXEC" ]] && command -v fish >/dev/null; then
  export FISH_EXEC=1
  exec fish
fi
```

The `FISH_EXEC` guard prevents an exec loop if fish spawns a bash subshell. Not perfectly declarative but the pragmatic choice on non-NixOS hosts.

### 2. Add the home-manager fish path to `/etc/shells` imperatively

Once, from a root shell:

```bash
echo "$HOME/.nix-profile/bin/fish" | sudo tee -a /etc/shells
chsh -s "$HOME/.nix-profile/bin/fish"
```

This works, but the path you added to `/etc/shells` is specific to your current generation's store path — after the next `home-manager switch` the binary at `~/.nix-profile/bin/fish` still exists (as a symlink), so this does keep working in practice. The objection is purely philosophical: system state is drifting outside your config repo.

### 3. Install fish from a real system-level Nix — i.e., NixOS

The fully declarative answer. On NixOS, you set `programs.fish.enable = true;` in `configuration.nix`, which adds fish to `environment.shells` and (indirectly) to `/etc/shells`. This only works once you're running NixOS — which is why [Level 3](#level-3-replacing-kali-with-nixos) exists.

**Recommendation on Kali: use strategy 1** until you switch to NixOS. It's pragmatic, reversible, and keeps `/etc/shells` out of your declarative scope.

---

## The Kali pentest toolchain — a strategy

Kali's value proposition is a curated, security-researcher-maintained set of hundreds of pentest tools, versioned together, with backported fixes. Nixpkgs has good coverage of the popular tools but does not offer the same curation. You have three realistic strategies while you're still on Kali:

### Strategy A — leave apt in charge of pentest tools

Home-manager owns your daily user tools (shell, editor, CLI utilities). apt (Kali) continues to own nmap, metasploit, burpsuite, wireshark, john, hashcat, aircrack-ng, and the rest of the Kali metapackage. You only cherry-pick into Nix the pentest tools that are either missing from Kali or whose Nix version you specifically want.

**Pros:** zero risk to your working Kali setup. Kali's update stream keeps the pentest toolchain current. No re-learning where anything lives.
**Cons:** your pentest toolchain is not portable — it lives on *this machine only*.
**When:** always, as the starting point. Also the right strategy if replacing Kali is not on your roadmap.

### Strategy B — duplicate, pick the winner, gradually

Install the nixpkgs version alongside apt's, run both briefly, retire whichever you like less. The PATH shadow makes this cheap — `nix run nixpkgs#nmap` runs Nix's nmap without even affecting PATH; `home.packages = [ pkgs.nmap ];` installs it persistently and shadows the apt one.

Watchouts:

- **Version skew.** Apt's Kali-tuned `metasploit-framework` may be ahead of or behind nixpkgs's `metasploit`. Check with `metasploit --version` (or `msfconsole -v`) after switching.
- **Shared state.** Many pentest tools keep session state in `~/.msf4`, `~/.john`, etc. — independent of PATH. Both copies will happily read/write the same state. That is usually what you want.
- **Tools that need setuid or special capabilities.** `nmap` wants raw sockets (cap_net_raw); nixpkgs provides a `nmap` and a `wrapped-nmap` (or similar) — setting capabilities on a `/nix/store/...` binary is not straightforward and is usually solved on NixOS via `security.wrappers`, which doesn't exist outside NixOS. On Kali, keep using apt's nmap for privileged scans.

### Strategy C — build a `pentest.nix` module and replace wholesale

This is the bridge to [Level 3](#level-3-replacing-kali-with-nixos). You pick the 20-40 pentest tools you actually use out of the hundreds Kali installs, declare them in a `pentest.nix` module, and use that module to reproduce Kali's essentials in any environment — a Nix home on Kali today, a NixOS VM tomorrow, a replacement NixOS install later. See the Level 3 section for the module skeleton.

### Tools with good nixpkgs coverage (as of unstable, 2025)

- Recon/scan: `nmap`, `masscan`, `zmap`, `rustscan`, `naabu`, `amass`, `subfinder`, `ffuf`, `gobuster`, `wfuzz`.
- Web: `burpsuite`, `zaproxy`, `nikto`, `sqlmap`, `wpscan`, `nuclei`, `httpx`.
- Exploitation: `metasploit`, `msfpc`, `exploitdb` (searchsploit), `routersploit`.
- Credentials / hashes: `john`, `hashcat`, `hydra`, `thc-hydra`, `medusa`.
- Wireless / RF: `aircrack-ng`, `reaver`, `bully`, `hcxtools`.
- Forensic: `binwalk`, `foremost`, `volatility3`, `autopsy`.
- Network: `wireshark`, `tcpdump`, `ettercap`, `bettercap`, `mitmproxy`.
- Windows-adjacent: `impacket`, `crackmapexec`, `bloodhound`, `responder`.
- Reverse engineering: `ghidra`, `radare2`, `rizin`, `cutter`, `binaryninja-free`.

### Tools where apt/Kali is materially better

- **Very recent CVE-specific PoCs** — Kali backports faster than nixpkgs-unstable cycles them in.
- **Niche scripts bundled in `kali-linux-*` metapackages** that have no upstream release — those aren't in nixpkgs at all.
- **Anything needing `CAP_NET_RAW` / setuid** — on Kali it's set during install; on non-NixOS nix you'd be manually setting capabilities on store paths.

---

## Level 2b — NixOS VM on Kali via libvirt/KVM

Kali has KVM available out of the box; libvirt + virt-manager gives you the desktop experience you want for iterating on `configuration.nix`.

### Install the virtualization stack

```bash
sudo apt install qemu-system-x86 qemu-utils libvirt-daemon-system virt-manager ovmf
sudo adduser "$USER" libvirt kvm
# log out / log in to pick up the new groups
sudo systemctl enable --now libvirtd
```

### Two workflows

**B1 — Install NixOS in the VM the "normal" way.** Download the NixOS installer ISO, create a new VM in virt-manager (4–8 GB RAM, 4 cores, 40 GB disk, UEFI firmware), attach the ISO, install as you would on real hardware. Once up, clone your flake inside the VM, pull `configuration.nix`, `sudo nixos-rebuild switch`. Take a libvirt snapshot after first boot — now you have a clean baseline you can roll back to.

**B2 — Build a QEMU VM directly from your flake.** The `nixos-rebuild build-vm` flow described in [getting-started.md — Level 2b](getting-started.md#option-c--libvirtqemu-vm-on-kali-the-native-linux-venue) is the fastest iteration loop: every edit to `configuration.nix`, one `nixos-rebuild build-vm --flake .#kali-vm`, one `./result/bin/run-*-vm`, observe. No installer involved, no snapshots, the VM disk is a `./nixos.qcow2` you can delete for a clean boot.

B1 is closer to the real-hardware experience; B2 is closer to unit-tested infrastructure. Use both for different purposes.

### Passing through a wireless card for wifi testing

If the point of the NixOS VM is to rehearse a pentest workflow, you'll want to pass a USB wifi card to the guest:

```xml
<!-- in virt-manager, Add Hardware → USB Host Device → pick the card -->
```

In `configuration.nix` on the guest side, make sure `aircrack-ng` and the card's driver module are present. Full disclosure: USB passthrough is more reliable than full PCI wifi passthrough. For PCI passthrough, `vfio-pci` + IOMMU groups apply (see [windows-to-nixos.md — Last resort — Windows VM with GPU passthrough](windows-to-nixos.md#last-resort-windows-vm-with-gpu-passthrough) for the same general approach applied to a GPU).

---

## Level 3 — replacing Kali with NixOS

The late step. Do this only when (a) your `home.nix` has been your daily driver on Kali for weeks, (b) you've run your target NixOS `configuration.nix` in a VM on this machine and are happy with it, and (c) you've identified the pentest-tools subset you actually use.

### The criteria — am I ready?

- [ ] `home.nix` is in git, deployed on both Kali and in the NixOS VM, and feels like home on both.
- [ ] `configuration.nix` for this machine runs in a libvirt VM for ≥2 weeks without you reaching back into apt for something.
- [ ] I have a `pentest.nix` module listing the pentest tools I actually reach for, not a wish-list of everything Kali installs.
- [ ] External backups are current — full `$HOME`, GPG keys, SSH keys, browser profiles, ~/.msf4, ~/.john, saved Burp/ZAP projects.
- [ ] I can boot a NixOS installer USB on this hardware and reach the internet.

### The `pentest.nix` module skeleton

```nix
# modules/pentest.nix
{ pkgs, ... }:
{
  environment.systemPackages = with pkgs; [
    # Recon / scan
    nmap masscan rustscan amass subfinder ffuf gobuster

    # Web
    burpsuite zaproxy nikto sqlmap nuclei httpx

    # Exploitation
    metasploit exploitdb

    # Credentials / hashes
    john hashcat hydra

    # Wireless
    aircrack-ng reaver hcxtools

    # Network
    wireshark tcpdump mitmproxy

    # Windows-adjacent
    impacket crackmapexec bloodhound

    # Reverse engineering
    ghidra radare2 rizin
  ];

  # Capabilities for nmap-style raw sockets without needing sudo
  security.wrappers.nmap = {
    owner = "root"; group = "root";
    capabilities = "cap_net_raw,cap_net_admin,cap_net_bind_service+eip";
    source = "${pkgs.nmap}/bin/nmap";
  };

  # Wireshark: add users to 'wireshark' group so dumpcap works unprivileged
  programs.wireshark.enable = true;

  # Kali-style Metasploit user data stays in ~/.msf4 — that's fine on NixOS too.
}
```

Then in `configuration.nix`:

```nix
{
  imports = [ ./modules/pentest.nix ];
}
```

Trim the list to what you actually use. You can always add more later with `sudo nixos-rebuild switch`. That's the entire point.

### The swap

1. Flash the NixOS installer ISO to a USB stick (Rufus from Windows, or `dd` from the Kali machine itself).
2. Make a final backup. This is an install, not an upgrade — nothing survives unless you copy it out.
3. Boot the installer USB. Partition (your choice: classic ext4/LVM, zfs, btrfs, or declarative via [disko](tricks.md)). Mount root at `/mnt`.
4. Clone your config repo into the target:

   ```bash
   nix-shell -p git
   git clone https://github.com/you/nixos-config.git /mnt/etc/nixos
   nixos-generate-config --root /mnt --dir /mnt/etc/nixos/hosts/kali-replacement
   ```

5. Review the generated `hardware-configuration.nix` — it will declare your exact disks, GPU, and kernel modules for this machine. Commit it.

6. Install:

   ```bash
   nixos-install --flake /mnt/etc/nixos#kali-replacement
   ```

7. Set the root password when prompted, reboot, remove the USB, log in as yourself, `home-manager switch --flake ~/nixos-config#alice`. Your user environment is back, identical to how it was on Kali.

### What you lose, what you gain

**Lose:**

- Kali's curated update stream. Security-critical pentest tools are maintained upstream anyway, but "fresh Kali rolling release" is a different rhythm from "nixpkgs unstable".
- A small number of tools that exist in Kali but not in nixpkgs. For those, you either skip them, package them yourself ([snippets.md](snippets.md) has `buildFHSEnv` / `autoPatchelfHook` recipes), or keep a Kali VM for the rare case.
- Kali's documentation assumes Kali. "Kali Linux Revealed" instructions will sometimes not directly apply.

**Gain:**

- `nixos-rebuild switch` to update the whole machine atomically, with a rollback one keystroke away.
- One git repo you can clone to stand up a new machine in minutes.
- A reproducible pentest environment — `pentest.nix` is the same file on your Kali replacement, on a coworker's NixOS box, and inside a throwaway NixOS VM for a specific engagement.
- Impermanence ([impermanence.md](impermanence.md)) becomes a natural next step: wipe `/` on every boot, declare exactly what persists. For a pentest box, this is remarkably clean.

---

## `nixos-anywhere` from Kali to a target machine

Once your Kali box has Nix and a working flake, it can provision a remote Linux host into NixOS over SSH with no installer media:

```bash
nix run github:nix-community/nixos-anywhere -- \
  --flake ~/nixos-config#portable-ssd \
  root@<target-ip>
```

`nixos-anywhere` takes the target machine (currently running any Linux), kexec-loads a NixOS installer environment, partitions according to your `disko` config, installs your flake, reboots into NixOS. Typical wall time is 15–30 minutes. See [tricks.md](tricks.md) for the full walkthrough.

Use cases:

- Installing NixOS onto a laptop whose BIOS boot menu is annoying to reach.
- Installing NixOS onto a VPS rented from a provider that only offers a Debian/Ubuntu base image.
- Reproducing a freshly-imaged lab machine into its "final" NixOS configuration in one step.

---

## Reverse direction — Kali in a VM on NixOS

Once you're on NixOS (whether as the Kali replacement or on an external SSD), you may still want a full Kali environment occasionally — for a specific class, a client that expects Kali, or a tool that never made it into your `pentest.nix`. The answer is a Kali VM on NixOS.

Enable libvirt in `configuration.nix`:

```nix
virtualisation.libvirtd = {
  enable = true;
  qemu.ovmf.enable = true;
  qemu.swtpm.enable = true;
};
programs.virt-manager.enable = true;
users.users.alice.extraGroups = [ "libvirtd" "kvm" ];
```

Rebuild, download Kali's official ISO, create the VM in virt-manager, install as you would on any other host. Snapshot after first boot so you can roll back to a clean baseline.

For pentest engagements that need a fresh environment per target, clone the Kali VM, work inside the clone, discard at the end. Same workflow people used to do on Kali itself with VirtualBox — just now the host is declarative.

---

## See also

- [getting-started.md](getting-started.md) — the big picture, the levels, and the experimentation matrix.
- [configuration.md](configuration.md) — the `configuration.nix` mental model.
- [packages.md](packages.md) — `nix-ld`, `steam-run`, `buildFHSEnv` for running prebuilt/third-party binaries not packaged in nixpkgs.
- [windows-to-nixos.md](windows-to-nixos.md) — Windows-side experimentation (WSL2, VirtualBox, external-SSD install), plus the gaming stack for the day you boot NixOS on real desktop hardware.
- [tricks.md](tricks.md) — `nixos-anywhere`, live USB build from your flake, generation management, `disko`-based declarative partitioning.
- [impermanence.md](impermanence.md) — wipe root on every boot; the cleanest expression of "my system is a git repo", particularly apt for a pentest box.
