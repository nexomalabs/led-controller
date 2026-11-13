# D2 Wi-Fi — round 2 digest

Run date 2026-11-12. 19 tool calls. In-depth reads: npm registry and reactnative.directory API (scripted), Expo New Architecture guide, GitHub issues/releases API, Apple forum thread 838025, Android Local Network Permission page, WLED JSON API page, flux_led/WLED source greps, tuya.com App SDK page. Everything else comes from search snippets and is marked "(snippet)".

## Findings

### Q1 — React Native local-networking libraries and New Architecture

- claim: react-native-udp's latest version is 4.1.7, published 2023-01-26. It has had no release in the last 12 months (previous releases were 4.1.6 on 2022-12-05 and 4.1.5 on 2022-03-22).
  source: https://registry.npmjs.org/react-native-udp
  publisher: npm registry (tradle/react-native-udp)
  pub_date: 2023-01-26
  accessed: 2026-11-12
  confidence: high
  class: ecosystem-health

- claim: reactnative.directory flags react-native-udp as `unmaintained: true` with no `newArchitecture` flag (358 stars, last push 2023-01-26).
  source: https://reactnative.directory/api/libraries?search=react-native-udp
  publisher: React Native Directory
  pub_date: 2026-11-12 (live API)
  accessed: 2026-11-12
  confidence: high
  class: ecosystem-health

- claim: The open issue tradle/react-native-udp#259, "New Architecture Support (TurboModules/Fabric)", was filed 2026-01-16 and shows no activity since (its updated date equals its created date).
  source: https://github.com/tradle/react-native-udp/issues/259
  publisher: GitHub (tradle/react-native-udp)
  pub_date: 2026-01-16
  accessed: 2026-11-12
  confidence: medium
  class: version-compatibility

- claim: react-native-tcp-socket's latest version is 6.4.3, published 2026-09-10. It had 5 releases in the last 12 months (6.3.1 on 2026-01-03, 6.4.0 on 01-15, 6.4.1 on 01-16, 6.4.2 on 07-26, 6.4.3 on 09-10), all bug fixes or small features such as connectTimeout and iOS startTLS fixes.
  source: https://registry.npmjs.org/react-native-tcp-socket ; https://github.com/Rapsssito/react-native-tcp-socket/releases
  publisher: npm registry / Rapsssito
  pub_date: 2026-09-10
  accessed: 2026-11-12
  confidence: high
  class: ecosystem-health

- claim: react-native-tcp-socket is still a legacy bridge module. Its master package.json has no `codegenConfig`, reactnative.directory shows no `newArchitecture` flag, and issue #187, "Request: Support React Native Architecture", has been open since 2024-04-12. On RN 0.82+ it would therefore run only through the interop layer. This is my inference from those three signals; I did not test it on a device.
  source: https://raw.githubusercontent.com/Rapsssito/react-native-tcp-socket/master/package.json ; https://github.com/Rapsssito/react-native-tcp-socket/issues/187
  publisher: Rapsssito / GitHub
  pub_date: 2024-04-12 (issue); package.json read live
  accessed: 2026-11-12
  confidence: medium
  class: version-compatibility

- claim: An open issue filed 2026-11-13 (UTC) reports an Android crash in 6.4.3: IllegalArgumentException "No socket with id" when writing to a socket that has just closed. This is one report and is not confirmed.
  source: https://github.com/Rapsssito/react-native-tcp-socket/issues/248
  publisher: GitHub (Rapsssito/react-native-tcp-socket)
  pub_date: 2026-11-13
  accessed: 2026-11-12
  confidence: low
  class: ecosystem-health

- claim: react-native-zeroconf's latest stable version is 0.17.7 and its latest prerelease is 1.0.0-rc.3, both published 2026-10-29. It had 17 releases between 2025-12-31 and 2026-10-29, most of them in a burst during Sept–Oct 2026. reactnative.directory flags it `newArchitecture: true`.
  source: https://registry.npmjs.org/react-native-zeroconf ; https://reactnative.directory/api/libraries?search=react-native-zeroconf
  publisher: npm registry / React Native Directory (Apercu/react-native-zeroconf)
  pub_date: 2026-10-29
  accessed: 2026-11-12
  confidence: high
  class: version-compatibility

- claim: react-native-nitro-dgram, a Nitro-based UDP library, has only 3 versions (0.1.4 to 0.1.6), all published between 2025-12-26 and 2026-01-01, with nothing since. reactnative.directory does not list it. It looks immature or stalled.
  source: https://registry.npmjs.org/react-native-nitro-dgram
  publisher: npm registry (iwater/react-native-nitro-dgram)
  pub_date: 2026-01-01
  accessed: 2026-11-12
  confidence: high
  class: ecosystem-health

- claim: The same author's react-native-nitro-net is more active: 0.6.2 was published 2026-09-23, after 16 releases since 2025-12-26. From the name it is a Nitro `net` (TCP) module, but I did not verify its scope.
  source: https://registry.npmjs.org/react-native-nitro-net
  publisher: npm registry (iwater/react-native-nitro-net)
  pub_date: 2026-09-23
  accessed: 2026-11-12
  confidence: low
  class: ecosystem-health

- claim: Other UDP alternatives exist. react-native-jsi-udp (mybigday) is a JSI UDP socket, and react-native-udp-turbo is a TurboModule that supports only Android at v0.1.4. I did not check the maintenance status of either. (snippet)
  source: https://socket.dev/npm/package/react-native-jsi-udp ; https://www.pkgstats.com/pkg:react-native-udp-turbo
  publisher: socket.dev / pkgstats (aggregators)
  pub_date: unknown
  accessed: 2026-11-12
  confidence: low
  class: landscape

- claim: "React Native 0.82 was the first version to remove the option to disable the New Architecture." Expo SDK 54 is the last SDK where it can be disabled, and SDK 55 uses RN 0.83. This is single-source (Expo); I did not cross-check it against reactnative.dev this run.
  source: https://docs.expo.dev/guides/new-architecture/
  publisher: Expo
  pub_date: 2026-11-02 (page modificationDate)
  accessed: 2026-11-12
  confidence: high
  class: version-compatibility

- claim: Interop layers have been on by default since RN 0.74 and "allow many libraries built for the old architecture to work on the New Architecture without any changes". However, "the interop is not perfect and some libraries will need to be updated", and libraries with third-party native code are the most likely to need updates.
  source: https://docs.expo.dev/guides/new-architecture/
  publisher: Expo
  pub_date: 2026-11-02
  accessed: 2026-11-12
  confidence: high
  class: version-compatibility

### Q2 — iOS multicast entitlement

- claim: In July 2026 a developer reported that Apple denied their com.apple.developer.networking.multicast request for an SSDP-based UPnP/DLNA control point, "with guidance to use Bonjour instead". Apple DTS (Quinn) replied that there is no supported way to send multicasts on iOS without the entitlement, and suggested re-applying while making clear that SSDP is the only option. This is one report.
  source: https://developer.apple.com/forums/thread/838025
  publisher: Apple Developer Forums (DTS reply)
  pub_date: 2026-07
  accessed: 2026-11-12
  confidence: high
  class: platform-constraint

- claim: Apple's entitlement documentation says the entitlement "requires permission from Apple before you can use it". It is required to send or receive IP multicast or broadcast on iOS, and it also allows browsing and advertising arbitrary Bonjour service types. (snippet)
  source: https://developer.apple.com/documentation/bundleresources/entitlements/com.apple.developer.networking.multicast
  publisher: Apple
  pub_date: unknown
  accessed: 2026-11-12
  confidence: medium
  class: platform-constraint

- claim: Older forum reports put approvals at about 3–6 days, with some applicants waiting more than 2 weeks. DTS has said there is no way to check a request's status, that it cannot grant the entitlement itself, and that it is granted per team. These reports are STALE (2020–2022) and come from snippets.
  source: https://developer.apple.com/forums/thread/683526 ; https://developer.apple.com/forums/thread/666240
  publisher: Apple Developer Forums
  pub_date: 2020–2022 (stale)
  accessed: 2026-11-12
  confidence: low
  class: platform-constraint

- claim: Several protocols need the iOS entitlement for auto-discovery: Govee (multicast scan, per round 1), Magic Home (UDP broadcast to port 48899, see Q4) and Yeelight (multicast). Hub-style APIs that advertise over mDNS/Bonjour (WLED `_wled._tcp`, confirmed below; Hue and Nanoleaf mDNS not confirmed this run) can be discovered without it if the service types are listed in NSBonjourServices. This is derived by combining TN3179 (round 1) with the discovery mechanisms found here; it is not an Apple statement.
  source: https://developer.apple.com/forums/thread/838025 ; https://raw.githubusercontent.com/wled/WLED/main/wled00/wled.cpp ; https://raw.githubusercontent.com/lightinglibs/flux_led/master/flux_led/scanner.py
  publisher: Apple / WLED / lightinglibs
  pub_date: 2026-07 (forum); source code read live
  accessed: 2026-11-12
  confidence: medium
  class: pattern

### Q3 — Android local network access

- claim: Apps targeting Android 17 (API 37) or higher have local network access blocked by default. They must declare and request the runtime permission ACCESS_LOCAL_NETWORK, which belongs to the NEARBY_DEVICES group. Apps targeting 36 or lower are implicitly granted access through INTERNET and "don't add ACCESS_LOCAL_NETWORK" before targeting 37.
  source: https://developer.android.com/privacy-and-security/local-network-permission
  publisher: Google / Android Developers
  pub_date: unknown
  accessed: 2026-11-12
  confidence: high
  class: platform-constraint

- claim: The permission gates outgoing and incoming TCP, and sending and receiving UDP unicast, multicast and broadcast. It "appl[ies] to all networking APIs", including OkHttp and Cronet, and resolving `.local` names needs it too. A blocked TCP connection typically times out; a blocked UDP call returns EPERM.
  source: https://developer.android.com/privacy-and-security/local-network-permission
  publisher: Google / Android Developers
  pub_date: unknown
  accessed: 2026-11-12
  confidence: high
  class: platform-constraint

- claim: Android's permission-free path is the NsdManager system picker (DiscoveryRequest FLAG_SHOW_PICKER). The user picks a single device, and connections to that device's IP need no permission. The docs say "home automation or IoT device management that need broad, persistent access" should request ACCESS_LOCAL_NETWORK instead (Path B).
  source: https://developer.android.com/privacy-and-security/local-network-permission
  publisher: Google / Android Developers
  pub_date: unknown
  accessed: 2026-11-12
  confidence: high
  class: platform-constraint

- claim: If the user has already granted another NEARBY_DEVICES permission, such as Bluetooth scan/connect, the app is not re-prompted and ACCESS_LOCAL_NETWORK is treated as granted. So an app that already asks for BLE permissions gets LAN access without a second prompt.
  source: https://developer.android.com/privacy-and-security/local-network-permission
  publisher: Google / Android Developers
  pub_date: unknown
  accessed: 2026-11-12
  confidence: high
  class: platform-constraint

- claim: The Android 16 opt-in (adb `am compat enable RESTRICT_LOCAL_NETWORK` plus NEARBY_WIFI_DEVICES) is deprecated and replaced by the Android 17 enforcement.
  source: https://developer.android.com/privacy-and-security/local-network-permission
  publisher: Google / Android Developers
  pub_date: unknown
  accessed: 2026-11-12
  confidence: high
  class: platform-constraint

### Q4 — Vendor local APIs

- claim: WLED's JSON API serves full state, info, effects and palettes at `GET /json`. `POST /json/state` (or `/json`) accepts a partial state with these keys: `on` (true, false, or "t" to toggle), `bri` 0–255, `seg[].col` (up to 3 colors as RGB(W) arrays or hex), `seg[].fx` (effect id) and `ps` (preset -1..250). A separate WebSocket interface exists (its path is not on this page).
  source: https://kno.wled.ge/interfaces/json-api/
  publisher: WLED project
  pub_date: unknown (references 0.15.0 and features "since 16.0.0")
  accessed: 2026-11-12
  confidence: high
  class: protocol

- claim: WLED firmware advertises mDNS `_wled._tcp` on port 80, with a `mac` TXT record, plus `_http._tcp`.
  source: https://raw.githubusercontent.com/wled/WLED/main/wled00/wled.cpp (L942–944)
  publisher: WLED project (source code)
  pub_date: main branch, read 2026-11-12
  accessed: 2026-11-12
  confidence: high
  class: protocol

- claim: WLED's latest GitHub release is v16.0.1 (2026-07-07). WLED also has its own UDP sync groups (`udpn.sgrp`/`udpn.rgrp`, groups 1–8) that can propagate state between WLED nodes.
  source: https://api.github.com/repos/wled/WLED/releases/latest ; https://kno.wled.ge/interfaces/json-api/
  publisher: WLED project
  pub_date: 2026-07-07
  accessed: 2026-11-12
  confidence: high
  class: ecosystem-health

- claim: Magic Home/Zengge (flux_led) discovery sends the UDP broadcast payload `HF-A11ASSISTHREAD` to port 48899 with SO_BROADCAST (BROADCAST_FREQUENCY = 6, "at least 6 for 0xA1 models"). Control uses TCP port 5577 (`DEFAULT_PORT = 5577`). No authentication appears in the library README or the code I grepped; that absence is weak evidence of no auth.
  source: https://raw.githubusercontent.com/lightinglibs/flux_led/master/flux_led/scanner.py ; https://raw.githubusercontent.com/lightinglibs/flux_led/master/flux_led/base_device.py
  publisher: lightinglibs/flux_led (source code)
  pub_date: master, read 2026-11-12
  accessed: 2026-11-12
  confidence: high (ports/discovery); low (no-auth)
  class: protocol

- claim: flux_led's latest GitHub release is 1.2.0 (2025-04-05), about 19 months ago. Its reference value is as protocol documentation, not as a fast-moving dependency.
  source: https://api.github.com/repos/lightinglibs/flux_led/releases/latest
  publisher: lightinglibs/flux_led
  pub_date: 2025-04-05
  accessed: 2026-11-12
  confidence: high
  class: ecosystem-health

- claim: Nanoleaf Open API pairing works like this. Hold the controller's on/off button for 5–7 s until the LED flashes, then within about 30 s send an empty `POST http://<ip>:16021/api/v1/new`, which returns `{"auth_token": "..."}`. If the device is not in pairing mode it returns 403. The API is plain HTTP with no TLS. This comes from community libraries and integrator docs; I did not open the official doc (forum.nanoleaf.me/docs/openapi). (snippet)
  source: https://nanoleafapi.readthedocs.io/en/latest/api.html ; https://v51.openhab.org/addons/bindings/nanoleaf/
  publisher: nanoleafapi (community) / openHAB
  pub_date: unknown
  accessed: 2026-11-12
  confidence: medium
  class: protocol

- claim: An integrator says its Nanoleaf Open API template does not support Thread-only Nanoleaf Essentials or Skylight. Essentials bulbs would therefore need the Matter/HomeKit path rather than the Open API. (snippet)
  source: https://taphome.com/en/compatibility/nanoleaf-rgb-light/
  publisher: TapHome
  pub_date: unknown
  accessed: 2026-11-12
  confidence: low
  class: landscape

- claim: Hue Bridge pairing works like this. Send `POST https://<bridge>/api` with `{"devicetype": "...", "generateclientkey": true}`. The bridge refuses until the physical link button is pressed; after the press, a retry returns `username`, which is used as the `hue-application-key` header on CLIP v2 calls (for example `GET /clip/v2/resource/device`). `generateclientkey` matters mainly for Entertainment streaming. The source is an unpkg mirror of the developers.meethue.com Getting Started page plus openHAB's v2 binding docs; I did not read the official page directly. (snippet)
  source: https://unpkg.com/node-red-contrib-knx-ultimate@4.0.24/tutorial/HUEApiV2/developers.meethue.com/develop/hue-api-v2/getting-started/index.html ; https://www.openhab.org/addons/bindings/hue/doc/readme_v2.html
  publisher: Signify (mirrored) / openHAB
  pub_date: unknown
  accessed: 2026-11-12
  confidence: medium
  class: protocol

- claim: I found no official third-party Bluetooth API for Hue Bluetooth bulbs. Philips' FAQ positions Bluetooth control as the Hue app's own bridgeless mode. Reviews report a 10-light limit and no grouping in that mode. Bridgeless BLE control exists only in reverse-engineered community libraries (libhueble, hueble).
  source: https://www.philips-hue.com/en-us/explore-hue/faq/controls/how-to-connect-philips-hue-without-a-bridge ; https://github.com/alexhorn/libhueble ; https://hueble.readthedocs.io/en/latest/
  publisher: Signify / community
  pub_date: unknown
  accessed: 2026-11-12
  confidence: medium (no official API, based on absence); low (10-light/grouping limits, snippet)
  class: platform-constraint

### Q5 — Tuya App SDK

- claim: Tuya's international App SDK page (© 2026, undated) lists two editions. The Development Edition is free, allows up to 100 registered users in total and up to 1 million cloud requests per month, and is not for launch or commercial use. The Official Edition is "$5000 per year, $2000 next year", with unlimited users, up to 100 million cloud requests per month, 1-year validity, and all UI BizBundles. The page does not show a currency code.
  source: https://www.tuya.com/platform/appdev/app-sdk
  publisher: Tuya Inc.
  pub_date: unknown (© 2026)
  accessed: 2026-11-12
  confidence: high
  class: licensing

- claim: My best resolution of the pricing conflict: CNY 33,500/yr on the VAS commodity page (round 1) is roughly US$4.7k at about 7.1 CNY/USD, consistent with "$5,000 first year", so the two pages are likely regional or currency storefronts for the same Official Edition. The dev-edition quota conflict (100,000 vs 1 million calls/month) remains UNRESOLVED. Both are undated first-party Tuya pages, so neither wins on recency, and the figure should be confirmed with Tuya before budgeting. This is inference.
  source: https://www.tuya.com/platform/appdev/app-sdk ; https://www.tuya.com/vas/commodity/SDK_APP_CREATE
  publisher: Tuya Inc.
  pub_date: unknown
  accessed: 2026-11-12
  confidence: low
  class: licensing

- claim: Neither Tuya page says that an App SDK app can control devices consumers already paired in the Smart Life app. The only official route I found for third-party software to control already-paired Smart Life devices is Tuya's own "Smart Life" integration (tuya/tuya-smart-life). It uses tuya-device-sharing-sdk for Powered-by-Tuya devices and is commonly used with Home Assistant. (snippet)
  source: https://github.com/tuya/tuya-smart-life ; https://github.com/tuya/tuya-smart-life-app-sdk
  publisher: Tuya Inc. (GitHub)
  pub_date: unknown
  accessed: 2026-11-12
  confidence: medium
  class: landscape

## Leads worth chasing
- tuya-device-sharing-sdk / Smart Life QR "user code" authorization. Could a third-party mobile app use this flow, or is its client ID or terms tied to Home Assistant? This is the most promising official way to reach retail Smart Life lights without an IoT-project Local Key per user.
- Unverified belief: Tuya App SDK users are scoped to the developer's own app account system, so devices paired in the consumer Smart Life app do not appear in a custom SDK app. Check developer.tuya.com FAQ.
- Unicast subnet-sweep viability per vendor: does Govee answer a unicast scan on 4001, does Magic Home answer a unicast `HF-A11ASSISTHREAD` on 48899, and does LIFX answer unicast GetService on 56700? No evidence found this run. A sweep is the only entitlement-free iOS discovery path for these vendors.
- Apple forum threads 814969 ("Multicast Entitlement now not working") and 798399 ("Multicast Entitlements") look like 2025–2026 reports and were not read. The request form's questions are behind login.
- Hue Bridge HTTPS uses a Signify-signed or self-signed certificate, and RN `fetch` will likely reject it without a native trust override. Nanoleaf and WLED use plain HTTP, which needs iOS ATS `NSAllowsLocalNetworking` and an Android cleartext network-security-config. All three are unverified this run.
- Google Play target-API deadline for API 37 (likely Aug 2027, unverified): this decides when ACCESS_LOCAL_NETWORK becomes mandatory for the app.
- react-native-zeroconf 1.0.0-rc changelog (rewrite? Nitro/TurboModule?) and whether it exposes Android NsdManager FLAG_SHOW_PICKER.
- react-native-nitro-net scope and maturity, as a TurboModule/Nitro replacement for react-native-tcp-socket.
- WLED settings PIN / OTA lock (whether the JSON API is open by default) and WebSocket path (`/ws` expected).
- Nanoleaf mDNS service type (`_nanoleafapi._tcp` expected), to confirm in official Open API docs.

## Looked for, not found
- 2025–2026 data on multicast entitlement approval times. Only 2020–2022 anecdotes, plus one July 2026 denial.
- Any Apple statement on whether unicast sweeps or user-entered IPs are an acceptable substitute (DTS thread 838025 did not address it).
- WifiManager.MulticastLock requirement. The Android LNP page does not mention it, and I did not fetch the WifiManager reference this run.
- An official WLED statement that the JSON API is unauthenticated by default.
- An official Tuya statement on interoperability between SDK apps and Smart Life accounts.
- Any maintainer response on react-native-udp New Architecture support.
- An official Hue Bluetooth (BLE) API for third parties.
- Dates on the Android LNP page and the Tuya pricing pages.
