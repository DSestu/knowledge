# Secrets

Putting secrets in `configuration.nix` or `home.nix` does not work: every file referenced from a Nix derivation ends up in `/nix/store`, which is world-readable. This page covers how to declare secrets in the flake (under encryption) and have them decrypted at the right moment on the target machine, with no plaintext ever entering the store.

Two established tools: **agenix** and **sops-nix**. Both solve the same problem. Pick one per repo; don't mix.

## In this page

- [Threat model first](#threat-model-first)
- [What NOT to do](#what-not-to-do)
- [agenix — SSH-key-encrypted secrets](#agenix-ssh-key-encrypted-secrets)
- [sops-nix — SOPS/age-encrypted secrets](#sops-nix-sopsage-encrypted-secrets)
- [Side-by-side comparison](#side-by-side-comparison)
- [Consuming secrets in modules](#consuming-secrets-in-modules)
- [home-manager secrets](#home-manager-secrets)
- [Bootstrap and re-keying](#bootstrap-and-re-keying)
- [Persisted-secret strategies with impermanence](#persisted-secret-strategies-with-impermanence)
- [Secrets that don't belong in the flake](#secrets-that-dont-belong-in-the-flake)

---

## Threat model first

Before picking a tool, be specific about what you're defending against. The answer to "how secure is enough?" is very different across these:

- **A casual reader of my public flake repo** — keeps me honest; both tools solve this.
- **Another user on the same machine running `cat /nix/store/<hash>-...`** — both tools solve this too (neither writes plaintext into the store).
- **Root on the machine** — once root, game over, end of story. **No secret-management tool defends against a compromised root.** Both agenix and sops-nix decrypt to `/run/secrets/*` readable by root.
- **The backup of my `/persist` partition** — that's a separate question; your backup tool needs to encrypt independently.
- **A lost SSD sold on eBay** — full-disk encryption (LUKS) solves this; secrets tooling does not.

All the recipes below defend against the first two. The last three are out of scope and need other tools.

---

## What NOT to do

### Don't inline secrets as strings

```nix
# BAD — plaintext lands in /nix/store, world-readable
services.myservice.apiKey = "sk_live_abc123";
```

### Don't `readFile` from a path in the flake

```nix
# BAD — the file's content is evaluated at build time
# and copied into the store.
services.myservice.apiKey = builtins.readFile ./secrets/api-key;
```

### Don't assume `.gitignore` is enough

If a file is referenced from Nix with `./foo`, it's imported into the store regardless of git ignore status. `.gitignore` prevents commits; it doesn't prevent the store copy. The fix is always "ship only the ciphertext; decrypt into `/run/secrets/*` at activation".

### Don't put secrets in environment variables via `environment.variables`

```nix
# BAD — /etc/profile lands in the store
environment.variables.OPENAI_API_KEY = "sk-...";
```

Systemd units that need secrets in the environment should use `EnvironmentFile=` pointing at a file outside the store — which is exactly what the two tools below produce.

---

## agenix — SSH-key-encrypted secrets

[`agenix`](https://github.com/ryantm/agenix) encrypts each secret file with the public SSH keys of the machines that are allowed to read it. At activation, the matching host decrypts with its own `ssh_host_ed25519_key` into `/run/secrets/<name>`.

### Install (flake)

```nix
{
  inputs.agenix.url = "github:ryantm/agenix";
  inputs.agenix.inputs.nixpkgs.follows = "nixpkgs";

  outputs = { self, nixpkgs, agenix, ... }: {
    nixosConfigurations.myhost = nixpkgs.lib.nixosSystem {
      system = "x86_64-linux";
      modules = [
        ./configuration.nix
        agenix.nixosModules.default
      ];
    };
  };
}
```

### Declare which keys can read which secrets

One file per repo, `secrets/secrets.nix`:

```nix
let
  alice   = "ssh-ed25519 AAAAC3Nz... alice@laptop";
  desktop = "ssh-ed25519 AAAAC3Nz... root@desktop";
  vm      = "ssh-ed25519 AAAAC3Nz... root@kali-vm";

  allHosts = [ desktop vm ];
  allUsers = [ alice ];
in
{
  "wifi-password.age"       = { publicKeys = allHosts ++ allUsers; };
  "tailscale-authkey.age"   = { publicKeys = allHosts; };
  "cachix-signing-key.age"  = { publicKeys = allHosts; };
  "smtp-password.age"       = { publicKeys = [ desktop alice ]; };
}
```

The user-side key (`alice`) is needed if **you** want to edit the secret with `agenix -e`. The host keys are what the target machine will use to decrypt at activation.

### Grab the host key

Each host has a pre-existing SSH host key (`/etc/ssh/ssh_host_ed25519_key.pub`). Read its **public** part:

```bash
cat /etc/ssh/ssh_host_ed25519_key.pub
```

Paste into `secrets/secrets.nix`. If the host was just installed and OpenSSH isn't running yet, generate one:

```bash
ssh-keygen -t ed25519 -f /etc/ssh/ssh_host_ed25519_key -N ''
```

### Create a secret

```bash
cd secrets/
nix run github:ryantm/agenix -- -e wifi-password.age
```

Your `$EDITOR` opens with plaintext; save and exit; the file is written back encrypted. Commit it to git.

### Declare in a module

```nix
{ config, ... }:
{
  age.secrets.wifi-password.file = ../secrets/wifi-password.age;

  networking.wireless.environmentFile = config.age.secrets.wifi-password.path;
  # or any other consumer — the path is /run/agenix/wifi-password
}
```

### Re-keying

When you add a new host or a user rotates their key, edit `secrets/secrets.nix`, then:

```bash
cd secrets/
nix run github:ryantm/agenix -- -r     # re-encrypt all files
```

---

## sops-nix — SOPS/age-encrypted secrets

[`sops-nix`](https://github.com/Mic92/sops-nix) is the Nix integration for Mozilla's [SOPS](https://github.com/getsops/sops), which encrypts YAML/JSON/INI/env files field-by-field using AWS KMS, GCP KMS, age, PGP, or HashiCorp Vault.

In practice on a personal setup, you use SOPS with **age keys** (same primitive agenix uses), but arranged so **one** secrets file holds many values.

### Install (flake)

```nix
{
  inputs.sops-nix.url = "github:Mic92/sops-nix";
  inputs.sops-nix.inputs.nixpkgs.follows = "nixpkgs";

  outputs = { self, nixpkgs, sops-nix, ... }: {
    nixosConfigurations.myhost = nixpkgs.lib.nixosSystem {
      system = "x86_64-linux";
      modules = [
        ./configuration.nix
        sops-nix.nixosModules.sops
      ];
    };
  };
}
```

### Generate an age key for the host

```bash
# On the host:
sudo mkdir -p /var/lib/sops-nix
sudo ssh-to-age -private-key -i /etc/ssh/ssh_host_ed25519_key \
  | sudo tee /var/lib/sops-nix/key.txt
sudo chmod 600 /var/lib/sops-nix/key.txt

# Read the corresponding public key (goes into .sops.yaml):
ssh-to-age < /etc/ssh/ssh_host_ed25519_key.pub
```

The `ssh-to-age` trick converts your SSH host key into an age keypair, so you don't maintain two key infrastructures.

### `.sops.yaml` at the repo root

```yaml
keys:
  - &alice age1abc...                  # your personal age pubkey
  - &desktop age1xyz...                # desktop host, converted from ssh host key
  - &vm age1ghi...                     # kali-vm host
creation_rules:
  - path_regex: secrets/shared\.yaml$
    key_groups:
      - age: [*alice, *desktop, *vm]
  - path_regex: secrets/desktop/[^/]+\.yaml$
    key_groups:
      - age: [*alice, *desktop]
```

### Create / edit a secrets file

```bash
sops secrets/shared.yaml
```

Inside, edit a normal YAML document:

```yaml
wifi:
  home: correcthorsebatterystaple
tailscale:
  authkey: tskey-auth-...
smtp:
  password: abc123
```

SOPS encrypts each value separately on save. Commit the resulting file to git — the keys stay in plaintext (so diffs are readable), only values are ciphertext.

### Declare in a module

```nix
{ config, ... }:
{
  sops = {
    defaultSopsFile = ../secrets/shared.yaml;
    age.keyFile = "/var/lib/sops-nix/key.txt";

    secrets = {
      "wifi/home"           = { };
      "tailscale/authkey"   = { owner = "root"; };
      "smtp/password"       = { mode = "0400"; owner = "alice"; };
    };
  };

  # Consume them — path is /run/secrets/<name>
  services.tailscale.authKeyFile = config.sops.secrets."tailscale/authkey".path;
}
```

The `sops.secrets.<name>` attribute exposes `.path`; point each consumer at it.

---

## Side-by-side comparison

| Axis                               | agenix                                | sops-nix                                                |
| ---------------------------------- | ------------------------------------- | ------------------------------------------------------- |
| **One file, one secret**           | Yes — many small `.age` files         | No — one YAML, many values                              |
| **Multi-backend (KMS, Vault)**     | No                                    | Yes (age, AWS KMS, GCP KMS, PGP, Vault)                 |
| **Uses SSH host keys directly**    | Yes                                   | Via `ssh-to-age` conversion                             |
| **Diff-friendly commits**          | Whole-file ciphertext changes         | Only changed values change                              |
| **Setup complexity**               | Minimal                               | Slightly more (`.sops.yaml`, key derivation)            |
| **Ecosystem / CI compatibility**   | NixOS-only                            | SOPS files work outside Nix too (CI, Docker, Terraform) |
| **Best for**                       | Small personal fleet, few secrets     | Larger repos, many secrets, cross-tool usage            |

Rule of thumb for this repo's audience (portable personal config, 2–4 hosts): **agenix is lighter; sops-nix scales better**. Either works fine for the 10-secrets-or-fewer case.

---

## Consuming secrets in modules

Whatever tool you pick, the pattern is the same from the consumer side: the module system exposes a `.path` you pass to the service.

### Pattern 1 — service option that takes a path

```nix
services.tailscale.authKeyFile = config.age.secrets.tailscale-authkey.path;
```

Most well-designed NixOS modules expose `*KeyFile`, `*SecretFile`, `*PasswordFile`, `environmentFile`, etc. Prefer these to inline strings.

### Pattern 2 — systemd unit needs the secret in its environment

```nix
systemd.services.myservice = {
  serviceConfig.EnvironmentFile = config.sops.secrets."myservice/env".path;
};
```

The file at `.path` should be in `KEY=VALUE\nKEY2=VALUE2` format. Put it in SOPS as:

```yaml
myservice:
  env: |
    DATABASE_URL=postgres://...
    API_KEY=sk_live_...
```

### Pattern 3 — config file needs the secret interpolated

Don't interpolate at Nix eval time (lands in store). Use `templates`:

```nix
# sops-nix
sops.templates."myservice.conf".content = ''
  database_url = "${config.sops.placeholder."db/url"}"
'';

services.myservice.configFile = config.sops.templates."myservice.conf".path;
```

`placeholder.*` is replaced on the host at activation, not at Nix eval — the rendered file lives at `/run/secrets/rendered/myservice.conf`, outside the store.

agenix has a similar facility via `age.secrets.<name>.file` + manual templating in an activation script; sops-nix's `templates` is the cleaner path.

---

## home-manager secrets

Both tools have home-manager modules. The mental model is the same; scope is per-user.

### agenix + home-manager

[`agenix-rs`](https://github.com/yaxitech/ragenix) and `agenix`'s own home-manager module both exist; the NixOS-module path (decrypt system-wide, bind-mount into `~/.config/...`) is often simpler than running agenix per-user.

### sops-nix + home-manager

```nix
{ config, ... }:
{
  sops = {
    defaultSopsFile = ./secrets/alice.yaml;
    age.keyFile = "${config.home.homeDirectory}/.config/sops/age/keys.txt";
    secrets = {
      "git/github-token" = { path = "${config.home.homeDirectory}/.config/github/token"; };
    };
  };
}
```

### When to use which scope

| Secret                                             | Scope       |
| -------------------------------------------------- | ----------- |
| WiFi passwords, VPN keys, host auth keys           | System      |
| Service passwords (Postgres, SMTP, Tailscale)      | System      |
| GitHub tokens, API keys for personal CLI tools     | User (home-manager) |
| Kerberos, password manager unlock keys             | User        |

Everything a systemd unit needs → system. Everything only your shell/editor needs → user.

---

## Bootstrap and re-keying

Two moments are annoying without a plan: the first install, and rotating a host key.

### First install of a new host

1. Boot the live installer (or `nixos-anywhere`).
2. Generate the SSH host key: `ssh-keygen -t ed25519 -f /mnt/etc/ssh/ssh_host_ed25519_key -N ''`.
3. Read the **public** key and paste it into `secrets/secrets.nix` (agenix) or `.sops.yaml` (sops-nix).
4. Re-key all secrets: `agenix -r` or `sops updatekeys secrets/shared.yaml`.
5. Commit; finish the install.

### Rotating a key

Same procedure. The old ciphertext still decrypts with the old key; after `updatekeys`/`-r` it also decrypts with the new one. Once every host has rotated, remove the old key from the config.

### Lost the key

If you lose the **only** key that can decrypt a secret, the secret is gone. For important secrets (wifi passwords, VPN keys, backup encryption passphrases) always include **two** keys in `publicKeys` / `key_groups` — your laptop and a backup kept offline — so one loss is recoverable.

---

## Persisted-secret strategies with impermanence

When the root filesystem is wiped on every boot ([impermanence.md](impermanence.md)), the secret-decryption key must survive reboots too. Two approaches:

### Approach 1 — persist `/var/lib/sops-nix` (or the agenix key location)

```nix
environment.persistence."/persist".files = [
  "/etc/ssh/ssh_host_ed25519_key"
  "/etc/ssh/ssh_host_ed25519_key.pub"
  "/etc/ssh/ssh_host_rsa_key"
  "/etc/ssh/ssh_host_rsa_key.pub"
  "/var/lib/sops-nix/key.txt"             # only for sops-nix
];
```

Simplest option. The bootstrap key file is not a secret-management secret in its own right — it's the **only** key not in the flake, by necessity.

### Approach 2 — TPM-sealed key (advanced)

Bind the decryption key to the machine's TPM so it only unseals when booted with the correct firmware + kernel + initrd chain. `systemd-cryptenroll` + `clevis` are the two common tools. Adds complexity; skip unless you specifically need evil-maid resistance.

### Approach 3 — re-auth on every boot

For very short-lived hosts (ephemeral VMs, CI), don't persist secrets at all: bake in a one-use Tailscale authkey, join the tailnet, let the flake pull the rest over SSH. Works for transient infrastructure; impractical for a desktop.

---

## Secrets that don't belong in the flake

Some things shouldn't be encrypted in the repo at all:

- **Private keys you want physical safeguarding** — the backup of your `/persist` SSH host keys belongs on a YubiKey or offline USB, not in a GitHub repo (even encrypted).
- **Very short-lived credentials** — Tailscale one-time authkeys, OAuth tokens that expire in an hour. Re-generate at use time instead.
- **Machine-derived secrets** — LUKS master keys, TPM-sealed keys. Generate on first boot, never put them in the flake.
- **Anything you wouldn't want to exist as *ciphertext* in a public git history** — if leaking the ciphertext would itself be a problem (regulatory reasons, or the mere existence of a credential is sensitive), keep it out of the repo.

---

## See also

- [snippets.md](snippets.md) — the pointer in the "Secrets management" section links here.
- [impermanence.md](impermanence.md) — where the "persist the key" question lives.
- [multi-host-flake.md](multi-host-flake.md) — adding a new host's public key to the secrets list fits in the `mkHost` workflow.
- [personal-setup.md](personal-setup.md) — SSH host keys (which agenix and sops-nix both depend on) and their persistence.
- [github.com/ryantm/agenix](https://github.com/ryantm/agenix) — agenix repo, README, and examples.
- [github.com/Mic92/sops-nix](https://github.com/Mic92/sops-nix) — sops-nix repo and examples.
- [getsops.io](https://getsops.io) — SOPS upstream.
