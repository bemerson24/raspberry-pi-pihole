# DNS Investigation — Evidence Collection Worksheet

**Project:** Raspberry Pi 400 / Pi-hole home DNS filtering
**Purpose:** Collect repeatable, privacy-safe observations to support a troubleshooting case study.
**Status:** Worksheet only. Blank fields are **not** measurements or completed tests.

## Scope

Use this worksheet when an unexpected client or query pattern appears in Pi-hole. A high query count alone does not establish compromise, successful connections, or even abnormal device behavior.

## Pre-test record

| Field | Value |
|---|---|
| Date, time, and timezone | To record |
| Pi-hole version | To record |
| Raspberry Pi OS / release | To record |
| Test device (generic description) | To record |
| Device power state | To record |
| DNS configuration and upstream resolver | To verify |
| Observation window length | To record |
| Query filter used | To record |

Keep raw client identifiers, internal addresses, and sensitive domains **out of the public repository**.

## Suggested controlled observation

1. **Baseline:** While the suspected device is plugged in, select a defined time window in Pi-hole and record its query behavior.
2. **Isolation:** Unplug that device and record the start/end times. Check whether queries attributed to that client stop during the interval.
3. **Reconnect:** Reconnect the device and record whether the same client begins generating queries again.
4. **Repeat if needed:** Use matched observation windows, the same filters, and time-synchronized timestamps.
5. **Check alternative explanations:** Consider retries, blocked requests, background updates, application telemetry, and misleading client attribution.
6. **Document limitations:** Pi-hole DNS logs alone are not packet captures and do not prove the reason behind each lookup.

## Measurement log — fill only after testing

| Interval | Start–end (local time) | Device state | Queries observed | Blocked queries | Notes |
|---|---|---|---:|---:|---|
| A: baseline | Not recorded | Plugged in | Not recorded | Not recorded | |
| B: isolated | Not recorded | Unplugged | Not recorded | Not recorded | |
| C: reconnected | Not recorded | Plugged in | Not recorded | Not recorded | |

Avoid computing query rates unless the observation durations and counts were actually recorded.

## Safe evidence checklist

- [ ] Pi-hole dashboard screenshot showing **only the relevant time window**, with personal domains, addresses, and client names redacted
- [ ] Versions of Pi-hole and Raspberry Pi OS
- [ ] Test start/end timestamps and local timezone
- [ ] Notes describing how the device was identified
- [ ] Record of any disabled VPN, alternate DNS, or browser secure-DNS setting that could affect attribution
- [ ] Distinction between **observed**, **inferred**, and **still unknown** findings

## Outcome

Write a brief conclusion supported by the recorded measurements. Do not describe the behavior as malicious or as an incident unless additional evidence establishes that.

**Related case study:** [Smart TV DNS investigation — October 15, 2026](../case-studies/smart-tv-dns-investigation-2026-10-15.md).
