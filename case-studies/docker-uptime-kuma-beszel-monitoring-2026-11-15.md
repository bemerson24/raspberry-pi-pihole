# Case Study: Adding Docker-Based Monitoring to My Pi-hole Raspberry Pi

**Dates:** November 15–16, 2026 (Pacific time)  
**Status as of November 16:** Pi-hole-to-Unbound forwarding previously confirmed; post-reboot native DNS tests passed; Uptime Kuma slim image deployed and monitoring checks reverified. Beszel Hub/Agent reporting and a cgroup memory controller repair were validated with both nonzero Docker memory statistics and refreshed Beszel Docker CPU, memory and network graphs. Root storage, I/O, temperature, system load and Wi-Fi signal panels also populated. Four privacy-reviewed Beszel dashboard screenshots are now published; the Uptime Kuma screenshot remains unpublished.  
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

**Evidence handling:** The screenshot was inspected during the investigation; a sanitized copy is **not yet committed to this repository**. It remains a session observation rather than a publicly archived image. Disk/network/temperature graphs were still unverified at this point in the sequence and were subsequently checked in the post-reboot follow-up below. Alert delivery remains untested.

## November 16 follow-up — Beszel CPU and memory panels

I revisited Beszel's `home-dns` dashboard and captured its one-hour view. The system remained **Up** over the local Unix socket `beszel.sock`. The page identified Debian GNU/Linux 13 (Trixie), Cortex-A72, **3.71 GB** system memory and approximately **34 days** of host uptime.

- **Host CPU Usage:** A populated timeline showed roughly **0.7–0.9%** system-wide utilization through most of the visible interval, with a brief peak near **1.8%**. These are screen-read approximations, not exported samples.
- **Host Memory Usage:** A populated, nearly flat memory chart appeared. This verifies collection and display of host memory data; its colored bands were not independently decoded into an exact used/free memory amount.
- **Docker CPU Usage and Docker Memory Usage:** Both charts were visible but appeared **flat at zero**. This is **not** sufficient to conclude that per-container metrics are correctly collected, that no container resources were consumed, or that the Agent is broken. Container-level monitoring remains to be checked against an independent observation.
- **Disk, network, temperature:** Not visible in this screenshot, and therefore not yet verified from it.

The screenshot provides stronger evidence of operational **host** monitoring than the earlier cropped view, but it does not validate alert delivery, long-term retention, per-container resource accounting, or any security-event detection. The screenshot has been reviewed in the working session but **not published as a sanitized artifact**.

## November 16 follow-up — Docker memory accounting root cause and verified repair

While checking Beszel, the Docker CPU and memory charts appeared flat. Instead of changing Beszel without evidence, I compared the dashboard with Docker's own read-only statistics.

### Pre-change observations

```text
sudo docker stats --no-stream
beszel-agent   0.00% CPU   0B / 0B memory
beszel         0.00% CPU   0B / 0B memory
uptime-kuma    0.18% CPU   0B / 0B memory
```

`sudo docker info` showed `Cgroup Driver: systemd` and `Cgroup Version: 2`, together with `WARNING: No memory limit support` and `WARNING: No swap limit support`. The effective kernel command line contained `cgroup_disable=memory`; `/sys/fs/cgroup/cgroup.controllers` listed only `cpuset cpu io pids`, confirming that the memory controller was unavailable.

### Controlled change

I confirmed that `/boot/firmware/cmdline.txt` existed, contained **one line** and did not already include a cgroup override. Before editing, I copied it to a timestamped backup:

```text
/boot/firmware/cmdline.txt.bak-20261116-110739
```

I appended `cgroup_enable=memory` to the existing single line rather than replacing existing boot parameters. A line-count check returned **1**; `grep` found the new parameter; and a `diff` after removing only the new trailing parameter printed `Original parameters preserved`. I then rebooted the Pi as a planned service interruption. **This procedure temporarily interrupted the household DNS/monitoring host; it was not a live Docker change.**

### Post-reboot evidence

```text
cat /sys/fs/cgroup/cgroup.controllers
cpuset cpu io memory pids
```

`sudo docker info` continued to report systemd/cgroup v2 and the same `6.18.50+rpt-rpi-v8` kernel, without the earlier memory/swap warning lines in the filtered output. `sudo docker stats --no-stream` now showed:

| Container | CPU at snapshot | Memory used / reported limit | Memory percentage |
| --- | ---: | ---: | ---: |
| `beszel-agent` | 0.00% | 16.92 MiB / 3.707 GiB | 0.45% |
| `beszel` | 0.00% | 36.03 MiB / 3.707 GiB | 0.95% |
| `uptime-kuma` | 0.19% | 200.6 MiB / 3.707 GiB | 5.28% |

All three containers appeared in the stats output with nonzero memory readings. The observed network/block I/O values also showed activity for some containers. These values are **one post-reboot sample**, not sustained load measurements.

**Disposition:** The explicitly disabled kernel memory cgroup controller was the evidence-supported cause of Docker reporting `0B / 0B` memory for every container. The override successfully restored kernel memory-controller availability and Docker memory accounting. A refreshed Beszel dashboard was **not yet inspected at the time of that Docker command**; subsequent November 16 screenshots confirmed per-container graphs, as documented below. Pi-hole and Unbound service recovery was subsequently tested by service status and direct local DNS queries as documented below; Docker stats alone did not establish DNS service health. No secrets or LAN addresses were included in this report.

## November 16 follow-up — Native DNS recovery verified after reboot

After the cgroup memory controller repair and reboot, I checked the two native DNS services and sent individual DNS lookups to their local listeners:

```text
sudo systemctl is-active pihole-FTL unbound
active
active

dig @127.0.0.1 example.com +short
104.20.23.154
172.66.147.243

dig @127.0.0.1 -p 5335 example.com +short
172.66.147.243
104.20.23.154
```

**Result:** Both `pihole-FTL` and `unbound` were active and both local resolvers returned A records for `example.com` after reboot. The differing order of returned addresses is not a failure. This validates each service's local lookup path at the time of observation. The earlier, separate Pi-hole forwarding log established that a specific query was forwarded to Unbound; the two direct queries here do not independently trace the forwarding path or demonstrate DNS availability for every household client. A refreshed Beszel dashboard was subsequently checked, as documented below.

## November 16 follow-up — Beszel dashboard confirms post-reboot recovery

I re-opened Beszel's `home-dns` system after the controlled reboot. The header showed **Up**, with the local socket connection `beszel.sock`, Debian GNU/Linux 13 (Trixie), Cortex-A72, and 3.71 GB RAM. The reported host uptime was about **10 minutes** at capture, consistent with a recent reboot.

The new one-hour dashboard views showed the following **observed** results (approximate screen-read values, not a dataset export):

| Beszel panel | Observation | Conclusion |
| --- | --- | --- |
| Host CPU and memory | Continuous host CPU and memory curves; host CPU briefly spiked during restart | Host-level metrics collecting |
| Docker CPU | Newly populated utilization curves and brief activity spike following restart | Container CPU metrics displayed |
| Docker Memory | Previously flat at zero, now three colored series totaling approximately 260 MB near capture | **Container memory reporting restored end-to-end** in Beszel |
| Docker Network I/O | Newly populated incoming and outgoing traffic series | Container network accounting displayed |
| Root Usage and Root I/O | Root disk usage displayed around 5–6 GB; root throughput showed nonzero I/O peaks | Host storage and I/O panels active |
| Bandwidth / Load Average | Host traffic and load curves populated | Host network/load panels active |
| Temperature | CPU thermal sensor plotted around 30–35°C in this view | Temperature sensor reporting (not a long-duration thermal test) |
| Wi-Fi signal | Interface strength fluctuated approximately −44 to −54 dBm | Wi-Fi monitoring reporting |

The plotted transition after reboot aligns with the `docker stats` result of nonzero memory usage for all three containers. The plots substantiate monitoring recovery at the observed time; the data do **not** establish uninterrupted monitoring across the reboot, alert delivery, precise per-container attribution without inspecting labels, or a production security-detection capability.

**Privacy and provenance:** The original screenshots remain private. I published four privacy-reviewed copies, preserving the actual plotted data. In the fourth image I obscured the private Wi-Fi SSID with an opaque mask and cropped out an incomplete container table containing a private LAN address. The source files, editing scope and SHA-256 hashes are documented in the [Beszel evidence notes](../evidence/beszel/README.md). I did not redraw chart lines, fabricate measurements, or disclose credentials.

### Published Beszel dashboard evidence — November 16

The captured panels cover a selected approximately **11:40 AM–12:40 PM Pacific one-hour view**, not a demonstrated period of uninterrupted uptime. They show `home-dns` Up, populated host and Docker CPU timelines, nonzero container memory, and active storage, network, load, temperature and Wi-Fi charts. These are dashboard observations, not exported raw telemetry or independent proof that every background service or household DNS client was continuously healthy.

- [01 — Beszel host and Docker CPU, with host memory](../evidence/beszel/01-beszel-host-and-docker-cpu-2026-11-16.png)
- [02 — Docker memory, root disk usage and root I/O](../evidence/beszel/02-beszel-docker-memory-and-storage-2026-11-16.png)
- [03 — Host bandwidth, Docker network I/O and load averages](../evidence/beszel/03-beszel-network-and-load-2026-11-16.png)
- [04 — CPU temperature and Wi-Fi signal (SSID redacted)](../evidence/beszel/04-beszel-temperature-wifi-2026-11-16.png)

The independent `docker stats --no-stream` measurements above support the conclusion that memory accounting was restored. These screenshots corroborate that Beszel subsequently displayed nonzero per-container memory metrics. They do **not** demonstrate alert delivery, a clean backup restoration or security-event detection.

## Verification boundaries and next checks

| Claim | Status as of November 16 |
| --- | --- |
| Docker Engine / Compose available | **Verified** by version and service checks |
| Existing Pi-hole / Unbound local DNS working after maintenance | **Verified** by status and local DNS query |
| Three Uptime Kuma checks showed Up | **Verified snapshot before image migration** |
| Uptime Kuma application-state archive and transferred copy intact | **Verified** gzip test, expected entries and matching SHA-256 |
| Slim Uptime Kuma image started; old image removed | **Verified**, with before/after disk-space observations |
| Uptime Kuma checks reverified after slim migration | **Verified in November 16 dashboard screenshot** (all four shown Up); public evidence image not yet committed |
| Backup can be restored into a clean instance | **Not tested** |
| Beszel hub running and system registered | **Verified** |
| Beszel agent connected / host metrics collecting | **Verified in November 16 screenshots** for host CPU/memory, Docker CPU/memory/network, root disk/I/O, bandwidth, load, CPU temperature and Wi-Fi signal panels; four privacy-reviewed images now published |
| Long-term availability, alert delivery or incident-detection capability | **Not demonstrated** |

The post-reboot Beszel graphs and native Pi-hole/Unbound DNS checks have passed. Four sanitized Beszel evidence images are now published. Remaining optional work includes privacy-reviewed Uptime Kuma image publication, alert-delivery testing, a clean backup restore and longer observation. None requires repeating the completed cgroup repair or redeploying the Agent. I will not change the working Pi-hole/Unbound DNS setup or present an availability monitor as a security detection system.

## Why this matters for my security-operations development

This project moves beyond a single DNS installation into **service observability and operational validation**. I exercised Linux administration, container lifecycle management, persistent storage and backup verification, network and DNS availability checks, resource constraints, and honest reporting of incomplete deployment.

These are foundational skills for understanding the availability and health of infrastructure monitored by a SOC. They **do not replace** event-log analysis or alert triage. My planned final controlled security-event exercise will address that additional analytical step.

## Evidence and privacy

This report is based on the November 15–16 command outputs and dashboard observations. Four sanitized Beszel screenshots and their provenance notes are now public in [evidence/beszel](../evidence/beszel/README.md). The original unredacted screenshots, Uptime Kuma dashboard screenshot, Docker Compose files, monitoring database, secrets and raw host logs are not published. LAN addresses in the written report remain placeholders. Commands and configurations not independently captured should not be treated as fully reproducible deployment artifacts.

*Documentation note: I used writing assistance to organize this report from the commands, results and screenshots I reviewed during my investigation. I distinguish observed results from interpretations and unfinished tests; writing assistance is not evidence of a technical result.*
