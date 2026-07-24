# petabyte familiar

**Monitor your homelab from inside your network — no port forwarding, no tunnels, no external access required.**

A familiar is a lightweight connector that runs inside your private network to check services that public probes can't reach. Your Jellyfin server, Home Assistant dashboard, pfSense admin panel, NAS web interface, or that printer at 192.168.1.50 — if it's on your LAN, a familiar can watch it.

Feed the monster. Keep your services up.

---

## What is petabyte.monster?

**Uptime monitoring where every service is a creature.** Uptime feeds it, downtime starves it, streaks evolve it.

Unlike traditional uptime monitors with boring status pages, petabyte.monster turns your infrastructure into a menagerie of digital creatures. Each service you monitor becomes a pet that needs feeding (uptime checks). Keep them alive long enough and they evolve. Let them starve and they die.

- **Plus tier:** Run familiars to monitor your private network
- **Max tier:** High availability with multiple familiar instances

**Try it:** [petabyte.monster](https://petabyte.monster)

---

## Why familiars?

Most uptime monitors can only check public-facing services. If you want to monitor internal services, you usually need to:

- Open ports through your firewall (security risk)
- Set up VPN access (complexity)
- Use reverse tunnels (another thing to maintain)
- Settle for basic SNMP/ICMP only (limited visibility)

**Familiars solve this.** Pure outbound HTTPS — no inbound connections, no firewall changes, no tunnels. Install it, give it a token, and it starts reporting.

---

## How it works

```
┌─────────────────────────────────────┐
│ Your Private Network                │
│                                     │
│  familiar ──checks──> LAN services  │
│      │                              │
│      │ HTTPS (outbound only)        │
└──────┼──────────────────────────────┘
       │
       ▼
  ping.petabyte.monster
       │
       ▼
  petabyte.monster core
```

1. Familiar runs inside your network (Docker, systemd, or bare Node)
2. Polls petabyte.monster for work assignments (~60s)
3. Runs checks against your LAN targets (HTTP, ping, DNS, TCP)
4. Posts results back over HTTPS
5. Your creatures stay fed (or starve if services go down)

**Outbound-only:** The familiar makes ordinary HTTPS requests out. Nothing reaches into your network. NAT/CGNAT don't matter. No firewall changes needed.

**Fail-safe:** If the familiar loses connectivity, your creatures freeze in `unknown` state — they don't die from an unfair timeout.

---

## Quick Start

### 1. Create a familiar

Dashboard → **Familiars** → **New familiar**

Copy the token (shown once — it's your familiar's authentication credential).

### 2. Deploy the connector

**Docker (easiest):**

```bash
docker run -d --restart unless-stopped \
  --name petabyte-familiar \
  -e FAMILIAR_TOKEN=paste-your-token-here \
  -e ALLOW_PRIVATE_TARGETS=true \
  petabyte/familiar:latest
```

**Docker Compose:**

```yaml
services:
  petabyte-familiar:
    image: petabyte/familiar:latest
    restart: unless-stopped
    environment:
      FAMILIAR_TOKEN: your-token-here
      ALLOW_PRIVATE_TARGETS: "true"
```

**Node (no Docker):**

```bash
# Download the connector
curl -L https://github.com/leighNZ/petabyte.monster-familiar/releases/latest/download/familiar.cjs -o familiar.cjs

# Run it
FAMILIAR_TOKEN=your-token ALLOW_PRIVATE_TARGETS=true node familiar.cjs
```

### 3. Create monitors

Dashboard → **Monitors** → **New monitor**

- Set **check via:** to your familiar
- Enter LAN target: `http://192.168.1.50:8096`
- Configure thresholds (response time, downtime tolerance)

The familiar picks up new assignments within ~60 seconds.

---

## Deployment Options

| Method | Best For | Setup |
|--------|----------|-------|
| **Docker** | Synology, Unraid, QNAP, Proxmox, Raspberry Pi | `docker run` above |
| **Docker Compose** | Declarative infrastructure | `docker-compose.yml` |
| **systemd + Docker** | Supervised container on Linux | `petabyte-familiar.service` |
| **systemd + Node** | LXC containers, VMs, bare metal | `petabyte-familiar-node.service` |
| **Standalone binary** | No Docker, no Node | *Planned* |

The connector is a single self-contained file (`familiar.cjs`) with no dependencies. Pick whatever runtime fits your infrastructure.

---

## High Availability

A single familiar instance is a single point of failure. If its host reboots or loses network, your monitors go dark until you fix it.

**Max tier** supports multiple instances per familiar (active/active). All instances check the same monitors. If one dies, the others keep reporting.

**Real HA means different failure domains:**
- Separate physical hosts
- Separate network paths
- Separate power sources

Two containers on the same box die together. Full HA guide in the dashboard: **Help → Familiars → High availability**.

---

## Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `FAMILIAR_TOKEN` | *(required)* | Authentication token from dashboard (rotate/revoke anytime) |
| `PETABYTE_PING_URL` | `https://ping.petabyte.monster` | Ping node endpoint. **Keep the default** — auto-routes to nearest node with fleet failover. Pinning a region creates a single point of failure. |
| `ALLOW_PRIVATE_TARGETS` | `true` | Allow checks against RFC1918/private IPs (the whole point of familiars) |
| `FAMILIAR_POLL_SECONDS` | `60` | How often to fetch new work assignments |
| `FAMILIAR_TICK_SECONDS` | `5` | Check scheduler granularity |

---

## ICMP Ping Checks

Ping monitors use unprivileged ICMP where supported (`net.ipv4.ping_group_range`).

If ping checks fail with "operation not permitted":

```bash
# Option 1: Widen ping_group_range
sudo sysctl -w net.ipv4.ping_group_range="0 2147483647"

# Option 2: Add NET_RAW capability (Docker)
docker run --cap-add=NET_RAW ...
```

HTTP/TCP/DNS checks need no special permissions.

---

## Security

### Authentication

- One secret: `FAMILIAR_TOKEN`
- Sent as bearer token over TLS
- Stored as SHA-256 hash (database leak can't replay tokens)
- Rotate or revoke anytime from dashboard

### Authorization

- Familiar can **only** receive your own monitors
- Familiar can **only** report on your own monitors
- Ownership enforced server-side, not trusted from agent

### Runtime

- Runs as unprivileged `node` user
- No disk writes
- No host filesystem access needed
- Only network access: LAN targets + HTTPS to ping nodes

---

## Open Source + Code Signing

This connector is **open source** (MIT license). You can read the code, fork it, and modify it.

**However:** Only signed releases can authenticate to petabyte.monster.

### Why signing?

Code signing ensures the connector running in your network is the exact code published in official releases — no backdoors, no malicious modifications, no supply chain attacks.

- **Transparent:** Full source available for auditing
- **Secure:** Modified versions can't connect to the platform
- **Safe:** Protects all users from compromised forks

### What this means

✅ **You can:**
- Read the source code
- Fork and modify it
- Use it for any purpose (MIT licensed)
- Redistribute modified versions

❌ **Modified versions:**
- Won't authenticate to petabyte.monster
- Will be rejected with HTTP 403

**Official signed releases only:** [GitHub Releases](https://github.com/leighNZ/petabyte.monster-familiar/releases)

### Building from source

```bash
git clone https://github.com/leighNZ/petabyte.monster-familiar.git
cd petabyte.monster-familiar
npm install
npm run build
```

Builds from source are **unsigned** and will be rejected by the platform (after the grace period ending Oct 25, 2026).

Want to contribute? PRs welcome. Accepted changes are signed and released by the maintainer.

**Details:** [Code Signing Documentation](docs/CODE_SIGNING.md)

---

## Connectivity & Failover

The familiar talks to **ping nodes**, not the core directly.

**Default:** `ping.petabyte.monster` (GeoDNS → nearest available node)

The connector learns the full fleet on every poll and **automatically fails over** if its current node goes down. A regional outage is invisible to you.

**Don't pin a region** unless you have a specific reason. Pinning `https://eu1.ping.petabyte.monster` makes that one node a single point of failure. The default provides automatic geographic routing and fleet-wide failover.

---

## Support & Contributing

- **Documentation:** [Full docs on GitHub](https://github.com/leighNZ/petabyte.monster-familiar)
- **Issues:** [Report bugs](https://github.com/leighNZ/petabyte.monster-familiar/issues)
- **Contributing:** [See CONTRIBUTING.md](CONTRIBUTING.md)
- **Platform help:** Dashboard → Help → Familiars

---

## License

MIT License. See [LICENSE](LICENSE) for full terms.

Copyright © 2026 petabyte.monster

**Note:** Only signed releases can authenticate to petabyte.monster. See LICENSE for authentication requirements.
