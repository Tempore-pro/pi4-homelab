# pi4-homelab

**Status: ✅ Complete**

A Raspberry Pi 4 running as an offsite, unattended, remotely-managed home server — hosted at a family member's house, administered entirely over the network.

---

## Overview

This project turns a Raspberry Pi 4 (128GB SD card, Ubuntu Server, hostname `wjw`) into a small always-on home server that lives at a family member's house and is managed entirely remotely. It runs network-wide ad-blocking DNS, monitors itself for outages, and keeps its own containers updated — all without a monitor, keyboard, or anyone physically touching it after initial setup.

The defining constraint of the project is the access model: the sole way in is [Tailscale](https://tailscale.com/) (a WireGuard-based mesh VPN), for both SSH and every web-based service. There are zero forwarded ports and zero public exposure on the home router — the device is reachable only from other machines on the same Tailscale network. That single decision shapes most of the firewall and networking work described below, and it's the difference between "a Pi with some Docker containers on it" and a server that can sit unattended on someone else's network without being a liability.

**Why it matters:** this project is a practical demonstration of Linux system administration, containerized service deployment, zero-trust remote access, and — most importantly — the kind of debugging that only shows up once a system is actually running in the real world under someone else's roof. The two incidents documented below (a Docker/firewall conflict and a UniFi routing rollback) are the real substance of the project: both were live problems, not staged exercises.

---

## Architecture

Each service runs as its own Docker Compose stack, kept in its own folder under `~/docker/<service>/`:

- **AdGuard Home** — network-wide DNS-based ad blocking, replacing an older setup that ran on Raspberry Pi OS.
- **Portainer** — web GUI for managing containers.
- **Uptime Kuma** — uptime monitoring for the Pi's services, with push notifications to [ntfy](https://ntfy.sh/) when something goes down.
- **Watchtower** — scheduled automatic container updates, with image cleanup, also notifying via ntfy.

Running alongside the containers, but not containerized itself:

- **Cockpit** — native host-level system management (installed directly on Ubuntu Server, not in Docker).

**Reliability and hardening details:**

- Every container is set to `restart: unless-stopped`, so the stack comes back on its own after a reboot or crash.
- Docker's logging is capped globally via `/etc/docker/daemon.json` (10MB per file, 3 files max) so unbounded container logs can't fill up and wear out the SD card.
- `ufw` is configured to default-deny all incoming traffic, allowing only the `tailscale0` interface — with one narrow, deliberate carve-out: `172.16.0.0/12` is allowed to reach port `9090` (Cockpit), because Docker's internal container network needs to reach Cockpit and doesn't come in over `tailscale0`.
- Network-wide ad blocking is rolled out via UniFi's DHCP settings, which hand out the Pi as the primary DNS server to every device on the network, with `1.1.1.1` set as the fallback DNS so the house doesn't lose DNS resolution entirely if the Pi ever goes down.

---

## Build Phases

**Phase 1 — Docker install**
Base Docker Engine and Docker Compose installed on Ubuntu Server.

**Phase 2 — Reliability foundations**
Global Docker log rotation configured, and `ufw` locked down to default-deny with only the `tailscale0` interface allowed — establishing the security posture before any services went on top of it.

**Phase 3 — Management GUIs**
Portainer and Cockpit set up for container and host management, respectively. A full desktop environment (Xfce + xrdp) was considered and deliberately passed over in favor of these two lighter-weight GUIs, to avoid the extra resource usage and disk writes a desktop environment would put on the SD card.

**Phase 4 — AdGuard Home restore**
AdGuard Home deployed in Docker, replacing the previous Raspberry Pi OS-based install.

**Phase 5 — Uptime Kuma monitoring**
Uptime Kuma deployed to monitor the health of the Pi's services, wired up to send push alerts via ntfy on outage.

**Phase 6 — Watchtower automation**
Watchtower deployed to keep all containers auto-updated on a schedule, with cleanup of old images and ntfy notifications on update.

**Phase 7 — House-wide DNS rollout**
UniFi's DHCP configuration updated to hand the Pi out as the network's primary DNS server (with `1.1.1.1` as fallback), making the ad-blocking effective for every device on the network rather than just the Pi itself.

---

## Known Pitfalls

- **Docker bypasses `ufw` for published container ports.** Docker manipulates `iptables` directly, and its rules take precedence over `ufw`'s — so a container port published in the usual way can end up reachable regardless of what `ufw` says. The mitigation used here is to bind Docker-published ports to the Tailscale IP specifically, rather than `0.0.0.0`, so the port is never exposed on the LAN or WAN interfaces in the first place.
- **`systemd-resolved` holds port 53 by default.** Ubuntu Server ships with `systemd-resolved` bound to port 53 out of the box, which conflicts with AdGuard Home trying to bind the same port. Its stub listener has to be disabled before AdGuard can take over DNS on the Pi.

---

## Troubleshooting Postmortems

### Postmortem 1: Cockpit showing false-down in Uptime Kuma

**Symptom:** Uptime Kuma reported Cockpit as down, flagging repeated outages — but manually running `curl` against Cockpit from the Pi itself confirmed the service was up and responding the whole time. Meanwhile, the Docker-published services (AdGuard, Portainer) showed no such issue.

**Investigation:** The discrepancy between "Kuma says down" and "curl says up" pointed at the network path rather than the service itself. Cockpit is a native, non-containerized service, so Uptime Kuma's HTTP health check to it had to cross the Docker bridge network to reach the host — a different path than a check made directly on the host. That distinction was the key: it meant Kuma's probe and a local `curl` weren't actually taking the same route to Cockpit.

**Root cause:** `ufw`'s default-deny-incoming policy was silently dropping Kuma's probe, because it arrived via the Docker bridge interface rather than `tailscale0`, the only interface `ufw` was allowing. The Docker-published services didn't show the same symptom because Docker's own `iptables` rules bypass `ufw` for published container ports — so they were reachable regardless of what `ufw` allowed, while Cockpit, a native service with no such bypass, was fully subject to `ufw`'s rules and got blocked.

**Fix:** Added one scoped `ufw` rule allowing the Docker internal network range (`172.16.0.0/12`) to reach port `9090` specifically — Cockpit's port — rather than opening the interface more broadly. This closed the gap for Kuma's health check without loosening the firewall for anything else.

This incident is a clean illustration of the seam between how `ufw` governs a native service versus how Docker-published ports evade it entirely — the same firewall behaved completely differently depending on which side of that line a service sat on.

### Postmortem 2: UniFi DNAT/Masquerade rule for DNS-hardcoding devices (built, debugged, then rolled back)

**Goal:** Some devices on the network — smart TVs and similar — ignore the DHCP-assigned DNS server entirely and hardcode a public DNS server like `8.8.8.8`, which meant they were bypassing AdGuard Home's ad-blocking no matter what DHCP said. The plan was to build a UniFi NAT policy (DNAT) that would intercept any device's outbound port-53 traffic and transparently redirect it to AdGuard, regardless of what DNS server the device thought it was talking to.

**Investigation:** The first version of the rule failed immediately — `dig` queries against the hardcoded DNS servers started coming back rejected as being from an "unexpected source." The cause turned out to be asymmetric routing: because the Pi and the client device were on the same subnet, the DNS reply from AdGuard was taking a direct path back to the client instead of routing back out through the gateway where the NAT translation was expected to happen — so the client saw a reply that didn't match up with the connection it thought it had made.

**Root cause:** The DNAT rule alone wasn't enough on a same-subnet setup; without also rewriting the source address on the way back, replies could route around the gateway and arrive looking like they came from the wrong place.

**Fix:** Added a Masquerade (SNAT) rule alongside the DNAT rule, so return traffic was forced back through the gateway and NAT'd correctly. To avoid masquerading the entire LAN and losing per-device visibility in AdGuard's query logs, the rule was scoped to a source allowlist of the specific known DNS-hardcoding devices rather than applied network-wide. Verified working via `dig` tests and by confirming the redirected queries showed up correctly attributed in AdGuard's query log.

**Later decision — rollback:** After the rule had been running for a while, a Pi outage exposed a reliability trade-off that hadn't been obvious during testing: the devices on the DNAT intercept list lost DNS resolution *entirely* during the outage, because the rule was redirecting even their fallback DNS queries to the now-unreachable Pi — they had no way to fall through to a working DNS server the way the rest of the network could via the `1.1.1.1` fallback. Once an actual outage left the TVs unable to resolve anything, the call was made to remove the DNAT/Masquerade rules entirely, accepting incomplete ad-blocking coverage on the handful of hardcoded-DNS devices in exchange for not creating a single point of failure for those devices' internet access.

This is as much a judgment call as a technical fix: the rule worked exactly as designed, but the outage made clear that "more ad-blocking coverage" wasn't worth "some devices go fully offline if the Pi does." The decision to roll it back was about home-network reliability taking priority over blocking coverage completeness.

---

## Future Work

- [ ] `restic` backups to a second machine
- [ ] Self-hosted `ntfy` (currently using the hosted service)
- [ ] [Homepage](https://gethomepage.dev/) dashboard for the service stack
# pi4-homelab
Raspberry Pi 4 offsite home server: Docker services, AdGuard Home DNS, Tailscale-only access
