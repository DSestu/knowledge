# Tailscale: two tailnets at once

Tailscale is deliberately **single-tailnet**: one `tailscaled` daemon speaks to exactly one tailnet at a time. There is still **no native multi-tailnet client** ([FR #183](https://github.com/tailscale/tailscale/issues/183), open since 2020). To be on two tailnets simultaneously you either share individual nodes across them, or run a **second daemon**.

Pick the lightest approach that covers the actual need:

| Need | Approach |
|---|---|
| Reach a *few* machines on the other tailnet | **Node sharing** — no second client |
| Full membership on both tailnets | **Second `tailscaled`** (own socket/state/tun/port) |
| Avoid routing/firewall collisions | **Userspace mode** (SOCKS proxy) or a container/netns |

## 1. Node sharing (simplest — try this first)

If you just need to hit a handful of hosts on the other tailnet, don't run a second client. On the *other* tailnet, share the specific nodes to your account (**Machines → Share**). They then show up in your normal `tailscale status` alongside your own devices. Zero extra daemons, zero routing headaches.

## 2. Second `tailscaled` instance (full membership)

One daemon = one tailnet, so join the second tailnet with a second daemon that has its **own socket, state file, TUN device, and UDP port**, plus `netfilterMode: off` so the two daemons don't fight over iptables/routing.

### Quick foreground test

```bash
sudo mkdir -p /run/tailscale1 /var/lib/tailscale1
sudo tailscaled \
  --tun=tailscale1 \
  --socket=/run/tailscale1/tailscaled.sock \
  --state=/var/lib/tailscale1/tailscaled.state \
  --port=41642                       # primary uses 41641
```

Bring it up and log into the **second** account by pointing the CLI at that socket. Every command for tailnet #2 needs `--socket`:

```bash
sudo tailscale --socket=/run/tailscale1/tailscaled.sock up --accept-dns=false
tailscale --socket=/run/tailscale1/tailscaled.sock status
tailscale --socket=/run/tailscale1/tailscaled.sock ping <host>
```

### Persistent via a systemd template

`/etc/systemd/system/tailscaled@.service`:

```ini
[Unit]
Description=Tailscale node agent instance #%i
After=network-pre.target NetworkManager.service systemd-resolved.service
Wants=network-pre.target

[Service]
EnvironmentFile=/etc/tailscale.d/tailscale%i.env
ExecStart=/usr/sbin/tailscaled --config=/etc/tailscale.d/tailscale%i.json \
  --state=/var/lib/tailscale%i/tailscaled.state \
  --socket=/run/tailscale%i/tailscaled.sock --tun tailscale%i --port=${PORT}
ExecStopPost=/usr/sbin/tailscaled --cleanup \
  --state=/var/lib/tailscale%i/tailscaled.state \
  --socket=/run/tailscale%i/tailscaled.sock --tun tailscale%i
Restart=on-failure
RuntimeDirectory=tailscale%i
StateDirectory=tailscale%i
Type=notify

[Install]
WantedBy=multi-user.target
```

`/etc/tailscale.d/tailscale1.json`:

```json
{ "version": "alpha0", "netfilterMode": "off", "acceptDNS": true, "enabled": true }
```

`/etc/tailscale.d/tailscale1.env` — copy `/etc/default/tailscaled` and set `PORT=41642` (unique). Then:

```bash
sudo systemctl enable --now tailscaled@1.service
```

⚠️ **The catch:** with `netfilterMode: off`, instance #2 installs **no** routing/firewall rules, so overlapping `100.x` CGNAT ranges or subnet routes across the two tailnets can collide — you manage routing/DNS yourself. Running two MagicDNS resolvers also conflicts; on the secondary it's usually simpler to keep `--accept-dns=false` and address hosts by IP.

## 3. Userspace / container isolation (cleanest routing story)

To sidestep the collisions entirely, run instance #2 in **userspace networking** and expose it as a SOCKS5/HTTP proxy instead of touching the host routing table:

```bash
sudo tailscaled --tun=userspace-networking \
  --socket=/run/tailscale1/tailscaled.sock \
  --state=/var/lib/tailscale1/tailscaled.state \
  --socks5-server=localhost:1055 \
  --outbound-http-proxy-listen=localhost:1055 --port=41642
```

Reach tailnet #2 through the proxy:

```bash
ALL_PROXY=socks5://localhost:1055 curl http://host2
```

A Docker container running the `tailscale/tailscale` image gives the same isolation with less fiddling and is the tidiest durable option. A Linux **network namespace** works too, but you then own the traffic routing in/out of the namespace.

## Rule of thumb

Home + work laptop → try **node sharing** first. Need both tailnets fully → prefer **userspace + SOCKS** or a **container**; the plain second-daemon approach (#2) works but is fragile because of the netfilter/routing overlap.

## Sources

- [FR #183 — Connect to multiple tailnets simultaneously](https://github.com/tailscale/tailscale/issues/183)
- [FR #5721 — workaround discussion](https://github.com/tailscale/tailscale/issues/5721)
- [Multiple Tailnets Guide (gist)](https://gist.github.com/Seas0/f6e9501fcccd2ca7eacdb22441986cc3)
- [Multi-Tailnet: Unlocking Access to Multiple Tailscale Networks](https://jamesguthrie.ch/blog/multi-tailnet-unlocking-access-to-multiple-tailscale-networks/)
- [Connecting to multiple tailscale networks on a single host](https://bnjoroge.com/posts/connecting-to-multiple-tailscale-networks-on-a-single-host/)
