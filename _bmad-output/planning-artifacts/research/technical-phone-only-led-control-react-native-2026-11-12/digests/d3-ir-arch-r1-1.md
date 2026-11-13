# D3 IR + architecture — round 1 digest

Run notes: 6 sources read in depth (HA light entity docs, Android ConsumerIrManager reference, smartprix IR listing, Hue core-concepts page [login-walled, no content], Expo changelog index, Expo SDK 58 post). Everything else below comes from search-result snippets that themselves state the fact, and is marked that way. About 19 tool calls used.

## Findings

### Q1 — Android IR (ConsumerIrManager, devices, RN modules, remote protocol)

- claim: android.hardware.ConsumerIrManager is still in the current Android API reference (added in API level 19). The page shows no deprecation notice and exposes exactly three methods: hasIrEmitter(), getCarrierFrequencies() and transmit(int carrierFrequency, int[] pattern).
  source: https://developer.android.com/reference/android/hardware/ConsumerIrManager
  publisher: Google / Android Developers
  pub_date: unknown (living reference page)
  accessed: 2026-11-12
  confidence: high
  class: platform-constraint
  (Single source, but it is the primary one. The page does not say which newest API level it was checked against.)

- claim: ConsumerIrManager needs the PackageManager.FEATURE_CONSUMER_IR system feature (checked with hasSystemFeature). transmit() is synchronous ("when it returns the pattern has been transmitted"), "only patterns shorter than 2 seconds will be transmitted", and the pattern is alternating on/off durations in microseconds at a carrier frequency given in Hz.
  source: https://developer.android.com/reference/android/hardware/ConsumerIrManager
  publisher: Google / Android Developers
  pub_date: unknown
  accessed: 2026-11-12
  confidence: high
  class: platform-constraint
  (Design implication: transmit blocks, so a native module must call it off the JS/UI thread. One NEC frame is about 108 ms, well under the 2 s cap.)

- claim: A phone-spec aggregator, updated around 2026-11-13, lists 239 phones with an IR blaster. The first page shows Xiaomi/Poco (Poco F7, Jul 2025; Xiaomi 18 / 18 Pro Max / 16 Ultra marked upcoming), OnePlus 16 series (upcoming or rumored), iQOO 16 / Vivo X300 Pro, OPPO Find X10 / X10 Pro Max, Huawei (Pura 90s Pro Max, Pura X Max, Mate XT 2, upcoming) and Nubia Red Magic 11 Pro (Feb 2026). It shows no Honor, Motorola, Infinix or Tecno models.
  source: https://us.smartprix.com/mobiles/with-ir_blaster?ref=country_redir
  publisher: Smartprix (aggregator)
  pub_date: 2026-11-13 (as shown on the page)
  accessed: 2026-11-12
  confidence: low
  class: device-availability
  (The aggregator mixes rumored and upcoming phones, and its model names conflict with other listings, so treat it only as a rough signal of which brands still ship IR.)

- claim: Search-result snippets from 2026 buyer guides say that no current Samsung Galaxy phone (including the Z Fold line) or iPhone has an IR blaster. They name Xiaomi/Redmi/Poco as the most consistent IR brand (for example Redmi Note 14 4G, Xiaomi 14T, Poco X7 Pro), plus OnePlus 15/15R, Red Magic 11 Pro and Unihertz rugged phones.
  source: https://fypower.org/best-smartphones-with-ir-blasters/ ; https://gadgets.beebom.com/mobile-list/ir-blaster-phones?page=2
  publisher: fypower.org; Beebom (aggregators)
  pub_date: 2026-08 (fypower title); unknown (Beebom)
  accessed: 2026-11-12
  confidence: low
  class: device-availability
  (Snippet-level only. The brand lists differ between aggregators.)

- claim: The best-known React Native IR package is danielr18/react-native-ir-manager. It is Android-only, wraps ConsumerIrManager, and includes a helper that converts Pronto Hex to a pattern. A cached listing shows it was last updated about 4 years ago, so it looks unmaintained.
  source: https://awesome.ecosyste.ms/projects/github.com%2Fdanielr18%2Freact-native-ir-manager ; https://npmjs.com/package/react-native-ir-manager
  publisher: ecosyste.ms; npm
  pub_date: unknown
  accessed: 2026-11-12
  confidence: medium
  class: ecosystem-health
  (Snippet-level. Release history was not checked.)

- claim: Smaller alternatives exist, all Android-only and low-adoption: react-native-infrared-interface (built with create-react-native-library, has hasIrEmitter, getCarrierFrequencies and transmit); lucas-react-native-ir-manage (a fork described as updated for the New Architecture, very few downloads); react-native-ir (old, needs react-native link and manual Gradle edits); and sachin-acharya-projects/irmanager (a small Expo Module wrapper whose docs say the app must run as a local or development build, not Expo Go).
  source: https://pkgstats.com/pkg:react-native-infrared-interface ; https://www.pkgstats.com/pkg:lucas-react-native-ir-manage ; https://npmjs.com/package/react-native-ir ; https://awesome.ecosyste.ms/projects/github.com%2Fsachin-acharya-projects%2Firmanager
  publisher: pkgstats / npm / ecosyste.ms
  pub_date: unknown
  accessed: 2026-11-12
  confidence: low
  class: ecosystem-health
  (No package found that is clearly maintained and works with the New Architecture. Writing an in-house Expo Module that wraps the three methods looks like the lower-risk path.)

- claim: iodn/android-ir-blaster is a native Android app, not an RN library, that supports external USB IR dongles (discovery, permission, bulk transfer). This is a reference for phones without a built-in emitter, but a dongle counts as extra hardware.
  source: https://github.com/iodn/android-ir-blaster
  publisher: iodn (GitHub)
  pub_date: unknown
  accessed: 2026-11-12
  confidence: low
  class: landscape

- claim: Cheap 44-key RGB strip remotes are widely documented as using NEC. Community code tables exist (an Electro-Tech Online thread of "NEC IR protocol 'Command' codes" for 44-key remotes, and GitHub gists of 44-button codes). Captured codes take the form 0x00FFxxxx (address 0x00 with its complement), for example 00FF4CB3 = Power and 00FF807F = Bright+ on one 5050-strip remote, captured with a 40 kHz carrier.
  source: https://electro-tech-online.com/threads/44-key-ir-remote-nec-command-codes.161412 ; https://gist.github.com/kylefmohr/e9811a6e34552efcd11783e44f0d8685 ; https://gist.github.com/kilip/caa23120cd41c3e7e4d38ca3d30cc664
  publisher: Electro-Tech Online forum; GitHub gist authors
  pub_date: unknown
  accessed: 2026-11-12
  confidence: medium (NEC is the protocol) / low (any one code table is universal)
  class: protocol

- claim: An Arduino forum thread warns that "many projects online feature incomplete codes for the 44 key IR RGB LED remote/controller boxes", and sellers use the "44-key" label for different button layouts. A fixed built-in code table will not cover every controller, so the app needs a way to pick a code set or learn codes.
  source: https://forum.arduino.cc/t/44-key-rgb-led-ir-controller-clone/1149943
  publisher: Arduino Forum
  pub_date: 2023 (approx., thread ID era; not confirmed)
  accessed: 2026-11-12
  confidence: low
  class: protocol
  (Phones have IR transmitters only, no receivers, so the app cannot learn codes itself. That last point is inference, not evidenced this run.)

### Q2 — iOS IR

- claim: 2026 buyer-guide snippets say no current iPhone has an IR blaster. No Apple source was retrieved this run confirming the absence of IR hardware or of any public IR API.
  source: https://fypower.org/best-smartphones-with-ir-blasters/
  publisher: fypower.org (aggregator)
  pub_date: 2026-08
  accessed: 2026-11-12
  confidence: low (single aggregator source; matches the widely held belief that iOS has no IR transmit API, which is not verified here)
  class: device-availability

### Q3 — How established controllers model a "light"

- claim: Home Assistant models light capability as a required set of supported_color_modes drawn from ONOFF, BRIGHTNESS, COLOR_TEMP, HS, XY, RGB, RGBW, RGBWW and WHITE (UNKNOWN can appear only as the current color_mode). ONOFF and BRIGHTNESS must each be the only mode in the set, WHITE must be paired with at least one color mode, and invalid combinations raise an error when state is written.
  source: https://contributors.home-assistant.io/core/entities/light/ (redirected from developers.home-assistant.io/docs/core/entity/light/)
  publisher: Home Assistant
  pub_date: unknown (living doc; links a core commit)
  accessed: 2026-11-12
  confidence: high
  class: pattern

- claim: In Home Assistant, a light reports a single current color_mode, which is required while the light is on. Only the color attribute matching that mode is set by the integration, and HA fills in the other formats from it. Value ranges are brightness 1–255, hs (hue 0–360, saturation 0–100), xy (CIE), rgb/rgbw/rgbww (0–255 per channel) and color_temp_kelvin with min_color_temp_kelvin and max_color_temp_kelvin (defaults 2000 K and 6535 K).
  source: https://contributors.home-assistant.io/core/entities/light/
  publisher: Home Assistant
  pub_date: unknown
  accessed: 2026-11-12
  confidence: high
  class: pattern

- claim: Home Assistant separates optional behaviours from color modes using LightEntityFeature flags: EFFECT (effect_list plus effect, with "off" meaning no effect), FLASH (short or long) and TRANSITION (seconds). Parameters for unsupported features are removed before driver methods are called. If brightness or white resolves to 0, HA calls turn_off instead.
  source: https://contributors.home-assistant.io/core/entities/light/
  publisher: Home Assistant
  pub_date: unknown
  accessed: 2026-11-12
  confidence: high
  class: pattern

- claim: The Matter Color Control cluster (0x0300) advertises its capabilities as a FeatureMap bitmap: bit 0 HS, bit 1 EHUE (requires HS), bit 2 CL color loop (requires EHUE), bit 3 XY and bit 4 CT. Every feature is optional, and the ColorTemperatureMireds attribute (0x0007) exists only when CT is supported. Matter expresses color temperature in mireds, whereas HA uses kelvin.
  source: https://matter-js.github.io/docs/enums/_project_chip_matter_js.cluster.ColorControl.Feature.html ; https://pigweed.googlesource.com/third_party/github/project-chip/connectedhomeip/+/29cfb7f1201f8fff2fe54b3dc8b61b50c579d4b4/data_model/clusters/ColorControl.xml
  publisher: matter-js project; Project CHIP / connectedhomeip (CSA reference implementation)
  pub_date: unknown (matter-js cites Matter 1.3 §3.2.5; XML is cluster revision 6)
  accessed: 2026-11-12
  confidence: medium (snippet-level; the CSA spec itself was not read)
  class: pattern

- claim: Google's Home APIs expose the Matter ColorControlTrait.Feature as boolean flags (hueAndSaturation, enhancedHue, colorLoop, xy, colorTemperature), so Google's own SDK uses the same capability-flags-per-device pattern as Matter.
  source: https://developers.home.google.com/reference/kotlin/com/google/home/matter/standard/ColorControlTrait.Feature
  publisher: Google Home Developers
  pub_date: unknown
  accessed: 2026-11-12
  confidence: medium (snippet-level)
  class: pattern

- claim (synthesis, inference from the three models above, not a sourced fact): Across HA, Matter and Google the shared pattern is (a) on/off as the base capability, (b) a separate brightness or level capability, (c) a set of color capabilities from which the device declares a subset, (d) a single "current mode" in state, and (e) optional features such as effects and transitions as separate flags. A driver interface should declare capabilities statically per device and receive commands already converted to a format it supports. HA does this conversion centrally rather than in each driver.
  source: (synthesis of the HA, matter-js and Google sources above)
  publisher: n/a
  pub_date: n/a
  accessed: 2026-11-12
  confidence: medium
  class: pattern

### Q4 — One preset across heterogeneous devices

- claim: Home Assistant core converts colors centrally. Drivers "never receive a color mode" and get at most one color attribute, "always in a format your light supports". The documented example converts hs_color to rgb_color for an RGB-only light. Percentage or step brightness is normalised to brightness 1–255.
  source: https://contributors.home-assistant.io/core/entities/light/
  publisher: Home Assistant
  pub_date: unknown
  accessed: 2026-11-12
  confidence: high
  class: pattern
  (The page does not describe how it converts RGB to RGBW or RGBWW, or color temperature to RGB. Not verified.)

- claim: For RGB-family modes, HA treats perceived output as the brightness attribute combined with the channel values. Its example, brightness 128 with rgb (192, 64, 32), gives about 38%, so a preset has to define brightness and color together to look the same across devices.
  source: https://contributors.home-assistant.io/core/entities/light/
  publisher: Home Assistant
  pub_date: unknown
  accessed: 2026-11-12
  confidence: high
  class: pattern

### Q5 — Groups, rate limits, sync

- claim: Philips Hue's developer guidance, as quoted in search snippets, recommends at most about 10 commands per second to /lights and 1 per second to /groups. A forum post quoting the same guidance adds a 100 ms gap between calls.
  source: https://developers.meethue.com/?p=497 (snippet); https://developers.meethue.com/develop/get-started-2/core-concepts/ (fetched: login wall, no content)
  publisher: Signify / Philips Hue Developer Program
  pub_date: unknown; STALE (CLIP v1-era guidance)
  accessed: 2026-11-12
  confidence: medium
  class: performance
  (Only one source per number. The primary page is behind a login and could not be read.)

- claim: openHAB Hue binding developers state the bridge's CLIP v2 API is rate-limited to about 20 requests per second. A third-party troubleshooting site says the bridge returns HTTP 429 when the limit is exceeded and that the limit is not per application.
  source: https://community.openhab.org/t/philips-hue-clip-2-api-v2-discussion-thread/142111?page=20 ; https://hue-log-website.pages.dev/help/philips-hue-bridge-api-429-too-many-requests.html
  publisher: openHAB community; unknown third party
  pub_date: unknown
  accessed: 2026-11-12
  confidence: low
  class: performance
  (Single source for each number, and neither is official. This conflicts with the v1 figures above.)

- claim: The LIFX LAN protocol's recommended maximum is 20 messages per second to a single device. The LIFX protocol-docs repository states it, Go and Python LIFX libraries encode or repeat it (controlifx MessageRate = 20; lifx-async), and LIFX's official LAN site warns that "excessive message rates … may result in unexpected device or protocol message behavior".
  source: https://github.com/moritzs/lifx-protocol-docs (mirror of LIFX/lifx-protocol-docs) ; https://beta.pkg.go.dev/github.com/yath/controlifx ; https://djelibeybi.github.io/lifx-async/user-guide/advanced-usage ; https://lan.developer.lifx.com/
  publisher: LIFX (protocol docs) plus library authors
  pub_date: unknown; likely STALE (legacy protocol-docs repo)
  accessed: 2026-11-12
  confidence: medium
  class: performance
  (Several domains, but they all trace back to one upstream LIFX document.)

- claim: A Particle community user reported sending about 20 messages per second to each of 1, 2 or 10 LIFX bulbs without problems, in a thread about UDP trouble when targeting more than 10 addresses. This hints that fan-out to many devices hits a separate limit.
  source: https://community.particle.io/t/strange-udp-issue-with-more-than-10-target-ip-addresses/55119
  publisher: Particle community
  pub_date: unknown
  accessed: 2026-11-12
  confidence: low
  class: performance

### Q6 — Expo vs bare RN

- claim: The latest Expo SDK is SDK 58, released 2026-10-15 and shipping React Native 0.88 (up from 0.86 in SDK 57). Recent cadence: SDK 56 on 2026-05-21, SDK 57 on 2026-06-30, SDK 58 beta on 2026-09-15.
  source: https://expo.dev/changelog ; https://expo.dev/changelog/sdk-58
  publisher: Expo
  pub_date: 2026-10-15
  accessed: 2026-11-12
  confidence: high
  class: version-compatibility
  (Single publisher, but primary. Within the 1-month freshness bar.)

- claim: In SDK 58, Expo Go is being replaced by a new build that will support only SDK 58. Until it ships, create-expo-app creates SDK 57 projects, and SDK 58 is used through `--template default@sdk-58` with a development build. Since 2026-09-03, running projects in Expo Go requires a login.
  source: https://expo.dev/changelog/sdk-58 ; https://expo.dev/changelog
  publisher: Expo
  pub_date: 2026-10-15; 2026-09-03
  accessed: 2026-11-12
  confidence: high
  class: version-compatibility

- claim: Expo's New Architecture guide states "SDK 55 and later run entirely on the New Architecture. The New Architecture is always enabled and cannot be disabled". SDK 54 was the last SDK with Legacy Architecture support, the newArchEnabled app.json option was removed, and React Native 0.82 removed the option to disable the New Architecture. SDK 55 shipped RN 0.83.
  source: https://docs.expo.dev/guides/new-architecture ; https://expo.dev/changelog/sdk-55-beta
  publisher: Expo
  pub_date: unknown (docs); SDK 55 beta changelog date not retrieved
  accessed: 2026-11-12
  confidence: high (two primary Expo pages, both snippet-level)
  class: version-compatibility

- claim: SDK 58 ships "Expo Modules 2.0" in beta on iOS and Android. Modules can be written as ordinary Swift or Kotlin classes with annotations instead of the existing DSL, and iOS synchronous calls are 2.5–5.6x faster than the DSL. The iOS macros were experimental in SDK 57.
  source: https://expo.dev/changelog/sdk-58
  publisher: Expo
  pub_date: 2026-10-15
  accessed: 2026-11-12
  confidence: high
  class: version-compatibility

- claim: SDK 58 keeps Continuous Native Generation (prebuild) and config plugins as the path for native changes. Prebuild now generates SceneDelegate.swift, because iOS 27 requires the scene life cycle. Release builds enable R8 by default. Teams must create a new development build after upgrading. This confirms the custom-native-code path is dev build + prebuild/config plugins, not Expo Go.
  source: https://expo.dev/changelog/sdk-58
  publisher: Expo
  pub_date: 2026-10-15
  accessed: 2026-11-12
  confidence: high
  class: version-compatibility

## Leads worth chasing

- Expo releases look unusually frequent (56 in May, 57 in June, 58 in October 2026). Check whether SDK 57 was a short-lived release and what React Native 0.86 and 0.88 change for BLE and UDP libraries (react-native-ble-plx, react-native-udp) on the New Architecture-only stack.
- Expo Modules 2.0 is in beta. Decide whether a v1 IR module should use the stable DSL Expo Modules API or the 2.0 annotation style.
- The Hue rate limits conflict: 10/s for lights and 1/s for groups (v1 docs) versus about 20 req/s (openHAB, CLIP v2). Read the CLIP v2 docs with a developer login, or the source of aiohue (Home Assistant's Hue library) and openHAB's Hue binding for coded throttles.
- Do any IR packages built on the old architecture work through the New Architecture interop layer on RN 0.88? Not verified. Building our own Expo Module seems safer.
- 44-key remote codes: the Electro-Tech table is an image attachment. The kylefmohr and kilip gists may hold full tables, and the WLED IR docs (Adafruit PDF) list remote types WLED supports (24-key, 40-key, 44-key), which could serve as a curated code source.
- One capture shows a 40 kHz carrier for a strip remote, while NEC is normally 38 kHz. Check which carriers ConsumerIrManager accepts on target phones (getCarrierFrequencies) and whether cheap receivers tolerate 38 kHz.
- Aggregator device lists disagree with each other (Xiaomi "16 Ultra" listed as upcoming alongside "Xiaomi 17/18"; Honor absent). Check a few specific 2025–2026 models (Redmi Note 14/15, Poco F7, Xiaomi 15T, Honor 400/Magic 7) on manufacturer spec pages.

## Looked for, not found

- Any data on the share of phones (installed base or shipments) with an IR emitter. Only listing counts found (smartprix: 239 phones), no market share.
- Honor phones with IR blasters in 2024–2026. None in the sources retrieved. Huawei appears only as "upcoming" on one aggregator.
- An Apple primary source confirming that no iPhone has IR hardware and that iOS has no IR API (not searched for lack of budget; only an aggregator snippet).
- HomeKit/Homebridge Lightbulb service characteristics (On, Brightness, Hue, Saturation, ColorTemperature) and Google Home device traits (OnOff, Brightness, ColorSetting). Not retrieved this run.
- Home Assistant's documentation of assumed_state or optimistic versus confirmed state. The light entity page does not cover it; the base entity page was not read.
- How Home Assistant converts RGB to RGBW/RGBWW and color temperature to RGB, and how Hue gamuts (A/B/C) are handled. The light page does not document the algorithms (color_util source not read).
- Documented "popcorn effect" mitigation or measured group sync latency for app-side fan-out.
- Current official Hue CLIP v2 rate-limit documentation (behind the developer login).
- An actively maintained React Native IR transmit library that works with the New Architecture, with recent releases.
