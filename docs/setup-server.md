# Server setup runbook — EC2 + Docker + Tailscale

Goal of this document: get to a URL that opens on the phone **over cellular**, serving
a hello-world container over real HTTPS, with nothing exposed to the public internet.
No application code involved. This is step 0 of the build order.

Rationale for these choices is in [`decisions.md`](decisions.md) §3 and §6.

---

## 1. What Tailscale actually is

A mesh VPN built on WireGuard. Three ideas are enough to use it:

**The tailnet.** Every device you log in from joins one private network of your own.
Each gets a stable address in the `100.64.0.0/10` range (e.g. `100.92.14.7`) and, with
MagicDNS on, a name like `pace.tailnet-name.ts.net`. Those addresses work identically
from your flat, from cellular, from anywhere.

**It is peer-to-peer, not a gateway.** Unlike a corporate VPN, traffic goes directly
between your devices, encrypted end to end. A central coordination server is used only
to exchange public keys and help the two ends find each other. If the two devices
cannot reach each other directly through NAT, Tailscale falls back to an encrypted
relay (DERP) — slower, still private. Practically: the coordination server being down
does not kill existing connections, but it does block enrolling a new device.

**It is outbound-only.** The EC2 instance dials *out* to establish its links. That is
why the security group can allow **zero inbound** and the phone can still reach the app.
Nothing on the public internet can open a connection to the box at all — the app is not
"hidden", it is unreachable, which is a much stronger property than a login page.

Two commands are easy to confuse:

| Command | What it does | Us? |
|---|---|---|
| `tailscale serve` | Reverse-proxies a local port to your tailnet, with a valid TLS cert. Tailnet-only. | ✅ yes |
| `tailscale funnel` | Publishes that same service **to the entire internet**. | ❌ never |

Free plan covers personal use comfortably (on the order of 3 users / 100 devices).

---

## 2. Before touching the server

In the Tailscale admin console (login with a personal identity — GitHub/Google):

1. **DNS → enable MagicDNS.** Gives devices names instead of bare IPs.
2. **DNS → enable HTTPS certificates.** Required for `tailscale serve` to get a cert.
3. **Settings → Keys → Generate auth key.** For a headless server you cannot do the
   interactive browser login, so you paste a key instead.
   - Reusable: **off** (one key, one machine).
   - Ephemeral: **off** — ephemeral nodes are removed when they go offline, which is
     wrong for a server that will reboot.
   - Pre-approved: **on**, if device approval is enabled on the tailnet.
   - Treat it like a password. It goes in the `tailscale up` command once and is then
     stored in the node's own state.

---

## 3. EC2 instance

- **Architecture: arm64 (`t4g` / Graviton) if the free tier allows it.** Oracle's
  Always Free tier is ARM, so matching now makes the later migration a copy rather
  than a rebuild. If the free tier only offers x86, take it — it is one word in the
  build command — but note it in the migration plan.
- Size: the smallest thing available. This app idles at a few MB of RAM.
- Storage: default is plenty. Remember the DB lives on this disk; snapshots are not a
  backup strategy on their own.
- **Security group inbound: SSH from your IP only, temporarily.** This is break-glass
  access while setting Tailscale up. Do not remove it until §6.
- Outbound: allow all.

---

## 4. Install Docker and Tailscale on the instance

```bash
# Docker (Amazon Linux 2023)
sudo dnf install -y docker
sudo systemctl enable --now docker
sudo usermod -aG docker "$USER"   # log out and back in for this to take effect
```

```bash
# Tailscale
curl -fsSL https://tailscale.com/install.sh | sh
```

Bring the node up. `--ssh` enables Tailscale SSH, which is what lets you close port 22
later:

```bash
sudo tailscale up --authkey=tskey-auth-REPLACE_ME --hostname=pace --ssh
```

Then, **in the admin console, open the `pace` node and disable key expiry.** Node keys
expire by default (commonly 180 days). An expired server silently vanishes from the
tailnet, and it will happen at the least convenient moment.

Verify:

```bash
tailscale status          # should list pace plus your laptop
tailscale ip -4           # the 100.x address
```

---

## 5. A hello-world container, bound to localhost

Create `/srv/pace/compose.yaml`:

```yaml
services:
  pace:
    image: nginxdemos/hello          # placeholder until the real image exists
    restart: unless-stopped
    ports:
      - "127.0.0.1:8080:80"          # note the 127.0.0.1 — this matters
    volumes:
      - /srv/pace/data:/data
```

```bash
sudo mkdir -p /srv/pace/data
cd /srv/pace && docker compose up -d
curl -s localhost:8080 | head -5
```

**Why `127.0.0.1:8080:8080` and not `8080:8080`:** Docker writes its own iptables
rules, and a plainly published port can end up reachable from the internet even when
the cloud firewall says otherwise. Binding to the loopback interface means the only
way in is through Tailscale. Keep this in the real compose file too.

---

## 6. Expose it to the tailnet over HTTPS

```bash
sudo tailscale serve --bg 8080
sudo tailscale serve status
```

The CLI for `serve` has changed shape across releases — if that form is rejected, check
`tailscale serve --help` for the version installed, which may want
`tailscale serve --bg http://localhost:8080` or a `https / <port>` argument form.

`serve status` prints the public-to-your-tailnet URL, something like
`https://pace.tailnet-name.ts.net/`. TLS is terminated by Tailscale with a real
certificate, so the browser sees a proper secure origin and the app never has to know
about certificates at all.

**Now close the door:**

1. Install Tailscale on the laptop, log in with the same identity.
2. Confirm `ssh pace` works over Tailscale SSH.
3. Only then remove the port 22 rule from the security group.

Keep EC2 Instance Connect or Session Manager available as a fallback. Locking yourself
out of a cloud VM is an easy own-goal and the whole point of doing this in order.

---

## 7. The phone

1. Install Tailscale from the Play Store, log in with the same identity.
2. Open `https://pace.tailnet-name.ts.net`. **Any** connection works — home wifi,
   cellular, anywhere. Day to day this will mostly be wifi; nothing about the setup
   depends on which.
3. If that page loads, the architecture in `decisions.md` §2 is proven and you can
   start writing Go.
4. Optional 30-second check: try it once with wifi off too. Not a requirement — the
   server is in the cloud, so there is no same-LAN shortcut to accidentally rely on.
   It only tells you whether a given network blocks Tailscale's preferred UDP path and
   forces the encrypted relay fallback, which still works but is slower. Useful to know
   before blaming your own code for being sluggish.
5. Later, once the real app serves a web manifest: Chrome → ⋮ → Add to Home screen.

Leave Tailscale running on the phone. Battery cost is negligible; it is idle unless
you are using it.

---

## 8. Deploying the real app later

From the laptop:

```bash
docker build --platform linux/arm64 -t ghcr.io/<user>/pace:latest .
docker push ghcr.io/<user>/pace:latest
```

On the server:

```bash
cd /srv/pace && docker compose pull && docker compose up -d
```

No registry, no account, same result:

```bash
docker save pace:latest | gzip | ssh pace 'gunzip | docker load'
```

Worth wrapping in a `make deploy` target the first time you do it twice.

---

## 9. Checklist

- [ ] MagicDNS and HTTPS certificates enabled in the admin console
- [ ] Auth key generated: not reusable, not ephemeral
- [ ] Instance is arm64 (or the exception is written down)
- [ ] Docker installed, user in the `docker` group
- [ ] `tailscale up --ssh` done, node named `pace`
- [ ] **Key expiry disabled on the `pace` node**
- [ ] Container published to `127.0.0.1` only
- [ ] `tailscale serve` running, URL loads on the laptop
- [ ] `ssh pace` works over Tailscale, *then* port 22 removed from the security group
- [ ] **Phone loads the URL** (wifi is fine; cellular is an optional extra check)
- [ ] `tailscale funnel` is not running anywhere
