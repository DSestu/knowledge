# Tailscale

Two things worth writing down: exposing a machine **publicly** with Funnel, and the (bad) state of running **two tailnets at once**.

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
| Full membership on both | **Second `tailscaled`** — fragile, see below |
| Avoid routing collisions | **Userspace mode** (SOCKS) or a container |

**Node sharing first.** On the *other* tailnet, share the specific nodes to your account (Machines → Share). They appear in your normal `tailscale status`. Zero extra daemons, zero routing headaches. This covers the home-plus-work-laptop case entirely.

**Second daemon** needs its own socket, state file, TUN device and UDP port, plus `netfilterMode: off`:

```bash
sudo tailscaled --tun=tailscale1 \
  --socket=/run/tailscale1/tailscaled.sock \
  --state=/var/lib/tailscale1/tailscaled.state \
  --port=41642                                    # primary uses 41641

sudo tailscale --socket=/run/tailscale1/tailscaled.sock up --accept-dns=false
```

Every CLI command for tailnet #2 needs that `--socket`. The catch: with `netfilterMode: off` the second instance installs **no** routing/firewall rules, so overlapping `100.x` ranges or subnet routes collide and you own the routing yourself. Two MagicDNS resolvers also conflict — keep `--accept-dns=false` on the secondary and address hosts by IP. A systemd template for this is in the [gist below](https://gist.github.com/Seas0/f6e9501fcccd2ca7eacdb22441986cc3).

**Prefer userspace or a container** if you really need both tailnets. Userspace mode exposes tailnet #2 as a proxy instead of touching the host routing table:

```bash
sudo tailscaled --tun=userspace-networking \
  --socket=/run/tailscale1/tailscaled.sock \
  --state=/var/lib/tailscale1/tailscaled.state \
  --socks5-server=localhost:1055 --port=41642

ALL_PROXY=socks5://localhost:1055 curl http://host2
```

A container running the `tailscale/tailscale` image gives the same isolation with less fiddling and is the tidiest durable option.

## Sources

- [Tailscale Funnel docs](https://tailscale.com/kb/1223/funnel)
- [`tailscale serve` / `funnel` CLI reference](https://tailscale.com/kb/1242/tailscale-serve)
- [FR #183 — Connect to multiple tailnets simultaneously](https://github.com/tailscale/tailscale/issues/183)
- [Multiple Tailnets Guide (gist)](https://gist.github.com/Seas0/f6e9501fcccd2ca7eacdb22441986cc3)
- [Multi-Tailnet: Unlocking Access to Multiple Tailscale Networks](https://jamesguthrie.ch/blog/multi-tailnet-unlocking-access-to-multiple-tailscale-networks/)
