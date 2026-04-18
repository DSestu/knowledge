# Disko — declarative partitioning

`disko` turns disk partitioning from a manual `parted` session into a Nix expression. Your flake describes the disk layout; `disko` runs on a fresh machine and creates exactly that. It's the missing piece that makes "reinstall from git" actually reinstall everything, including the disk.

This is a Level-3 concern: it matters when you install NixOS onto real hardware (or over SSH via `nixos-anywhere`). For VMs and NixOS-WSL you don't need it — the installer handles the disk and you don't want your flake second-guessing a virtualization platform's disk layout. Skip this page until you're past Level 2b.

## In this page

- [Why declarative partitioning](#why-declarative-partitioning)
- [Install and run](#install-and-run)
- [Layout 1 — single disk, EFI, ext4](#layout-1-single-disk-efi-ext4)
- [Layout 2 — single disk, EFI, btrfs with subvolumes](#layout-2-single-disk-efi-btrfs-with-subvolumes)
- [Layout 3 — single disk, LUKS-encrypted root](#layout-3-single-disk-luks-encrypted-root)
- [Layout 4 — impermanence-friendly (tmpfs root, /persist, /nix)](#layout-4-impermanence-friendly-tmpfs-root-persist-nix)
- [Layout 5 — two disks, one for Nix store, one for data](#layout-5-two-disks-one-for-nix-store-one-for-data)
- [Integrating with `nixos-anywhere`](#integrating-with-nixos-anywhere)
- [Testing a disko layout in a VM first](#testing-a-disko-layout-in-a-vm-first)
- [Common traps](#common-traps)

---

## Why declarative partitioning

Without disko, reinstallation goes:

1. Boot installer.
2. Partition by hand, remember which UUIDs you assigned to what.
3. Format filesystems.
4. Mount everything.
5. Run `nixos-install`.

With disko:

1. `nix run github:nix-community/disko -- --mode disko --flake .#myhost`
2. Run `nixos-install`.

The partition layout is part of the flake, version-controlled, identical across reinstalls. On the **external-SSD** strategy from [windows-to-nixos.md](windows-to-nixos.md) and [kali-to-nixos.md](kali-to-nixos.md), this means the SSD's layout is in your repo — reformat it at any time without remembering what you did last time.

---

## Install and run

### As a one-shot from the installer

```bash
# Boot the NixOS live installer, then:
nix run github:nix-community/disko -- --mode disko --flake .#myhost
nixos-install --flake .#myhost
```

`--mode disko` wipes and recreates. The alternative is `--mode mount`, which mounts an already-partitioned disk at the expected paths (used during nixos-anywhere reinstalls where the layout already exists).

### Wire into the flake

```nix
{
  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-24.11";
    disko.url = "github:nix-community/disko";
    disko.inputs.nixpkgs.follows = "nixpkgs";
  };

  outputs = { self, nixpkgs, disko, ... }: {
    nixosConfigurations.myhost = nixpkgs.lib.nixosSystem {
      system = "x86_64-linux";
      modules = [
        disko.nixosModules.disko
        ./hosts/myhost/configuration.nix
        ./hosts/myhost/disk.nix            # ← the disko layout
      ];
    };
  };
}
```

The `disko.nixosModules.disko` module translates your `disko.devices = { ... }` declaration into the `fileSystems.*` entries NixOS needs. You write the layout **once**; both disko-the-tool (for formatting) and NixOS (for mounting at boot) read it.

---

## Layout 1 — single disk, EFI, ext4

The simplest modern layout: one EFI System Partition, one ext4 root, no swap (use zram if you want compressed swap).

`hosts/myhost/disk.nix`:

```nix
{ lib, ... }:
{
  disko.devices = {
    disk.main = {
      type = "disk";
      device = "/dev/disk/by-id/nvme-CT1000P5PSSD8_...";  # by-id, never /dev/sda
      content = {
        type = "gpt";
        partitions = {
          ESP = {
            size = "512M";
            type = "EF00";
            content = {
              type = "filesystem";
              format = "vfat";
              mountpoint = "/boot";
              mountOptions = [ "umask=0077" ];
            };
          };
          root = {
            size = "100%";
            content = {
              type = "filesystem";
              format = "ext4";
              mountpoint = "/";
            };
          };
        };
      };
    };
  };
}
```

> **Always use `/dev/disk/by-id/...`**, not `/dev/sda` or `/dev/nvme0n1`. Kernel device enumeration order is not stable; `by-id` is. The one exception is a live installer where the target disk is identified interactively before the flake exists — but by the time you're running disko from the flake, use `by-id`.

---

## Layout 2 — single disk, EFI, btrfs with subvolumes

Btrfs gives you snapshots, compression, and copy-on-write for free. The subvolume pattern below separates `/`, `/home`, `/nix`, and `/persist` so each can be snapshotted independently.

```nix
{
  disko.devices = {
    disk.main = {
      type = "disk";
      device = "/dev/disk/by-id/nvme-...";
      content = {
        type = "gpt";
        partitions = {
          ESP = {
            size = "512M";
            type = "EF00";
            content = {
              type = "filesystem";
              format = "vfat";
              mountpoint = "/boot";
            };
          };
          root = {
            size = "100%";
            content = {
              type = "btrfs";
              extraArgs = [ "-L" "nixos" "-f" ];   # label + force overwrite
              subvolumes = {
                "/rootfs" = {
                  mountpoint = "/";
                  mountOptions = [ "compress=zstd" "noatime" ];
                };
                "/home" = {
                  mountpoint = "/home";
                  mountOptions = [ "compress=zstd" "noatime" ];
                };
                "/nix" = {
                  mountpoint = "/nix";
                  mountOptions = [ "compress=zstd" "noatime" ];
                };
                "/persist" = {
                  mountpoint = "/persist";
                  mountOptions = [ "compress=zstd" "noatime" ];
                };
                "/swap" = {
                  mountpoint = "/.swapvol";
                  swap.swapfile.size = "16G";
                };
              };
            };
          };
        };
      };
    };
  };
}
```

Later, `btrfs subvolume snapshot` / `btrfs send` gives you point-in-time restores outside Nix's own generation system — useful for `/home` in particular, which Nix doesn't manage.

---

## Layout 3 — single disk, LUKS-encrypted root

Full-disk encryption, prompted for passphrase at boot. EFI stays unencrypted (it has to — the bootloader lives there).

```nix
{
  disko.devices = {
    disk.main = {
      type = "disk";
      device = "/dev/disk/by-id/nvme-...";
      content = {
        type = "gpt";
        partitions = {
          ESP = {
            size = "512M";
            type = "EF00";
            content = {
              type = "filesystem";
              format = "vfat";
              mountpoint = "/boot";
            };
          };
          luks = {
            size = "100%";
            content = {
              type = "luks";
              name = "cryptroot";
              settings.allowDiscards = true;        # TRIM through LUKS for SSDs
              content = {
                type = "filesystem";
                format = "ext4";
                mountpoint = "/";
              };
            };
          };
        };
      };
    };
  };
}
```

At boot, systemd-initrd (or the classic initrd) prompts for the passphrase, unlocks, mounts. No additional `configuration.nix` needed — disko generates the right `boot.initrd.luks.devices.*` entries.

### Unlock with a TPM or FIDO2 key

```nix
luks.settings = {
  allowDiscards = true;
  crypttabExtraOpts = [ "tpm2-device=auto" ];   # or "fido2-device=auto"
};
```

Enrol the key once after install with `systemd-cryptenroll --tpm2-device=auto /dev/nvme0n1p2` (or similar). Now the passphrase is only needed as a fallback.

### LUKS + a separate unencrypted `/nix`

Common optimization: the Nix store is rebuild-able cache, doesn't need encryption, and skipping it makes unlock-time disk I/O much faster. Trade-off: anyone with physical access can see what packages you have installed (not your data, just the package list).

```nix
partitions = {
  ESP = { /* ... */ };
  nix = {
    size = "100G";
    content = { type = "filesystem"; format = "ext4"; mountpoint = "/nix"; };
  };
  luks = {
    size = "100%";
    content = {
      type = "luks";
      name = "cryptroot";
      content = { type = "filesystem"; format = "ext4"; mountpoint = "/"; };
    };
  };
};
```

---

## Layout 4 — impermanence-friendly (tmpfs root, /persist, /nix)

Paired with [impermanence.md](impermanence.md). Root is a tmpfs (wiped on every boot); real partitions hold `/nix` (Nix store) and `/persist` (declared state).

```nix
{
  disko.devices = {
    disk.main = {
      type = "disk";
      device = "/dev/disk/by-id/nvme-...";
      content = {
        type = "gpt";
        partitions = {
          ESP = {
            size = "512M";
            type = "EF00";
            content = {
              type = "filesystem";
              format = "vfat";
              mountpoint = "/boot";
            };
          };
          nix = {
            size = "100G";
            content = {
              type = "filesystem";
              format = "ext4";
              mountpoint = "/nix";
              mountOptions = [ "noatime" ];
            };
          };
          persist = {
            size = "100%";
            content = {
              type = "filesystem";
              format = "ext4";
              mountpoint = "/persist";
              mountOptions = [ "noatime" ];
            };
          };
        };
      };
    };
  };

  # Root as tmpfs is declared in the NixOS config, not disko.
  # See impermanence.md for the fileSystems."/" tmpfs entry.
}
```

The `configuration.nix` side then declares `fileSystems."/"` as tmpfs and sets `fileSystems."/persist".neededForBoot = true;` — see [impermanence.md](impermanence.md) for the full picture.

---

## Layout 5 — two disks, one for Nix store, one for data

Desktop scenario: a fast small NVMe for the Nix store + root, a large spinning disk for `/home` and media.

```nix
{
  disko.devices = {
    disk.nvme = {
      type = "disk";
      device = "/dev/disk/by-id/nvme-...";
      content = {
        type = "gpt";
        partitions = {
          ESP = { size = "512M"; type = "EF00"; content = { type = "filesystem"; format = "vfat"; mountpoint = "/boot"; }; };
          root = { size = "100%"; content = { type = "filesystem"; format = "ext4"; mountpoint = "/"; }; };
        };
      };
    };

    disk.data = {
      type = "disk";
      device = "/dev/disk/by-id/ata-ST4000DM004-...";
      content = {
        type = "gpt";
        partitions = {
          data = {
            size = "100%";
            content = {
              type = "filesystem";
              format = "ext4";
              mountpoint = "/home";
              mountOptions = [ "noatime" ];
            };
          };
        };
      };
    };
  };
}
```

For a LUKS-encrypted `/home` on the data disk, wrap the inner partition in a `luks` block as in Layout 3.

---

## Integrating with `nixos-anywhere`

`nixos-anywhere` runs disko automatically on the target before installing:

```bash
nix run github:nix-community/nixos-anywhere -- \
  --flake .#myhost \
  root@target.example.com
```

What happens:

1. `nixos-anywhere` uploads a "kexec installer" over SSH and kexecs into it (replacing the running OS with a tiny NixOS in RAM).
2. It runs disko with your `disk.nix` — wiping and partitioning the target's disk.
3. It copies the closure from the build host into the new `/mnt`.
4. It runs `nixos-install`.
5. Reboot into the new system.

Time from `nix run` to a booted NixOS: typically 5–15 minutes depending on bandwidth.

> Same warning as always: **`nixos-anywhere` erases the target**. Never run it against a machine whose disk you want to keep. For the external-SSD workflow, it's the right tool — the SSD is the disposable target.

---

## Testing a disko layout in a VM first

Before running disko against real hardware, try it in a VM. `disko` ships a `runVMWithDisko` test harness, or you can use a simple QEMU loop:

```bash
# Create a scratch disk image
qemu-img create -f qcow2 test.qcow2 50G

# Run disko against it inside a VM, then boot and verify
nix run github:nix-community/disko -- --mode disko --flake .#myhost \
  --arg device '"/dev/vda"'
```

Alternatively, build a VM image of the whole thing and boot it:

```bash
nix build .#nixosConfigurations.myhost.config.system.build.diskoImages
./result/main.qcow2     # or whatever disko produced
qemu-system-x86_64 -enable-kvm -m 4G -drive file=./result/main.qcow2
```

This catches 90% of layout bugs (mount order, filesystem compatibility, labels) before you point disko at anything that matters.

---

## Common traps

- **Using `/dev/sda` instead of `/dev/disk/by-id/...`** — pick a different USB port and you wipe the wrong disk. Always by-id.
- **Forgetting `type = "EF00";` on the ESP** — the partition type GUID matters for UEFI to find it. `EF00` is the shorthand disko accepts; the underlying GUID is `c12a7328-f81f-11d2-ba4b-00a0c93ec93b`.
- **Mount-order mistakes** — when you have `/`, `/home`, `/nix`, `/persist`, each mountpoint's parent must mount first. Disko handles ordering automatically if you declare the layout correctly; mistakes tend to manifest as "bind-mount target does not exist".
- **Re-running `--mode disko` on a machine with data** — wipes. Use `--mode mount` on existing installs.
- **Mixing disko and manual `fileSystems.*` entries for the same path** — the merge is confusing and rarely does what you want. Pick one source of truth per mountpoint; disko is the source of truth when it's in the flake.
- **BitLocker-style resizable partitions** — disko is for first-install layouts, not for in-place resize of an existing install. Use standard tools (gparted, `btrfs balance`) for that.
- **Swap on encrypted disk without hibernation support** — if you need hibernate, the swap partition has to be unlocked before resume. Easiest: put swap *inside* the LUKS container and configure `boot.resumeDevice` accordingly; hardest: plain swap partition (unencrypted, leaks RAM to disk on hibernate). Pick consciously.

---

## See also

- [impermanence.md](impermanence.md) — the most common reason people reach for disko.
- [tricks.md](tricks.md) — `nixos-anywhere` from the tooling side.
- [windows-to-nixos.md](windows-to-nixos.md) and [kali-to-nixos.md](kali-to-nixos.md) — the external-SSD workflow, where disko is the partition step.
- [multi-host-flake.md](multi-host-flake.md) — how to slot `disk.nix` into the `hosts/<name>/` directory pattern.
- [github.com/nix-community/disko](https://github.com/nix-community/disko) — upstream, with many more example layouts (RAID, LVM, ZFS pools, bcachefs, tmpfs-on-real-FS).
