# Case Study: Investigating Repeated Smart TV DNS Requests

**Initial investigation:** October 15, 2026  
**Follow-up observation:** October 22–23, 2026  
**Environment:** Raspberry Pi 400 running Pi-hole on a home network  
**Status:** The October 15 unplug/reconnect test supported attribution to a smart TV. The later router-isolation test records reachability and aggregate DNS observations, but does not independently identify the target model or establish why the service was queried.

## Question

An unexpectedly active client appeared in Pi-hole's DNS-query telemetry. The purpose of this exercise was to identify the source device and test whether its network activity corresponded to a particular household device, before making assumptions about malicious behavior.

## Initial investigation — October 15

1. Reviewed the Pi-hole client and query views to identify the device generating repeated requests.
2. Correlated a local client IP address and MAC-related device information with an Insignia smart TV. Identifiers have been omitted from this public case study.
3. Unplugged the suspected TV to test whether its power state correlated with the observed traffic.
4. Observed that the requests stopped while the TV was unplugged and that the client disappeared from the current-client view.
5. After the TV was reconnected and used normally, observed that DNS requests resumed.

## Follow-up — router-level isolation, October 22–23

### Evidence captured

- **Pi-hole query log:** The target client generated lookups for a long, identifier-like hostname under an Amazon-associated Minerva service domain. The log showed multiple A (IPv4) and AAAA (IPv6) queries at the displayed time 21:30:55 on October 22, plus another query at 21:22:54. The device-specific hostname and local network identifiers are intentionally omitted from this public report.
- **Router/client inventory:** The client table displayed a vendor attribution of Shenzhen Cultraview Digital Technology Co., Ltd. for the target's hardware address. A vendor attribution can identify a registered MAC-address prefix or network-interface supplier; it does not establish the consumer product brand or exact device model.
- **Router access control and ping:** A router-level block was applied to the target client. A subsequent Windows ping received no successful echo reply: the output showed one local “Destination host unreachable” message from the testing laptop and three request timeouts (75% packet loss as reported by Windows). The local error is not a response from the target device.
- **Pi-hole dashboard:** The supplied 24-hour-style graphs showed network-wide query activity falling after midnight and increasing again in the early morning. These graphs aggregate multiple clients and do not isolate the target client's contribution.
- **Timing limitation:** The exact time the router block was applied and a matched per-client before/after query count were not captured in the supplied evidence.

### Interpretation

The ping output is consistent with the target being unreachable from the laptop during the test. It does not by itself distinguish a router access-control effect from a powered-off device, a disconnected client, or another local network condition.

The Minerva-related DNS lookups indicate that Pi-hole observed DNS requests for that service hostname. DNS logs do not establish that an application connection succeeded, what data was transmitted, why the hostname was requested, or whether the activity was malicious. A lack of later Pi-hole queries would also not prove that the device stopped attempting communication: a router block may prevent DNS requests from reaching Pi-hole.

The current evidence does not independently establish that the router-isolation target is the same physical device as the Insignia TV from the October 15 test. A private comparison of the target's MAC address against the TV's network settings would help resolve that question.

No service-impact observations, packet capture, or application-layer traffic analysis have been recorded for this follow-up.

## Reproducible validation approach

- Note the client and query activity within a defined time window.
- Record the suspected device's local identifiers privately; do not publish household IP/MAC information.
- Power the suspected device off or unplug it, or apply a clearly time-stamped router isolation rule.
- Compare query activity during the disconnected interval, using the same client filter and matched time windows.
- Reconnect or restore access only when safe, and observe whether the same client activity returns.
- Repeat with time-stamped screenshots or exported, sanitized metrics if a stronger evidentiary record is needed.
- Distinguish DNS lookups, block decisions, ping reachability, and successful application-layer connections; they are different observations.

## Skills practiced

DNS troubleshooting · Pi-hole monitoring · Client identification · Hypothesis testing · Controlled isolation · Router access controls · Technical documentation · Distinguishing observation from inference

## Limitations / next steps

- The target device's exact make and model remain unconfirmed for the router-isolation follow-up.
- The exact time the router block was applied was not recorded.
- No matched, target-specific query counts before and after the block have been recorded.
- No packet-level analysis was performed.
- The specific service's purpose on this device and the reasons for the requests were not conclusively characterized.
- A later follow-up could compare filtered queries by client and timestamp, verify the device privately against its MAC address, and test whether a particular feature depends on a blocked service without weakening network-wide filtering unnecessarily.

**Takeaway:** Controlled unplug/reconnect and router-isolation tests can help narrow down the source of network activity. Each result has limits: DNS logs show lookups, ping tests show reachability at a moment in time, and aggregate graphs do not prove what a single client did. The evidence supports further identification and validation, not a claim of compromise or data exfiltration.
