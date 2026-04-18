# Personal setup

This is the worked example that ties together every piece up to this point: once `configuration.nix` and `home.nix` are real (Level 2b or 3 in the [four-level framing](getting-started.md#the-four-safe-levels-of-commitment)), this is what goes in them. Every recipe here is the declarative translation of the dotfiles + setup steps from [Linux → Tools](../linux/tools.md).

Almost everything below is **equally valid in Level 2a** (home-manager standalone on WSL2 Debian or Kali). The `programs.git`, `programs.fish`, `programs.ssh`, `programs.eza`, `programs.yazi`, `programs.micro` modules live entirely inside `home.nix`; copy those sections wholesale into a Kali or WSL2 `home.nix` and they work without any `configuration.nix`. See [home-manager-modes.md](home-manager-modes.md) for the split.

The three scopes used on this page:

- **System level** (`configuration.nix`) — anything that affects all users, listens on a port, or needs root. SSH server, firewall, globally installed binaries, default shell choice. **NixOS only.**
- **User level — home-manager** (`home.nix`) — your shell plugins, prompt, editor, aliases, `~/.ssh/config`, theme files. Per user, no root needed. **Portable across NixOS, Kali, WSL2 Debian.**
- **User level — Plasma** (`programs.plasma` in `home.nix`) — panel layouts, shortcuts, KDE theme. Only applies on hosts that run KDE, so skip on headless / WSL hosts. See [kde.md](kde.md).

## In this page

- [SSH server](#ssh-server)
- [SSH client config (`~/.ssh/config` declaratively)](#ssh-client-config-ssh-config-declaratively)
- [Firewall / open ports](#firewall-open-ports)
- [Fish — translating linux/tools.md to declarative](#fish-translating-linux-tools-md-to-declarative)
- [EZA — aliases and theme](#eza-aliases-and-theme)
- [Yazi](#yazi)
- [Auto-launch byobu on SSH](#auto-launch-byobu-on-ssh)
- [Micro editor](#micro-editor)
- [Git preferences](#git-preferences)
- [Syncthing — shared folders across all my computers](#syncthing-shared-folders-across-all-my-computers)
- [Tailscale — zero-config private VPN between my computers](#tailscale-zero-config-private-vpn-between-my-computers)

---

## SSH server

### Minimal, secure default

```nix
services.openssh = {
  enable = true;
  openFirewall = true;            # opens TCP 22 (or whatever `ports` is) automatically
  settings = {
    PasswordAuthentication = false;
    PermitRootLogin = "no";
    KbdInteractiveAuthentication = false;
    X11Forwarding = false;
  };
};

# Public keys that are allowed to log in as your user
users.users.alice.openssh.authorizedKeys.keys = [
  "ssh-ed25519 AAAAC3Nz... alice@laptop"
  "ssh-ed25519 AAAAC3Nz... alice@desktop"
];
```

> `authorizedKeys.keys` is declarative — the list is the source of truth. Anything else anyone drops into `~/.ssh/authorized_keys` is overwritten at the next activation. That is usually what you want.

### Non-standard port

```nix
services.openssh = {
  enable = true;
  ports = [ 2222 ];
  openFirewall = true;          # opens whatever you put in `ports`
};
```

### Listen only on specific interface (e.g. Tailscale)

```nix
services.openssh = {
  enable = true;
  openFirewall = false;                        # don't blanket-open
  listenAddresses = [
    { addr = "100.64.0.1"; port = 22; }        # your Tailscale IP
  ];
};

networking.firewall.interfaces.tailscale0.allowedTCPPorts = [ 22 ];
```

### Host keys persisted across reinstalls

By default NixOS generates fresh SSH host keys on first boot. If you reinstall and don't preserve `/etc/ssh/ssh_host_*`, every client will yell about changed host keys. Persist them:

```nix
services.openssh.hostKeys = [
  { path = "/persist/ssh/ssh_host_ed25519_key"; type = "ed25519"; }
  { path = "/persist/ssh/ssh_host_rsa_key";     type = "rsa"; bits = 4096; }
];
```

Copy them from the old system (or treat them as recoverable secrets via sops-nix / agenix).

---

## SSH client config (`~/.ssh/config` declaratively)

Via home-manager's `programs.ssh`:

```nix
# in home.nix
programs.ssh = {
  enable = true;

  matchBlocks = {
    "github.com" = {
      user = "git";
      identityFile = "~/.ssh/id_ed25519_github";
      identitiesOnly = true;
    };

    "prod-*" = {
      user = "deploy";
      identityFile = "~/.ssh/id_ed25519_prod";
      proxyJump = "bastion";
      forwardAgent = false;
    };

    "bastion" = {
      hostname = "bastion.example.com";
      user = "alice";
      port = 2222;
      identityFile = "~/.ssh/id_ed25519";
    };

    "*.internal" = {
      user = "alice";
      identityFile = "~/.ssh/id_ed25519_internal";
      extraOptions = {
        StrictHostKeyChecking = "accept-new";
      };
    };
  };

  extraConfig = ''
    AddKeysToAgent yes
    ServerAliveInterval 60
  '';
};
```

home-manager writes the expected `~/.ssh/config` with proper permissions (0600).

### SSH agent (keys unlocked once per session)

On NixOS, start the user-level agent via systemd:

```nix
# in home.nix
services.ssh-agent.enable = true;
```

And make sure your shell sees it:

```nix
home.sessionVariables = {
  SSH_AUTH_SOCK = "$XDG_RUNTIME_DIR/ssh-agent";
};
```

Keys configured with `AddKeysToAgent yes` in `~/.ssh/config` will be cached on first use.

> Alternatively, KDE's `kwallet-ssh` agent handles this graphically — enabled by default when KDE Wallet is running. Pick one; don't run both.

---

## Firewall / open ports

NixOS ships an nftables/iptables firewall controlled declaratively from `configuration.nix`.

### Basics

```nix
networking.firewall = {
  enable = true;
  allowedTCPPorts = [ 22 80 443 ];
  allowedUDPPorts = [ 51820 ];                # wireguard
  allowedTCPPortRanges = [ { from = 8000; to = 8100; } ];
};
```

### Per-interface rules

Open a port only on your LAN interface, not the WAN:

```nix
networking.firewall.interfaces.enp3s0 = {
  allowedTCPPorts = [ 8080 ];
};

networking.firewall.interfaces.tailscale0 = {
  allowedTCPPorts = [ 22 53 3000 ];
};
```

### Trusted interfaces (skip firewall entirely)

Useful for VPNs where you trust anyone on the tunnel:

```nix
networking.firewall.trustedInterfaces = [ "tailscale0" "wg0" ];
```

### Auto-open when enabling a service

Most NixOS service modules have an `openFirewall` option that handles the port for you:

```nix
services.openssh.openFirewall = true;
services.transmission.openFirewall = true;
services.syncthing.openFirewall = true;
services.jellyfin.openFirewall = true;
```

Prefer this over hardcoding port numbers — when the service's default port changes upstream, your firewall follows.

### ICMP / ping

```nix
networking.firewall.allowPing = true;       # default: true
```

### Fully disable (desktop on trusted network)

```nix
networking.firewall.enable = false;
```

> The KDE Connect module opens its own range (`programs.kdeconnect.enable = true;` handles that). Same for Steam's `remotePlay.openFirewall` and `dedicatedServer.openFirewall` flags — let the modules manage their ports.

### Inspect live state

```bash
sudo nft list ruleset                       # full nftables view
sudo iptables -L -n -v                      # if iptables compatibility enabled
```

---

## Fish — translating [linux/tools.md](../linux/tools.md) to declarative

The existing Fish setup in `linux/tools.md` uses `fisher` to install plugins and hand-edits `~/.config/fish/config.fish`. On NixOS, all of that moves into home-manager — and plugins come from nixpkgs, not fisher.

> Wondering where the option names like `shellAbbrs` and `interactiveShellInit` come from? The "Discovering what options a module exposes" section in [configuration.md](configuration.md) walks through how to find them.

### System-wide shell enablement

In `configuration.nix`:

```nix
programs.fish.enable = true;
users.users.alice.shell = pkgs.fish;
```

> `programs.fish.enable` must be set system-wide even if you configure the rest via home-manager — it installs the completion scripts and wires fish into `/etc/shells`.

### Full home-manager fish config

In `home.nix`:

```nix
{ pkgs, ... }:
{
  programs.fish = {
    enable = true;

    plugins = [
      { name = "tide";                src = pkgs.fishPlugins.tide.src; }
      { name = "fzf-fish";             src = pkgs.fishPlugins.fzf-fish.src; }
      { name = "z";                    src = pkgs.fishPlugins.z.src; }
      { name = "autopair";             src = pkgs.fishPlugins.autopair.src; }
      { name = "abbreviation-tips";    src = pkgs.fishPlugins.fish-abbreviation-tips.src; }
      { name = "done";                 src = pkgs.fishPlugins.done.src; }
    ];

    shellAbbrs = {
      g  = "git";
      gs = "git status";
      gp = "git pull --rebase";
      gc = "git commit -v";
      gco = "git checkout";
      gd = "git diff";
    };

    shellAliases = {
      l  = "eza -Bhm --icons --no-user --git --time-style long-iso --group-directories-first --color=always --color-scale=age -F --no-permissions -s extension --git-ignore";
      la = "l -a";
      ll = "l -la";
      lt = "ll -T";
    };

    interactiveShellInit = ''
      set -U fish_greeting ""

      # ctrl-backspace → delete previous word (terminal sends ctrl-h)
      bind \cH backward-kill-word

      # !! last command, !$ last argument of last command
      function bind_bang
        switch (commandline -t)[-1]
          case "!"
            commandline -t -- $history[1]
            commandline -f repaint
          case "*"
            commandline -i !
        end
      end

      function bind_dollar
        switch (commandline -t)[-1]
          case "!"
            commandline -f backward-delete-char history-token-search-backward
          case "*"
            commandline -i '$'
        end
      end

      function fish_user_key_bindings
        bind ! bind_bang
        bind '$' bind_dollar
      end
    '';

    functions = {
      y = ''
        set tmp (mktemp -t "yazi-cwd.XXXXXX")
        command yazi $argv --cwd-file="$tmp"
        if read -z cwd < "$tmp"; and [ "$cwd" != "$PWD" ]; and test -d "$cwd"
          builtin cd -- "$cwd"
        end
        rm -f -- "$tmp"
      '';
    };
  };
}
```

One `home-manager switch` later, everything from the Debian recipe is applied, reproducibly, without ever running `fisher install`.

> `fisher` is unnecessary on Nix — packaging a plugin means adding it to `fishPlugins` in nixpkgs (or referencing a fetched src). [search.nixos.org/packages](https://search.nixos.org/packages) with query `fishPlugins` lists ~60 maintained plugins.

### Tide prompt configuration

Tide stores its config in universal variables the first time you run `tide configure`. That's imperative. To keep it declarative, either re-run `tide configure` after each reinstall, or pin the chosen style via home-manager:

```nix
programs.fish.interactiveShellInit = ''
  set -U tide_left_prompt_items pwd git newline character
  set -U tide_right_prompt_items status cmd_duration context jobs time
  set -U tide_pwd_color_dirs blue
  # ... (rest of tide_* variables after you've tuned with `tide configure`)
'';
```

After `tide configure`, dump the current values with `set -U | grep tide_` and paste them into the snippet.

### Auto-launch Fish from Zsh (no longer needed on NixOS)

On NixOS, set Fish as your user's login shell directly:

```nix
users.users.alice.shell = pkgs.fish;
```

No more `exec fish -l` hack in `~/.zshrc`.

---

## EZA — aliases and theme

`eza` is installed and enabled via home-manager, with the theme you already have in [linux/tools.md](../linux/tools.md):

```nix
# in home.nix
programs.eza = {
  enable = true;
  enableFishIntegration = true;     # adds a default `ls` → `eza` abbr
  git = true;
  icons = "auto";
  extraOptions = [ "--group-directories-first" "--header" ];
};

# Theme file — drop the YAML from linux/tools.md here verbatim
xdg.configFile."eza/theme.yml".source = ./files/eza-theme.yml;
```

Keep `eza-theme.yml` alongside `home.nix` under `./files/` (or inline it with `.text = '' ... ''`). The theme is then committed to your repo — one less piece of state to migrate manually.

---

## Yazi

```nix
# in home.nix
programs.yazi = {
  enable = true;
  enableFishIntegration = true;           # installs the `y` wrapper function
  settings = {
    manager = {
      show_hidden = true;
      show_symlink = true;
    };
  };
};
```

`enableFishIntegration = true` already provides the `y` wrapper (same behavior as the one in [linux/tools.md](../linux/tools.md)), so you can remove the hand-rolled version from the fish `functions` block if you prefer the module's.

---

## Auto-launch byobu on SSH

The tools.md recipe appends a line to `~/.zprofile`. In declarative Fish:

```nix
# in home.nix
programs.fish.loginShellInit = ''
  if status is-login; and set -q SSH_TTY; and not set -q BYOBU_BACKEND
    exec byobu
  end
'';
```

Install byobu system-wide:

```nix
# in configuration.nix
environment.systemPackages = with pkgs; [ byobu tmux ];
```

---

## Micro editor

`cargo install`-free:

```nix
environment.systemPackages = with pkgs; [ micro ];
environment.variables.EDITOR = "micro";
```

Or, via home-manager if you want it user-only with config:

```nix
programs.micro = {
  enable = true;
  settings = {
    colorscheme = "atomdark";
    tabsize = 2;
    softwrap = true;
    savehistory = true;
  };
};
```

---

## Git preferences

Completing the picture — usually lives in home-manager, not system config:

```nix
programs.git = {
  enable = true;
  lfs.enable = true;
  userName = "Alice Example";
  userEmail = "alice@example.com";
  extraConfig = {
    init.defaultBranch = "main";
    pull.rebase = true;
    push.autoSetupRemote = true;
    rerere.enabled = true;
    diff.algorithm = "histogram";
  };
  aliases = {
    st = "status";
    co = "checkout";
    br = "branch";
    lg = "log --oneline --graph --decorate";
  };
  ignores = [ ".direnv" ".envrc.local" "*.swp" ".DS_Store" ];
};
```

---

## Syncthing — shared folders across all my computers

[Syncthing](https://syncthing.net) is a peer-to-peer continuous file sync between devices (laptop, desktop, phone, server). No central server, end-to-end encrypted, declaratively configurable on NixOS.

### Enable the service

```nix
# in configuration.nix, on every machine that should participate
services.syncthing = {
  enable = true;
  user = "alice";
  group = "users";
  dataDir = "/home/alice";                       # where synced folders live
  configDir = "/home/alice/.config/syncthing";
  openDefaultPorts = true;                       # opens 22000/tcp + 22000/udp + 21027/udp
  guiAddress = "127.0.0.1:8384";                 # loopback-only Web GUI
};
```

Rebuild and the daemon runs as your user. `systemctl status syncthing.service` should show it healthy.

### Step 1 — get each machine's device ID

On every machine, after the first rebuild:

```bash
syncthing --device-id
# e.g. ABCD1234-EFGH5678-IJKL9012-MNOP3456-QRST7890-UVWX1234-YZAB5678-CDEF9012
```

> The device ID is derived from the machine's Syncthing TLS cert (which lives under `configDir`). It stays stable across reboots and even across NixOS reinstalls *if* you persist `configDir`. With impermanence, put `/home/alice/.config/syncthing` in your persistence list — or declare `cert` and `key` with agenix/sops-nix for truly portable identities.

Write every machine's ID somewhere handy (a `devices.nix` you import from each host).

### Step 2 — declare devices and folders

```nix
# devices.nix — a shared module imported by every host
{ lib, ... }:
{
  services.syncthing.settings = {
    devices = {
      "laptop"   = { id = "ABCD1234-EFGH5678-...LAPTOP...";  };
      "desktop"  = { id = "WXYZ9876-VUTS5432-...DESKTOP..."; };
      "nas"      = { id = "MNOP1234-QRST5678-...NAS...";     };
    };

    folders = {
      "documents" = {
        path = "/home/alice/Documents";
        devices = [ "laptop" "desktop" "nas" ];
        versioning = {
          type = "staggered";
          params = {
            cleanInterval = "3600";
            maxAge = "31536000";                 # 1 year in seconds
          };
        };
      };

      "photos" = {
        path = "/home/alice/Pictures";
        devices = [ "laptop" "nas" ];            # not synced to desktop
        type = "sendreceive";
      };

      "dotfiles" = {
        path = "/home/alice/Dotfiles";
        devices = [ "laptop" "desktop" ];
        ignorePerms = true;
      };
    };
  };
}
```

Import it from each host's `configuration.nix`:

```nix
imports = [ ./devices.nix ];
```

> Each folder is identified by its **key** (`"documents"` above). The same key on different machines means "this is the same folder" — Syncthing will pair them automatically. Use descriptive names, not hashes.

### Step 3 — override the Web GUI

Syncthing's Web GUI lets users accept devices and folders imperatively. By default NixOS *merges* your declared config with GUI changes, so manual additions survive a rebuild. If you want truly declarative configs (GUI changes wiped on rebuild):

```nix
services.syncthing = {
  overrideDevices = true;     # GUI-added devices wiped on next rebuild
  overrideFolders = true;     # GUI-added folders wiped on next rebuild
};
```

This is what you want once you're comfortable declaring everything in Nix. Until then, leave them `false` so GUI exploration survives rebuilds.

### Step 4 — access the Web GUI from another machine

By default the GUI is loopback-only (secure). To access from a laptop on the same Tailscale network:

```nix
services.syncthing.guiAddress = "100.64.0.1:8384";   # your Tailscale IP
```

Or use SSH port-forwarding for a one-off: `ssh -L 8384:localhost:8384 myhost`.

### Step 5 — ignore patterns (the `.stignore` file)

Per-folder ignore rules live in a `.stignore` file at each folder root. On NixOS you'd ideally declare them:

```nix
services.syncthing.settings.folders."dotfiles" = {
  path = "/home/alice/Dotfiles";
  devices = [ "laptop" "desktop" ];
  # .stignore contents:
  ignore = [
    ".direnv"
    ".venv"
    "node_modules"
    "*.log"
    ".DS_Store"
  ];
};
```

> The `ignore` list here gets written into the folder's `.stignore` at activation. Syncthing rereads it on the next rescan.

### Versioning strategies

Per-folder, inside `settings.folders.<name>.versioning`:

| Type                 | What it keeps                                                                  |
| -------------------- | ------------------------------------------------------------------------------ |
| `trashcan`           | Deleted files go to `.stversions/` for N days.                                 |
| `simple`             | Last N versions of each file.                                                  |
| `staggered`          | Exponentially thinning history (hour/day/week/…) up to `maxAge`.               |
| `external`           | Hand off to a script (e.g., backup to Borg).                                   |

Example for `trashcan`:

```nix
versioning = {
  type = "trashcan";
  params.cleanoutDays = "30";
};
```

### The recommended fleet pattern

```
~/nixos-config/
├── flake.nix
├── hosts/
│   ├── laptop/configuration.nix
│   ├── desktop/configuration.nix
│   └── nas/configuration.nix
└── modules/
    └── syncthing.nix       ← shared devices + folders, imported by all hosts
```

One source of truth, every machine imports it. Add a new machine → add its device ID to `modules/syncthing.nix` and `nixos-rebuild switch` on each existing machine. No Web GUI clicks ever.

> This is exactly the layout documented in [multi-host-flake.md](multi-host-flake.md). The Syncthing module is a small, concrete example of why that structure pays off — you want the device list defined once, not copy-pasted into every `configuration.nix`.

### Android / iOS / Windows / Mac

Syncthing has native clients for all of them; they just need to know your device IDs. No NixOS-specific wiring — install the app, scan the QR code shown by `syncthing --device-id`, accept the peering. The mobile apps ship with sensible defaults (WiFi-only, battery-aware).

---

## Tailscale — zero-config private VPN between my computers

[Tailscale](https://tailscale.com) is a zero-config mesh VPN built on WireGuard. Every device gets a stable `100.64.0.0/10` IP; connections punch through NATs automatically; MagicDNS gives you `myhost` instead of `100.64.3.12`. A NixOS-native alternative to Syncthing for remote access and to ZeroTier for fleet networking.

### Enable the service

```nix
# on every machine
services.tailscale = {
  enable = true;
  useRoutingFeatures = "client";        # or "server" for exit nodes / subnet routers
};

# Trust anyone on the tailnet — same as your LAN, no firewall between Tailscale peers
networking.firewall.trustedInterfaces = [ "tailscale0" ];
```

### Step 1 — interactive login (first time on each machine)

```bash
sudo tailscale up
```

This opens a URL; sign in with your existing Tailscale account (Google / GitHub / Microsoft / email). After the first auth, the device key is remembered forever (unless you `tailscale logout` or the tailnet admin expires it).

### Step 1, alternative — auth key (for headless / reproducible)

Generate a pre-auth key at [login.tailscale.com/admin/settings/keys](https://login.tailscale.com/admin/settings/keys). Mark it *reusable* if you need to auth several machines, or *ephemeral* for short-lived hosts (CI, VMs).

Store the key as a secret (agenix / sops-nix), then:

```nix
services.tailscale = {
  enable = true;
  authKeyFile = config.age.secrets.tailscale-authkey.path;
  # or: config.sops.secrets."tailscale/authkey".path
  extraUpFlags = [ "--ssh" "--accept-routes" ];
};
```

Now a fresh `nixos-install` joins the tailnet automatically on first boot, no interactive step.

> The `authKeyFile` points at a file the Tailscale unit reads at startup. It's one-use (after the machine is registered, the file can be gone). sops-nix / agenix provides the file at `/run/secrets/*` securely.

### MagicDNS — call machines by name

Enable it once in the [Tailscale admin panel](https://login.tailscale.com/admin/dns) (tailnet-wide setting). Each device becomes reachable as `<hostname>` (or `<hostname>.tailXXXX.ts.net`):

```bash
ssh alice@laptop          # if MagicDNS is on
ssh alice@laptop.tailXXXX.ts.net
```

Match the hostname you want Tailscale to use with your NixOS hostname:

```nix
networking.hostName = "laptop";     # drives both MagicDNS and local hostname
```

### Tailscale SSH — drop `~/.ssh/authorized_keys` entirely

Tailscale can authenticate SSH for you based on tailnet identity — no SSH keys to exchange, no passwords. Add `--ssh` to `extraUpFlags` (or run `sudo tailscale up --ssh` once):

```nix
services.tailscale.extraUpFlags = [ "--ssh" ];
```

Then `ssh alice@laptop` from any machine on the same tailnet succeeds based on the tailnet ACL, not on `~/.ssh/authorized_keys`. You can restrict who can SSH to what in the [ACL editor](https://login.tailscale.com/admin/acls).

### Exit node — route all traffic via a specific machine

Make one machine act as an exit node:

```nix
# on the "gateway" machine
services.tailscale.useRoutingFeatures = "server";
services.tailscale.extraUpFlags = [ "--advertise-exit-node" ];
boot.kernel.sysctl."net.ipv4.ip_forward" = 1;
boot.kernel.sysctl."net.ipv6.conf.all.forwarding" = 1;
```

Approve it in the admin console. Then on any client:

```bash
sudo tailscale up --exit-node=gateway --exit-node-allow-lan-access
```

All traffic leaves through `gateway`, while LAN traffic (printer, NAS) bypasses it.

### Subnet router — expose a LAN to the tailnet

Make one machine advertise a LAN subnet so tailnet peers can reach LAN-only devices (e.g. your printer at `192.168.1.50`):

```nix
# on the router machine
services.tailscale.useRoutingFeatures = "server";
services.tailscale.extraUpFlags = [ "--advertise-routes=192.168.1.0/24" ];
boot.kernel.sysctl."net.ipv4.ip_forward" = 1;
```

Approve in admin. Remote peers add `--accept-routes` (see the minimal config above — it's there by default in `extraUpFlags`) and can now reach `192.168.1.x` through the router machine.

### ACLs (who can talk to whom)

The admin console's **Access Controls** tab holds a JSON ACL that restricts tailnet traffic. Default is "everyone can talk to everyone"; tighten it up for shared tailnets:

```json
{
  "acls": [
    { "action": "accept", "src": ["autogroup:member"], "dst": ["autogroup:self:*"] },
    { "action": "accept", "src": ["tag:laptop"], "dst": ["tag:nas:22,445,80,443"] }
  ],
  "tagOwners": {
    "tag:laptop": ["alice@example.com"],
    "tag:nas":    ["alice@example.com"]
  }
}
```

Not a NixOS concern — all tailnet-wide — but useful to know exists.

### Tailscale + Syncthing

The two complement each other. Make Syncthing listen only on the tailnet:

```nix
services.syncthing.guiAddress = "100.64.0.3:8384";     # your Tailscale IP on this host
```

Syncthing's sync port (22000) travels encrypted anyway; you can either leave it world-reachable (its own TLS) or bind it to `tailscale0` only. The simplest approach: just trust `tailscale0` in the firewall as above, and let Syncthing do its thing.

### Useful commands

```bash
sudo tailscale up                           # bring up (interactive auth on first run)
sudo tailscale down                         # disconnect
tailscale status                            # list peers, IPs, online status
tailscale ip -4                             # my IPv4 tailnet address
tailscale ping laptop                       # latency to a peer
tailscale ssh alice@laptop                  # SSH via Tailscale (with Tailscale SSH)
sudo tailscale up --exit-node=gateway       # route via an exit node
sudo tailscale up --exit-node=""            # stop using exit node
tailscale file cp ./report.pdf alice@laptop:
tailscale file get ./incoming/
```

---

## See also

- [configuration.md](configuration.md) — how `configuration.nix` modules fit together.
- [home-manager-modes.md](home-manager-modes.md) — lift the `home.nix` half of this page into Level 2a on Kali or WSL2.
- [multi-host-flake.md](multi-host-flake.md) — share one `modules/` directory across laptop, desktop, NAS.
- [snippets.md](snippets.md) — more single-purpose copy-paste recipes.
- [linux/tools.md](../linux/tools.md) — the Debian-flavored originals these translations come from.
