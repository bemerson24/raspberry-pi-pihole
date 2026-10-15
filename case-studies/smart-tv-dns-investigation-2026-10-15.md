# Case Study: Investigating Repeated Smart TV DNS Requests

**Date:** October 15, 2026  
**Environment:** Raspberry Pi 400 running Pi-hole on a home network  
**Status:** Client identification verified; root cause and necessity of individual requests not established

## Question

An unexpectedly active client appeared in Pi-hole's DNS-query telemetry. The purpose of this exercise was to identify the source device and test whether its network activity corresponded to a particular household device, before making assumptions about malicious behavior.

## Investigation

1. Reviewed the Pi-hole client and query views to identify the device generating repeated requests.
2. Correlated a local client IP address and MAC-related device information with an Insignia smart TV. Identifiers have been omitted from this public case study.
3. Unplugged the suspected TV to test whether its power state correlated with the observed traffic.
4. Observed that the requests stopped while the TV was unplugged and that the client disappeared from the current-client view.
5. After the TV was reconnected and used normally, observed that DNS requests resumed.

## Observations and interpretation

The unplug/reconnect experiment supports attribution of the observed DNS activity to the smart TV. It does **not** prove that the requests were malicious, that the device was compromised, or that every query was necessary for normal TV operation. A repeated DNS request may have multiple explanations, including application behavior, advertising/telemetry services, retry behavior, and a response blocked by a DNS filtering rule.

Pi-hole exposes DNS lookups and block decisions; those alone are not a packet capture or a full view of application-layer traffic. A blocked lookup should not be described as proof that a connection to the queried service succeeded.

## Reproducible validation approach

- Note the client and query activity within a defined time window.
- Record the suspected device's local identifiers privately; do not publish household IP/MAC information.
- Power the suspected device off or unplug it.
- Compare query activity during the disconnected interval.
- Reconnect the device and observe whether the same client activity returns.
- Repeat with time-stamped screenshots or exported, sanitized metrics if a stronger evidentiary record is needed.

## Skills practiced

DNS troubleshooting · Pi-hole monitoring · Client identification · Hypothesis testing · Controlled isolation · Technical documentation · Distinguishing observation from inference

## Limitations / next steps

- No sanitized query samples or timestamps have been attached yet.
- No packet-level analysis was performed.
- The specific service domains, retry patterns, and reasons for the requests were not conclusively characterized.
- A later follow-up could compare filtered queries by domain and timestamp and test whether a particular TV feature depends on a blocked service, without weakening network-wide filtering unnecessarily.

**Takeaway:** A simple controlled isolation test transformed an ambiguous monitoring signal into a supported device attribution, without treating high-volume DNS traffic as proof of a security incident.
