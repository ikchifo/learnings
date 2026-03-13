# Tailscale on Android TV for DNS-based ad blocking

How to force a Smart TV's DNS through AdGuard Home using
Tailscale, bypassing the TV's hardcoded DNS.

---

## Date

- 2026-03-12

## Context

- Device: Sony BRAVIA XR-55X90J (Google TV, Android 12)
- DNS: AdGuard Home running on HA Pi 5, configured as the
  Tailscale tailnet's DNS via "Override local DNS"
- Problem: Smart TVs hardcode DNS (typically Google 8.8.8.8)
  or use DNS-over-HTTPS, bypassing router-level DNS settings

## Core learning

Smart TVs (especially Google TV / Android TV) ignore
router-assigned DNS and resolve ads/telemetry directly through
Google's DNS. The most reliable way to force ad blocking is to
put the TV on a Tailscale tailnet with "Override local DNS"
enabled, pointing to AdGuard Home. The VPN tunnel prevents the
TV from bypassing the DNS -- all queries must go through
Tailscale's DNS configuration.

## Setup steps

### 1. Install Tailscale on the TV

The Tailscale Android TV app is available on the Google Play
Store. If not listed, sideload via ADB:

```bash
adb connect <TV_IP>:5555
adb install tailscale-android-tv.apk
```

APK source: https://www.apkmirror.com/apk/tailscale-inc/tailscale-android-tv/

### 2. Authenticate

The TV app shows a QR code or alphanumeric code. Enter it at:
Tailscale admin console > Add device > Android > Input code.

SSO-based login is not available on Android TV.

### 3. Set always-on VPN

Tailscale does not auto-reconnect after a full TV reboot
(known upstream issue: https://github.com/tailscale/tailscale/issues/7824).
Fix with ADB:

```bash
adb shell settings put secure always_on_vpn_app com.tailscale.ipn
adb shell settings put secure always_on_vpn_lockdown 0
```

The second command is critical: setting lockdown to 0 ensures
the TV still has internet if Tailscale fails to start. Setting
it to 1 would block all connectivity on Tailscale failure.

Verify:

```bash
adb shell settings get secure always_on_vpn_app
# Output: com.tailscale.ipn
```

### 4. Enable Developer Options on the TV

Settings > System > About > Android TV OS build -- tap 7 times
to unlock Developer Options. Then enable USB debugging under
Settings > System > Developer options.

Disable USB debugging after setup for security.

## What gets blocked

With AdGuard Home as the DNS:

- In-app ads (streaming service recommendations, Sony splash
  ads)
- Google TV telemetry (phones home aggressively)
- Analytics and tracking domains
- Smart TV data collection

## Reboot behavior

| Event | Tailscale status |
|-------|-----------------|
| TV standby (remote off/on) | Stays connected |
| Full restart (Settings > Restart) | Reconnects via always-on VPN |
| Firmware update reboot | Reconnects via always-on VPN |
| Power loss and recovery | Reconnects via always-on VPN |

Note: "reboot" means a full OS restart. Normal remote on/off
is standby mode and does not disconnect Tailscale.

## Tailscale on Android TV capabilities

| Feature | Supported |
|---------|-----------|
| Join tailnet | Yes |
| Use exit node | Yes |
| Run as exit node | Yes (slow, userspace routing) |
| Run as subnet router | Yes (intended for always-on devices) |
| MagicDNS | Yes |
| Taildrop | Yes |
| SSO auth | No (QR/code only) |
| Auto-start on boot | No (use always-on VPN workaround) |

## Why not just set DNS at the router?

Smart TVs bypass router DNS in multiple ways:

1. Hardcoded DNS (8.8.8.8, 8.8.4.4) in the OS
2. DNS-over-HTTPS to Google's resolvers
3. Fallback DNS servers ignoring DHCP-assigned DNS

A VPN tunnel is the only reliable way to force DNS because the
TV cannot route around it -- all traffic goes through the
Tailscale interface.

Alternative (without Tailscale): block outbound DNS
(port 53/443 to known DoH servers) at the firewall and redirect
to AdGuard. This requires router-level firewall rules and
doesn't cover DoH over non-standard ports.

## References

- https://tailscale.com/kb/1079/install-android
- https://github.com/tailscale/tailscale/issues/7824
- https://tailscale.com/blog/android-tv
