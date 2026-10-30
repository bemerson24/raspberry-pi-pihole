# Case Study: Laptop DNS Portability and VPN Connection Failure

**Date:** October 29, 2026  
**Environment:** HP EliteBook 840 G8 running Windows; Bitdefender VPN; home network with Pi-hole on a Raspberry Pi 400; college Wi-Fi  
**Status:** VPN recovered. Suspected DNS configuration problem still requires controlled testing on campus.

## What happened

During an in-person Python class, my laptop could join the campus Wi-Fi but some websites would not load. The browser reported DNS-related failures. This interrupted hands-on exercises even though the Python material itself was not the issue.

Bitdefender VPN also became unreliable. Disconnecting and reconnecting could take several minutes. At times a connection appeared to work briefly before access failed again.

My home network uses Pi-hole for DNS filtering, so I wanted to check whether a laptop configuration that works at home was interfering with access on another network. A later check of the Windows Wi-Fi adapter showed that IPv4 DNS was **manually configured** rather than set to obtain DNS server addresses automatically. The actual configured DNS addresses were not recorded for this case study, so I have **not confirmed** that the laptop was pointed directly at the home Pi-hole.

## Troubleshooting and recovery

1. While troubleshooting under time pressure during class, I disabled the TAP-Windows Adapter V9 associated with the VPN.
2. At home, Windows Network Connections showed the TAP adapter disabled; I re-enabled it. Wi-Fi was connected to the home network.
3. Bitdefender VPN then displayed **error code 9** when attempting to connect.
4. I restarted Windows after re-enabling the adapter.
5. The VPN worked again after the restart.

The VPN recovery is a recorded outcome. I cannot isolate whether re-enabling the adapter, restarting Windows, or the combination was required. This also did not prove that the original campus browser failures were caused by the VPN.

## Working hypothesis: DNS settings that do not travel

A private DNS resolver on a home LAN generally cannot be reached directly from campus Wi-Fi. If the Windows Wi-Fi adapter is configured with that resolver instead of accepting DNS via the current network, ordinary network connectivity may appear available while hostname lookups fail.

A VPN can also change DNS resolution behavior while its tunnel is active. That could explain some differences between connected and disconnected states, but **the five-minute delays and intermittent VPN drops have not been attributed to DNS**. VPN software, Wi-Fi conditions, firewall policy, captive portals, and other settings remain alternative explanations.

## Next validation steps

These are planned tests, **not completed results**:

- [ ] Record the current Wi-Fi IPv4 and IPv6 DNS configuration privately before making changes. Do not publish actual home IP addresses, device names, or network identifiers.
- [ ] Verify whether the manually entered DNS address is the Pi-hole, a public resolver, or something else.
- [ ] Test automatic DNS configuration on the laptop and confirm whether the home router supplies Pi-hole through DHCP.
- [ ] At home and then on campus, test website access and DNS resolution **with the VPN disconnected first**. Record the network, approximate time, and outcome.
- [ ] Use `ipconfig /all` and `nslookup example.com` to check which resolver is used and whether a query succeeds; sanitize any saved output.
- [ ] If DNS resolution still fails, inspect browser secure DNS / DNS-over-HTTPS, IPv6 settings, campus sign-in requirements, and Bitdefender VPN DNS or kill-switch behavior individually.
- [ ] Once basic network access is stable, test the VPN separately and capture the exact message if the fault returns.

Avoid broad network resets or changes to the Pi-hole, router, VirtualBox, or WSL configuration unless later evidence points there.

## Outcome and lesson

The immediate VPN failure was recovered without reinstalling the software or resetting the Windows network stack. The suspected **home-versus-campus DNS portability problem remains open**.

This incident highlights the difference between joining Wi-Fi, resolving names, accessing websites, and establishing a VPN tunnel. Troubleshooting is more useful when each layer is tested separately and a plausible explanation is clearly labeled as a hypothesis until reproduced.

**Skills:** Windows networking · DNS troubleshooting · VPN troubleshooting · Layered diagnosis · Hypothesis testing · Evidence-based technical documentation
