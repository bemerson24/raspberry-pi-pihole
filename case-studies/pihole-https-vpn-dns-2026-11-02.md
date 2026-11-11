# Pi-hole HTTPS and VPN DNS troubleshooting

**Date:** November 2, 2026  
**Status:** HTTPS dashboard access restored on the Windows laptop. Local DNS behavior compared with VPN connected and disconnected. Earlier campus and battery-related browsing failures remain unresolved.

## What I was trying to do

I wanted to secure access to my Pi-hole dashboard. HTTPS was already available, but getting to the dashboard exposed separate problems with name resolution, certificate names and certificate trust.

This continues the [laptop DNS portability investigation](laptop-dns-portability-vpn-2026-10-29.md). The tests below took place at home, not on the campus network.

## Observations and tests

| Test or observation | Result | What it established |
| --- | --- | --- |
| Open `https://pi.hole/admin/` in Firefox | Firefox reported that its Cloudflare secure DNS provider could not find the name | The browser was asking an external resolver for a local name |
| Open HTTPS using the Pi's LAN IP | Bitdefender blocked an untrusted certificate | An HTTPS endpoint was responding; certificate trust still needed work |
| Inspect the certificate through SSH | Subject `CN=pi.hole`; issuer `CN=pi.hole, O=Pi-hole, C=DE`; SAN `DNS:pi.hole` | The certificate covered the hostname, not the LAN IP |
| Check certificate dates | October 12–November 28, 2026 | The server certificate was within its validity period on the test date |
| Allow Firefox to use the system resolver for `pi.hole` | Name resolution still failed | The browser exception alone did not resolve the problem |
| Windows lookup with VPN connected | Default resolver `198.18.0.33` returned non-existent domain for `pi.hole`; direct queries to Pi-hole timed out | The VPN-connected state affected local DNS access |
| Repeat both lookups after disconnecting Bitdefender VPN | Both resolved `pi.hole` to the Pi's local addresses | Disconnecting the VPN restored local DNS resolution in this test |
| Import Pi-hole's CA into the Windows current-user trusted root store | Dashboard access reported working afterward | Dashboard access worked after the trust change, with the VPN disconnected; this does not isolate every contributing setting |
| Query `example.com` with VPN on and off | Both returned the same set of IPv4 and IPv6 addresses | Public DNS worked through both paths during this later comparison |

The Pi also resolved its own name successfully. That was a server-side check, not evidence that the Windows client could reach its DNS service. Repeating the commands from Windows was necessary.

## What the DNS comparison supports

Pi-hole filters DNS queries that reach it; its presence on the home network does not establish that every application or VPN uses it. In the recorded VPN-connected test, the default resolver returned a negative answer for `pi.hole` and direct queries to Pi-hole timed out. With the VPN disconnected, both lookups succeeded.

The later successful `example.com` lookups establish that public-name resolution worked during that comparison. Matching returned addresses do not establish that both queries used the same resolver or passed through Pi-hole. Local-name resolution and public-name resolution must be tested separately.

**Evidence boundary:** These are documented observations from the November 2 session. The exact VPN routing, DNS interception or filtering mechanism remains unverified.

## SSH and certificate handling

My first SSH attempt failed with `Permission denied (publickey)`. The working key was stored in Windows under a custom filename. Specifying it with `ssh -i` allowed login. WSL's SSH directory only contained `known_hosts`; switching shells was not the fix.

On the Pi, I inspected the existing server certificate:

```bash
sudo openssl x509 -in /etc/pihole/tls.pem -noout -subject -issuer -dates -ext subjectAltName
```

The initial SCP request for `/etc/pihole/tls_ca.crt` failed because the SSH account could not read that protected path directly. I used sudo to make a readable copy of the **public CA certificate** in the account's home directory, then downloaded it through the authenticated SSH connection.

I imported that downloaded CA on Windows:

```powershell
Import-Certificate -FilePath "$env:USERPROFILE\Downloads\pihole-ca.crt" -CertStoreLocation Cert:\CurrentUser\Root
```

This deliberately trusts certificates signed by that CA for applications using this Windows trust store. The file imported was the CA certificate, not `tls.pem`, which can contain private key material. No private keys or certificate files are included in this repository.

## What I learned

Cookies maintain a login session; HTTPS protects traffic in transit. They are separate parts of the connection.

A certificate can be within its valid dates and still fail validation because its name does not match the URL or its issuer is not trusted. Using `pi.hole` addressed the name mismatch. Importing the CA addressed trust.

Browser secure DNS, Windows DNS and VPN DNS can follow different paths. A working SSH connection to the Pi did not prove DNS queries to it would work. Likewise, a successful public DNS lookup did not prove local names would resolve.

## Limits and follow-up

- The successful dashboard test was performed with the VPN disconnected. A permanent arrangement for VPN use alongside local Pi-hole access is not yet validated.
- HTTPS here protects dashboard access. It does not establish encrypted DNS for household clients or upstream DNS traffic.
- The VPN on/off comparison supports a local DNS interaction. It does not identify the exact VPN setting or packet-filtering mechanism.
- No failure-state campus comparison was captured in this November 2 home session. The [November 3 campus follow-up](https://github.com/bemerson24/cybersecurity-virtual-lab-environment/blob/main/docs/tcp-tls-session-analysis-2026-11-03.md) later documented successful public DNS and command-line HTTPS checks alongside a Firefox-without-VPN failure. Its root cause and the earlier apparent battery association remain unresolved.
- A future recurrence should be tested before restarting: record the network, VPN state, resolver and `nslookup example.com` results with VPN on and off. Firefox secure DNS should also be considered if browser behavior differs from command-line results.
- Automatic certificate renewal and trust behavior after renewal have not been verified.

## References

- [Pi-hole TLS/SSL documentation](https://docs.pi-hole.net/api/tls/) — CA trust and hostname requirements.
- [Pi-hole webserver configuration](https://docs.pi-hole.net/ftldns/configfile/) — HTTPS and webserver settings.

*Documentation prepared with ChatGPT assistance from my command output, screenshots and reported results. The observations above are from this troubleshooting session; follow-up items are not completed tests.*

## Follow-up validation — November 11, 2026

**Purpose:** Repeat the Windows-to-Pi-hole DNS test under controlled Bitdefender VPN connection states, and record the Pi-hole baseline. These are home-network observations; this is not an identified security incident.

### Environment and baseline

- Raspberry Pi 400 running Linux kernel `6.18.50+rpt-rpi-v8` (reported by the SSH login banner).
- Pi-hole Core **v6.4.3**, Web **v6.6**, FTL **v6.7.1**, as reported by `pihole -v` on November 11. Each component reported an update available; no update was performed during these tests.
- `sudo pihole status` reported FTL listening on port 53 over TCP and UDP for both IPv4 and IPv6, and blocking enabled. An initial unprivileged status check produced a configuration-read permission warning, which did not recur with `sudo`.
- Windows PowerShell 5.1 / OpenSSH for Windows 9.5p2. The initial SSH attempt failed because default key filenames did not match the existing custom key name. Explicitly specifying the existing `home_dns_ed25519` private key allowed authenticated SSH access. The key contents are not published.

### DNS checks

| Source and query | Observed result | Evidence boundary |
| --- | --- | --- |
| Raspberry Pi: `dig @127.0.0.1 example.com A` at 12:54:25 PST | `NOERROR`; two A records; `EDE: 3 (Stale Answer)`; TTL 0; 3 ms | Demonstrates a stale response served by the local resolver; does not identify upstream cause |
| Raspberry Pi: same query at 12:55:43 PST | `NOERROR`; same two A records; TTL 118; no stale-answer EDE; 0 ms | Consistent with cache refresh; upstream transaction not captured |
| Raspberry Pi: `nslookup example.com 192.168.0.116` | Two IPv4 and two IPv6 answers | Confirms resolution via the Pi's LAN IPv4 address **from the Pi itself**, not from Windows |

### Windows client: A–B–A VPN-state test

With the Windows laptop querying the Pi-hole server explicitly using `nslookup example.com 192.168.0.116`:

| Sequence | Bitdefender VPN state | Observed result |
| --- | --- | --- |
| A1 | Connected | Repeated DNS request timeouts; no `example.com` answer |
| B | Disconnected | Server identified as `pi.hole`; two IPv4 and two IPv6 answers returned |
| A2 | Reconnected | Repeated DNS request timeouts; no `example.com` answer |

The successful Windows test proves a client-to-Pi-hole DNS path was available with the VPN disconnected. The repeated failure after reconnecting strengthens the association between VPN state and failure of **direct queries to Pi-hole**. It does not establish whether Bitdefender routing, packet filtering, DNS interception, a firewall interaction, or another mechanism caused the timeouts. The `Server: UnKnown` label in timed-out output is not, by itself, evidence of DNS failure. There was no packet capture, firewall-log correlation, or analysis of default Windows DNS traffic in this follow-up.

**Disposition:** Reproducible, VPN-associated local DNS connectivity issue; mechanism unresolved. No indication of compromise established. Do not change production DNS or VPN security settings solely on this evidence.

**Evidence still needed:** Redacted Windows terminal screenshots for A1/B/A2, a redacted Pi-hole query-log view showing whether the VPN-off client's successful lookup is recorded, and optional versions/service-status screenshots. Do not upload unredacted client identifiers, sensitive domains, private key contents, or certificates containing private keys.
