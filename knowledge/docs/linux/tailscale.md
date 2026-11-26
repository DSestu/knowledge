# Tailscale

Four things worth writing down: exposing a machine **publicly** with Funnel, the (bad) state of running **two tailnets at once**, the **WSL mirrored-mode** trap that makes the Windows host the one peer WSL cannot reach, and **root ADB over the tailnet** without Wi-Fi.

## Funnel — a public DNS name for a tailnet machine

Machines in a tailnet have `100.x` CGNAT addresses that are unroutable from outside. Publishing an A record for one does nothing. **Funnel** is the supported way out: it gives the node a public name in the `ts.net` zone with a publicly-trusted TLS certificate, over an outbound connection to Tailscale's relays — no port forwarding, no inbound firewall rule.

```bash
# Once, in the tailnet policy (admin console → Access controls):
#   "nodeAttrs": [{ "target": ["autogroup:member"], "attr": ["funnel"] }]
# Admin console → DNS: MagicDNS and HTTPS Certificates must both be enabled.

tailscale funnel --bg --https=8443 8443     # public :8443 → local :8443
tailscale funnel status                     # prints the real public URL
tailscale serve status                      # shows serve *and* funnel entries
tailscale funnel off
```

The resulting name is `https://<host>.<tailnet>.ts.net`, e.g. `desktop-tvtdome.tail6c627.ts.net`. Get it from `funnel status` rather than assembling it — the tailnet part is a generated `tailXXXXX`, not your login domain.

### Constraints

* **Public ports are 443, 8443, 10000. Only those.** TCP only, no UDP.
* Traffic is relayed through DERP, so expect added latency and don't plan on it for sustained high-bandwidth media.
* **The name is fixed.** You can `CNAME foo.example.com` to it, but the certificate only covers the `ts.net` SAN, so any client using your own name gets a TLS error. There is no custom-domain support.
* **It is genuinely public.** Anyone with the URL is in. Funnel provides no authentication.

`tailscale serve` is the same machinery restricted to the tailnet — useful for HTTPS without public exposure, and the source of the collision below.

### Gotchas that cost real time

**`tailscale funnel 8443` does not mean 8443 → 8443.** The bare argument is the *local* target; the public port defaults to 443. Pass `--https=8443` when you want the port preserved.

**`sending serve config: updating config: listener already exists for port 443`** means that port already holds a `serve` entry. Inspect and clear it before funneling:

```bash
tailscale serve status
tailscale serve --https=443 off
```

**A port left as `serve` instead of `funnel` fails in a confusing way.** The relay accepts the TCP connection and then drops the TLS handshake, identically from every relay IP:

```
curl: (35) TLS connect error: error:0A000126:SSL routines::unexpected eof while reading
```

That reads like a certificate problem but it is a missing funnel listener. Check `funnel status` lists the port before debugging TLS.

**Self-signed backends need `https+insecure://`.** If the local service serves its own untrusted cert, Funnel refuses the upstream connection unless told not to validate. Funnel still presents its own valid cert publicly, so clients see a clean site:

```bash
tailscale funnel --bg --https=8443 https+insecure://localhost:8443
```

**Without `--bg` the proxy is foreground-only** and dies with the command.

### When the name doesn't resolve

Check more than one resolver before assuming the setup is broken:

```bash
nslookup <host>.<tailnet>.ts.net 8.8.8.8
nslookup <host>.<tailnet>.ts.net 1.1.1.1
```

A resolver queried *before* the funnel existed caches the `NXDOMAIN` for its negative TTL while others answer correctly. Observed in practice: Google and Quad9 returning both A and AAAA while Cloudflare `1.1.1.1` held `NXDOMAIN` for over an hour. Purge it at <https://1.1.1.1/purge-cache/> (A and AAAA both), or use another resolver meanwhile.

Distinguish the two failure modes by status code — `NXDOMAIN` is negative caching, `SERVFAIL` is DNSSEC validation and not something you can fix locally.

Test reachability with the IP pinned, bypassing DNS entirely:

```bash
curl -sS -o /dev/null -w 'http=%{http_code} tls=%{ssl_verify_result}\n' \
  --resolve <host>.<tailnet>.ts.net:8443:<ip> https://<host>.<tailnet>.ts.net:8443
```

Also note that a machine *inside* the tailnet resolves the name to its `100.x` address via MagicDNS instead of the public IP. That's expected and still connects — but a machine with Tailscale installed and **disconnected** can be left with broken DNS config. The cleanest external test is a phone on cellular.

### Custom domain / real bandwidth — use an ingress node instead

When Funnel's ports, fixed name, or relay bandwidth don't fit: put a cheap VPS in the tailnet, point your own DNS at its public IP, and reverse-proxy over the tailnet.

```caddyfile
jellyfin.example.com {
    reverse_proxy http://myhost.tailnet-name.ts.net:8096
}
```

Caddy gets a Let's Encrypt cert for your domain, the backend never opens a port to the internet, and you're on real 443 with no port constraints. Cost is the VPS and its egress.

## Two tailnets at once

Tailscale is deliberately **single-tailnet**: one `tailscaled` speaks to exactly one tailnet. There is still **no native multi-tailnet client** ([FR #183](https://github.com/tailscale/tailscale/issues/183), open since 2020).

| Need | Approach |
|---|---|
| Reach a *few* machines on the other tailnet | **Node sharing** — no second client |
| Full IP access to the other tailnet | **Second `tailscaled` in a netns**, see below |
| Avoid routing collisions | **Userspace mode** (SOCKS) or a container |

**Node sharing first.** On the *other* tailnet, share the specific nodes to your account (Machines → Share). They appear in your normal `tailscale status`. Zero extra daemons, zero routing headaches. This covers the home-plus-work-laptop case entirely.

**A second kernel-mode daemon on the same host breaks the first one.** Verified against the 1.98.4 source (`wgengine/router/osrouter/router_linux.go`, `cmd/tailscaled/tailscaled.go`):

* `--tun`, `--port`, `--socket` and `--statedir` separate the daemons, and routes do not collide: each daemon adds per-peer `/32`s to table 52 on its own interface only.
* Both share ip rules 5210–5270. When one stops it deletes them, and the other restores them within about 250 ms (`onIPRuleDeleted`). That self-heals.
* **Every `tailscaled` start runs `router.CleanUp`, which deletes *all* `ts-*` iptables/nft chains**, including the other daemon's. `--netfilter-mode=off` does not prevent this.
* With `NetfilterMode: on` (the default), the first daemon's `ts-input` holds `! -i tailscale0 -s 100.64.0.0/10 -j DROP`. Replies from tailnet #2 arriving on `tailscale1` are dropped. Only a tailnet admin can lift that, via the `disable-linux-cgnat-drop-rule` node attribute.

Check what your primary uses with `tailscale debug prefs` (`NetfilterMode` 2 = on) and `sudo iptables -S ts-input`.

### Working setup: second tailnet in a network namespace (Kali laptop)

Use case: the laptop stays fully on the work tailnet (`--accept-routes`, MagicDNS, nothing changed), and reaches two nodes on the personal tailnet (`desktop-tvtdome`, `nixos-wsl`) with plain IP access for SMB, Steam Remote Play, SSH and self-hosted services. No SOCKS proxy, no node sharing (work must never see the personal nodes).

The personal `tailscaled` runs inside netns `tsperso`, so its cleanup, firewall and rules can't touch the host. The host reaches it over a veth pair. Each personal node gets a host-side alias in `10.251.0.0/24`, which the netns DNATs to the node's `100.x`. Replies come back from the alias, never from a `100.x` source, so work's CGNAT DROP never sees them.

```
host ── 10.251.0.1 ──► vperso0 10.250.0.1 ═ veth ═ vperso1 10.250.0.2 [netns tsperso]
                                                   DNAT → 100.113.113.87, MASQUERADE
                                                   └─► tailscale-perso ──► desktop-tvtdome
```

Three files, installed by hand (Kali is not NixOS, and Home Manager can't run a root daemon):

`/etc/tailscale-perso/nodes`:

```
# <alias on this host> <perso tailnet 100.x> <hostnames for /etc/hosts>
# After editing: sudo systemctl restart tailscale-perso
10.251.0.1 100.113.113.87 desktop-tvtdome
10.251.0.2 100.88.56.95   nixos-wsl
```

`/etc/systemd/system/tailscale-perso.service`:

```ini
[Unit]
Description=Tailscale perso tailnet in netns tsperso
Wants=network-online.target
After=network-online.target tailscaled.service docker.service

[Service]
ExecStartPre=/usr/local/sbin/tailscale-perso setup
ExecStart=/usr/sbin/ip netns exec tsperso /usr/sbin/tailscaled --tun=tailscale-perso --port=41642 --statedir=/var/lib/tailscale-perso --socket=/run/tailscale-perso/tailscaled.sock
ExecStopPost=/usr/local/sbin/tailscale-perso teardown
Restart=on-failure
RuntimeDirectory=tailscale-perso
StateDirectory=tailscale-perso
StateDirectoryMode=0700
Type=notify

[Install]
WantedBy=multi-user.target
```

`/usr/local/sbin/tailscale-perso` (mode 0755):

```bash
#!/usr/bin/env bash
# tailscale-perso: run the perso tailnet in netns "tsperso" next to the pro tailscaled,
# reached from the host through 10.251.0.x aliases listed in /etc/tailscale-perso/nodes.
# Why a netns: pro's ts-input drops 100.64.0.0/10 arriving outside tailscale0, and every
# tailscaled start wipes all ts-* chains, so a second daemon on the host breaks pro.
set -uo pipefail
NS=tsperso
CONF=/etc/tailscale-perso/nodes
SOCK=/run/tailscale-perso/tailscaled.sock
TAG='# tailscale-perso'

nodes() { grep -vE '^\s*(#|$)' "$CONF"; }   # lines: <alias> <perso 100.x> <name>...

setup() {
  teardown >/dev/null 2>&1   # clear leftovers from a crash
  set -ex
  ip netns add $NS
  ip link add vperso0 type veth peer name vperso1
  ip link set vperso1 netns $NS
  ip addr add 10.250.0.1/30 dev vperso0
  ip link set vperso0 up
  ip netns exec $NS ip addr add 10.250.0.2/30 dev vperso1
  ip netns exec $NS ip link set vperso1 up
  ip netns exec $NS ip link set lo up
  ip netns exec $NS ip route add default via 10.250.0.1
  ip netns exec $NS sysctl -qw net.ipv4.ip_forward=1
  # host: let the netns reach the internet (Docker sets FORWARD policy DROP)
  iptables -I FORWARD 1 -i vperso0 -m comment --comment tsperso -j ACCEPT
  iptables -I FORWARD 1 -o vperso0 -m conntrack --ctstate RELATED,ESTABLISHED -m comment --comment tsperso -j ACCEPT
  iptables -t nat -I POSTROUTING 1 -s 10.250.0.0/30 ! -o vperso0 -m comment --comment tsperso -j MASQUERADE
  ip route add 10.251.0.0/24 via 10.250.0.2
  # netns: alias -> perso node, and leave as our perso IP
  ip netns exec $NS iptables -t nat -A POSTROUTING -o tailscale-perso -j MASQUERADE
  local alias target names
  while read -r alias target names; do
    ip netns exec $NS iptables -t nat -A PREROUTING -i vperso1 -d "$alias" -j DNAT --to-destination "$target"
    echo "$alias $names $TAG" >> /etc/hosts
  done < <(nodes)
  mkdir -p /etc/netns/$NS
  printf 'nameserver 1.1.1.1\nnameserver 9.9.9.9\n' > /etc/netns/$NS/resolv.conf
}

teardown() {
  set -x
  ip netns del $NS             # also removes vperso0/vperso1, tailscale-perso and netns rules
  ip link del vperso0
  ip route del 10.251.0.0/24
  iptables -D FORWARD -i vperso0 -m comment --comment tsperso -j ACCEPT
  iptables -D FORWARD -o vperso0 -m conntrack --ctstate RELATED,ESTABLISHED -m comment --comment tsperso -j ACCEPT
  iptables -t nat -D POSTROUTING -s 10.250.0.0/30 ! -o vperso0 -m comment --comment tsperso -j MASQUERADE
  sed -i "/ $TAG\$/d" /etc/hosts
  rm -rf /etc/netns/$NS
  return 0
}

case "${1:-}" in
  setup) setup ;;
  teardown) teardown ;;
  ts) shift; exec tailscale --socket=$SOCK "$@" ;;   # e.g. tailscale-perso ts status
  *) echo "usage: $0 setup|teardown|ts <tailscale args>" >&2; exit 2 ;;
esac
```

Install and first login:

```bash
sudo install -m 0755 tailscale-perso /usr/local/sbin/
sudo install -d /etc/tailscale-perso && sudo install -m 0644 nodes /etc/tailscale-perso/
sudo install -m 0644 tailscale-perso.service /etc/systemd/system/
sudo systemctl daemon-reload && sudo systemctl enable --now tailscale-perso
sudo tailscale-perso ts up --hostname=kali-perso --accept-routes=false --accept-dns=false   # open the URL, log in with the perso account
sudo tailscale-perso ts status        # read the 100.x of each node into the nodes file, then restart the unit
```

The login is stored in `/var/lib/tailscale-perso`, so later starts reconnect by themselves.

Use it with `ping desktop-tvtdome`, `ssh nixos-wsl`, `smb://desktop-tvtdome/`, or the Steam client with `10.251.0.1` added by hand (LAN discovery can't cross this link).

Rollback:

```bash
sudo systemctl disable --now tailscale-perso     # removes netns, veth, host rules, hosts lines
# full uninstall:
sudo rm /etc/systemd/system/tailscale-perso.service /usr/local/sbin/tailscale-perso
sudo rm -r /etc/tailscale-perso /var/lib/tailscale-perso && sudo systemctl daemon-reload
```

Verified 2026-11-26 on Tailscale 1.98.4:

* With perso running, the host's ip rules and table 52 were identical to a pre-install snapshot.
* After `systemctl stop`, `iptables-save` differed only by the order of work's own `ts-forward`/`ts-postrouting` jumps, which a work `tailscaled` restart re-inserts.
* Restarting work's `tailscaled` with perso up kept both working.

Limits:

* Connections go laptop → personal node only. The personal nodes can't initiate to the laptop.
* If a node re-registers and its `100.x` changes, update the nodes file.
* Personal MagicDNS names are not resolved (they'd return `100.x`). `/etc/hosts` lines cover it, and the host reads `files` before `dns` in `nsswitch.conf`.

**Prefer userspace or a container** if you really need both tailnets. Userspace mode exposes tailnet #2 as a proxy instead of touching the host routing table:

```bash
sudo tailscaled --tun=userspace-networking \
  --socket=/run/tailscale1/tailscaled.sock \
  --state=/var/lib/tailscale1/tailscaled.state \
  --socks5-server=localhost:1055 --port=41642

ALL_PROXY=socks5://localhost:1055 curl http://host2
```

A container running the `tailscale/tailscale` image gives the same isolation with less fiddling and is the tidiest durable option.

## WSL mirrored mode makes the Windows host unreachable from WSL

Symptom: from a WSL distro running its own `tailscaled`, the Windows host it lives on is the *only* tailnet peer you cannot reach — `\\host` SMB, Jellyfin, anything — while phones and other machines reach it fine, and WSL reaches every other peer. `Connection refused` on the host's `100.x` *and* on its LAN IP; loopback still works.

Cause: `.wslconfig` `networkingMode=mirrored` projects every Windows adapter into the VM, including the Tailscale one, so the host's own `100.x` shows up as a **local** address inside WSL:

```bash
ip route show table local | grep 100.        # local 100.113.113.87 dev eth2 ...
```

The kernel delivers packets for that address inside WSL — where nothing listens — and they never leave. The host is not broken; WSL is talking to itself.

### Do not debug this from WSL

Every probe from WSL is poisoned, in both directions, and it is easy to conclude the host's tunnel is dead:

* `tailscale nc <host> <port>` returns nothing — for *every* port, including ones a phone reaches. Run a positive control against a port you know works before trusting a negative.
* Windows → WSL tests fail too: WSL's reply to the host's `100.x` is delivered locally and never returns.
* `tailscale ping` still pongs (handled inside tailscaled, never touches the adapter), which makes the picture look like "daemon fine, tunnel dead".

Valid vantages: a phone on cellular, or the node's own Funnel URL via a public resolver (`curl --doh-url https://8.8.8.8/dns-query https://<host>.<tailnet>.ts.net:8443`). Windows-side checks that *are* worth doing first, because they are cheap: `Get-SmbShare`, `Get-NetTCPConnection -LocalPort 445 -State Listen`, and `Get-NetFirewallRule -DisplayName 'File and Printer Sharing (SMB-In)' | Get-NetFirewallAddressFilter` — the stock SMB-In rule is scoped to `LocalSubnet`, which excludes `100.64.0.0/10`; a dedicated allow rule for that range on TCP 445 is required.

### Fix — on the Windows side, in `%UserProfile%\.wslconfig`

```ini
[experimental]
hostAddressLoopback=true
```

Mirrored-mode only, IPv4 only (Windows 11 22H2+, WSL ≥ 2.0). It extends what `127.0.0.1` already does to every IP assigned to the host: connections from WSL to the host's `100.x` or LAN IP are forwarded to Windows instead of dying inside the VM. Apply with `wsl --shutdown` from PowerShell, then reopen — this kills everything running in WSL, including anything funneled from it. The Windows side needs nothing else, provided SMB-In is allowed from `100.64.0.0/10` (see above).

No-restart fallback, from inside WSL, ephemeral (gone at the next `wsl --shutdown`): send the CGNAT range through tailscaled's table before the local-table lookup, so the packet takes the real tunnel.

```bash
sudo ip rule add pref 1 to 100.64.0.0/10 lookup 52    # 52 = tailscaled's table
ip route get 100.113.113.87                           # ... dev tailscale0 table 52
```

Prefs 0–1 are WSL's own mirroring rules and `lookup local` is pref 2, so pref 1 is the slot; a table miss falls through, so WSL's own `100.x` still resolves locally. `ip rule add` exits 2 on a duplicate.

Either way, test as any other client would — over the tailnet name, not `127.0.0.1`:

```bash
smbclient //desktop-tvtdome/'The lake' -U david
```

### Phone can't open the share by name — check by IP first

An Android file manager (CX, Solid, MiXplorer…) failing on `desktop-tvtdome` with no credential prompt looks like a host problem but usually isn't: try `100.113.113.87` in the same app. If the IP works, the host, tunnel and firewall are fine and the app's own name lookup is the issue — these clients tend to treat a dotless, ≤15-character name as a NetBIOS name and try broadcast resolution, which cannot cross the tunnel. Use the MagicDNS FQDN (`desktop-tvtdome.tail6c627.ts.net`) or the IP; after one FQDN connection the short name may start working again (cached). Server-side proof that a phone got through, no admin needed: `Get-SmbSession` and `Get-NetTCPConnection -LocalPort 445`.

## Root ADB on a phone over the tailnet, without Wi-Fi

Goal: `adb shell su -c …` on the rooted OnePlus 11 (`oneplus-11-5g`, `100.108.95.2`) from WSL, phone on mobile data.

The Android 11+ **Wireless debugging** toggle is greyed out off Wi-Fi ("only available on Wi-Fi") and its port changes every reboot, so ignore it. Legacy TCP adbd has neither problem and listens on every interface, including `tailscale0`. Only **USB debugging** needs to be enabled in Developer options.

On the phone, once, in Termux (grant root in Magisk):

```bash
su -c 'setprop persist.adb.tcp.port 5555; setprop service.adb.tcp.port 5555; stop adbd; start adbd'
# if stop/start is denied on your ROM: su -c 'killall adbd' — init restarts it on the new port
```

`persist.adb.tcp.port` is meant to survive reboots. If it doesn't on yours, drop the `setprop` line in `/data/adb/service.d/adb_tcp.sh` (Magisk runs it at boot).

From WSL:

```bash
adb connect 100.108.95.2:5555     # first time: "Allow USB debugging?" prompt on the phone — tick "Always allow"
adb devices -l                    # "unauthorized" until you accept, then "device"
adb -s 100.108.95.2:5555 shell 'su -c id'   # uid=0(root) … context=u:r:magisk:s0
```

The auth prompt appears for TCP connections too, so no cable is ever needed.

**Security:** anyone reaching `:5555` with an authorised key owns the phone. Add a tailnet ACL allowing 5555 from the WSL node only.

### adb hangs forever from Claude Code / a sandboxed shell

`adb connect` (and even `adb start-server`) blocks with no output and nothing listening on `:5037`. The daemonised server does not survive the sandbox's process handling; the client waits for an ACK that never comes. Run the server in foreground mode, detached, then use the client normally:

```bash
setsid nohup adb nodaemon server >/tmp/adb-server.log 2>&1 &
adb connect 100.108.95.2:5555
```

There is no `adb` in the WSL profile; `nix build nixpkgs#android-tools` gives a store path with `bin/adb` (`nix run` fails: the package has no binary named `android-tools`).

## Sources

- [Tailscale Funnel docs](https://tailscale.com/kb/1223/funnel)
- [`tailscale serve` / `funnel` CLI reference](https://tailscale.com/kb/1242/tailscale-serve)
- [FR #183 — Connect to multiple tailnets simultaneously](https://github.com/tailscale/tailscale/issues/183)
- [Multiple Tailnets Guide (gist)](https://gist.github.com/Seas0/f6e9501fcccd2ca7eacdb22441986cc3)
- [Multi-Tailnet: Unlocking Access to Multiple Tailscale Networks](https://jamesguthrie.ch/blog/multi-tailnet-unlocking-access-to-multiple-tailscale-networks/)
