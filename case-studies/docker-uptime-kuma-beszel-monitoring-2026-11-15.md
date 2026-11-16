# Case Study: Adding Docker-Based Monitoring to My Pi-hole Raspberry Pi

**Date:** November 15, 2026 (Pacific time)  
**Status:** Docker and Uptime Kuma running; Beszel hub running, **agent collection not yet operational**  
**Project:** Existing Raspberry Pi 400 home-DNS server, not a new lab or a production SOC environment

## Objective and constraints

I extended my existing Raspberry Pi 400 DNS server with containerized monitoring while keeping the working Pi-hole and Unbound services in place. I wanted to monitor three different service dependencies—my router, public internet reachability, and DNS resolution—without confusing a network reachability check with a security alert.

I also needed to account for the Pi's limited microSD storage and protect the Uptime Kuma monitoring configuration before switching Docker images.

**Scope:** This report records what I actually configured, observed, and verified on November 15. It does not claim a long-term uptime history, an attack detection, or a functioning Beszel agent where one has not been verified.

## Existing host and DNS services

| Component | Observed configuration |
| --- | --- |
| Hardware | Raspberry Pi 400, ARM64 |
| Operating system | Debian 13.7 / Trixie |
| Memory | Approximately 3.7 GiB available to the system |
| Storage | Approximately 15 GiB microSD filesystem |
| Host | `home-dns` (local hostname; private LAN address omitted) |
| Networking | Wi-Fi (`wlan0`); local router and Pi-hole addresses redacted in this public report |
| Pi-hole | Existing **native host service**, not moved into Docker |
| Unbound | Existing recursive resolver active on `127.0.0.1:5335` |
| Docker Engine | Version `29.9.3`, verified active |
| Docker Compose | Version `v5.7.1`, verified installed |

I kept Pi-hole/Unbound separate from the new containers. After the Uptime Kuma image change, a Pi-hole status check and a local `dig @127.0.0.1 example.com` test succeeded. Those results establish a working local DNS response at the time of the test, **not** universal DNS reachability from every client or proof of uninterrupted service throughout the maintenance window.

## Uptime Kuma deployment and monitoring

I deployed Uptime Kuma in Docker using Compose, with its application state on a persistent named volume (`uptime_kuma_data`), a restart policy of `unless-stopped`, and the web interface published only on my Pi's LAN address at TCP port `3001`.

Three Uptime Kuma monitors were configured and shown **Up / green** in the captured dashboard state:

| Monitor name | Check type | Target | Observed evidence |
| --- | --- | --- | --- |
| Home Router | ICMP ping | My LAN router (`<ROUTER_LAN_IP>`) | Up; approximately 4.06 ms latency in the observed snapshot |
| Internet Connectivity | ICMP ping | `1.1.1.1` | Up; approximately 13.10 ms latency in the observed snapshot |
| Pi-hole DNS Resolution | DNS A-record lookup | `example.com` using `<PIHOLE_LAN_IP>` on port `53` | Up; approximately 5 ms current / 8.33 ms average response time in the observed snapshot |

The screenshot displayed 100% for these monitors **at that point in the recorded observation**. It is not evidence of weeks or months of uninterrupted availability. The dashboard verification occurred **before** the later slim-image migration; a new post-migration monitor screenshot has not been archived in this case study.

The checks isolate three useful failure categories: local gateway access, outside-network reachability, and a DNS query answered by the specific Pi-hole resolver. They are **availability and diagnostic checks**, not intrusion-detection rules.

## Backup verification before changing the image

Before changing the Uptime Kuma image, I backed up its persistent application data and verified the archive rather than relying only on a successful copy.

- Archive size recorded: **35,159 bytes**.
- `gzip -t` passed the compressed-archive integrity check.
- The archive's contents included `data/kuma.db` and `data/db-config.json`.
- A SHA-256 comparison between the Raspberry Pi copy and the Windows copy matched.

These tests support **archive readability and copy integrity**. I did not perform and verify a full restore into a clean Uptime Kuma instance, so restore success remains untested. I have not published the database or backup archive because it can contain internal monitoring configuration.

## Storage-conscious image migration

The initial Uptime Kuma image, `louislam/uptime-kuma:2`, was observed at approximately **2.57 GB**. I migrated to `louislam/uptime-kuma:2-slim`, observed at approximately **880 MB**, retained the persistent data volume, started the slim-image container, and removed the old image.

| Storage observation | Before | After |
| --- | ---: | ---: |
| Host available disk space | Approximately 6.3 GB | Approximately 8.1 GB |
| Uptime Kuma image size observed | Approximately 2.57 GB (`:2`) | Approximately 880 MB (`:2-slim`) |

The space recovery is consistent with the image replacement and old-image removal. These readings are the observed measurements, not a controlled storage benchmark. The container was started following migration, but I am **not** claiming a fresh, captured A/B/A monitor-availability test after that change.

## Beszel host-monitoring extension — partial completion

I also deployed the Beszel **Hub** container, based on `henrygd/beszel:latest`, with the interface bound to the Pi's LAN address at TCP port `8090`.

- The hub container was observed **running**; its interface reported version **0.21.2**.
- Container mount locations `/beszel_data` and `/beszel_socket` were verified.
- I added/registered the `home-dns` system in the hub interface.
- The dashboard initially displayed no systems. Subsequently, the registered `home-dns` system was **Down/pending**.
- A container inventory showed **only the `beszel` hub container**: the **Beszel agent was not installed or running** in the verified session.

**Open issue:** Beszel cannot report host CPU, memory, disk or other system metrics until the agent is actually deployed, configured and shown connected. The agent's required `KEY` / `TOKEN` values were not supplied in the verified work. Any later credentials must remain in private configuration and must **never** be committed to this repository.

The successful hub deployment is therefore **not** proof of end-to-end Beszel monitoring.

## Verification boundaries and next checks

| Claim | Status on November 15 |
| --- | --- |
| Docker Engine / Compose available | **Verified** by version and service checks |
| Existing Pi-hole / Unbound local DNS working after maintenance | **Verified** by status and local DNS query |
| Three Uptime Kuma checks showed Up | **Verified snapshot before image migration** |
| Uptime Kuma application-state archive and transferred copy intact | **Verified** gzip test, expected entries and matching SHA-256 |
| Slim Uptime Kuma image started; old image removed | **Verified**, with before/after disk-space observations |
| Uptime Kuma checks reverified after slim migration | **Not independently archived** |
| Backup can be restored into a clean instance | **Not tested** |
| Beszel hub running and system registered | **Verified** |
| Beszel agent connected / host metrics collecting | **Not complete** |
| Long-term availability, alert delivery or incident-detection capability | **Not demonstrated** |

A safe follow-up is to verify the three monitors again after the image switch, separately deploy and validate the Beszel agent with its credentials kept outside Git, and capture only sanitized dashboard evidence. I will not change the working Pi-hole/Unbound DNS setup or present an availability monitor as a security detection system.

## Why this matters for my security-operations development

This project moves beyond a single DNS installation into **service observability and operational validation**. I exercised Linux administration, container lifecycle management, persistent storage and backup verification, network and DNS availability checks, resource constraints, and honest reporting of incomplete deployment.

These are foundational skills for understanding the availability and health of infrastructure monitored by a SOC. They **do not replace** event-log analysis or alert triage. My planned final controlled security-event exercise will address that additional analytical step.

## Evidence and privacy

This report is transcribed from the November 15 command outputs and dashboard observations. No fresh evidence images, Docker Compose file, monitoring database, secrets or raw host logs are published with this case study. LAN addresses are represented by placeholders. Commands and configurations not independently captured should not be treated as fully reproducible deployment artifacts.

*Documentation note: I used writing assistance to organize this report from the commands, results and screenshots I reviewed during my investigation. I distinguish observed results from interpretations and unfinished tests; writing assistance is not evidence of a technical result.*
