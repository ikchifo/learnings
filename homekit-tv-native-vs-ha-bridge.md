# HomeKit TV controls: native HomeKit vs HA bridge

How to get full Apple Home TV controls (Remote widget with
d-pad, volume, play/pause) for a Sony Bravia in a Home
Assistant setup.

---

## Date

- 2026-03-12

## Context

- Device: Sony BRAVIA XR-55X90J (Google TV, Android 12)
- HA integration: `braviatv` (PSK auth)
- Goal: full TV controls in Apple Home, including the Control
  Center Remote widget (d-pad, back, play/pause, volume)

## Problem statement

- What was expected: exposing the TV through HA's HomeKit
  Bridge in accessory mode would provide full TV controls in
  Apple Home, including the Remote widget in Control Center.
- What actually happened: the Home app showed the TV as a
  proper TV icon with power and input switching, but the
  Control Center Remote widget showed "Choose a TV" with no
  TVs listed.

## Core learning

HA's HomeKit Bridge exposes TVs as HAP Television accessories
correctly (category 31, with RemoteKey, TelevisionSpeaker, and
InputSource services), but the **Control Center Remote widget
does not work without a Home Hub** (Apple TV or HomePod). For
TVs with built-in HomeKit (Sony, LG, Samsung), using the TV's
**native HomeKit** is far simpler and provides the Remote
widget without requiring a Home Hub. The HA integration
(`braviatv`) continues to work independently for automations
and dashboard controls.

## Investigation steps

### 1. Verified HA HomeKit Bridge config

The TV was being exposed **three times**:

```
HASS Bridge:21064 (bridge)    -- media_player domain included,
                                 only Cast entity excluded
Living Room TV:21066 (accessory) -- dedicated, unpaired
HASS Bridge ZX:21065 (accessory) -- dedicated, paired
```

Fix: excluded the Bravia entity from the main bridge, deleted
the duplicate unpaired entry.

### 2. Verified HAP service structure

Checked the IID file and confirmed correct services:

| Service type | HAP code | Present |
|-------------|----------|---------|
| AccessoryInformation | 3E | Yes |
| Television | D8 | Yes |
| InputSource (x5) | D9 | Yes |
| TelevisionSpeaker | 113 | Yes |

TelevisionSpeaker had `VolumeControlType: 1` (RELATIVE) with
VolumeSelector and Mute characteristics.

### 3. Verified mDNS advertisement

```python
# Using zeroconf to check TXT records
Name: Living Room TV 9504C5._hap._tcp.local.
  Category (ci): 31     # Television -- correct
  Status Flags (sf): 0  # Paired
  Feature Flags (ff): 0
```

Category 31 is correct for Television. The accessory was
properly advertised and paired.

### 4. Identified root cause: no Home Hub

The Control Center Remote widget requires a Home Hub (Apple TV
or HomePod) to communicate with third-party HomeKit Television
accessories. Without one, only the Home app tile works (power +
inputs), but the Remote widget cannot discover the TV.

### 5. Switched to native HomeKit

Enabled the TV's built-in HomeKit (Settings > Apps > AirPlay &
HomeKit). Initial pairing failed with "unable to add
accessory" because HA had a `homekit_controller` ignore entry
auto-discovered for the TV. Deleting that entry and
resetting/re-enabling HomeKit on the TV resolved the pairing.

## Fix summary

| Step | Action |
|------|--------|
| 1 | Delete HA HomeKit TV accessory entry |
| 2 | Delete `homekit_controller` ignore entry for the TV |
| 3 | Exclude `media_player.living_room_tv_2` from HA bridge |
| 4 | Enable native HomeKit on the TV |
| 5 | Pair TV directly to Apple Home |

## Key takeaway: when to use native HomeKit vs HA bridge

| Scenario | Recommendation |
|----------|---------------|
| TV has built-in HomeKit | Use native HomeKit for Apple Home; keep HA integration for automations |
| TV lacks native HomeKit | Use HA bridge in accessory mode (needs Home Hub for Remote) |
| No Home Hub available | Native HomeKit is the only way to get the Remote widget |
| Home Hub available | Either approach works; native is simpler |

## Avoiding duplicates

When using native HomeKit alongside the HA bridge:

1. Exclude the TV entity from the HA HomeKit Bridge
2. Delete any `homekit_controller` ignore entries HA
   auto-discovers for the TV
3. Disable the TV's native HomeKit only if using HA's bridge
   exclusively

## Debugging HomeKit TV issues checklist

1. Check how many HomeKit entries expose the TV
   (`/api/config/config_entries/entry`, filter `domain:
   homekit`)
2. Check pairing state in `.storage/homekit.<entry_id>.state`
   (`paired_clients` field)
3. Verify mDNS advertisement with `zeroconf` (category `ci`
   should be 31 for Television)
4. Check IID file for correct services (D8=Television,
   D9=InputSource, 113=TelevisionSpeaker)
5. Check for `homekit_controller` entries that might block
   native pairing

## References

### Official docs

- https://www.home-assistant.io/integrations/homekit/
- https://github.com/HomeSpan/HomeSpan/blob/master/docs/TVServices.md

### HAP Television service codes

| Code | Service |
|------|---------|
| 3E | AccessoryInformation |
| D8 | Television |
| D9 | InputSource |
| 113 | TelevisionSpeaker |

### HAP RemoteKey values

| Code | Key | In Remote widget |
|------|-----|-----------------|
| 4-7 | Arrow keys | Yes |
| 8 | Select | Yes |
| 9 | Back | Yes |
| 11 | Play/Pause | Yes |
| 15 | Information | Yes |
| 0-3 | Rewind/FF/Track | No |
