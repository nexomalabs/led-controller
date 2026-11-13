# D1 BLE — round 2 digest

Scope: round-2 lead-following for D1 (BLE LED strips + React Native BLE stack). Round-1 facts (ELK FFF0/FFF3 9-byte frames, HA led_ble name matching, ble-manager 13.0.0 codegen, ble-plx 3.5.1 no codegen, Android 12+ BLUETOOTH_SCAN neverForLocation) were not re-researched.

Budget note: the tool-call budget ran out before Q4 (Apple CoreBluetooth docs) and the saberlight protocol.md could be fetched. Those items are listed under "Looked for, not found" and are not asserted as findings.

## Findings

### Q1 — react-native-ble-plx vs New Architecture; ble-manager Expo plugin

- claim: dotintent/react-native-ble-plx PR #1331, "V4 turbomodule rewrite" by iotashan, opened 2026-03-16, is still Open and in Draft (not merged). It targets 4.0.0-alpha.0 and states "Requires React Native >= 0.82.0 (New Architecture only)", iOS 17.0+ and Android API 23+.
  source: https://github.com/dotintent/react-native-ble-plx/pull/1331
  publisher: dotintent / iotashan (GitHub)
  pub_date: 2026-03-16
  accessed: 2026-11-12
  confidence: high
  class: version-compatibility

- claim: Breaking changes listed in PR #1331 for v4: `enable()`/`disable()` removed, `setLogLevel()` removed, TS enums become `as const` objects, `BleManager` becomes a true singleton, and `createClient()` must be called explicitly before any API. The PR also says the existing CI matrix "will fail by design" for RN < 0.82.
  source: https://github.com/dotintent/react-native-ble-plx/pull/1331
  publisher: dotintent / iotashan (GitHub)
  pub_date: 2026-03-16
  accessed: 2026-11-12
  confidence: high
  class: version-compatibility

- claim: The PR author says that before the rewrite, "the library had no path to New Architecture support", and that "v3 could continue to support old versions". Read with the next claim, this means 3.x has no native TurboModule implementation. The PR does not discuss whether 3.x runs through the interop layer.
  source: https://github.com/dotintent/react-native-ble-plx/pull/1331
  publisher: dotintent / iotashan (GitHub)
  pub_date: 2026-03-16
  accessed: 2026-11-12
  confidence: high
  class: version-compatibility

- claim: The npm registry shows only a `latest` dist-tag for react-native-ble-plx, pointing to 3.5.1. No `next`/`beta`/`alpha` tag and no 4.x version appeared in the portion read, so the v4 rewrite has not been published to npm. (Only the first ~100k of ~145k characters of the registry document were read.)
  source: https://registry.npmjs.org/react-native-ble-plx
  publisher: npm registry
  pub_date: unknown (live registry document)
  accessed: 2026-11-12
  confidence: medium
  class: version-compatibility

- claim: React Native Directory lists react-native-ble-plx with `"newArchitecture":true` (no newArchitectureNote), `"configPlugin":true`, `"latestRelease":"3.5.1"`, `"pushedAt":"2026-03-09T19:59:48Z"`, 109,142 weekly downloads, 3,445 stars and 33 open issues. Since round 1 found no codegenConfig in 3.5.1, this "true" must mean "works via the New-Arch interop layer", not native TurboModule support.
  source: https://reactnative.directory/api/libraries?search=react-native-ble
  publisher: React Native Directory (Expo / community)
  pub_date: 2026-11 (live API)
  accessed: 2026-11-12
  confidence: medium (the flag is a directory assertion; I did not confirm it from the issue tracker this run)
  class: version-compatibility

- claim: The default branch of ble-plx was last pushed 2026-03-09, so the main branch has seen no commits for about 8 months. The only active New-Arch work is the unmerged draft PR from a contributor's fork. This is a maintenance-risk signal for a v1 app.
  source: https://reactnative.directory/api/libraries?search=react-native-ble ; https://github.com/dotintent/react-native-ble-plx/pull/1331
  publisher: React Native Directory; dotintent (GitHub)
  pub_date: 2026-11 (live) / 2026-03-16
  accessed: 2026-11-12
  confidence: medium
  class: ecosystem-health

- claim: React Native Directory lists react-native-ble-manager with `"newArchitecture":true`, the note "The old architecture is not supported since version 12.x", `"configPlugin":true`, `"latestRelease":"13.0.0"`, `"pushedAt":"2026-11-05T07:45:52Z"`, 38,177 weekly downloads, 2,336 stars and 1 open issue.
  source: https://reactnative.directory/api/libraries?search=react-native-ble
  publisher: React Native Directory
  pub_date: 2026-11 (live API)
  accessed: 2026-11-12
  confidence: high for the flags as listed; medium for the config plugin, which comes only from the directory (single source; I did not inspect app.plugin.js in the package)
  class: version-compatibility

- claim: A community fork, `@sfourdrinier/react-native-ble-plx` (3.9.2 on jsDelivr), is a TypeScript-first TurboModule with a React Native 0.86+ floor. Its readme points users of older RN versions back to upstream dotintent.
  source: https://cdn.jsdelivr.net/npm/@sfourdrinier/react-native-ble-plx@3.9.2/ROADMAP.md (snippet)
  publisher: sfourdrinier (npm fork)
  pub_date: unknown
  accessed: 2026-11-12
  confidence: low (snippet only)
  class: ecosystem-health

**How the contradiction resolved (Q1 summary):** Both libraries are flagged New-Arch-compatible on React Native Directory, but in different ways. react-native-ble-manager 12.x+ is New-Arch-only and native: it ships codegenConfig and dropped the old architecture. react-native-ble-plx 3.5.1 is a legacy bridge module with no codegen. Its "newArchitecture: true" can only come from the interop layer. Its native rewrite (v4, PR #1331, RN ≥ 0.82, iOS 17+) is an unmerged draft that was never published to npm, and main has been quiet since 2026-03-09. I found no first-party issue-tracker evidence this run either confirming or refuting ble-plx 3.5.1 on RN 0.82+/Expo SDK 55 in production. Treat it as "claimed working via interop, not verified". On current evidence, react-native-ble-manager is the lower-risk choice for a New-Arch-only v1.

### Q2 — Zengge/LEDnetWF and BJ_LED/MohuanLED protocols

- claim: LEDnetWF (Zengge) BLE devices expose service `0000ffff-0000-1000-8000-00805f9b34fb`, with write characteristic `0000ff01-…` and notify characteristic `0000ff02-…`.
  source: https://raw.githubusercontent.com/8none1/lednetwf_ble/main/protocol_docs/01_quick_reference.md
  publisher: 8none1 (GitHub, lednetwf_ble)
  pub_date: unknown (repo last pushed ~2026-01-17 per ecosyste.ms snippet)
  accessed: 2026-11-12
  confidence: medium (one primary repo; reverse-engineered)
  class: protocol

- claim: The LEDnetWF integration identifies devices primarily from manufacturer data (AD type 0xFF) with a company-ID range of 0x5A00–0x5AFF (23040–23295). Device names contain "LEDnetWF" or "IOTWF". "IOTBT"-prefixed devices (e.g., IOTBT6BA) also exist and use one of two sub-protocols, "Telink" (hue-based) or "segment" (per-segment addressable), which the integration auto-detects.
  source: https://raw.githubusercontent.com/8none1/lednetwf_ble/main/protocol_docs/01_quick_reference.md ; https://raw.githubusercontent.com/8none1/lednetwf_ble/main/README.md
  publisher: 8none1
  pub_date: unknown
  accessed: 2026-11-12
  confidence: medium
  class: protocol

- claim: LEDnetWF inner commands follow the Magic Home style. Set colour is `[0x31, R, G, B, WW, CW, 0x5A, persist(0xF0|0x0F), chk]`. Power on is `[0x3B, 0x23, …]` on BLE v5+ firmware or `[0x71, 0x23, 0x0F, chk]` on legacy firmware. Power off is `[0x3B, 0x24, …]` or `[0x71, 0x24, 0x0F, chk]`. Brightness is `[0x47, pct 0–100, chk]`. The quick reference does not define how `chk` is computed.
  source: https://raw.githubusercontent.com/8none1/lednetwf_ble/main/protocol_docs/01_quick_reference.md
  publisher: 8none1
  pub_date: unknown
  accessed: 2026-11-12
  confidence: medium
  class: protocol

- claim: Every LEDnetWF command is wrapped in an 8-byte transport header:
  - byte 0: flags (version bits 0–1, message type bits 2–3 JSON/HEX/ACK, ack-expected bit 4, "protection" bit 5, segmentation bit 6)
  - byte 1: sequence number 0–255
  - bytes 2–3: 0x8000 for a single complete segment
  - bytes 4–6: total length (big-endian) and payload length + 1
  - byte 7: cmdId (10 = expects response, 11 = no response)

  Worked example (state query): `00 01 80 00 00 04 05 0A 81 8A 8B 40`.
  source: https://raw.githubusercontent.com/8none1/lednetwf_ble/main/protocol_docs/04_connection_transport.md
  publisher: 8none1
  pub_date: unknown
  accessed: 2026-11-12
  confidence: medium
  class: protocol

- claim: LEDnetWF connection guidance is to request MTU 512 when the device's bleVersion ≥ 8, otherwise 255. Notification payloads may be JSON (first byte `{`/`[`) carrying a hex `payload` field, and the header type bits are unreliable for detecting JSON.
  source: https://raw.githubusercontent.com/8none1/lednetwf_ble/main/protocol_docs/04_connection_transport.md
  publisher: 8none1
  pub_date: unknown
  accessed: 2026-11-12
  confidence: medium
  class: protocol

- claim: Neither LEDnetWF doc read mentions encryption, pairing or bonding. Commands are documented as plain bytes. However, the header defines a "protection" bit whose meaning is not explained, so "unencrypted" is inferred from absence and not confirmed.
  source: https://raw.githubusercontent.com/8none1/lednetwf_ble/main/protocol_docs/04_connection_transport.md
  publisher: 8none1
  pub_date: unknown
  accessed: 2026-11-12
  confidence: low
  class: protocol

- claim: The lednetwf_ble README labels the project WIP and says new Zengge-platform devices "are released regularly". It keeps a `needs_hardware` issue label. Repo signals from a third-party index: 44 stars, 11 forks, last push 2026-01-17.
  source: https://raw.githubusercontent.com/8none1/lednetwf_ble/main/README.md ; https://awesome.ecosyste.ms/projects/github.com%2F8none1%2Flednetwf_ble (snippet)
  publisher: 8none1; ecosyste.ms
  pub_date: 2026-01 (last push)
  accessed: 2026-11-12
  confidence: medium
  class: ecosystem-health

- claim: BJ_LED devices (companion app "MohuanLED", MACs reportedly starting `FF:FF:`) take plaintext commands:
  - power on: `69 96 06 01 01`
  - power off: `69 96 02 01 00`
  - colour: `69 96 05 02 RR GG BB WW` (WW optional because the strips have no white LED)
  - mode: `69 96 03 03 MM SS` (MM 00–15, SS 01 fastest to 0a slowest; values above 0a misbehave)

  There is no brightness command; brightness is done by scaling RGB.
  source: https://raw.githubusercontent.com/8none1/bj_led/main/README.md
  publisher: 8none1 (GitHub, bj_led)
  pub_date: unknown
  accessed: 2026-11-12
  confidence: medium (single reverse-engineered source)
  class: protocol

- claim: The BJ_LED README does not list service or characteristic UUIDs. It notes the controller advertises `notify` but the author never received data on it, so an app cannot read state back from BJ_LED.
  source: https://raw.githubusercontent.com/8none1/bj_led/main/README.md
  publisher: 8none1
  pub_date: unknown
  accessed: 2026-11-12
  confidence: medium
  class: protocol

### Q3 — Protocol detection by GATT characteristics; ELK command bytes

- claim: The HA led-ble library (Bluetooth-Devices/led-ble) defines no service UUIDs. It detects the transport by probing characteristic UUIDs:
  - write candidates, in order: ff01, ffd5, ffd9, ffe9, ffe5 (the comment says ffe5 is "potentially invalid" and is tried last)
  - read/notify candidates: ff02, ffd0, ffd4, ffe0, ffe4
  - all candidates use the base `0000{}-0000-1000-8000-00805f9b34fb`
  - code comments: ff01 is for "0x97 socket" devices and ffd4 is for legacy Triones
  source: https://raw.githubusercontent.com/Bluetooth-Devices/led-ble/main/src/led_ble/const.py
  publisher: Bluetooth-Devices (HA ecosystem org)
  pub_date: unknown (main branch)
  accessed: 2026-11-12
  confidence: high (primary source code)
  class: protocol

- claim: led-ble's state-query command is `EF 01 77` (`STATE_COMMAND = b"\xef\x01\x77"`). Sending it after connecting is the library's way to read device state.
  source: https://raw.githubusercontent.com/Bluetooth-Devices/led-ble/main/src/led_ble/const.py
  publisher: Bluetooth-Devices
  pub_date: unknown
  accessed: 2026-11-12
  confidence: high
  class: protocol

- claim: Detection collision: LEDnetWF's write characteristic ff01 (under service ffff) is also first in led-ble's candidate list (for 0x97 sockets). Probing characteristics alone therefore cannot tell these families apart. A detector should combine service UUID (ffff vs fff0 vs ffd5/ffe5) with manufacturer data (0x5A00–0x5AFF for Zengge) and the name prefix.
  source: https://raw.githubusercontent.com/Bluetooth-Devices/led-ble/main/src/led_ble/const.py ; https://raw.githubusercontent.com/8none1/lednetwf_ble/main/protocol_docs/01_quick_reference.md
  publisher: Bluetooth-Devices; 8none1
  pub_date: unknown
  accessed: 2026-11-12
  confidence: medium (inference from two primary sources)
  class: pattern

- claim: elkbledom's model data moved to `custom_components/elkbledom/models.json`, which `model.py` loads at runtime. It has 24 name-prefix entries. ELK-BTC, ELK-BTCW, ELK-BLEDOB, ELK-BLEDDM, ELK-BLEDOM0E, ELK-BLEDOM, ELK-BLE, MELK-* (OG10/OF10/OC10/OA10/OA21), MELK, ELK-BULB, ELK-BULB2, ELK-LAMPL, "XSL-" and "LED LIGHT STRIP" use write `0000fff3-…` and read `0000fff4-…`. LEDBLE, "LED-", JACKYLED and XROCKER use `0000ffe1-…`. DMRRBA-007 uses Nordic UART `6e400002-b5a3-f393-e0a9-e50e24dcca9e`.
  source: https://raw.githubusercontent.com/dave-code-ruiz/elkbledom/main/custom_components/elkbledom/models.json
  publisher: dave-code-ruiz (GitHub, elkbledom)
  pub_date: unknown (main branch)
  accessed: 2026-11-12
  confidence: high
  class: protocol

- claim: ELK-BLEDOM (entry 1) commands in hex (file values are decimal):
  - power on: `7E 04 04 F0 00 01 FF 00 EF`
  - power off: `7E 04 04 00 00 00 FF 00 EF`
  - colour: `7E 07 05 03 RR GG BB 0A EF`
  - brightness: `7E 04 01 ii 01 FF 02 01 EF`, where ii = intensity*100/255
  - white: `7E 07 01 ii 00 00 00 00 EF`
  - effect: `7E 07 03 vv 03 FF FF 00 EF`
  - effect speed: `7E 07 02 vv 00 00 00 00 EF`
  - colour temp: `7E 07 05 02 ww cc 00 00 EF`
  - query: `7E 00 01 FA 00 00 00 00 EF`

  A second ELK-BLEDOM variant uses `00` as byte 1, e.g. on `7E 00 04 F0 00 01 FF 00 EF` and colour `7E 00 05 03 RR GG BB 00 EF`.
  source: https://raw.githubusercontent.com/dave-code-ruiz/elkbledom/main/custom_components/elkbledom/models.json
  publisher: dave-code-ruiz
  pub_date: unknown
  accessed: 2026-11-12
  confidence: high
  class: protocol

- claim: elkbledom has three per-family behaviour quirks:
  - MELK/MODELX devices (name prefixes "melk"/"modelx") need a login sequence first: write `7E 07 83`, then `7E 04 04`, without response.
  - ELK-BLEDOB and ELK-BLEDDM get different write pacing.
  - Notification subscription is skipped for MELK and LEDBLE devices.

  Other hard-coded commands: mic enable `7E 04 07 01 FF FF FF 00 EF`, mic effect `7E 05 03 vv 04 FF FF 00 EF`, scheduler `7E 00 82 hh mm 00 00 vv EF`.
  source: https://raw.githubusercontent.com/dave-code-ruiz/elkbledom/main/custom_components/elkbledom/elkbledom.py
  publisher: dave-code-ruiz
  pub_date: unknown
  accessed: 2026-11-12
  confidence: high
  class: protocol

### Q4 — iOS constraints / AccessorySetupKit

- claim: Apple's AccessorySetupKit (iOS/iPadOS 18.0+) lets an app "discover, select, and set up a specific Bluetooth accessory without requesting permission to use Bluetooth". The system picker matches scan results against all rules in an ASDiscoveryDescriptor.
  source: https://developer.apple.com/documentation/accessorysetupkit (snippet)
  publisher: Apple
  pub_date: 2024-06 (iOS 18 framework)
  accessed: 2026-11-12
  confidence: medium (snippet of a primary doc)
  class: platform-constraint

- claim: With AccessorySetupKit, pairing/bonding happens without the usual OS pairing prompt. Non-pairing devices still appear in the picker. CoreBluetooth may still prompt for pairing on an accessory already bonded via ASK.
  source: https://punchthrough.com/ios18-accessorysetupkit-everything-ble-developers-need-to-know/ (snippet)
  publisher: Punch Through
  pub_date: 2024 (stale for ecosystem bar)
  accessed: 2026-11-12
  confidence: low (snippet; secondary)
  class: platform-constraint

- claim: A developer reports that once AccessorySetupKit Bluetooth support was declared in Info.plist (iOS 18.3.1), a plain CBCentralManager stayed `poweredOff` until an accessory was authorized. If true, adopting ASK closes off the ordinary "scan for any strip" CoreBluetooth path. This is a key architecture risk for generic, multi-brand strip discovery.
  source: https://developer.apple.com/forums/thread/852603 (snippet)
  publisher: Apple Developer Forums (developer post)
  pub_date: unknown
  accessed: 2026-11-12
  confidence: low (single forum report, snippet)
  class: platform-constraint

- claim: A forum report says the AccessorySetupKit picker shows only 3 accessories when `filterDiscoveryResults` is used (iOS 27.1, FB25362032). This matters for onboarding many identical strips for group control.
  source: https://origin-devforums.apple.com/forums/tags/accessorysetupkit (snippet)
  publisher: Apple Developer Forums (developer post)
  pub_date: unknown (2026, given iOS 27.1)
  accessed: 2026-11-12
  confidence: low (snippet)
  class: platform-constraint

### Q5 — Max simultaneous BLE connections

- claim: In AOSP's current Bluetooth module (packages/modules/Bluetooth), GATT_MAX_PHY_CHANNEL is described as a compile-time static-allocation size. The runtime maximum comes from the system property `bluetooth.core.le.max_number_of_concurrent_connections`, with a floor of 8 that the commit ties to the CDD requirement of at least 8 GATT channels. The diff appears to raise the compile-time value to 16, but the value could not be confirmed from the snippet.
  source: https://gitlab.e.foundation/e/os/android_packages_modules_bluetooth/-/commit/ad4bb564ab97ee0b53360df0e8f4bbea38598703 (snippet; mirror of AOSP)
  publisher: /e/OS mirror of AOSP Bluetooth module
  pub_date: unknown
  accessed: 2026-11-12
  confidence: low (snippet; needs AOSP primary confirmation)
  class: performance

- claim: History: the Bluedroid commit "LE: Increase number of simultaneous connections" (KitKat, klp-dev) raised the default GATT connection limit from 4 to 7 in bt_target.h. These limits are global per phone, not per app.
  source: https://android.googlesource.com/platform/external/bluetooth/bluedroid/+/kitkat-mr1-release%5E1..kitkat-mr1-release/ (snippet)
  publisher: AOSP
  pub_date: 2013 (stale; historical baseline only)
  accessed: 2026-11-12
  confidence: low
  class: performance

- claim: A developer report on a Pixel 5 running Android 13 shows connection attempts failing with GATT_FAILURE alongside a "max GATT client reached" log line. The practical ceiling on shipping phones is lower than the spec ideal and varies by vendor.
  source: https://blog.retronyms.com/2023/05/developer-notes-navigating-androids.html (snippet)
  publisher: Retronyms (developer blog)
  pub_date: 2023-04/05
  accessed: 2026-11-12
  confidence: low (single anecdotal source)
  class: performance

## Leads worth chasing

- Q1: Search dotintent/react-native-ble-plx issues for "0.82", "bridgeless", "interop" and "Expo 55" to confirm runtime behaviour of 3.5.1 on New-Arch-only stacks. Also check whether the v4 branch is installable from git. Inspect `react-native-ble-manager@13.0.0` for `app.plugin.js` (unpkg file listing) to confirm the directory's `configPlugin:true`.
- Q1: Evaluate the `@sfourdrinier/react-native-ble-plx` fork (TurboModule, RN 0.86+): release cadence, downloads, and whether it keeps API parity.
- Q2: `8none1/lednetwf_ble/protocol_docs/05_basic_commands.md` (checksum definition and full wrapped power/colour packets), `02_manufacturer_data.md` (product-ID byte, firmware/bleVersion in the advertisement), and `custom_components/lednetwf_ble/data/ble_devices.json` (device table).
- Q2: Hypothesis to verify: the inner checksum is the Magic Home sum-of-bytes & 0xFF.
- Q3: Bluetooth-Devices/led-ble `led_ble.py`/`model_db.py`: how the reply to `EF 01 77` yields a model/protocol byte. That is the best "probe after connect" mechanism for the Triones family.
- Q3: Saberlight/madhead protocol.md for Triones `56 RR GG BB 00 F0 AA` and `CC 23 33`/`CC 24 33` verification. Not reached this run.
- Q4: Apple ASDiscoveryDescriptor fields (bluetoothServiceUUID, name substring, company identifier/manufacturer data) to check whether cheap strips without unique service UUIDs (ELK FFF0, Zengge FFFF) can be matched. Also whether declaring `NSAccessorySetupBluetoothServices` restricts CoreBluetooth to ASK-authorized accessories.
- Q5: The AOSP primary `system/stack/include/bt_target.h` / sysprop definitions on android.googlesource.com (current main) for the GATT_MAX_PHY_CHANNEL value and the default of `bluetooth.core.le.max_number_of_concurrent_connections`. For iOS, Apple Developer Forums threads on the concurrent connection count.

## Looked for, not found

- I found no first-party issue or discussion in dotintent/react-native-ble-plx confirming or denying that 3.5.1 works on RN 0.82+/Expo SDK 55 via interop. The PR #1331 page does not address interop.
- I found no 4.x, alpha or beta react-native-ble-plx on npm. The only dist-tag is `latest` = 3.5.1, though the registry document was only partly read.
- The repo `8none1/zengge_lednetwf_ble` does not exist (HTTP 404). The canonical repo is `8none1/lednetwf_ble`.
- The BJ_LED README gives no GATT service or characteristic UUIDs.
- led-ble const.py defines no service UUIDs, only characteristic candidates.
- The LEDnetWF quick reference and transport doc do not define the checksum algorithm.
- I could not fetch Apple CoreBluetooth docs on per-app peripheral UUIDs (no MAC), background scanning needing service-UUID filters and coalescing results, or NSBluetoothAlwaysUsageDescription (budget). These remain unverified beliefs and are not findings.
- I found no current (≤ 3 years) iOS figure for maximum simultaneous BLE connections.
- I could not reach the saberlight (madhead) protocol.md.
