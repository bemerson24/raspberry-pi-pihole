# Case Study: Adding Docker-Based Monitoring to My Pi-hole Raspberry Pi

**Dates:** November 15–16, 2026 (Pacific time)  
**Status as of November 16:** Pi-hole forwarding to Unbound confirmed; Uptime Kuma slim image deployed and monitors rechecked; Beszel Hub and Agent connected with live host CPU and memory graphs. Container-level Docker metrics and some other hardware panels still need verification.  
**Project:** Existing Raspberry Pi 400 home-DNS server, not a new lab or a production SOC environment

## Objective and constraints

I extended my existing Raspberry Pi 400 DNS server with containerized monitoring while keeping the working Pi-hole and Unbound services in place. I wanted to monitor three different service dependencies—my router, public internet reachability, and DNS resolution—without confusing a network reachability check with a security alert.

I also needed to account for the Pi's limited microSD storage and protect the Uptime Kuma monitoring configuration before switching Docker images.

**Scope:** This report records my November 15 baseline and November 16 Agent follow-up separately. It does not claim a long-term uptime history, an attack detection, or metrics not visible in the evidence.

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

## Pi-hole to Unbound forwarding verification — November 15

I separately installed and configured Unbound to listen only on `127.0.0.1:5335` for TCP and UDP. I preserved Debian's root trust-anchor configuration, validated it with `sudo unbound-checkconf`, and kept the `unbound-resolvconf.service` integration disabled.

My direct resolver tests returned `NOERROR` and the `ad` flag for `dnssec.works`, and `SERVFAIL` for the intentionally invalid `fail01.dnssec.works` domain. The invalid-domain test had an initial timeout before the final SERVFAIL response.

Pi-hole initially reported Quad9 upstream servers (`9.9.9.9` and `149.112.112.112`). I changed `dns.upstreams` to the local Unbound service and re-read the setting as `[ 127.0.0.1#5335 ]`. A subsequent `dig @127.0.0.1 en.wikipedia.org A` succeeded; Pi-hole's log explicitly recorded:

```text
forwarded en.wikipedia.org to 127.0.0.1#5335
```

This verifies that the particular request was forwarded to Unbound. It does not prove every household client uses this resolver or that DNS is continuously available.

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

## Beszel host-monitoring extension — November 15 baseline

I also deployed the Beszel **Hub** container, based on `henrygd/beszel:latest`, with the interface bound to the Pi's LAN address at TCP port `8090`.

- The hub container was observed **running**; its interface reported version **0.21.2**.
- Container mount locations `/beszel_data` and `/beszel_socket` were verified.
- I added/registered the `home-dns` system in the hub interface.
- The dashboard initially displayed no systems. Subsequently, the registered `home-dns` system was **Down/pending**.
- A container inventory showed **only the `beszel` hub container**: the **Beszel agent was not installed or running** in the verified session.

**November 15 open issue (resolved November 16):** At this point the Hub alone could not report host metrics. The Agent was not yet operational. See the dated follow-up below. Agent credentials must remain in private configuration, never in Git.

The successful hub deployment is therefore **not** proof of end-to-end Beszel monitoring.

## November 16 follow-up — Beszel Agent connection verified

I extended the existing Beszel Compose deployment with an Agent container and a local Unix socket shared with the Hub. The Hub's mount list confirmed `/beszel_data` and `/beszel_socket`, and the registered system used `beszel.sock` as its connection.

The Agent initially entered a restart loop with exit code 1. Its logs specifically reported `Failed to parse key: YOUR_BESZEL_PUBLIC_KEY`. I traced that failure to a placeholder instead of the real Beszel-generated public key. A second validation failure arose because the public key appeared as a bare first line in `.env`, not in the `BESZEL_KEY='...'` assignment expected by Docker Compose.

I corrected the private environment-file structure, kept the file permission at `600`, and successfully ran `sudo docker compose config --quiet`. I then recreated only the Agent with `sudo docker compose up -d --no-deps --force-recreate beszel-agent`. The next `docker compose ps` output showed both `beszel` and `beszel-agent` **Up**, with the Agent still up at a subsequent three-minute check.

A later Beszel dashboard screenshot showed the `home-dns` system **Up** using `beszel.sock`, with Debian GNU/Linux 13 (Trixie), Cortex-A72, approximately 3.71 GB RAM and 34 days host uptime. Live host CPU readings began to populate: system CPU briefly spiked to approximately 29% in the small sample and then fell near idle; the visible Docker CPU plot remained near zero. That first image did not establish successful collection of per-container metrics.

**Verified:** Agent process stability over the observed interval, successful Hub-to-Agent connection, reported system metadata, and initial CPU metrics. **Not yet verified from that first cropped screenshot:** complete memory, disk, temperature and network metric graphs, alert delivery, long-term retention, or a post-slim Uptime Kuma capture. Later screenshots address some of these items below. I did not publish the real Agent key, token, private `.env` file, or raw configuration.

## November 16 follow-up — Uptime Kuma monitors reverified after slim-image migration

After the earlier `louislam/uptime-kuma:2-slim` migration, I revisited the Uptime Kuma dashboard and captured a screenshot. **All four displayed monitors were green, showed 100% in the dashboard's reported periods, and displayed current green history bars:** Uptime Kuma (self-check), Home Router, Internet Connectivity, and Pi-hole DNS Resolution.

The selected **Pi-hole DNS Resolution** monitor was **Up**, querying `example.com` for an A record every **60 seconds**. The latest result showed IPv4 answers `104.20.23.154` and `172.66.147.243`, a **2 ms** current response and **4.19 ms** displayed average. Its visible green-bar timeline covered approximately **28 minutes**.

**Interpretation:** This screenshot demonstrates the monitor configuration and a functioning DNS check **after** the slim-image migration, together with currently Up status for the other three displayed monitors. The interface's reported 24-hour / 30-day / 1-year 100% values do **not** independently prove continuous monitoring over those entire periods. This is **not** a clean restore test, a measurement of alert delivery, or security-event detection.

**Evidence handling:** The screenshot was inspected during the investigation; a sanitized copy is **not yet committed to this repository**. It remains a session observation rather than a publicly archived image. Beszel resource details are assessed in the following follow-up; disk/network/temperature graphs and alert delivery remain outstanding.

## November 16 follow-up — Beszel CPU and memory panels

I revisited Beszel's `home-dns` dashboard and captured its one-hour view. The system remained **Up** over the local Unix socket `beszel.sock`. The page identified Debian GNU/Linux 13 (Trixie), Cortex-A72, **3.71 GB** system memory and approximately **34 days** of host uptime.

- **Host CPU Usage:** A populated timeline showed roughly **0.7–0.9%** system-wide utilization through most of the visible interval, with a brief peak near **1.8%**. These are screen-read approximations, not exported samples.
- **Host Memory Usage:** A populated, nearly flat memory chart appeared. This verifies collection and display of host memory data; its colored bands were not independently decoded into an exact used/free memory amount.
- **Docker CPU Usage and Docker Memory Usage:** Both charts were visible but appeared **flat at zero**. This is **not** sufficient to conclude that per-container metrics are correctly collected, that no container resources were consumed, or that the Agent is broken. Container-level monitoring remains to be checked against an independent observation.
- **Disk, network, temperature:** Not visible in this screenshot, and therefore not yet verified from it.

The screenshot provides stronger evidence of operational **host** monitoring than the earlier cropped view, but it does not validate alert delivery, long-term retention, per-container resource accounting, or any security-event detection. The screenshot has been reviewed in the working session but **not published as a sanitized artifact**.

## Verification boundaries and next checks

| Claim | Status on November 15 |
| --- | --- |
| Docker Engine / Compose available | **Verified** by version and service checks |
| Existing Pi-hole / Unbound local DNS working after maintenance | **Verified** by status and local DNS query |
| Three Uptime Kuma checks showed Up | **Verified snapshot before image migration** |
| Uptime Kuma application-state archive and transferred copy intact | **Verified** gzip test, expected entries and matching SHA-256 |
| Slim Uptime Kuma image started; old image removed | **Verified**, with before/after disk-space observations |
| Uptime Kuma checks reverified after slim migration | **Verified in November 16 dashboard screenshot** (all four shown Up); public evidence image not yet committed |
| Backup can be restored into a clean instance | **Not tested** |
| Beszel hub running and system registered | **Verified** |
| Beszel agent connected / host metrics collecting | **Verified** for host CPU and memory in November 16 dashboard; Docker CPU/memory panels show flat zero and need independent corroboration; disk/network/temperature not yet verified |
| Long-term availability, alert delivery or incident-detection capability | **Not demonstrated** |

The next safe follow-up is to inspect Beszel's disk, network and temperature panels, independently check whether Docker resource metrics should be nonzero, and separately validate any alerting behavior if needed. Capture sanitized monitoring evidence before publishing. The agent is already connected and the Uptime Kuma monitors have been rechecked; neither needs to be redeployed simply to repeat the completed work. I will not change the working Pi-hole/Unbound DNS setup or present an availability monitor as a security detection system.

## Why this matters for my security-operations development

This project moves beyond a single DNS installation into **service observability and operational validation**. I exercised Linux administration, container lifecycle management, persistent storage and backup verification, network and DNS availability checks, resource constraints, and honest reporting of incomplete deployment.

These are foundational skills for understanding the availability and health of infrastructure monitored by a SOC. They **do not replace** event-log analysis or alert triage. My planned final controlled security-event exercise will address that additional analytical step.

## Evidence and privacy

This report is transcribed from the November 15–16 command outputs and dashboard observations. No fresh evidence images, Docker Compose file, monitoring database, secrets or raw host logs are published with this case study. LAN addresses are represented by placeholders. Commands and configurations not independently captured should not be treated as fully reproducible deployment artifacts.

*Documentation note: I used writing assistance to organize this report from the commands, results and screenshots I reviewed during my investigation. I distinguish observed results from interpretations and unfinished tests; writing assistance is not evidence of a technical result.*
