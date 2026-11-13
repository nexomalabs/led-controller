---
title: 'technical research: Phone-only LED light control from React Native (iOS + Android)'
type: 'technical'
topic: 'Phone-only LED light control from React Native (iOS + Android)'
decision: 'Which LED light protocols a phone-only React Native app (iOS + Android) can control directly in v1, and which adapter architecture and libraries make generic control (presets, single lights, groups) work'
source: 'native run (deep-recon)'
status: complete
preset: 'standard'
validation: 'normal'
created: '2026-11-12'
updated: '2026-11-13'
claims_verified: 18
claims_unverified: 4
claims_disputed: 1
---

# technical research: Phone-only LED light control from React Native (iOS + Android)

**Decision this research serves:** Which LED light protocols a phone-only React Native app (iOS + Android) can control directly in v1, and which adapter architecture and libraries make generic control (presets, single lights, groups) work.

## Executive summary

**Answer.** A phone-only React Native app can control a wide range of LED lights. It cannot control *any* LED light. No common protocol exists, so "generic" has to mean three parts:
- one internal light model,
- a growing catalog of per-protocol drivers,
- a discovery layer that works out which driver a device needs.

The evidence supports this v1 scope:

| Transport | Support in v1 | Platforms |
|---|---|---|
| Bluetooth LE | ELK-BLEDOM (duoCo / Lotus Lantern), Triones / HappyLighting / LEDnet, BJ_LED; Zengge after a hardware check | iOS + Android |
| Wi-Fi, found by Bonjour | WLED, Hue (through the user's Bridge), Nanoleaf | iOS + Android |
| Wi-Fi, found by UDP | Govee LAN, LIFX, Yeelight, Magic Home | Android; on iOS only if Apple grants the multicast entitlement, otherwise by manual IP entry |
| IR | 44-key NEC remotes, with the user picking a code set | Android phones that have an IR blaster only |
| Defer to v2 | Tuya / Smart Life (cloud-account flow), Matter (a separate native project), Govee BLE (differs per model) | — |

**What drives this answer.**
1. **iOS sets the limits.** iPhones have no IR transmitter [57][80]. Auto-discovering UDP-based Wi-Fi lights needs a multicast entitlement that Apple must approve and has refused at least once in 2026 [36][38][73]. Apple's own BLE shortcut, AccessorySetupKit, blocks generic Bluetooth scanning [78]. Android is broader on every one of these points [40][49].
2. **The New Architecture decides the libraries.** React Native 0.82+ / Expo SDK 55+ run only on the New Architecture [45][77]. `react-native-ble-manager` supports it natively and `react-native-ble-plx` does not [10][12]. No maintained UDP or IR library exists [41][52]. So you will write two small native modules (UDP and IR transmit) with Expo Modules in an Expo development build.
3. **The light model is a solved problem.** Home Assistant, Matter, Google Home and HomeKit all describe a light the same way. Each light declares a set of colour capabilities and reports a single current mode. One central converter turns any request into a format that light supports. Lights that can't report their state get an explicit "assumed state" flag [58][59][60][61][62]. Home Assistant's conversion code can be ported directly [63].

**Biggest caveat.** Every Bluetooth protocol, and several Wi-Fi ones, is reverse-engineered, and variants differ even within one family [2][6][9]. Device support will grow one device at a time, and you need real strips on your desk to test against. Tuya, one of the brands you want, is the least accessible [21][24].

## 1. Bluetooth LE strips and the React Native BLE stack

**Findings.** None of the cheap-strip vendors publishes its Bluetooth protocol. Every protocol found was reverse-engineered by the community, mostly for Home Assistant integrations [1][2][3][6][7]. The protocols are simple, though: plain GATT writes of short byte frames, with no encryption or key exchange in any family examined [1][2][7]. Zengge is the exception to check. Its protocol has a single source and no independent confirmation. "No encryption" is inferred from the docs not mentioning any, the header has an unexplained "protection" bit, and a secondary write-up mentions one capture that looked AES-encrypted. Treat it as low confidence and check on hardware [6][81].

| Family (typical app / name) | Service · write characteristic | Command shape | State readback | Sources |
|---|---|---|---|---|
| ELK-BLEDOM (duoCo Strip, Lotus Lantern; `ELK-*`, `MELK*`) | FFF0 · FFF3 (some variants FFE1 or Nordic UART) | 9-byte `7E … EF` frames, e.g. colour `7E 07 05 03 RR GG BB 0A EF` | Some variants only | [1][2] |
| Triones / HappyLighting / LEDnet (`Triones*`, `LEDnet*`, `QHM-*`, `Dream~*`) | FFD5 · FFD9 (library probes ff01/ffd5/ffd9/ffe9/ffe5) | e.g. on `CC 23 33`, colour `56 RR GG BB 00 F0 AA`; state query `EF 01 77` | Yes | [3][4][5] |
| Zengge / Magic Home BLE (`LEDnetWF`, `IOTWF`, `IOTBT`) | FFFF · FF01 (notify FF02) | Magic-Home bytes (e.g. `0x31` colour) inside an 8-byte header with a sequence counter | Yes | [6] |
| BJ_LED (MohuanLED) | not documented | `69 96 …`, e.g. colour `69 96 05 02 RR GG BB WW` | **No** | [7] |
| Govee BLE (H6xxx/H61xx) | per model | 20-byte `0x33 …` packets with an XOR checksum (documented for H6001 only) | varies | [8][9] |

- **Variants inside one family differ.** The ELK integration keeps 24 name-prefix model entries. Each specifies its own characteristic UUIDs and byte-1 variant. Some also need a fixed "login" sequence (MELK/MODELX), extra write pacing (BLEDOB/BLEDDM) or have no notifications [2]. Govee command sets also differ between models, and community integrations lag new models [9].
- **Identifying the protocol.** Home Assistant core matches these devices by advertised name only [4]. Names collide: `LEDBLE` is claimed by two integrations [2][4]. Users can rename devices in the vendor app [5]. Characteristic probing also collides: FF01 is used by both the led-ble library and Zengge [3][6]. A reliable fingerprint combines name prefix, manufacturer data (Zengge uses company IDs 0x5A00–0x5AFF [6]), service UUIDs probed after connecting, and, where available, a state-query reply [3].
- **Write-only devices exist.** BJ_LED never answers on its notify characteristic, and some ELK variants send no notifications [2][7]. The app has to keep an assumed (optimistic) state for them.

**React Native library.** `react-native-ble-manager` released 13.0.0 on 2026-11-05 and has averaged about one release a month for 12 months [10]. It ships a TurboModule spec (codegenConfig) and has supported only the New Architecture since 12.x. React Native Directory lists an Expo config plugin for it [10][13]. Version 13 changed the byte layout of Android manufacturer data, which matters for fingerprinting code [10].

`react-native-ble-plx` has had one release in 12 months (3.5.1, 2026-02-18) and has no codegenConfig [11]. Its New Architecture rewrite (v4, React Native ≥ 0.82, iOS 17+) is an open draft PR that is not on npm, and the main branch has had no pushes since 2026-03-09 [12][13]. The directory's "New Architecture: yes" flag for ble-plx can only reflect React Native's interop layer, which Expo describes as "not perfect" [13][45]. *Choose react-native-ble-manager* (high confidence on the release facts; medium that ble-plx would cause trouble in practice, since no failure reports were found).

**Platform limits.**
- **Android 12+ permissions:** Android 12 and later require `BLUETOOTH_SCAN` and `BLUETOOTH_CONNECT`. Declaring `neverForLocation` avoids the location permission [14].
- **Connection limits:** Neither Apple nor Google publishes a maximum number of simultaneous BLE connections. Android's runtime limit is a system property with a floor of 8 (low confidence) [17]. Older reports put the ceiling at about 7 on Android and 8–10 on iOS (stale) [15][16]. Group control of many strips therefore needs either a connection cap or connect-send-disconnect cycling. Expect visible lag across large BLE groups.
- **iOS AccessorySetupKit is not a fit for generic discovery.** On iOS 18+ it can pair accessories without a Bluetooth permission prompt [18]. Apple's WWDC24 session confirms the cost (verified): once an app declares it, CoreBluetooth turns on only after the app has paired accessories through it, and scans return only those accessories [19][78]. A generic "find any strip" app should use plain CoreBluetooth with the standard Bluetooth permission instead.
- **Bluetooth Mesh:** It has standard lighting models, but a phone can reach a mesh only through a GATT proxy node [20]. No evidence was found that cheap strips use it.

## 2. Wi-Fi lights, Matter, and the local-network stack

**Findings.** Each brand has its own local protocol. They fall into three groups.

| Access model | Brand: discovery → control | Sources |
|---|---|---|
| **Open, no pairing** | WLED: mDNS `_wled._tcp` → HTTP JSON `/json/state` | [30][31] |
| | LIFX: UDP broadcast → official binary UDP protocol | [28] |
| | Govee: UDP multicast 239.255.255.250:4001 → UDP 4003 (LAN toggle per device in the Govee app) | [26][27] |
| | Yeelight: multicast :1982 → TCP 55443 JSON-RPC (owner must enable "LAN Control") | [29] |
| | Magic Home Wi-Fi: UDP broadcast :48899 → TCP 5577 (no auth seen) | [32] |
| **Pairing token** | Hue Bridge: press the link button → application key → HTTPS CLIP v2 | [34] |
| | Nanoleaf: hold power 5–7 s → token → HTTP :16021 | [33] |
| **Cloud-keyed** | Tuya / Smart Life: each device needs a Local Key that comes from Tuya's cloud | [21][72] |

- **Tuya** is the hardest for an "any brand" app.
  - **Local control:** Every device needs its own Local Key, and the key changes whenever the device is re-paired [21].
  - **Getting the key:** The manual route is a Tuya IoT developer cloud project. Its API access is a one-month trial that can be refreshed only every 6 months [21][72]. *Verification disputed the claim that this is the only route:* Tuya-Local fetches keys through a cloud-assisted sign-in to the Smart Life app, with no developer account [72].
  - **Official App SDK:** It costs about $5,000 for the first year and $2,000 a year to renew. The free edition is capped at 100 users [23]. Tuya's own pages disagree on the free edition's call quota (100k vs 1M a month) [22][23]. Neither page says an SDK app can reach lights already paired in the consumer Smart Life app.
  - **Official integration route:** Tuya's own `tuya-device-sharing-sdk`, which Home Assistant's built-in Tuya integration uses, controls lights already paired in Smart Life. The user enters a user code and scans a QR code, with no developer project needed [24][75]. It is a Python SDK. Two things are unconfirmed: whether a mobile app may use the same flow, and whether control then goes through Tuya's cloud rather than the LAN.
  - **Matter:** Some Tuya lights are now Matter-certified, which opens a standard path to them [25].
- **Hue Bluetooth bulbs:** There is no official third-party Bluetooth API. Bridgeless control is a feature of Philips' own app [35].

**iOS local-network rules (the main iOS constraint).**
- **Local Network permission:** Nearly all LAN traffic (TCP, UDP unicast, Bonjour) needs this permission and an `NSLocalNetworkUsageDescription` string [36].
- **Multicast entitlement:** Sending or receiving UDP multicast or broadcast also needs the `com.apple.developer.networking.multicast` entitlement, which Apple must approve. This was independently verified [36][37][73]. In July 2026 Apple denied one such request and told the developer to use Bonjour instead [38]. Older reports put approval at 3–6 days (stale) [39].
- **What that means here:** Without the entitlement, the app cannot auto-discover Govee, Yeelight, Magic Home, LIFX (broadcast) or Tuya devices on iOS. WLED, and probably Hue and Nanoleaf, can still be found through Bonjour types declared in `NSBonjourServices` [31][36]. Their mDNS types were not confirmed this run.

**Android local-network rules.** Apps targeting Android 17 (API 37) are blocked from the LAN by default. They must request `ACCESS_LOCAL_NETWORK`, which covers TCP and all UDP. This was independently verified [40][74]. A system device picker avoids the prompt, but it grants access to one device only, and Google points home-automation apps to the permission instead [40]. This permission is in the same "Nearby devices" group as the Bluetooth permissions, so an app that already holds BLE permission isn't prompted a second time [40].

**React Native networking libraries.**

| Library | Status | Sources |
|---|---|---|
| `react-native-udp` | Unmaintained. Last release 2023-01-26; its New Architecture issue has had no response | [41][13] |
| `react-native-tcp-socket` | Maintained (6.4.3, 2026-09-10), but a legacy bridge module | [42] |
| `react-native-zeroconf` (mDNS) | Active, flagged New Architecture (1.0.0-rc.3, 2026-10-29) | [43] |
| `react-native-nitro-dgram` (Nitro UDP) | Stalled since 2026-01 | [44] |

No healthy UDP library exists for current React Native. UDP is required for Govee, LIFX, Magic Home discovery and Tuya, so plan a small in-house native UDP module.

**Matter.**
- **iOS:** A third-party app can commission Matter lights into its own ecosystem. It uses a MatterSupport extension plus Apple's Matter framework. This was verified against Apple's docs, with corrections: it needs MatterSupport on iOS 16.1+, and the `allow-setup-payload` entitlement is needed only when the app supplies the setup payload itself [46][76]. Espressif ships this from React Native through a custom native module; no reusable React Native Matter library was found [46]. Developers report that the flow is unreliable for Thread devices, still as of iOS 26.2 [47].
- **Android:** The Google Home APIs can commission a device to the app's own fabric plus Google's [48].
- **Fit for v1:** Matter is the only standards-based route to "any brand", but it is a native-heavy project of its own.

## 3. IR from phones, and the generic adapter architecture

**IR findings.**
- **Android API:** `ConsumerIrManager` is a current Android API with no deprecation in AOSP main (verified). `transmit()` blocks until the pattern is sent and accepts only patterns shorter than 2 s [49][79].
- **Which phones have IR:** IR blasters ship mainly on Xiaomi/Redmi/POCO phones (POCO F7 [55]), OnePlus 15 [56], and some vivo/iQOO, OPPO and Red Magic models (aggregators, low confidence) [50][51]. Buyer guides say no current Samsung phone has one [51]. No iPhone has an IR transmitter. Apple's iPhone 17 spec page lists none, and an independent guide states that no iPhone has ever shipped with one [57][80]. No market-share data was found.
- **React Native library:** No maintained React Native IR library supports the New Architecture; the best-known one was last updated about 4 years ago [52]. A thin custom native module (3 methods) is the realistic path.
- **Remote codes:** Cheap 44-key RGB remotes use NEC codes of the form `0x00FFxxxx`. Community code tables are incomplete and differ between controllers that look alike [53][54]. Phones can only transmit, not receive, so the app can't learn codes. The user will have to pick a code set (inference). All IR lights are write-only.

**How established platforms model a light.** Four ecosystems converged on the same shape:

- **Home Assistant:** each light declares a fixed set of supported colour modes (onoff, brightness, color_temp, hs, xy, rgb, rgbw, rgbww, white) and reports a single current mode. HA core converts any request into a mode the light supports before the driver sees it. Effects and transitions are separate feature flags [58].
- **Matter:** Colour Control advertises optional HS / enhanced hue / colour loop / XY / CT features, with colour temperature in mireds [60]. Google's Matter traits mirror these as boolean flags [71].
- **Google Home:** ColorSetting declares an rgb or hsv colour model and/or a Kelvin range [61].
- **HomeKit:** Lightbulb requires only On; Brightness, Hue, Saturation and ColorTemperature are optional [62].
- **Write-only devices** get an explicit flag in each system: HA's `assumed_state` (the UI shows separate On and Off buttons instead of a toggle) and Google's `commandOnlyColorSetting` [59][61].

**One preset, many devices.** Home Assistant's `color.py` is a complete, readable reference [63]:
- sRGB → CIE xy with snapping to the device's gamut
- RGB → RGBW using the minimum channel, plus an RGBWW variant
- Kelvin → RGB using Helland's approximation (1000–40000 K)

Philips' archived note defines the Hue gamut triangles and the "closest point on the triangle" rule [64]. HA's matrix and Philips' differ, so choose one matrix pair and use it in both directions [63][64]. Perceived output depends on brightness and channel values together, so define each preset as colour plus brightness [58].

**Groups and rate limits.**
- **Hue bridge:** Allow no more than 3 requests in flight. Two independent, maintained libraries cap concurrency at 3, and aiohue backs off on 429/503 responses [65][66]. openHAB paces requests at 50 ms (about 20/s) [66]. An undated aggregator gives a soft cap of about 10 light commands/s and 1 room command/s (low confidence; no Signify source was readable) [67]. A single grouped_light write, or a scene recall, changes a whole room in one request [67].
- **LIFX:** At most 20 messages/s per device [28][68].
- **"Popcorn effect":** Sending one command per bulb makes group members change one after another. The usual fix is to use the vendor's own groups or scenes (old, low-confidence community reports) [69]. Across brands there is no shared group, so the app will have to fan commands out.

**Expo or bare React Native.**
- **Current Expo:** Expo SDK 58 (2026-10-15) ships React Native 0.88 [70].
- **New Architecture:** SDK 55 and later run only on the New Architecture. This was verified: React Native 0.82 was the first release that runs entirely on it [45][77].
- **Custom native code:** Code such as BLE, UDP and IR modules goes into development builds through prebuild and config plugins, not Expo Go [70]. Expo Modules 2.0, which lets modules be written as plain Swift/Kotlin classes, is in beta [70].
- **Verdict:** Expo with development builds supports everything this app needs.

## 4. Cross-dimension insights

1. **Android can do more than iOS, so feature parity isn't possible.**
   - **Android:** IR works on phones that have a blaster [49]. The "Nearby devices" permission group covers both Bluetooth and LAN access, so users see one prompt [14][40].
   - **iOS:** There is no IR at all [57][80]. UDP discovery depends on an entitlement Apple must grant [36][73], and AccessorySetupKit can't be used for generic Bluetooth discovery [78].
   - **For the PRD:** It needs a per-platform feature matrix, not a promise of parity.
2. **Native code is needed from day one.** The New Architecture cutover leaves UDP and IR without maintained libraries [41][44][52], and both are required: UDP for Govee, LIFX, Magic Home and Tuya; IR for remote-controlled strips. "Expo + development builds + in-house Expo Modules" is therefore a starting requirement, not a later optimization [45][70].
3. **Write-only lights are normal, not an edge case.** IR lights, BJ_LED and some ELK variants can't report their state [2][7][53], so the app must default to an assumed state. For UX, a preset is something the app *applies*, not a state it can confirm. Lights with an assumed state need separate On and Off buttons, following Home Assistant's pattern [59].
4. **Discovery is harder than control.**
   - **Control is simple:** In every family the commands are short byte frames or small JSON [1][3][30].
   - **Discovery is the hard part:** Variants collide by name and characteristic UUID [2][3][4][6], iOS gates multicast [36], and Android 17 gates the whole LAN [40].
   - **For the architecture:** Treat discovery and fingerprinting as their own module, backed by a data-driven device catalog. The ELK integration's `models.json` is a working example [2].
5. **"All lights" presets will show stagger across brands.** The slowest transport sets the pace:
   - **BLE:** a phone holds only about 7–8 connections, and some variants need write pacing (low confidence) [2][17].
   - **Hue:** the bridge accepts only 3 requests in flight [65][66].
   - **LIFX:** at most about 20 messages/s per device [68].

   Use vendor-native groups where they exist, such as Hue grouped_light and WLED sync groups [30][67]. Fan out per transport, in parallel, everywhere else.
6. **Tuya, one of the brands you want, is reachable only through Tuya's cloud accounts.** The workable routes are a cloud sign-in that fetches Local Keys and the official device-sharing SDK [72][75]. The growing set of Matter-certified Tuya lights [25] would let a later Matter adapter cover part of Tuya without either route.

## 5. Recommendations

| # | Recommendation | Feeds | Confidence basis |
|---|---|---|---|
| R1 | **Scope v1 as in the summary table.** Bluetooth: ELK, Triones/LEDnet and BJ_LED. Wi-Fi: WLED, Hue, Nanoleaf, Govee LAN, LIFX, Yeelight and Magic Home. IR: Android only. Defer Tuya, Matter and Govee BLE. | PRD scope | High for the documented or two-source Wi-Fi APIs [26][27][28][30]. Medium for the Bluetooth families, which are reverse-engineered and independently verified only for ELK [1][2]. Low for Zengge, which is unverified [6]. |
| R2 | **Write the PRD as a per-platform feature matrix.** IR is Android-only. iOS UDP discovery depends on the entitlement, with manual IP entry as the fallback. | PRD | High: the constraints were independently verified [36][40][73][74][78][80]. |
| R3 | **Driver interface.** Each driver declares its capabilities statically: a set of colour modes (HA-style), feature flags (effects, transitions) and a state mode (`reported` or `assumed`). It receives commands already converted. The canonical internal colour is hue/saturation + brightness + Kelvin. One central converter, ported from HA `color.py`, uses a single matrix pair in both directions. | Architecture | High: primary sources, with the same pattern in four ecosystems [58][59][60][61][62][63]. |
| R4 | **Transport layer.** Give each transport its own command queue and limits. Hue: at most 3 requests in flight, with backoff on 429/503 (high confidence) and about 10 light commands/s and 1 group command/s (low confidence). LIFX: at most 20 messages/s per device (medium). BLE: pacing per variant and a connection cap (low–medium). Use native groups when every member of a group sits behind one hub. | Architecture | Mixed, as marked per item [2][17][65][66][67][68]. |
| R5 | **Discovery and fingerprinting module.** Combine name prefix, manufacturer data, service-UUID probe and state-query reply. Keep a data-driven device catalog so new variants can be added without code changes. | Architecture | Medium: inferred from two primary codebases [2][3][6]. |
| R6 | **Stack.** Expo SDK 58 with development builds. `react-native-ble-manager` for Bluetooth and `react-native-zeroconf` for mDNS. In-house Expo Modules for UDP and IR transmit. For TCP (Yeelight, Magic Home), prototype `react-native-tcp-socket` through the interop layer early, or build TCP into the in-house module. Don't adopt AccessorySetupKit. | Architecture | High on today's versions; re-check monthly [10][42][43][45][70][77][78]. |
| R7 | **Platform tasks now.** Apply for the iOS multicast entitlement at project start; one 2026 request was denied [38]. Plan for `ACCESS_LOCAL_NETWORK` before targeting API 37. The shared "Nearby devices" group means no extra prompt once Bluetooth permission is granted. | Roadmap risk | High [38][40][74]. |
| R8 | **Presets in UX.** Define each preset as colour + brightness, convert it per light, and show assumed-state lights with separate On and Off buttons. Tell users that groups mixing brands won't change in perfect sync. | UX / PRD | High for the model [58][59]. Low for sync timing, which wasn't measured [69]. |

## 6. Open questions

1. **Will Apple grant this app the multicast entitlement?** Apply. The only 2026 data point is one denial [38]. If it's denied, test whether unicast discovery (a subnet sweep or a remembered IP) works for Govee, Magic Home and LIFX. No evidence was found either way.
2. **How many BLE connections can a phone hold at once?** Measure on target iOS and Android phones. Only stale or low-confidence figures exist [15][16][17].
3. **What exactly is the Zengge / LEDnetWF protocol, and is it encrypted?** Capture a real device with nRF Connect. It has one source, and one capture looked AES-encrypted [6][81].
4. **Does `react-native-tcp-socket` work through the interop layer on React Native 0.88?** Build a prototype [42][45].
5. **Can a mobile app (not Home Assistant) use Tuya's device-sharing QR flow, and is control then cloud-only?** Read the SDK's terms and source [24][75].
6. **What are Signify's official Hue rate limits?** The docs are login-gated, so join the Hue Developer Program [67].
7. **What CoreBluetooth background and identifier rules apply on iOS?** Not researched. Read Apple's CoreBluetooth docs during architecture.
8. **How does the app handle the Hue Bridge's HTTPS certificate, and cleartext HTTP for WLED and Nanoleaf?** This involves iOS ATS settings and Android's network security config; nothing was verified this run [33][34].
9. **Does Android need `WifiManager.MulticastLock` to receive discovery replies?** Not checked.
10. **How many of your users' phones have an IR blaster?** No market-share data was found [50][51]. Decide in the PRD whether IR is worth building in v1.
11. **Do newer Govee BLE models use different framing or encryption?** Coverage is per model and lags new releases [9].

## 7. Source appendix

| # | Supports | Publisher | Pub date | Accessed | Confidence |
|---|---|---|---|---|---|
| [1] | ELK-BLEDOM UUIDs (FFF0/FFF3) and 9-byte frames; reverse-engineered | [FergusInLondon/ELK-BLEDOM](https://github.com/FergusInLondon/ELK-BLEDOM) | undated | 2026-11-12 | medium |
| [2] | ELK variants, quirks, command bytes, plaintext, models.json catalog | [dave-code-ruiz/elkbledom](https://github.com/dave-code-ruiz/elkbledom) | main branch, undated | 2026-11-12 | high |
| [3] | Triones/LEDnet characteristic probing, `EF 01 77` state query | [Bluetooth-Devices/led-ble](https://github.com/Bluetooth-Devices/led-ble) | main branch, undated | 2026-11-12 | high |
| [4] | HA led_ble matches devices by name only | [Home Assistant core: led_ble manifest](https://github.com/home-assistant/core/blob/dev/homeassistant/components/led_ble/manifest.json) | dev branch, undated | 2026-11-12 | high |
| [5] | Triones command bytes; renamed devices escape detection | [Home Assistant Community thread](https://community.home-assistant.io/t/custom-component-integration-for-triones-happylighting-based-ble-lights/356011) | undated | 2026-11-12 | medium (snippet) |
| [6] | Zengge/LEDnetWF UUIDs, header, manufacturer-data range | [8none1/lednetwf_ble](https://github.com/8none1/lednetwf_ble) | 2026-01 (last push) | 2026-11-12 | low (unverified) |
| [7] | BJ_LED commands; no state readback | [8none1/bj_led](https://github.com/8none1/bj_led) | undated | 2026-11-12 | medium |
| [8] | Govee BLE 20-byte packets (H6001) | [chvolkmann/govee_btled](https://github.com/chvolkmann/govee_btled) | undated (stale) | 2026-11-12 | medium |
| [9] | Govee BLE varies per model; integrations lag | [Home Assistant Community: Govee H617A](https://community.home-assistant.io/t/govee-h617a-ble-led-strip-lights-reverse-engineering/901353) | ~2025 | 2026-11-12 | medium |
| [10] | ble-manager 13.0.0, monthly releases, codegenConfig, manufacturer-data change | [innoveit/react-native-ble-manager releases](https://github.com/innoveit/react-native-ble-manager/releases) | 2026-11-05 | 2026-11-12 | high |
| [11] | ble-plx 3.5.1, one release in 12 months, no codegen | [dotintent/react-native-ble-plx releases](https://github.com/dotintent/react-native-ble-plx/releases) | 2026-02-18 | 2026-11-12 | high |
| [12] | ble-plx v4 TurboModule rewrite is an open draft | [react-native-ble-plx PR #1331](https://github.com/dotintent/react-native-ble-plx/pull/1331) | 2026-03-16 (state observed 2026-11-12) | 2026-11-12 | high |
| [13] | Library flags: New Architecture, config plugin, unmaintained, last push | [React Native Directory](https://reactnative.directory/) | live, 2026-11 | 2026-11-12 | medium |
| [14] | Android 12+ BLUETOOTH_SCAN/CONNECT, neverForLocation | [Android Developers: Bluetooth permissions](https://developer.android.com/develop/connectivity/bluetooth/bt-permissions) | live | 2026-11-12 | high |
| [15] | iOS ~8–10 connection anecdote | [Apple bluetooth-dev list](https://lists.apple.com/archives/bluetooth-dev/2013/Jul/msg00026.html) | 2013-07 (stale) | 2026-11-12 | low |
| [16] | Android parallel-connection problems at ~5+ | [Nordic DevZone](https://devzone.nordicsemi.com/f/nordic-q-a/77268/android-ble-issues-on-parallel-5-connection-establishments/319483) | ~2021 (stale) | 2026-11-12 | low |
| [17] | Android runtime connection limit property, floor 8 | [AOSP Bluetooth commit (e/OS mirror)](https://gitlab.e.foundation/e/os/android_packages_modules_bluetooth/-/commit/ad4bb564ab97ee0b53360df0e8f4bbea38598703) | undated | 2026-11-12 | low |
| [18] | AccessorySetupKit pairs without a Bluetooth prompt | [Apple: AccessorySetupKit](https://developer.apple.com/documentation/accessorysetupkit) | 2024-06 | 2026-11-12 | medium |
| [19] | Report that declaring ASK limits CoreBluetooth | [Apple Developer Forums 852603](https://developer.apple.com/forums/thread/852603) | undated | 2026-11-12 | medium (verified by [78]) |
| [20] | Bluetooth Mesh needs a GATT proxy for phones | [Silicon Labs: Bluetooth Mesh app](https://www.silabs.com/software-and-tools/bluetooth-mesh-mobile-app) | undated | 2026-11-12 | medium |
| [21] | Tuya Local Key requirement, ports, 1-month trial, key changes on re-pair | [jasonacox/tinytuya](https://github.com/jasonacox/tinytuya) | live | 2026-11-12 | medium (disputed on "only route") |
| [22] | Tuya SDK CNY 33,500/yr; free edition 100k calls, 100 users | [Tuya VAS: App SDK](https://www.tuya.com/vas/commodity/SDK_APP_CREATE) | undated (© 2026) | 2026-11-12 | medium |
| [23] | Tuya App SDK $5,000 / $2,000; free edition 100 users, 1M calls | [Tuya: App SDK](https://www.tuya.com/platform/appdev/app-sdk) | undated (© 2026) | 2026-11-12 | high |
| [24] | Official Smart Life integration via tuya-device-sharing-sdk | [tuya/tuya-smart-life](https://github.com/tuya/tuya-smart-life) | undated | 2026-11-12 | high (verified by [75]) |
| [25] | Matter-certified Tuya Wi-Fi lights exist | [CSA certified product: Tuya Wi-Fi strip](https://csa-iot.org/csa_product/tuya-wi-fi-strip-light-rgbc/) | undated | 2026-11-12 | medium |
| [26] | Govee LAN: UDP 4001/4002/4003, per-device toggle | [openHAB Govee binding](https://www.openhab.org/addons/bindings/govee/) | undated | 2026-11-12 | high (two sources) |
| [27] | Govee LAN implementation | [govee crate (wez)](https://docs.rs/crate/govee/0.7.2/source/src/backend/local.rs) | undated | 2026-11-12 | medium |
| [28] | LIFX official LAN protocol; warning about excessive rates | [LIFX LAN docs](https://lan.developer.lifx.com/docs/introduction) | 2025-04-21 | 2026-11-12 | high |
| [29] | Yeelight TCP 55443, LAN Control toggle, multicast :1982 | [Yeelight Inter-Operation Spec](https://www.yeelight.com/download/Yeelight_Inter-Operation_Spec.pdf) | undated | 2026-11-12 | medium (snippet) |
| [30] | WLED JSON API; UDP sync groups | [WLED JSON API](https://kno.wled.ge/interfaces/json-api/) | undated | 2026-11-12 | high |
| [31] | WLED mDNS `_wled._tcp`; v16.0.1 | [wled/WLED](https://github.com/wled/WLED) | 2026-07-07 | 2026-11-12 | high |
| [32] | Magic Home Wi-Fi: UDP 48899 discovery, TCP 5577 | [lightinglibs/flux_led](https://github.com/lightinglibs/flux_led) | 2025-04-05 (1.2.0) | 2026-11-12 | high (ports) / low (no auth) |
| [33] | Nanoleaf pairing and token, HTTP :16021 | [nanoleafapi docs](https://nanoleafapi.readthedocs.io/en/latest/api.html) | undated | 2026-11-12 | medium |
| [34] | Hue link-button pairing, application key, CLIP v2 | [openHAB Hue binding (API v2)](https://www.openhab.org/addons/bindings/hue/doc/readme_v2.html) | undated | 2026-11-12 | medium |
| [35] | No official third-party API for Hue Bluetooth | [Philips Hue FAQ](https://www.philips-hue.com/en-us/explore-hue/faq/controls/how-to-connect-philips-hue-without-a-bridge) | undated | 2026-11-12 | medium |
| [36] | iOS Local Network privacy; multicast entitlement; NSBonjourServices | [Apple TN3179](https://developer.apple.com/documentation/technotes/tn3179-understanding-local-network-privacy) | 2026-10-06 | 2026-11-12 | high (verified by [73]) |
| [37] | Multicast entitlement needs Apple's permission | [Apple: multicast entitlement](https://developer.apple.com/documentation/bundleresources/entitlements/com.apple.developer.networking.multicast) | undated | 2026-11-12 | medium (snippet) |
| [38] | July 2026 entitlement denial; DTS reply | [Apple Developer Forums 838025](https://developer.apple.com/forums/thread/838025) | 2026-07 | 2026-11-12 | high |
| [39] | Approval times of 3–6 days | [Apple Developer Forums 683526](https://developer.apple.com/forums/thread/683526) | 2020–2022 (stale) | 2026-11-12 | low |
| [40] | Android 17 ACCESS_LOCAL_NETWORK, scope, picker path, shared group | [Android Developers: Local network permission](https://developer.android.com/privacy-and-security/local-network-permission) | live | 2026-11-12 | high (verified by [74]) |
| [41] | react-native-udp last release 2023-01-26; New Architecture issue ignored | [tradle/react-native-udp #259](https://github.com/tradle/react-native-udp/issues/259) | 2023-01-26 (last release) | 2026-11-12 | high |
| [42] | react-native-tcp-socket 6.4.3, legacy bridge | [Rapsssito/react-native-tcp-socket](https://github.com/Rapsssito/react-native-tcp-socket) | 2026-09-10 | 2026-11-12 | high (versions) / medium (interop) |
| [43] | react-native-zeroconf 0.17.7 / 1.0.0-rc.3 | [npm: react-native-zeroconf](https://registry.npmjs.org/react-native-zeroconf) | 2026-10-29 | 2026-11-12 | high |
| [44] | react-native-nitro-dgram stalled | [npm: react-native-nitro-dgram](https://registry.npmjs.org/react-native-nitro-dgram) | 2026-01-01 | 2026-11-12 | high |
| [45] | Expo SDK 55+ New Architecture only; interop "not perfect" | [Expo: New Architecture guide](https://docs.expo.dev/guides/new-architecture/) | 2026-11-02 | 2026-11-12 | high (verified by [77]) |
| [46] | Third-party iOS Matter commissioning; RN native bridge | [Espressif RainMaker: iOS native bridge](https://docs.rainmaker.espressif.com/docs/sdk/rainmaker-matter-sdk/Commissioning/ios-native-bridge) | undated (© 2026) | 2026-11-12 | high (verified by [76], with corrections) |
| [47] | MatterSupport unreliable for Thread devices | [Apple Developer Forums 781444](https://developer.apple.com/forums/thread/781444) | 2025–2026 | 2026-11-12 | medium |
| [48] | Android Matter commissioning to the app's own fabric | [Google Home APIs: Commissioning](https://developers.home.google.com/apis/android/commissioning) | undated | 2026-11-12 | high |
| [49] | ConsumerIrManager API, synchronous transmit, < 2 s | [Android: ConsumerIrManager](https://developer.android.com/reference/android/hardware/ConsumerIrManager) | live | 2026-11-12 | high (verified by [79]) |
| [50] | Phones listed with IR blasters | [Smartprix](https://us.smartprix.com/mobiles/with-ir_blaster) | 2026-11 | 2026-11-12 | low |
| [51] | No current Samsung with IR; IR brands | [fypower buyer guide](https://fypower.org/best-smartphones-with-ir-blasters/) | 2026-08 | 2026-11-12 | low |
| [52] | react-native-ir-manager last updated ~4 years ago | [ecosyste.ms: react-native-ir-manager](https://awesome.ecosyste.ms/projects/github.com%2Fdanielr18%2Freact-native-ir-manager) | ~2022 | 2026-11-12 | medium |
| [53] | 44-key remotes use NEC `0x00FFxxxx` | [44-key NEC code gist](https://gist.github.com/kylefmohr/e9811a6e34552efcd11783e44f0d8685) | undated | 2026-11-12 | medium |
| [54] | Code tables online are incomplete | [Arduino Forum](https://forum.arduino.cc/t/44-key-rgb-led-ir-controller-clone/1149943) | ~2023 | 2026-11-12 | low |
| [55] | POCO F7 has an IR blaster | [POCO F7 specs](https://www.po.co/global/poco-f7/specs/) | 2025 | 2026-11-12 | medium (snippet) |
| [56] | OnePlus 15 has an infrared port | [GSMArena: OnePlus 15](https://m.gsmarena.com/oneplus_15-ampp-14206.php) | 2025 | 2026-11-12 | medium |
| [57] | iPhone 17 spec list has no IR | [Apple: iPhone 17 specs](https://www.apple.com/iphone-17/specs/) | 2025-09 | 2026-11-12 | high (verified by [80]) |
| [58] | HA colour modes, single current mode, central conversion, feature flags | [Home Assistant: light entity](https://contributors.home-assistant.io/core/entities/light/) | live | 2026-11-12 | high |
| [59] | HA `assumed_state`: separate On and Off buttons | [Home Assistant: entity class](https://contributors.home-assistant.io/core/entities/entity-class/) | live | 2026-11-12 | high |
| [60] | Matter Color Control features (HS/EHUE/CL/XY/CT, mireds) | [matter-js: ColorControl.Feature](https://matter-js.github.io/docs/enums/_project_chip_matter_js.cluster.ColorControl.Feature.html) | undated (Matter 1.3) | 2026-11-12 | medium |
| [61] | Google ColorSetting model; `commandOnlyColorSetting` | [Google Home: ColorSetting trait](https://developers.home.google.com/cloud-to-cloud/traits/colorsetting) | undated | 2026-11-12 | high |
| [62] | HomeKit Lightbulb: On required, others optional | [HAP-NodeJS service definitions](https://github.com/homebridge/HAP-NodeJS/blob/master/src/lib/definitions/ServiceDefinitions.ts) | undated | 2026-11-12 | high |
| [63] | HA colour conversion algorithms (xy, gamut, RGBW, Kelvin) | [Home Assistant core: util/color.py](https://github.com/home-assistant/core/blob/dev/homeassistant/util/color.py) | live | 2026-11-12 | high |
| [64] | Hue gamut triangles; closest-point rule | [Philips Hue SDK: RGB to xy note](https://github.com/PhilipsHue/PhilipsHueSDK-iOS-OSX/blob/00187a3db88dedd640f5ddfa8a474458dff4e1db/ApplicationDesignNotes/RGB%20to%20xy%20Color%20conversion.md) | archived 2021-08 | 2026-11-12 | high |
| [65] | Hue: at most 3 concurrent requests; 429/503 backoff | [home-assistant-libs/aiohue](https://github.com/home-assistant-libs/aiohue/blob/main/aiohue/v2/__init__.py) | live | 2026-11-12 | high |
| [66] | Hue: 50 ms interval, 3 concurrent streams, GO_AWAY recycling | [openHAB Clip2Bridge.java](https://github.com/openhab/openhab-addons/blob/main/bundles/org.openhab.binding.hue/src/main/java/org/openhab/binding/hue/internal/connection/Clip2Bridge.java) | live | 2026-11-12 | high |
| [67] | Hue ~10 light/s and 1 group/s soft cap; grouped_light single write | [Jentic: openhue CLIP v2 listing](https://jentic.com/apis/openhue.io/openhue.md) | undated | 2026-11-12 | low |
| [68] | LIFX at most 20 messages/s per device | [LIFX protocol docs (mirror)](https://github.com/moritzs/lifx-protocol-docs) | undated (stale) | 2026-11-12 | medium |
| [69] | Popcorn effect from per-bulb commands | [SmartThings Community](https://community.smartthings.com/t/hue-delay-between-bulbs/62888) | ~2016 (stale) | 2026-11-12 | low |
| [70] | Expo SDK 58, RN 0.88, development builds, Expo Modules 2.0 beta | [Expo SDK 58 changelog](https://expo.dev/changelog/sdk-58) | 2026-10-15 | 2026-11-12 | high |
| [71] | Google Matter ColorControl feature flags | [Google Home: ColorControlTrait.Feature](https://developers.home.google.com/reference/kotlin/com/google/home/matter/standard/ColorControlTrait.Feature) | undated | 2026-11-12 | medium |
| [72] | Local Key via cloud-assisted sign-in, no developer account; 1-month trial | [make-all/tuya-local](https://github.com/make-all/tuya-local) | live | 2026-11-12 | high |
| [73] | Independent: iOS broadcast needs the multicast entitlement | [Nabto iOS notes](https://www.nabto.com/docs/developer/platforms/ios/ios145.html) | ~2023 | 2026-11-12 | medium |
| [74] | Independent: Android 17 ACCESS_LOCAL_NETWORK | [Help Net Security](https://www.helpnetsecurity.com/?p=359446) | 2026-02 | 2026-11-12 | medium |
| [75] | Independent: HA Tuya integration uses the official SDK, user code + QR | [Home Assistant: Tuya integration](https://www.home-assistant.io/integrations/tuya/) | live | 2026-11-12 | high |
| [76] | Independent: MatterSupport for third-party ecosystems; iOS 16.1+ | [Apple: Adding Matter support to your ecosystem](https://developer.apple.com/documentation/mattersupport/adding-matter-support-to-your-ecosystem) | live | 2026-11-12 | high |
| [77] | Independent: RN 0.82 runs entirely on the New Architecture | [React Native 0.82 release post](https://reactnative.dev/blog/2025/10/08/react-native-0.82) | 2025-10 | 2026-11-12 | high |
| [78] | Independent: ASK limits CoreBluetooth to ASK-paired accessories | [Apple WWDC24 "Meet AccessorySetupKit"](https://developer.apple.com/videos/play/wwdc2024/10203/) | 2024-06 | 2026-11-12 | high |
| [79] | Independent: ConsumerIrManager not deprecated in AOSP main | [AOSP ConsumerIrManager.java](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/java/android/hardware/ConsumerIrManager.java) | live | 2026-11-12 | high |
| [80] | Independent: no iPhone has shipped with an IR blaster | [MobilityArena](https://mobilityarena.com/how-to-get-an-ir-blaster-on-my-phone/) | ≥2023 | 2026-11-12 | medium (snippet) |
| [81] | Retelling of the Zengge write-up; one capture looked AES-encrypted | [Habr](https://habr.com/ru/articles/781118) | 2023-12 | 2026-11-12 | low (not independent of [6]) |

## 8. Staleness map

The re-check dates below were computed with `recon_kit.py staleness`, using these freshness windows: versions 1 month; ecosystem health and pricing 6 months; platform constraints, protocols, device availability and performance 12 months; patterns 24 months.

| Re-check by | Class | Claim |
|---|---|---|
| **2025-06-01 (already stale)** | platform-constraint | AccessorySetupKit limits CoreBluetooth scanning [19][78]. The source is WWDC24; check it against current iOS docs |
| **2026-09-01 (already stale)** | device-availability | Which phones have IR; no iPhone IR [55][56][57] |
| 2026-11-15 | version-compatibility | Expo SDK 58 / RN 0.88 is current [70] |
| 2026-11-29 | version-compatibility | react-native-zeroconf is active and supports the New Architecture [43] |
| 2026-12-02 | version-compatibility | Expo SDK 55+ / RN 0.82+ are New Architecture only [45][77] |
| 2026-12-05 | version-compatibility | react-native-ble-manager 13.0.0 is native New Architecture [10] |
| 2026-12-12 | version-compatibility | ble-plx v4 is unmerged; npm latest is 3.5.1 [12] |
| 2027-01-01 | protocol | Zengge LEDnetWF protocol [6] |
| 2027-03-10 | ecosystem-health | react-native-tcp-socket is maintained but legacy [42] |
| 2027-05-12 | ecosystem-health / licensing | react-native-udp is unmaintained [41]; Tuya SDK pricing [23] |
| 2027-07-01 | platform-constraint | The 2026 multicast entitlement denial [38] |
| 2027-10-06 | platform-constraint | The iOS multicast entitlement rule [36][73] |
| 2027-11-12 | platform / protocol / performance | Tuya key routes [21][72], Android 17 permission [40][74], iOS Matter [46][76], ConsumerIrManager [49][79], ELK protocol [2], Govee LAN [26][27], Hue limits [65][66][67], BLE connection floor [17] |
| 2028-11-12 | pattern | HA colour-mode capability model [58] |

The earliest re-check is **2025-06-01**, already past, for the AccessorySetupKit claim. The next date that matters is **2026-11-15** (Expo SDK version). The library versions cluster in Nov–Dec 2026, so run a Refresh before architecture work starts.
