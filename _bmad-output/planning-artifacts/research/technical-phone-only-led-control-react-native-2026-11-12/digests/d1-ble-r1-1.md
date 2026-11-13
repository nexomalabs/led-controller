# D1 BLE — round 1 digest

## Findings

### Q1 — Cheap BLE controller families and their protocols

- claim: ELK-BLEDOM controllers use GATT service 0000fff0-0000-1000-8000-00805f9b34fb and write all commands to characteristic 0000fff3-0000-1000-8000-00805f9b34fb. They also advertise the HID service 0x1812, which does not answer characteristic requests.
  source: https://raw.githubusercontent.com/FergusInLondon/ELK-BLEDOM/master/PROTCOL.md
  publisher: FergusInLondon (GitHub)
  pub_date: unknown
  accessed: 2026-11-12
  confidence: medium
  class: protocol

- claim: ELK-BLEDOM commands are fixed 9-byte frames that start with 0x7E and end with 0xEF. RGB is `7E 00 05 03 R G B 00 EF`, brightness is `7E 00 01 <brightness> <mode> 00 00 00 EF`, and colour temperature is `7E 00 05 02 <warm> <cold> 00 00 EF` (warm + cold = 100). The author warns that the decompiled app code differs from the tables in byte 2 and in the brightness bytes 5 to 7.
  source: https://raw.githubusercontent.com/FergusInLondon/ELK-BLEDOM/master/PROTCOL.md
  publisher: FergusInLondon (GitHub)
  pub_date: unknown
  accessed: 2026-11-12
  confidence: medium
  class: protocol

- claim: The ELK-BLEDOM protocol was reverse-engineered from the vendor's Android app. It is not officially documented, and the writeup does not cover power on/off, effect modes or speed.
  source: https://github.com/FergusInLondon/ELK-BLEDOM
  publisher: FergusInLondon (GitHub)
  pub_date: unknown
  accessed: 2026-11-12
  confidence: high
  class: protocol

- claim: The maintained Home Assistant custom integration for the ELK family (dave-code-ruiz/elkbledom) handles several variants, each with its own quirks. ELK-BLEDOM is the default. ELK-BLEDOB uses write-without-response, needs a 0.1 s settle delay and has no state notifications. ELK-BLEDDM needs a 0.3 s delay between commands. MELK/MODELX need an unauthenticated "login" sequence (`7e 07 83`, wait 1 s, `7e 04 04`, wait 1 s) before service discovery, and LEDBLE skips notifications.
  source: https://raw.githubusercontent.com/dave-code-ruiz/elkbledom/main/custom_components/elkbledom/elkbledom.py
  publisher: dave-code-ruiz (GitHub)
  pub_date: unknown (fetched from main branch)
  accessed: 2026-11-12
  confidence: high
  class: protocol

- claim: ELK-family commands are plaintext writes. The elkbledom code has no encryption, key exchange or cryptographic authentication, only the fixed MELK/MODELX login bytes. Its 9-byte state notification reports power in byte 3 (0x23/0xF0/0x01 = on, 0x24/0x00 = off), RGB in bytes 4–6 and brightness % in byte 7.
  source: https://raw.githubusercontent.com/dave-code-ruiz/elkbledom/main/custom_components/elkbledom/elkbledom.py
  publisher: dave-code-ruiz (GitHub)
  pub_date: unknown
  accessed: 2026-11-12
  confidence: high
  class: protocol

- claim: elkbledom also hardcodes 9-byte 0x7E…0xEF frames for mic effects (`7e 05 03 <0x80–0x87> 04 ff ff 00 ef`), mic sensitivity (`7e 04 06 <0–100> ff ff ff 00 ef`) and an on-device scheduler (`7e 00 82 ...`). Power, colour, brightness and effect builders live in a separate Model class that I did not read.
  source: https://raw.githubusercontent.com/dave-code-ruiz/elkbledom/main/custom_components/elkbledom/elkbledom.py
  publisher: dave-code-ruiz (GitHub)
  pub_date: unknown
  accessed: 2026-11-12
  confidence: high
  class: protocol

- claim: Triones / HappyLighting devices take writes on service FFD5, characteristic FFD9. Power on is `CC 23 33`, power off is `CC 24 33`, RGB is `56 RR GG BB 00 F0 AA`, and warm white is `56 00 00 00 <ww> 0F AA`. The protocol was reverse-engineered by madhead (the saberlight project's protocol.md).
  source: https://community.home-assistant.io/t/custom-component-integration-for-triones-happylighting-based-ble-lights/356011 (also https://github.com/Aritzherrero4/python-trionesControl, https://flowcloud.github.io/ci20-bluetooth-LE/2015/09/09/reverse-engineering-the-control-protocol)
  publisher: Home Assistant Community / Aritzherrero4 / flowcloud (search snippets only; saberlight protocol.md not opened)
  pub_date: unknown (flowcloud 2015-09)
  accessed: 2026-11-12
  confidence: medium
  class: protocol

- claim: The Home Assistant core `led_ble` integration (codeowner @bdraco) pins led-ble==1.1.11 and supports one family of generic unbranded controllers and bulbs ("Triones", "LEDBLE", "LEDBlue", "QHM", "Dream~"). It explicitly excludes Govee and devices that use other protocols such as yeelight-bt or ALLOYSEED.
  source: https://github.com/Bluetooth-Devices/led-ble
  publisher: Bluetooth-Devices org (GitHub)
  pub_date: unknown (repo page 2026)
  accessed: 2026-11-12
  confidence: high
  class: landscape

- claim: Govee BLE light control is reverse-engineered per model, not documented. Community repos cover H6001, H6199, H7015, H6159, H613E, H617A and others. Contributors report that command sets differ slightly between models, and a 2025 HA forum post says no existing Govee BLE integration worked for the H617A.
  source: https://community.home-assistant.io/t/govee-h617a-ble-led-strip-lights-reverse-engineering/901353 (plus search snippets for egold555/Govee-Reverse-Engineering, Obi2000 H6199, ConsciousCode govee_h7015)
  publisher: Home Assistant Community / various GitHub authors (snippets)
  pub_date: unknown (thread id suggests 2025)
  accessed: 2026-11-12
  confidence: medium
  class: landscape

- claim: Govee BLE packets (documented for the H6001) are 20 bytes long. They start with 0x33, then a type byte (0x01 power, 0x04 brightness, 0x05 colour) and a value, and end with an XOR checksum of all earlier bytes. Power on is `33 01 01 …`, brightness is `33 04 <uint8>`, and manual colour is `33 05 02 RRGGBB 00 …`. White tones are app-hardcoded codes that cannot be set to arbitrary values. No encryption is described. STALE: the repo is old, has 2 commits and covers one model only.
  source: https://github.com/chvolkmann/govee_btled
  publisher: chvolkmann (GitHub)
  pub_date: unknown (stale; Python 3.7-era)
  accessed: 2026-11-12
  confidence: medium (single old source; model-specific)
  class: protocol

### Q2 — Identifying which protocol a discovered device speaks

- claim: Home Assistant's `led_ble` integration recognises its devices by BLE advertised local name alone. Its manifest has no service_uuid or manufacturer_id matchers, only these name patterns: `LEDnet*`, `BLE-LED*`, `LEDBLE*`, `Triones*`, `LEDBlue*`, `Dream~*`, `QHM-*`, `AP-*`, `LD-0003`.
  source: https://raw.githubusercontent.com/home-assistant/core/dev/homeassistant/components/led_ble/manifest.json
  publisher: Home Assistant (home-assistant/core)
  pub_date: unknown (dev branch, fetched 2026-11-12)
  accessed: 2026-11-12
  confidence: high
  class: pattern

- claim: The ELK-family integration also picks the protocol variant from name prefixes (ELK-BLEDOM default, ELK-BLEDOB, ELK-BLEDDM, `melk`, `modelx`, `ledble`). The "LEDBLE" prefix is claimed both by elkbledom and by core `led_ble` (`LEDBLE*`), so a name prefix alone can be ambiguous.
  source: https://raw.githubusercontent.com/dave-code-ruiz/elkbledom/main/custom_components/elkbledom/elkbledom.py ; https://raw.githubusercontent.com/home-assistant/core/dev/homeassistant/components/led_ble/manifest.json
  publisher: dave-code-ruiz; Home Assistant
  pub_date: unknown
  accessed: 2026-11-12
  confidence: medium (overlap is an inference from two sources; the two may target different hardware under the same name)
  class: pattern

- claim: Users report that a Triones device is not auto-discovered by HA unless its advertised name starts with "Triones-", which can be renamed in the vendor app. Name-based detection is fragile.
  source: https://community.home-assistant.io/t/custom-component-integration-for-triones-happylighting-based-ble-lights/356011 (snippet)
  publisher: Home Assistant Community
  pub_date: unknown
  accessed: 2026-11-12
  confidence: low (snippet)
  class: pattern

### Q3 — React Native BLE libraries

- claim: react-native-ble-manager is at 13.0.0, published 2026-11-05. It had 12 releases from 2025-11-10 to 2026-11-05 (12.4.0, 12.4.1, 12.4.2, 12.4.3, 12.4.4, 12.4.5, 12.5.0, 12.5.1, 12.5.2, 12.5.3, 12.6.0, 12.6.1, 13.0.0) plus an 11.7.1 backport on 2026-03-19, so roughly one release a month.
  source: https://registry.npmjs.org/react-native-ble-manager (time field, queried 2026-11-12); https://github.com/innoveit/react-native-ble-manager/releases
  publisher: npm registry; innoveit (GitHub)
  pub_date: 2026-11-05
  accessed: 2026-11-12
  confidence: high (two sources agree)
  class: version-compatibility

- claim: The react-native-ble-manager 13.0.0 package.json declares a `codegenConfig` (name BleManagerSpec, type modules), so it ships a TurboModule spec for the New Architecture. Its peer dependencies are react:* and react-native:*.
  source: https://registry.npmjs.org/react-native-ble-manager (latest version manifest)
  publisher: npm registry
  pub_date: 2026-11-05
  accessed: 2026-11-12
  confidence: high
  class: version-compatibility

- claim: react-native-ble-manager 13.0.0 has a breaking change: Android `advertising.manufacturerRawData` now uses the same byte layout as iOS. Android parsers must change company-ID decoding and the payload offset from 4 bytes to 2. This matters for protocol detection that relies on manufacturer data.
  source: https://github.com/innoveit/react-native-ble-manager/releases
  publisher: innoveit (GitHub)
  pub_date: 2026-11-05
  accessed: 2026-11-12
  confidence: high
  class: version-compatibility

- claim: react-native-ble-plx (dotintent) is at 3.5.1, published 2026-02-18. Before that, 3.5.0 came out on 2025-02-11, so it shipped only one release in the last 12 months and about one a year since 2024.
  source: https://registry.npmjs.org/react-native-ble-plx (time field); https://github.com/dotintent/react-native-ble-plx/releases
  publisher: npm registry; dotintent (GitHub)
  pub_date: 2026-02-18
  accessed: 2026-11-12
  confidence: high (two sources agree)
  class: ecosystem-health

- claim: The react-native-ble-plx 3.5.1 package.json has no `codegenConfig`. None of its release notes mention the New Architecture, TurboModules or Fabric. Expo support came as a config plugin in 3.1.0 (fixed in 3.1.1), and 3.4.0 dropped `expo` from dependencies and made `BleManager` a singleton.
  source: https://registry.npmjs.org/react-native-ble-plx ; https://github.com/dotintent/react-native-ble-plx/releases
  publisher: npm registry; dotintent (GitHub)
  pub_date: 2026-02-18
  accessed: 2026-11-12
  confidence: high
  class: version-compatibility

- claim: Upstream PR #1331 on react-native-ble-plx proposes a ground-up New Architecture rewrite. Its author says v3 has about 25 unpatchable known problems and "no path" to the New Architecture. The proposed floor is RN 0.82+ (New Arch only), iOS 17+ and Android API 23+. Merge status is unknown.
  source: https://github.com/dotintent/react-native-ble-plx/pull/1331 (seen only via mirror-domain snippets)
  publisher: dotintent repo contributor
  pub_date: unknown (2026 likely)
  accessed: 2026-11-12
  confidence: low (snippet via mirrors; not opened)
  class: ecosystem-health

- claim: A community fork, @sfourdrinier/react-native-ble-plx v3.9.2, presents itself as a TypeScript-first TurboModule targeting Expo SDK 57+ / RN 0.86+. It calls upstream "effectively unmaintained for current React Native and Expo stacks." It needs a dev build and does not work in Expo Go.
  source: https://cdn.jsdelivr.net/npm/@sfourdrinier/react-native-ble-plx@3.9.2/docs/FORK.md (snippet)
  publisher: sfourdrinier (npm/jsDelivr)
  pub_date: unknown (2026)
  accessed: 2026-11-12
  confidence: low (snippet; advocacy by fork author)
  class: ecosystem-health

### Q4 — Platform limits

- claim: On Android 12 (API 31) and later, BLE scanning needs the runtime permission BLUETOOTH_SCAN, shown to the user as "Nearby devices". An app can skip location permission by declaring `android:usesPermissionFlags="neverForLocation"`, but that filters some BLE beacons out of scan results. Apps on Android 11 and lower still need location permission to scan, and the guidance is to declare ACCESS_FINE_LOCATION with maxSdkVersion=30.
  source: https://developer.android.com/develop/connectivity/bluetooth/bt-permissions
  publisher: Google / Android Developers
  pub_date: unknown (live doc)
  accessed: 2026-11-12
  confidence: high (official doc via search snippet; not opened in depth)
  class: platform-constraint

- claim: Neither Apple nor Google publishes a maximum number of simultaneous BLE connections. Developer reports suggest roughly 8–10 on iOS (a 2013 Apple bluetooth-dev report could not connect an 11th device; iOS 9 users connected 8). On Android, connections become unreliable at around 5 or more in parallel and a "7" figure is often cited, varying by phone. STALE: sources are from 2012–2021.
  source: https://lists.apple.com/archives/bluetooth-dev/2013/Jul/msg00026.html ; https://devzone.nordicsemi.com/f/nordic-q-a/77268/android-ble-issues-on-parallel-5-connection-establishments/319483
  publisher: Apple bluetooth-dev mailing list; Nordic DevZone
  pub_date: 2013-07; ~2021
  accessed: 2026-11-12
  confidence: low (stale, anecdotal; two sources but neither current)
  class: performance

### Q5 — Bluetooth standard for lighting

- claim: Bluetooth Mesh has standard lighting models (Lightness, CTL, HSL, plus NLC profiles). Phones do not natively speak Mesh over the advertising bearer, so a phone app must connect over GATT to a Mesh node that has the optional Proxy feature, and implement provisioning and proxy logic itself. Vendor SDKs provide this for Android/iOS (Silicon Labs, ST BlueNRG-Mesh, Nordic).
  source: https://www.silabs.com/software-and-tools/bluetooth-mesh-mobile-app ; https://www.st.com/resource/en/user_manual/dm00369109-getting-started-with-the-st-bluenrgmesh-android-application-stmicroelectronics.pdf ; https://www.nordicsemi.com/Products/Bluetooth-mesh/What-is-Bluetooth-mesh
  publisher: Silicon Labs; STMicroelectronics; Nordic Semiconductor
  pub_date: unknown
  accessed: 2026-11-12
  confidence: medium (vendor docs via snippets)
  class: platform-constraint

## Leads worth chasing

- **Contradiction, ble-plx New Architecture:** a 2026-08-09 tutorial (reactnativerelay.com) says ble-plx v3.x supports the New Architecture (Fabric/TurboModules), but PR #1331's author says v3 has no New Arch path, and the 3.5.1 package has no codegenConfig. v3 may run through RN's interop layer. Check open issues about "new architecture", "bridgeless" and "RN 0.8x" on dotintent/react-native-ble-plx, and whether PR #1331 merged or shipped as v4.
- **react-native-ble-manager Expo support:** release notes say nothing about an Expo config plugin. Check the repo for app.plugin.js and the README's Expo section.
- **Missing ELK-BLEDOM power/effect bytes:** they live in elkbledom's Model class (another file in custom_components/elkbledom/). Unverified belief, not evidenced this run: on = `7e 00 04 f0 00 01 ff 00 ef`, off = `7e 00 04 00 00 00 ff 00 ef`, effect = `7e 00 03 <id> 03 00 00 00 ef`.
- **Variant drift within ELK:** ELK-BLEDOB and ELK-BLEDDM behave differently (write-without-response, settle delays, no notify). The adapter layer needs per-variant timing and an optimistic state model.
- **LEDBLE name collision** between core led_ble and elkbledom: protocol detection probably needs a GATT service probe after connecting (FFF0 → ELK, FFD5/FFD0 → Triones family), not just the name. Verify which service UUIDs led-ble's devices expose.
- **Govee:** newer Govee models (H6xxx/H61xx) may use different or extra BLE framing, and some community reports mention a keep-alive. Check egold555/Govee-Reverse-Engineering, wez/govee2mqtt docs, and beshelmek/govee_ble_lights for current model coverage and any encryption on 2024+ models. Govee also has an official LAN (UDP) API for Wi-Fi models, which belongs to the Wi-Fi dimension.
- **8none1** (author of pytrionesmqtt / nimble_triones) also appears to maintain BJ_LED and LEDnetWF/Zengge BLE integrations. This is a likely source for the BJ_LED/MohuanLED and Magic Home/Zengge BLE protocols.
- **OpenRGB issue #2354** ("Triones Bluetooth lights") may be a second protocol-catalog source.
- **@sfourdrinier/react-native-ble-plx fork:** check download counts and maintenance over time before considering it.

## Looked for, not found

- An official protocol document from any of these vendors (ELK/duoCo Strip/Lotus Lantern, Triones/HappyLighting, Govee BLE). Everything found is reverse-engineered.
- Power on/off and effect byte formats for ELK-BLEDOM in the sources I read (the PROTCOL.md table leaves them unimplemented).
- Magic Home / Zengge BLE ("LEDnetWF") and BJ_LED / MohuanLED protocol details. I ran out of budget before searching; the only lead is the `LEDnet*` name in HA's led_ble manifest.
- Any source showing that cheap Amazon/AliExpress RGB strips use Bluetooth SIG Mesh lighting models. No evidence found; treat it as unlikely but unverified.
- An official or current (≤2 years) source for the maximum simultaneous BLE connections on iOS or Android. Only stale anecdotal reports found.
- iOS CoreBluetooth constraints (no MAC exposure, per-app peripheral UUIDs, background scanning needing service UUID filters, the NSBluetoothAlwaysUsageDescription prompt). Not researched this round for budget reasons, so these remain unverified beliefs that need Apple-doc evidence.
- Typical BLE write latency or throughput figures for LED controllers. Not found.
- A single open-source multi-protocol catalog covering ELK + Triones + Govee + Zengge together. HA core led_ble covers only the Triones/LEDnet family and elkbledom covers only ELK. No unified catalog was found this round.
