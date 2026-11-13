# D2 Wi-Fi — round 1 digest

Budget note: 18 research tool calls. Sources read in depth: tinytuya README, Tuya VAS SDK listing, Espressif RainMaker iOS Native Bridge, Google Home APIs Android Commissioning, Apple TN3179 (JSON), LIFX LAN intro. Two in-depth reads failed (Govee manual is JS-rendered; Hue developer docs are behind a login). Search-snippet-only claims are marked low/medium and say "(snippet)".

## Findings

### Q1 — Tuya / Smart Life Wi-Fi lights

- claim: Tuya Wi-Fi devices can be controlled on the LAN without the cloud (TCP 6668 for control; UDP 6666/6667/7000 for discovery broadcasts), but every device needs its per-device Local Key to be controlled ("All devices will require a device Local_Key to control the device"). Protocol 3.1 devices can report status without the key.
  source: https://github.com/jasonacox/tinytuya
  publisher: Jason Cox (tinytuya maintainer)
  pub_date: unknown (living README)
  accessed: 2026-11-12
  confidence: high
  class: protocol

- claim: The Local Key is obtained through the Tuya IoT Cloud Platform: the developer creates a Tuya developer account and a Cloud Project, links the Smart Life app account by QR code, subscribes to the "IoT Core" and "Authorization" APIs, then pulls keys with an API ID, secret and region (cn/us/us-e/eu/eu-w/sg/in).
  source: https://github.com/jasonacox/tinytuya
  publisher: Jason Cox (tinytuya maintainer)
  pub_date: unknown
  accessed: 2026-11-12
  confidence: high
  class: platform-constraint

- claim: The IoT Core subscription that lets you fetch keys is a free trial that expires after one month. After that, keys for new devices can't be fetched until the user extends the trial (1, 3 or 6 months), and Tuya may not allow the trial edition to be selected again.
  source: https://github.com/jasonacox/tinytuya
  publisher: Jason Cox (tinytuya maintainer)
  pub_date: unknown
  accessed: 2026-11-12
  confidence: medium (single source; Tuya policy can change)
  class: platform-constraint

- claim: The Local Key changes every time a device is removed and re-added in the Tuya/Smart Life app, so a re-pair means fetching the key again.
  source: https://github.com/jasonacox/tinytuya
  publisher: Jason Cox (tinytuya maintainer)
  pub_date: unknown
  accessed: 2026-11-12
  confidence: high
  class: protocol

- claim: tinytuya supports Tuya local protocol versions 3.1, 3.2, 3.3, 3.4 and 3.5. All use AES, and v3.5 needs AES-GCM (pyaes can't handle it). The README doesn't document the per-version cipher modes or the 3.4/3.5 session-key negotiation.
  source: https://github.com/jasonacox/tinytuya
  publisher: Jason Cox (tinytuya maintainer)
  pub_date: unknown
  accessed: 2026-11-12
  confidence: high (versions); low (any cipher detail beyond "AES; GCM for 3.5")
  class: protocol

- claim: Battery-powered Tuya devices sleep, don't keep a local network connection, and can't be controlled locally. Some older-firmware devices may not work locally until their firmware is updated in the official app.
  source: https://github.com/jasonacox/tinytuya
  publisher: Jason Cox (tinytuya maintainer)
  pub_date: unknown
  accessed: 2026-11-12
  confidence: high
  class: platform-constraint

- claim: Tuya's official "Self-Developed Smart Life App" SDK has a free Development edition (100,000 base-SDK calls per month, at most 100 registered users, meant for debugging and not for scaled use) and a paid annual Business/Official edition (unlimited calls and users, independent domain), listed at CNY 33,500 per 12 months. When it expires, the shipped app keeps working, but SDK updates and platform services stop.
  source: https://www.tuya.com/vas/commodity/SDK_APP_CREATE
  publisher: Tuya Inc.
  pub_date: unknown (page footer © 2026)
  accessed: 2026-11-12
  confidence: medium (the price on the page isn't explicitly tied to an edition; it conflicts with the snippet below)
  class: licensing

- claim: (snippet) Tuya's app-SDK marketing page lists the Development edition as free with up to 1 million cloud requests per month, and the Official edition at $5,000 for the first year and $2,000 for renewal.
  source: https://tuya.com/platform/appdev/app-sdk
  publisher: Tuya Inc.
  pub_date: unknown
  accessed: 2026-11-12
  confidence: low (snippet only; the call quota conflicts with the VAS page's 100,000)
  class: licensing

- claim: Tuya-made Matter-certified Wi-Fi lights exist: the CSA certified-product database lists Tuya Wi-Fi RGBCW bulbs, Wi-Fi strip lights (RGBC) and "Wi-Fi Extended Color Light" products supporting on/off, brightness, color temperature and color control.
  source: https://csa-iot.org/csa_product/wi-fi-extended-color-light/ (also https://csa-iot.org/csa_product/tuya-wi-fi-strip-light-rgbc/)
  publisher: Connectivity Standards Alliance
  pub_date: unknown
  accessed: 2026-11-12
  confidence: medium (snippet of a primary registry; what share of new Tuya SKUs ship with Matter is unknown)
  class: landscape

### Q2 — Branded local APIs

- claim: Govee LAN control is unauthenticated plaintext UDP. Discovery is a multicast scan to 239.255.255.250:4001, replies arrive on port 4002, and control and status go unicast to the device on port 4003. LAN Control has to be switched on per device in the Govee Home app.
  source: https://www.openhab.org/addons/bindings/govee/ ; https://docs.rs/crate/govee/0.7.2/source/src/backend/local.rs
  publisher: openHAB; govee Rust crate (wez)
  pub_date: unknown
  accessed: 2026-11-12
  confidence: medium (two secondary sources agree; the official manual at app-h5.govee.com/user-manual/wlan-guide wouldn't render)
  class: protocol

- claim: (snippet) Some newer Govee models ignore the multicast scan but answer a directed subnet broadcast (x.x.x.255:4001). Only one process per host can bind port 4002.
  source: https://www.openhab.org/addons/bindings/govee/ (and the lasswellt/govee-homeassistant v2026.6.23 release note via newreleases.io)
  publisher: openHAB / lasswellt
  pub_date: 2026-06 (release note)
  accessed: 2026-11-12
  confidence: low
  class: protocol

- claim: LIFX publishes an official LAN Protocol for third-party developers. It is UDP over IP with a packed binary encoding, and the docs warn that "excessive message rates" can cause unexpected device behavior without giving a numeric limit on the intro page.
  source: https://lan.developer.lifx.com/docs/introduction
  publisher: LIFX Inc.
  pub_date: 2025-04-21
  accessed: 2026-11-12
  confidence: high (official); stale for "landscape", fine for "pattern"
  class: protocol

- claim: (snippet) Yeelight LAN control uses JSON-RPC over TCP port 55443, and bulbs advertise themselves by SSDP-style multicast to 239.255.255.250:1982. Port 55443 stays closed until the owner turns on "LAN Control" (on some firmware, "Developer Mode") in the Yeelight app. The channel isn't encrypted. Yeelight's official reference is the "Inter-Operation Spec" PDF, which one community reference calls incomplete.
  source: https://www.yeelight.com/download/Yeelight_Inter-Operation_Spec.pdf ; https://numberonebot.github.io/yeelight-client/protocol
  publisher: Yeelight; community protocol reference
  pub_date: unknown
  accessed: 2026-11-12
  confidence: medium (snippets; PDF not read)
  class: protocol

- claim: (snippet) Some Xiaomi-branded Yeelight-family bulbs lost LAN access in firmware updates, and the firmware couldn't be reverted.
  source: https://numberonebot.github.io/yeelight-client/devices
  publisher: community reference
  pub_date: unknown
  accessed: 2026-11-12
  confidence: low (single source)
  class: platform-constraint

- claim: Philips Hue API v2 developer docs (core concepts, rate limits) sit behind a Hue Developer Program login and couldn't be read. Third-party sources say the bridge soft-caps about 10 light commands per second and about 1 grouped_light command per second per room or zone. node-hue-api throttles single lights at 1 per 60 ms and groups at 1 per second, "as Signify state in the documentation".
  source: https://jentic.com/apis/openhue.io/openhue ; https://www.github.com/peter-murray/node-hue-api/issues/210
  publisher: Jentic (aggregator); node-hue-api maintainer (Peter Murray)
  pub_date: unknown
  accessed: 2026-11-12
  confidence: medium (two sources agree, but neither is Signify's own page)
  class: performance

- claim: (snippet) An openHAB forum thread reports a bridge-wide limit of about 20 requests per second for the Hue CLIP v2 API.
  source: https://community.openhab.org/t/philips-hue-clip-2-api-v2-discussion-thread/142111?page=20
  publisher: openHAB community
  pub_date: unknown
  accessed: 2026-11-12
  confidence: low (single source; the scope relative to the 10/s and 1/s figures isn't reconciled)
  class: performance

### Q3 — Matter

- claim: On iOS, a third-party app can commission Matter devices into its own fabric (not only Apple Home). MatterSupport (MatterAddDeviceRequest plus a MatterAddDeviceExtensionRequestHandler extension) runs the system discovery and Wi-Fi/Thread network-setup UI, and the Matter framework's MTRDeviceController then does PASE, commissioning, NOC install and cluster control. It needs iOS 16.4+, the com.apple.developer.matter.allow-setup-payload entitlement, an App Group shared between the app and the extension, and an extension target of type com.apple.matter.support.extension.device-setup.
  source: https://docs.rainmaker.espressif.com/docs/sdk/rainmaker-matter-sdk/Commissioning/ios-native-bridge
  publisher: Espressif Systems
  pub_date: unknown (footer © 2026)
  accessed: 2026-11-12
  confidence: high (vendor doc with a shipping app; Apple's own docs not read this round)
  class: platform-constraint

- claim: Espressif's RainMaker app drives Matter commissioning from React Native through a custom native module (ESPMatterModule via NativeModules, events via RCTEventEmitter), with the core commissioning in native Swift/Objective-C. This is precedent for an app-specific bridge, not a reusable RN Matter library.
  source: https://docs.rainmaker.espressif.com/docs/sdk/rainmaker-matter-sdk/Commissioning/ios-native-bridge
  publisher: Espressif Systems
  pub_date: unknown
  accessed: 2026-11-12
  confidence: high
  class: pattern

- claim: (snippet) Developers report that the MatterSupport flow is unreliable for Thread devices (blamed on HomeKit's Thread management in iOS 18) and still failing on iOS 26.2. In the same thread, an Apple DTS reply says the problem went away when the extension was removed and the Matter framework was used directly in developer mode.
  source: https://developer.apple.com/forums/thread/781444
  publisher: Apple Developer Forums (developers + DTS)
  pub_date: 2025–2026 (thread spans iOS 18.5 to iOS 26.2)
  accessed: 2026-11-12
  confidence: medium (forum anecdote, but includes DTS)
  class: platform-constraint

- claim: (snippet) The "Matter client developer profile" has to be installed on the commissioning device for Matter-framework testing, and a developer who asked whether release builds could skip it was told no. The exact production-path requirements need checking against Apple's docs.
  source: https://developer.apple.com/forums/thread/734828 ; https://developer.apple.com/forums/thread/795746
  publisher: Apple Developer Forums
  pub_date: unknown
  accessed: 2026-11-12
  confidence: low
  class: platform-constraint

- claim: On Android, the Google Home APIs Commissioning API can commission a Matter device to the app's own fabric plus the Google fabric, or to the Google fabric only. The flow is a CommissioningRequest with the onboarding payload and setStoreToGoogleFabric(true), and setCommissioningService adds the app's own fabric. commissionDevice() returns an IntentSender, and an ACTION_START_COMMISSIONING intent filter lets the app run commissioning without the Google Home app. If the app has no fabric of its own and no user permission grant, the device may not be accessible to the app.
  source: https://developers.home.google.com/apis/android/commissioning
  publisher: Google
  pub_date: unknown
  accessed: 2026-11-12
  confidence: high
  class: platform-constraint

### Q4 — React Native local-networking stack and OS constraints

- claim: Under iOS Local Network Privacy, outgoing TCP connections, UDP unicast sends, connected UDP sockets, sending and receiving UDP multicast/broadcast, all Bonjour operations, and resolving .local names all need local network access. Listening for incoming TCP and receiving incoming UDP unicast don't. The checks apply to every API (Network framework, BSD sockets, URLSession), so RN socket libraries are covered too.
  source: https://developer.apple.com/tutorials/data/documentation/technotes/tn3179-understanding-local-network-privacy.json (TN3179)
  publisher: Apple
  pub_date: 2026-10-06 (latest revision)
  accessed: 2026-11-12
  confidence: high
  class: platform-constraint

- claim: On iOS, sending or receiving any UDP multicast or broadcast requires the app to be signed with com.apple.developer.networking.multicast. UDP unicast doesn't. Browsing arbitrary or all Bonjour service types also needs the entitlement, but browsing the service types listed in NSBonjourServices doesn't. macOS doesn't require the entitlement.
  source: https://developer.apple.com/tutorials/data/documentation/technotes/tn3179-understanding-local-network-privacy.json (TN3179)
  publisher: Apple
  pub_date: 2026-10-06
  accessed: 2026-11-12
  confidence: high
  class: platform-constraint

- claim: TN3179 says to enable the multicast capability in Xcode's Capabilities editor (automatic signing) or otherwise via "provisioning with managed capabilities". It doesn't describe an approval form or how long approval takes.
  source: https://developer.apple.com/tutorials/data/documentation/technotes/tn3179-understanding-local-network-privacy.json (TN3179)
  publisher: Apple
  pub_date: 2026-10-06
  accessed: 2026-11-12
  confidence: medium (whether Apple still reviews the request isn't stated; see Leads)
  class: platform-constraint

- claim: Apps must add NSLocalNetworkUsageDescription to Info.plist and list the Bonjour types they browse or register in NSBonjourServices (both in the host app's Info.plist, not an extension's). AccessorySetupKit, DeviceDiscoveryUI, AirPlay and UIKit printing are exempt from local network privacy. An iOS 18 bug (FB14321888) that desynced the privacy state was fixed in iOS 18.6.
  source: https://developer.apple.com/tutorials/data/documentation/technotes/tn3179-understanding-local-network-privacy.json (TN3179)
  publisher: Apple
  pub_date: 2026-10-06
  accessed: 2026-11-12
  confidence: high
  class: platform-constraint

- claim: (snippet) react-native-udp still appears to use the asynchronous bridge with Base64 buffers, per a third-party comparison. JSI/Turbo alternatives exist (react-native-nitro-dgram, react-native-udp-turbo v0.1.4 with inconsistent iOS-support claims, react-native-jsi-udp), but their maintenance isn't verified.
  source: https://www.pkgstats.com/pkg:react-native-nitro-dgram ; https://www.pkgstats.com/pkg:react-native-udp-turbo
  publisher: pkgstats (aggregator)
  pub_date: unknown
  accessed: 2026-11-12
  confidence: low (aggregator; no maintainer statement found; ONE SOURCE, version-compatibility class)
  class: version-compatibility

- claim: (snippet) Expo docs say React Native 0.82 is the first version that runs entirely on the New Architecture, and that the legacy architecture stopped getting new features and bug fixes in June 2025, so bridge-only native modules are a risk on current RN.
  source: https://docs.expo.dev/guides/new-architecture
  publisher: Expo
  pub_date: unknown
  accessed: 2026-11-12
  confidence: medium (ONE SOURCE, version-compatibility class; the interop-layer behaviour for legacy modules was not checked)
  class: version-compatibility

## Leads worth chasing

- iOS multicast entitlement is the gating item for every UDP-discovery protocol: Govee (multicast 239.255.255.250:4001, and broadcast for newer models), Yeelight (multicast :1982), LIFX (broadcast discovery, assumed), Tuya (broadcast receive on 6666/6667) and Magic Home (broadcast discovery, assumed). Unicast control after discovery doesn't need it. Hue, Nanoleaf and WLED (mDNS, assumed) need only NSBonjourServices. Next: find Apple's current approval process and turnaround (developer.apple.com/contact/request/networking-multicast) and any 2025–2026 reports of rejections or delays.
- Workaround to verify: without the entitlement, can an app reach devices by unicast sweeps of the subnet, or by user-entered or remembered IPs? TN3179 says UDP unicast doesn't need the entitlement.
- Tuya price conflict: the VAS page shows CNY 33,500/yr and 100,000 calls/month for the dev edition, while the marketing snippet shows $5,000 first year, $2,000 renewal and 1M calls/month. CNY 33,500 is about US$4.7k, so the yearly price may match, but the quota doesn't. Also unresolved: does the Smart Life App SDK work with retail Tuya-brand lights bought by consumers, or only with the developer's own PID products? That decides whether a "control any Smart Life bulb" app is possible officially.
- The Tuya Local Key path needs each end user to create a Tuya IoT developer project, with a 1-month trial that has to be renewed. That's likely unacceptable consumer UX for v1 and needs checking against D-level UX constraints.
- Hue: get Signify's own rate-limit text (developer login needed) and confirm the link-button / generatekey flow, HTTPS cert pinning (Signify root CA) and the SSE eventstream. Hue Bluetooth bulbs weren't researched (likely belongs to the D1 BLE dimension).
- Matter on iOS: read Apple's MatterSupport and Matter framework docs directly. Questions: is the Matter client developer profile only for testing, does ecosystem commissioning need Apple's approval of the matter.allow-setup-payload entitlement, and does Thread need an Apple border router (HomePod/Apple TV)?
- Matter on Android: does the app's own-fabric control of lights (OnOff/LevelControl/ColorControl) need a Google hub, or can it control directly over IP? Check the Google Home APIs Device/Automation APIs and the Play services "Home Mobile SDK" docs, plus the Thread credential APIs.
- RN Matter bindings: no reusable community library surfaced; Espressif uses a custom native module. Search GitHub for "react-native-matter".
- React Native Directory (reactnative.directory) entries for react-native-udp, react-native-tcp-socket and react-native-zeroconf (newArchitecture flag, last release) weren't checked. Also check whether RN 0.82+'s interop layer keeps legacy-bridge modules like react-native-udp working.
- AccessorySetupKit (iOS 18+) is exempt from local network privacy and may be an alternative pairing path for Wi-Fi/BLE accessories; check whether it covers Wi-Fi LAN lights.

## Looked for, not found

- Per-version Tuya cipher details (3.3 AES-ECB; 3.4 session-key negotiation and HMAC; 3.5 AES-GCM framing): the tinytuya README confirms only "AES; GCM needed for 3.5". The tinytuya source or protocol docs are needed.
- Official Govee LAN API manual content (page is JS-rendered; fetch returned empty).
- Official Philips Hue API v2 docs (login-gated); Hue Bluetooth bulb control.
- Nanoleaf Open API, WLED JSON API and Magic Home / Zengge LAN protocol: not researched (budget).
- LIFX discovery and port specifics (UDP 56700, GetService broadcast) and any numeric rate limit: the intro page doesn't contain them.
- Latest versions, release cadence and New Architecture status of react-native-udp, react-native-tcp-socket and react-native-zeroconf: no primary-source evidence retrieved.
- Android multicast lock (WifiManager.MulticastLock) and Android 14–16 local-network changes: not researched this round. Unverified belief: Android 16 introduced an opt-in "local network protection" permission, with enforcement planned for a later release. Verify on developer.android.com.
- Thread border router requirements for third-party Matter controllers on iOS and Android: not found in the pages read.
