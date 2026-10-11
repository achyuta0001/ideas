# Syska Bulb CLI — Tuya WiFi bulb under command-line control

Control a Syska smart bulb (7 W or 9 W; the exact model was never read off the base) from a
terminal: on/off, brightness, colour, no phone app in the loop. It would not pair with the
Smart Life app. Diagnosed on 2026-08-29, then paused before picking a fix route.

## Verified facts (tested 2026-08-29, not assumed)

```text
  Bulb       Tuya WiFi. Not BLE, not Zigbee.
             Wireless module alive and the reset sequence works, so no
             soldering or serial flashing is needed.
             Previously paired to an older network: it boots, hunts for the
             stored credentials, never advertises. Hence steady-on and silent.

  Modes      5 power cycles      EZ mode (slow blink on this unit)
             5 more cycles       AP mode, open 2.4 GHz SSID     UNCONFIRMED

  Absent     no Tuya UDP broadcasts on 6666 / 6667 / 7000
             no port 6668 open on any LAN host
             no open AP across ~22 CoreWLAN scans
             no Tuya BLE advertisement (manufacturer ID 0x07D0)

  Mac        en0 on 5 GHz (channel 157) during testing
```

## Premise corrections

**1. A bulb in EZ mode cannot be found by scanning. That is by design.**
EZ (SmartConfig) mode puts the radio in promiscuous mode to listen for multicast. It transmits
nothing: no AP, no beacon. WiFi, BLE and LAN scans all come back empty, and most of the first
session went into proving that the hard way. On this unit slow blink means EZ; vendors vary,
and Tuya's own docs say slow blink is AP.

**2. The original pairing failure may just be 5 GHz.**
EZ provisioning only works from 2.4 GHz, and the bulb's radio cannot see 5 GHz at all. The Mac
was on 5 GHz. A phone on the 5 GHz half of a merged SSID would fail the same way.

**3. macOS hides WiFi names from Terminal without Location Services.**
Without the grant, CoreWLAN returns `ssid=None` and `bssid=None` for every network, and
`system_profiler SPAirPortDataType` returns cached, partial scans (8 networks one call, 1 the
next). Requesting authorization from a bare Python process does not raise the prompt; it stays
`notDetermined`. Toggle it by hand in System Settings → Privacy & Security → Location Services.
Security mode still reads correctly without the grant, so "is there an open AP" works even
while the names are hidden.

## Routes (none picked yet)

```text
  1  Keep Tuya firmware, provision from the Mac
     tuya-cli link, which needs a free Tuya IoT developer account for the
     pairing token. Then local control with tinytuya.
     The Mac has to be on 2.4 GHz for the EZ broadcast to reach the bulb.

  2  Flash open firmware, zero cloud
     OpenBeken or ESPHome via tuya-cloudcutter.
     Does not run on macOS: needs Linux plus a WiFi adapter that supports
     AP mode. Unknown whether this bulb's firmware is already patched
     against it.

  3  Pair in the app first, then go local
     Smart Life app in AP / Compatible mode, phone on 2.4 GHz. Then
     extract local_key (tinytuya's wizard pulls it through a Tuya IoT
     developer account linked to the app account) and control with tinytuya.
```

Routes 1 and 3 both end with the same Tuya IoT developer account and the same `tinytuya`
control. They differ only in how the bulb gets onto the network.

## Resume here

- Grant Terminal Location Services first, or every scan hides the network names.
- Put whatever does the pairing (phone or Mac) on 2.4 GHz.
- Try the five extra power cycles to reach AP mode. It is unconfirmed, and if it works, route 3
  becomes the easy path.
- The helper scripts from the first session (BLE scan, Tuya UDP listener, CoreWLAN open-AP
  watcher, AP-mode HTTP provisioner, `tinytuya` bulb control) lived in a temporary scratchpad
  and are gone. They are quick to rebuild.

## Open questions

- Exact model, 7 W or 9 W? Read it off the base. It matters for route 2: which chip the bulb
  uses, and whether cloudcutter has a profile for its firmware.
- Is a cloud account acceptable at all? If not, route 2 is the only one that ends with zero
  cloud dependency.
