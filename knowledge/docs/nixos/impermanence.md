# Impermanence

Wipe the entire root filesystem on every boot. Everything starts clean, except the paths you *explicitly* mark as persistent. The system becomes honest about its state — if you didn't declare it, it doesn't survive.

This is an advanced deployment pattern. You don't need it to enjoy NixOS, but it is the logical conclusion of "my system is a git repo of text files": anything not in the repo either shouldn't exist, or needs to be *explicitly* persisted.

## In this page

- [Why impermanence](#why-impermanence)
- [Two approaches](#two-approaches)
- [Approach 1 — tmpfs root + impermanence module](#approach-1-tmpfs-root-impermanence-module)
- [Approach 2 — ZFS / Btrfs snapshot rollback ("erase your darlings")](#approach-2-zfs-btrfs-snapshot-rollback-erase-your-darlings)
- [What to persist](#what-to-persist)
- [Per-user persistence (home-manager)](#per-user-persistence-home-manager)
- [Discovering hidden state](#discovering-hidden-state)
- [Pitfalls and gotchas](#pitfalls-and-gotchas)
- [Disaster recovery becomes trivial](#disaster-recovery-becomes-trivial)
- [Further reading](#further-reading)

---

## Why impermanence

- **Proves your config is complete.** Any hidden state — logs, caches, half-written configs, unknown service data — vanishes on reboot. If something breaks when it disappears, you just found a gap in your `configuration.nix`.
- **Reinstalls are identical.** Clone the repo, reformat, reboot: same system, no drift.
- **Security and forensics.** An attacker who dropped a file in `/etc`, `/usr`, or `/tmp` loses it at reboot. Same for rootkits that forget the boot path.
- **Less cruft.** Uninstalling a service removes its config. No leftover dotfiles from tools you tried once. The system is always the sum of its declared state.
- **Clear mental model.** Either a path is in `/persist`, or it resets. No "maybe it's cached somewhere" ambiguity.

The cost is upfront work: you must think about every piece of state and decide whether it deserves to persist. In practice that thinking is valuable — most people discover their machine was storing things they didn't know about.

---

## Two approaches

| Approach                          | Root FS                        | Complexity | Flexibility |
| --------------------------------- | ------------------------------ | ---------- | ----------- |
| **Tmpfs root + impermanence**     | RAM (tmpfs)                    | Low        | Medium      |
| **ZFS/Btrfs rollback**            | Real FS, snapshot-restored     | Medium     | High        |

- **Tmpfs root** is easier to set up and requires no special filesystem. RAM-bounded, which means root FS caps at a few GB. Fine for a desktop/laptop; not great for servers with heavy logging.
- **Snapshot rollback** keeps a real journaling FS (ZFS or Btrfs) and rolls back to a known-good snapshot before mount. No RAM limit, but requires ZFS/Btrfs familiarity and more careful dataset/subvolume planning.

Start with tmpfs. Migrate to ZFS rollback only if tmpfs bites you.

---

## Approach 1 — tmpfs root + impermanence module

### Partition layout

Typical layout for a single disk:

| Partition | Mount       | FS    | Purpose                                        |
| --------- | ----------- | ----- | ---------------------------------------------- |
| 1         | `/boot`     | vfat  | ESP (EFI System Partition).                    |
| 2         | `/nix`      | ext4  | The Nix store. Huge, cacheable, rebuildable.   |
| 3         | `/persist`  | ext4  | Everything that must survive a reboot.         |
| 4         | `[swap]`    | swap  | (Optional; for hibernate, must be ≥ RAM.)      |

Root is **not on disk** — it is a tmpfs.

> [Disko](https://github.com/nix-community/disko) lets you declare this partition layout in your flake and apply it from the installer. Recommended: your disk layout ends up in git alongside everything else. See [tricks.md](tricks.md) and the disko README for examples.

### Minimal `configuration.nix`

```nix
{
  # Root on tmpfs — wiped on every boot.
  fileSystems."/" = {
    device = "none";
    fsType = "tmpfs";
    options = [ "defaults" "size=4G" "mode=755" ];
  };

  # Persistent storage for things we declare below.
  fileSystems."/persist" = {
    device = "/dev/disk/by-label/persist";
    fsType = "ext4";
    neededForBoot = true;        # must be mounted before activation
  };

  # The Nix store itself.
  fileSystems."/nix" = {
    device = "/dev/disk/by-label/nix";
    fsType = "ext4";
    neededForBoot = true;
  };

  fileSystems."/boot" = {
    device = "/dev/disk/by-label/ESP";
    fsType = "vfat";
  };
}
```

> `neededForBoot = true;` on `/persist` is critical. Activation runs before fstab mounts, so any bind-mount from `/persist` will fail silently unless `neededForBoot` forces it to mount in the initrd.

### Wire in the impermanence module

Flake:

```nix
{
  inputs.impermanence.url = "github:nix-community/impermanence";

  outputs = { self, nixpkgs, impermanence, ... }: {
    nixosConfigurations.myhost = nixpkgs.lib.nixosSystem {
      system = "x86_64-linux";
      modules = [
        impermanence.nixosModules.impermanence
        ./configuration.nix
      ];
    };
  };
}
```

Classic (no flakes):

```nix
# somewhere in configuration.nix
imports = [
  (builtins.fetchTarball "https://github.com/nix-community/impermanence/archive/master.tar.gz" + "/nixos.nix")
];
```

### Declare what persists

```nix
environment.persistence."/persist" = {
  hideMounts = true;              # don't clutter `mount` and `df` output
  directories = [
    "/var/log"                    # system logs
    "/var/lib/nixos"              # UID/GID seeds, etc.
    "/var/lib/bluetooth"          # paired devices
    "/var/lib/systemd/coredump"
    "/var/lib/AccountsService"    # user avatars
    "/var/lib/NetworkManager"     # known networks
    "/etc/NetworkManager/system-connections"  # saved wifi passwords
    "/etc/ssh"                    # host keys
  ];
  files = [
    "/etc/machine-id"             # systemd/journald identity
  ];
};
```

`directories` are bind-mounted; anything the system writes to them lands on `/persist`. `files` do the same for individual paths.

---

## Approach 2 — ZFS / Btrfs snapshot rollback ("erase your darlings")

Your root filesystem is a real ZFS or Btrfs dataset. On every boot, before `switch-root`, a pre-boot hook restores a pristine empty snapshot of root. Persistent datasets/subvolumes (`/home`, `/nix`, `/var/log`, `/etc/ssh`, …) are mounted separately and untouched.

### ZFS variant

```nix
boot.initrd.postDeviceCommands = lib.mkAfter ''
  zfs rollback -r rpool/root@blank
'';
```

Take the `blank` snapshot once, immediately after install, when root is in its desired empty state:

```bash
sudo zfs snapshot rpool/root@blank
```

Everything written to `rpool/root` between the snapshot and the next reboot gets erased.

Persistent data lives in separate datasets:

- `rpool/persist` → `/persist`
- `rpool/home` → `/home`
- `rpool/nix` → `/nix`
- `rpool/var-log` → `/var/log`

None of these get rolled back. The impermanence module still helps declare bind-mounts if you prefer that style.

### Btrfs variant

Replace `zfs rollback` with a Btrfs subvolume swap:

```nix
boot.initrd.postDeviceCommands = lib.mkAfter ''
  mkdir -p /mnt
  mount -o subvol=/ /dev/disk/by-label/nixos /mnt
  btrfs subvolume delete /mnt/root
  btrfs subvolume snapshot /mnt/root-blank /mnt/root
  umount /mnt
'';
```

### Pros and cons vs tmpfs

- No RAM ceiling — root can be arbitrarily large during a session.
- Preserves real FS semantics (hardlinks, reflinks, inode numbers) which a few rare tools want.
- More setup; dataset / subvolume planning becomes part of your config.

---

## What to persist

A starter list that covers most desktops:

```nix
environment.persistence."/persist" = {
  hideMounts = true;

  directories = [
    # Logs and system state
    "/var/log"
    "/var/lib/nixos"
    "/var/lib/systemd/coredump"
    "/var/lib/systemd/timers"

    # Networking
    "/etc/NetworkManager/system-connections"
    "/var/lib/NetworkManager"
    "/var/lib/tailscale"

    # Auth and SSH
    "/etc/ssh"

    # Hardware state
    "/var/lib/bluetooth"
    "/var/lib/upower"
    "/var/lib/alsa"

    # Desktop session data
    "/var/lib/AccountsService"
    "/var/lib/sddm"
    "/var/lib/flatpak"
    "/var/lib/cups"

    # Service data (only the ones you actually run)
    # "/var/lib/docker"
    # "/var/lib/libvirt"
    # "/var/lib/postgresql"
  ];

  files = [
    "/etc/machine-id"
    "/var/lib/dbus/machine-id"
  ];
};
```

### What absolutely must be there

- **`/etc/machine-id`** — systemd, journald, and many services key off this.
- **`/etc/ssh/ssh_host_*_key`** — otherwise every SSH client on the planet yells about changed host keys after each reboot.
- **`/var/lib/nixos`** — NixOS's UID/GID state. Without it, users may get different numeric IDs across reboots, breaking file ownership.
- **`/var/log`** — otherwise you lose the journal on every reboot, which defeats debugging.
- **`/var/lib/NetworkManager`** / **`/etc/NetworkManager/system-connections`** — known wifi networks and saved passwords.

### Typically NOT persisted (deliberate)

- `/tmp` — by definition ephemeral.
- `/var/cache/*` — caches, rebuild on demand.
- `/var/tmp` — ephemeral.
- `/run/*` — ephemeral by kernel convention.

---

## Per-user persistence (home-manager)

The impermanence repo ships a home-manager module. You choose per-user what to persist.

Flake:

```nix
inputs.impermanence.url = "github:nix-community/impermanence";

# in home.nix
{
  imports = [ inputs.impermanence.homeManagerModules.impermanence ];

  home.persistence."/persist/home/alice" = {
    allowOther = true;
    directories = [
      "Documents"
      "Downloads"
      "Music"
      "Pictures"
      "Videos"
      "Projects"
      ".ssh"
      ".gnupg"
      ".mozilla"
      ".config/kate"
      ".config/kdeconnect"
      ".config/obsidian"
      ".local/share/keyrings"      # GNOME/KDE keyrings
      ".local/share/akonadi"       # KDE PIM
      ".cache/mozilla"             # optional — caches are cheap to rebuild
    ];
    files = [
      ".bash_history"
      ".zsh_history"
      ".local/share/fish/fish_history"
    ];
  };
}
```

> `allowOther = true;` is needed for any directory that root or a service must read (e.g., keyrings, mail indexes).

You can also split by concern: one persistence block for "documents and downloads" that gets backed up aggressively, another for "caches" that doesn't.

---

## Discovering hidden state

The first week of impermanence is spent finding paths you didn't know your system used. The workflow:

1. Reboot. Notice that X is broken / forgotten.
2. Find what service writes the state: `journalctl -u <service> -b` or `lsof | grep <path>`.
3. Identify the path — usually `/var/lib/<service>/...` or `~/.local/share/<app>/...`.
4. Add it to `directories` or `files` in the persistence block.
5. Rebuild. Reboot. Verify.

Common culprits not in the starter list:

- `/var/lib/docker` — docker's image and volume storage.
- `/var/lib/libvirt` — VM XML definitions, snapshots.
- `/var/lib/postgresql` — DB data directory (consider a real backup strategy on top).
- `/var/lib/mysql` / `/var/lib/mariadb` — same.
- `/var/cache/cups` — printer cache; re-probing takes a minute on each boot otherwise.
- `/var/lib/swtpm` — if you run VMs with emulated TPM.
- `/var/lib/tailscale` — login state; otherwise you re-auth every boot.
- `/var/lib/private/*` — systemd-managed services that run with `DynamicUser=yes`.
- `~/.local/share/applications` — custom `.desktop` files you added manually.
- `~/.local/state/*` — XDG state dir, increasingly used by modern apps.

### A diagnostic command

After a reboot, see what has been newly written to root (which would be lost on next reboot):

```bash
sudo find / -xdev -type f -mtime -1 \
  -not -path "/nix/*" \
  -not -path "/persist/*" \
  -not -path "/var/log/*" \
  -not -path "/tmp/*" \
  -not -path "/run/*" \
  -not -path "/proc/*" \
  -not -path "/sys/*" \
  2>/dev/null
```

Anything in that list is being written to the *volatile* root and will vanish. Decide: persist it or ignore it.

---

## Pitfalls and gotchas

- **`neededForBoot = true;` on `/persist`.** Non-negotiable. Without it, activation starts before `/persist` is mounted and every bind-mount fails silently.
- **Secret key bootstrap.** If your decryption keys (age, SSH) live in `/persist` and are needed at boot, ordering matters. agenix / sops-nix handle this, but think through the first-boot story.
- **Database writes.** Anything chatty (PostgreSQL, MySQL, Elastic) that you persist will hammer the real disk — plan size + IOPS accordingly. Tmpfs-root people sometimes put the data directory on a separate dataset entirely.
- **machine-id churn.** If you forget to persist `/etc/machine-id`, systemd regenerates it on every boot and the entire journal history becomes unreachable (it's indexed by machine-id).
- **bluetooth / keyring / GPG agents.** These store per-user state under `~/.local/share` or `~/.config`. If you don't persist the right subpaths, you re-pair your headphones or re-enter passphrases on every boot.
- **Systemd timers with "persistent" schedules.** These rely on `/var/lib/systemd/timers` to remember when they last ran. Missing it means they all fire on every boot.
- **`allowOther = true;` when sharing with system services.** KDE wallet, Akonadi, certain file indexers need it.
- **Disko + impermanence boot order.** If you go whole-hog declarative with disko, be sure the disko layout sets labels that match `fileSystems.*.device`.

---

## Disaster recovery becomes trivial

With impermanence and a git repo, reinstallation is:

1. Boot any NixOS installer — or your [personalized live USB](tricks.md) from this chapter.
2. Partition with disko from your flake: `nix run github:nix-community/disko -- --mode disko --flake .#myhost`.
3. Restore `/persist` from backup if you want continuity, or leave empty for a truly fresh start.
4. `nixos-install --flake <your-repo>#myhost`.
5. Reboot.

The system returns bit-for-bit identical — because anything the old system relied on outside `/persist` was either (a) not state at all, or (b) already in your git repo.

Combined with [`nixos-anywhere`](tricks.md), you can even skip the installer: run the install over SSH from any machine, wiping and rebuilding the target remotely.

---

## Further reading

- [`github.com/nix-community/impermanence`](https://github.com/nix-community/impermanence) — the module's source and README.
- [Graham Christensen — *Erase your darlings*](https://grahamc.com/blog/erase-your-darlings/) — the canonical blog post that popularized ZFS rollback.
- [nixos.wiki — Impermanence](https://nixos.wiki/wiki/Impermanence)
- [`github.com/nix-community/disko`](https://github.com/nix-community/disko) — declarative partitioning that pairs naturally with impermanence.

## See also

- [configuration.md](configuration.md) — the module system and `fileSystems` options.
- [tricks.md](tricks.md) — live USB, `nixos-anywhere`, and other reinstall tools that shine with impermanence.
- [personal-setup.md](personal-setup.md) — for `ssh.hostKeys`, which must point at `/persist` when impermanence is in use.
