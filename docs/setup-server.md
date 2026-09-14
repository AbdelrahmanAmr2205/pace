# Server setup runbook — EC2 + Docker + Tailscale

Goal of this document: get to a URL that **opens on the phone**, serving a hello-world
container over real HTTPS, with nothing exposed to the public internet. No application
code involved. This is step 0 of the build order.

Rationale for these choices is in [`decisions.md`](decisions.md) §3 and §6.

**Do [`setup-aws.md`](setup-aws.md) first.** It covers the account itself — root
hardening, budgets, the identity you log in as, and launching the instance with no
inbound ports. This document picks up once you have a shell on the box.

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

The free Personal plan covers this comfortably: up to 6 users with unlimited user
devices. (Checked 2026-09-14 — these limits have changed before.)

---

## 2. Before touching the server

In the Tailscale admin console (login with a personal identity — GitHub/Google):

1. **DNS → enable MagicDNS.** Gives devices names instead of bare IPs.
2. **DNS → enable HTTPS certificates.** Required for `tailscale serve` to get a cert.
3. **Nothing else.** You do *not* need an auth key. `tailscale up` on a headless box
   prints a login URL you paste into any browser — the machine never needs its own
   session. That is the default path in §4.

   An auth key is for unattended provisioning (user-data scripts, image bakes), where
   there is no human to open a URL. If you use one anyway: Reusable **off**, Ephemeral
   **off** (ephemeral nodes disappear when they go offline, which is wrong for a server
   that reboots), Pre-approved **on** if device approval is enabled. And keep it out of
   your shell history — it is a credential that grants tailnet membership, and a
   `tskey-auth-...` string sitting in `~/.bash_history` outlives every intention to
   clean it up.

---

## 3. EC2 instance

Covered in full by [`setup-aws.md`](setup-aws.md) §8. The summary:

- **arm64 (`t4g` / Graviton) if the free tier allows it.** Oracle's Always Free tier is
  ARM, so matching now makes the later migration a copy rather than a rebuild. If only
  x86 is free, take it — it is one word in the build command — but note it in the
  migration plan.
- Smallest available size. This app idles at a few MB of RAM.
- Storage: 20–30 GiB is plenty. Remember the DB lives on this disk; snapshots are not
  a backup strategy on their own.
- **Security group inbound: zero rules.** Not "SSH from my IP, temporarily". Break-glass
  during setup is Session Manager, which needs no open port — so there is nothing to
  open and nothing to remember to close. Rules can be added in seconds if you ever
  need one.
- An instance role granting `AmazonSSMManagedInstanceCore` and nothing else, and an
  IMDS hop limit of 1 so containers cannot reach the instance credentials.
- Outbound: allow all.

You should reach a shell through EC2 → Connect → Session Manager before continuing.

---

## 4. Install Docker and Tailscale on the instance

```bash
# Docker (Amazon Linux 2023)
sudo dnf install -y docker
sudo systemctl enable --now docker
```

**AL2023's `docker` package does not include the Compose plugin.** `docker compose`
will simply not exist, which is a confusing failure right at the point you need it.
Install it as a CLI plugin (arm64 binary; pin the version rather than tracking
`latest`):

```bash
sudo mkdir -p /usr/local/lib/docker/cli-plugins
sudo curl -fsSL -o /usr/local/lib/docker/cli-plugins/docker-compose \
  https://github.com/docker/compose/releases/download/v2.39.4/docker-compose-linux-aarch64
sudo chmod +x /usr/local/lib/docker/cli-plugins/docker-compose
docker compose version      # fail here, not three steps later
```

**Use `sudo docker`, and do not add yourself to the `docker` group.** Two reasons. The
group is root-equivalent — anyone in it can mount the host filesystem into a container
— so it buys convenience at the cost of the privilege boundary you just spent
`setup-aws.md` building. And it would not even work as intended here: connected through
Session Manager you are `ssm-user`, but over Tailscale SSH you will be `ec2-user`, so
`usermod -aG docker "$USER"` silently grants the group to whichever account you happened
to be using at the time.

```bash
# Tailscale
curl -fsSL https://tailscale.com/install.sh | sh
```

Bring the node up. `--ssh` enables Tailscale SSH, which is how you will reach this box
from the laptop without any port being open:

```bash
sudo tailscale up --hostname=pace --ssh
```

This prints a URL. Open it in the browser on your laptop, approve the machine, and the
command returns. No credential is typed on the server and none ends up in your history.

(If you ever need the unattended form, it is
`sudo tailscale up --hostname=pace --ssh --auth-key="$TS_KEY"` with the key supplied
through the environment rather than typed on the command line.)

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
cd /srv/pace && sudo docker compose up -d
curl -s localhost:8080 | head -5
```

**Why `127.0.0.1:8080:80` and not `8080:80`:** Docker writes its own iptables
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

**Now switch to your permanent way in:**

1. Install Tailscale on the laptop, log in with the same identity.
2. Confirm `ssh ec2-user@pace` works over Tailscale SSH. **Specify the user.** Tailscale
   SSH defaults to your *laptop's* username, which does not exist on Amazon Linux, so a
   bare `ssh pace` fails in a way that looks like Tailscale being broken when it is
   working perfectly. Add it to `~/.ssh/config` once and forget it:

   ```
   Host pace
       User ec2-user
   ```

There is no door to close — the security group never had an inbound rule. If you
followed `setup-aws.md`, you now have three independent ways onto the box: Tailscale
SSH (daily), Session Manager (when Tailscale is unhappy), and the `pace` key pair plus
a temporary firewall rule (when both are). Locking yourself out of a cloud VM is an
easy own-goal; the layering is deliberate.

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
cd /srv/pace && sudo docker compose pull && sudo docker compose up -d
```

No registry, no account, same result:

```bash
docker save pace:latest | gzip | ssh ec2-user@pace 'gunzip | sudo docker load'
```

Worth wrapping in a `make deploy` target the first time you do it twice.

---

## 9. Checklist

- [ ] [`setup-aws.md`](setup-aws.md) checklist complete
- [ ] MagicDNS and HTTPS certificates enabled in the admin console
- [ ] Auth key generated: not reusable, not ephemeral
- [ ] Instance is arm64 (or the exception is written down)
- [ ] Docker installed **and `docker compose version` works** (the plugin is separate)
- [ ] `tailscale up --ssh` done, node named `pace`
- [ ] **Key expiry disabled on the `pace` node**
- [ ] Container published to `127.0.0.1` only
- [ ] `tailscale serve` running, URL loads on the laptop
- [ ] `ssh ec2-user@pace` works over Tailscale SSH
- [ ] Security group still has zero inbound rules
- [ ] **Phone loads the URL** (wifi is fine; cellular is an optional extra check)
- [ ] `tailscale funnel` is not running anywhere
