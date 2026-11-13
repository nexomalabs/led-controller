# D3 IR + architecture — round 2 digest

Sources read in depth (8): aiohue v2/__init__.py, openHAB Clip2Bridge.java, HA util/color.py, Philips Hue SDK "RGB to xy Color conversion" note, Google ColorSetting trait page, HA contributors Entity class page, HAP-NodeJS ServiceDefinitions.ts, apple.com iPhone 17 specs. Also one aggregator page (jentic.com openhue listing), which is weak evidence and is labeled as such. Everything else is search snippets, marked "(snippet)".

## Findings

### Q1 — Philips Hue bridge rate limits; grouped_light / scenes

- claim: Home Assistant's Hue library (aiohue, v2 client) does not use a requests-per-second throttle. It caps concurrency with `aiohttp.TCPConnector(limit_per_host=3)`. Its source comment says "The bridge will deny more than 3 requests at the same time with a 429 error."
  source: https://raw.githubusercontent.com/home-assistant-libs/aiohue/main/aiohue/v2/__init__.py
  publisher: home-assistant-libs (aiohue maintainers)
  pub_date: unknown (main branch as fetched)
  accessed: 2026-11-12
  confidence: high (for what the code does); medium (for the bridge behaviour the comment describes)
  class: performance

- claim: aiohue retries HTTP 429 ("bridge is rate limiting/overloaded") and 503 ("temporarily unavailable") responses up to MAX_RETRIES = 25 times, with linear backoff of 0.25 s × retry count. After that it raises BridgeBusy ("probably overloaded"). Its comments also note that other apps share the bridge, so a well-behaved client can still hit the limit.
  source: https://raw.githubusercontent.com/home-assistant-libs/aiohue/main/aiohue/v2/__init__.py
  publisher: home-assistant-libs
  pub_date: unknown
  accessed: 2026-11-12
  confidence: high
  class: pattern

- claim: openHAB's current Hue CLIP v2 binding has a hard-coded Throttler with `REQUEST_INTERVAL = Duration.ofMillis(50)` (at most one GET/PUT per 50 ms, which is 20 req/s) and `MAX_CONCURRENT_STREAMS = 3` (a fair Semaphore). The Javadoc says the bridge "can get confused if they receive too many HTTP requests in a short period ... or if too many HTTP sessions are opened at the same time" and then returns an HTML error page. This code is the source of the "~20 req/s for CLIP v2" figure.
  source: https://raw.githubusercontent.com/openhab/openhab-addons/main/bundles/org.openhab.binding.hue/src/main/java/org/openhab/binding/hue/internal/connection/Clip2Bridge.java
  publisher: openHAB project (openhab-addons)
  pub_date: unknown (main branch as fetched)
  accessed: 2026-11-12
  confidence: high (for the code values); medium (as a description of the bridge's true limit)
  class: performance

- claim: The same openHAB source says the Hue bridge's nginx server sends HTTP/2 GO_AWAY frames after a set number of GET/PUT calls ("normally 999"), and the binding then reopens the session. A long-lived client must handle connection recycling.
  source: https://raw.githubusercontent.com/openhab/openhab-addons/main/bundles/org.openhab.binding.hue/src/main/java/org/openhab/binding/hue/internal/connection/Clip2Bridge.java
  publisher: openHAB project
  pub_date: unknown
  accessed: 2026-11-12
  confidence: medium (single source)
  class: platform-constraint

- claim: The figures "~10 light commands/s to a single light" and "1 grouped_light command/s per room or zone" appear as a "soft cap" in the Jentic listing of the openhue CLIP v2 spec. The listing gives no Signify attribution, date or version. node-hue-api reportedly codes a 60 ms per-light limit and a 1 s group limit (snippet only).
  source: https://jentic.com/apis/openhue.io/openhue.md ; https://www.github.com/peter-murray/node-hue-api/releases (snippet)
  publisher: Jentic (aggregator); peter-murray/node-hue-api
  pub_date: unknown
  accessed: 2026-11-12
  confidence: low (aggregator plus snippet; no Signify primary source retrieved; flag as a one-source class)
  class: performance

- claim: CONFLICT RESOLUTION. The two figures are not contradictory, because they limit different layers. "~20 req/s" (openHAB's 50 ms interval) plus "≤3 concurrent requests" (aiohue and openHAB) limit HTTP request admission at the bridge, and both are coded in maintained libraries today. "~10 commands/s to lights, ~1/s to groups" is v1-era guidance about Zigbee radio throughput: per-light unicasts versus group broadcasts. No library or source retrieved this run shows Signify retracting it, and none shows Signify stating it either. Recommended safe envelope for the app: ≤3 in-flight requests per bridge, ≤10 light-level commands/s in total, ≤1 grouped_light command/s per room or zone, and linear backoff on 429/503.
  source: synthesis of the three code sources above plus the Jentic listing
  publisher: (synthesis)
  pub_date: n/a
  accessed: 2026-11-12
  confidence: medium (no Signify primary source; two independent maintained codebases agree on the concurrency cap of 3)
  class: performance

- claim: In CLIP v2, a PUT to `/clip/v2/resource/grouped_light/{id}` addresses "all lights in a room or zone in a single update" (on, dimming, color). Scenes are recalled with a single `recall.action = 'active'` write. These two single-request paths are the ones that avoid per-light fan-out.
  source: https://jentic.com/apis/openhue.io/openhue.md
  publisher: Jentic (aggregator of the openhue spec)
  pub_date: unknown
  accessed: 2026-11-12
  confidence: medium (aggregator; consistent with the community reports in Q4)
  class: protocol

### Q2 — Color conversion for one preset across device types

- claim: HA's `color_RGB_to_xy_brightness` does the following. It normalises RGB to 0–1. It removes sRGB gamma (`((c+0.055)/1.055)^2.4` above 0.04045, otherwise `c/12.92`). It applies a "Wide RGB D65" matrix (e.g. `X = R*0.664511 + G*0.154324 + B*0.162028`). It computes x = X/(X+Y+Z) and y = Y/(X+Y+Z), with brightness = round(min(Y,1)*255). If a Gamut is given and xy is out of reach, it snaps xy to the closest point. Output is rounded to 3 decimals.
  source: https://raw.githubusercontent.com/home-assistant/core/dev/homeassistant/util/color.py
  publisher: Home Assistant (home-assistant/core)
  pub_date: unknown (dev branch as fetched)
  accessed: 2026-11-12
  confidence: high
  class: pattern

- claim: HA's `color_xy_brightness_to_RGB` works in this order: snap to gamut, rebuild XYZ from xy and brightness (vY floored at 1e-11), apply the inverse matrix, reapply gamma, clamp negatives to 0, divide all channels by the max if any exceeds 1, then scale to 0–255. `color_hs_to_xy` converts HS to RGB at full value and then calls RGB→xy.
  source: https://raw.githubusercontent.com/home-assistant/core/dev/homeassistant/util/color.py
  publisher: Home Assistant
  pub_date: unknown
  accessed: 2026-11-12
  confidence: high
  class: pattern

- claim: HA's white-channel conversions:
  - `color_rgb_to_rgbw` sets w = min(r,g,b), subtracts w from each channel, and rescales so the output max equals the input max (`match_max_scale`).
  - `color_rgbw_to_rgb` adds w back and rescales.
  - `color_rgb_to_rgbww` uses the white point at the midpoint (in mireds) of the device's Kelvin range, takes white_level = min(r/w_r, g/w_g, b/w_b), and assigns that level to both cold and warm white.
  - `color_rgbww_to_rgb` derives a CT ratio ww/(cw+ww) and adds the scaled white back.
  source: https://raw.githubusercontent.com/home-assistant/core/dev/homeassistant/util/color.py
  publisher: Home Assistant
  pub_date: unknown
  accessed: 2026-11-12
  confidence: high
  class: pattern

- claim: HA's `color_temperature_to_rgb` clamps input to 1000–40000 K and uses Tanner Helland's piecewise approximation. Mired↔Kelvin conversion is `floor(1e6 / value)`. `color_xy_to_temperature` uses McCamy's cubic approximation, documented as valid for roughly 2000–10000 K.
  source: https://raw.githubusercontent.com/home-assistant/core/dev/homeassistant/util/color.py
  publisher: Home Assistant
  pub_date: unknown
  accessed: 2026-11-12
  confidence: high
  class: pattern

- claim: HA's gamut code is adapted from Benjamin Knight's MIT-licensed rgbxy library. GamutType is three CIE xy points (red, green, blue). `check_valid_gamut` requires non-collinear primaries with all coordinates in 0–1. A point is in reach if its barycentric s ≥ 0, t ≥ 0 and s+t ≤ 1. Otherwise HA projects it onto each of the three edges (with t clamped to 0–1) and picks the nearest projection.
  source: https://raw.githubusercontent.com/home-assistant/core/dev/homeassistant/util/color.py
  publisher: Home Assistant
  pub_date: unknown
  accessed: 2026-11-12
  confidence: high
  class: pattern

- claim: Philips' own (archived) conversion note gives these gamut triangles:
  - "Hue bulbs": R(0.675, 0.322), G(0.4091, 0.518), B(0.167, 0.04). The community calls this gamut B.
  - "LivingColors Bloom, Aura, Iris": R(0.704, 0.296), G(0.2151, 0.7106), B(0.138, 0.08). The community calls this gamut A.
  - "Other lights": R(1.0, 0), G(0.0, 1.0), B(0.0, 0.0).
  For out-of-gamut colours the note says to "Calculate the closest point on the color gamut triangle and use that as xy value".
  source: https://github.com/PhilipsHue/PhilipsHueSDK-iOS-OSX/blob/00187a3db88dedd640f5ddfa8a474458dff4e1db/ApplicationDesignNotes/RGB%20to%20xy%20Color%20conversion.md
  publisher: Philips Hue (Signify) SDK repo, archived 2021-08-11
  pub_date: unknown (repo archived 2021-08); STALE against the 2-year pattern bar, but it is the canonical algorithm source
  accessed: 2026-11-12
  confidence: high (for its contents); medium (for the A/B labels)
  class: protocol

- claim: Gamut C (Hue white and color ambiance, e.g. LCT010–LCT016) is not in the Philips note. Community sources give G(0.17, 0.7), B(0.1532, 0.0475) and red y = 0.3083. The red x value was not verified this run (believed ~0.6915).
  source: https://community.openhab.org/t/hue-binding-hsbtype-color-discrepancy/79908/13 (snippet); https://docs.rs/crate/hues/latest/source/src/service/light.rs (snippet)
  publisher: openHAB community; hues crate
  pub_date: unknown
  accessed: 2026-11-12
  confidence: low
  class: protocol

- claim: There is a matrix inconsistency. HA cites the Philips note as the origin of its RGB→xy formula, but HA uses X = 0.664511R + … while the Philips note gives X = 0.649926R + …. The Philips note itself says its reverse conversion "is a D50 conversion currently". An app should pick one matrix pair and use it in both directions so that round trips stay consistent.
  source: https://raw.githubusercontent.com/home-assistant/core/dev/homeassistant/util/color.py ; https://github.com/PhilipsHue/PhilipsHueSDK-iOS-OSX/blob/00187a3db88dedd640f5ddfa8a474458dff4e1db/ApplicationDesignNotes/RGB%20to%20xy%20Color%20conversion.md
  publisher: Home Assistant; Philips Hue
  pub_date: unknown
  accessed: 2026-11-12
  confidence: high (that the discrepancy exists)
  class: pattern

### Q3 — Other capability models; write-only lights

- claim: The HomeKit Lightbulb service (UUID 00000043-0000-1000-8000-0026BB765291) requires only On. Its optional characteristics are Brightness, Hue, Saturation, ColorTemperature and Name, plus three transition characteristics used for Adaptive Lighting: CharacteristicValueTransitionControl, CharacteristicValueActiveTransitionCount and SupportedCharacteristicValueTransitionConfiguration.
  source: https://raw.githubusercontent.com/homebridge/HAP-NodeJS/master/src/lib/definitions/ServiceDefinitions.ts
  publisher: homebridge / HAP-NodeJS
  pub_date: unknown (master as fetched)
  accessed: 2026-11-12
  confidence: high
  class: protocol

- claim: The Google Home ColorSetting trait declares `colorModel` ("rgb" or "hsv") and/or `colorTemperatureRange` {temperatureMinK, temperatureMaxK}, in any combination on one device. State is reported as color.temperatureK, spectrumRgb (int) or spectrumHsv {hue, saturation 0–1, value 0–1}. The command is ColorAbsolute with color.{name?, temperature | spectrumRGB | spectrumHSV}. The page also has a capitalisation inconsistency (spectrumRgb vs spectrumRGB) between the states and Report State sections.
  source: https://developers.home.google.com/cloud-to-cloud/traits/colorsetting
  publisher: Google (Google Home Developers)
  pub_date: unknown
  accessed: 2026-11-12
  confidence: high
  class: protocol

- claim: Google's ColorSetting has `commandOnlyColorSetting` (default false). Setting it to true declares a one-way device that accepts ColorAbsolute but cannot answer QUERY or Report State. This maps directly onto IR lights and write-only BLE strips.
  source: https://developers.home.google.com/cloud-to-cloud/traits/colorsetting
  publisher: Google
  pub_date: unknown
  accessed: 2026-11-12
  confidence: high
  class: pattern

- claim: Home Assistant's Entity class defines `assumed_state` (bool, default False) as "True when the state is a guess, because the device can't report it". When it is set, the frontend "shows separate on and off buttons instead of a toggle". `should_poll` (default True) should be False for push integrations.
  source: https://contributors.home-assistant.io/core/entities/entity-class/ (developers.home-assistant.io/docs/core/entity/ now 301-redirects here)
  publisher: Home Assistant
  pub_date: unknown
  accessed: 2026-11-12
  confidence: high
  class: pattern

- claim: Pattern synthesis. Three major ecosystems model write-only lights the same way: a per-device or per-capability flag (HA `assumed_state`, Google `commandOnly*`) together with UI that does not imply a known state, such as separate On and Off buttons instead of a toggle. HomeKit's model is HSV-like (Hue + Saturation + Brightness) plus ColorTemperature. Google offers either rgb or hsv plus a Kelvin range. Both fit a canonical internal model of HS + brightness + CT, converted per device.
  source: synthesis of the HAP-NodeJS, Google and HA sources above
  publisher: (synthesis)
  pub_date: n/a
  accessed: 2026-11-12
  confidence: medium
  class: pattern

### Q4 — Group control across mixed transports; popcorn effect

- claim: Community reports agree that the "popcorn effect" (members of a group visibly changing one after another) comes from sending one command per bulb. Bridge-native scenes and groups are sent as a single groupcast, so all members change together. The recommended mitigation is to trigger the vendor-native scene or group (e.g. HA `hue.hue_activate_scene`) rather than fan out from the controller.
  source: https://community.smartthings.com/t/hue-delay-between-bulbs/62888 (snippet); https://community.home-assistant.io/t/ha-with-hue-hub/236993 (snippet)
  publisher: SmartThings community; Home Assistant community
  pub_date: unknown (threads are old; STALE)
  accessed: 2026-11-12
  confidence: low-medium (community, old, but consistent with the grouped_light design)
  class: pattern

- claim: For Hue, the v2 single-request group paths are grouped_light PUT (one write per room or zone) and scene recall. With the ~1 group command/s envelope (Q1), a Hue room is one write. Fanning out per light to 10 lights at ~10/s would take about 1 s and show visible stagger. This is an inference from the Q1 numbers, not measured.
  source: https://jentic.com/apis/openhue.io/openhue.md plus the Q1 synthesis
  publisher: Jentic; synthesis
  pub_date: n/a
  accessed: 2026-11-12
  confidence: low (inferred, not measured)
  class: performance

### Q5 — IR device availability

- claim: POCO's own global spec page for the POCO F7 lists "IR blaster" in its sensor line ("Proximity sensor | Ambient light sensor | Accelerometer | Electronic compass | IR blaster | Gyroscope | Flicker sensor"). 91mobiles also lists "IR Blaster: Yes".
  source: https://www.po.co/global/poco-f7/specs/ (snippet); https://www.91mobiles.com/poco-f7-price-in-india?ty=specs (snippet)
  publisher: POCO / Xiaomi; 91mobiles
  pub_date: unknown (device launched 2025)
  accessed: 2026-11-12
  confidence: medium (manufacturer page seen via snippet only)
  class: device-availability

- claim: The OnePlus 15 (late-2025 flagship) has an infrared port. GSMArena's spec table shows "Infrared port: Yes", and the Notebookcheck and BGR reviews describe using it as a TV or universal remote. The OnePlus 15R also has IR, per Kimovil.
  source: https://m.gsmarena.com/oneplus_15-ampp-14206.php (snippet); https://www.notebookcheck.net/OnePlus-15-smartphone-review-Gaming-at-165fps-despite-being-slower-than-the-predecessor.1171622.0.html (snippet); https://bgr.com/2047680/oneplus-15-review-battery-champion (snippet)
  publisher: GSMArena; Notebookcheck; BGR (third-party, no oneplus.com page retrieved)
  pub_date: 2025 (launch-era reviews)
  accessed: 2026-11-12
  confidence: medium
  class: device-availability

- claim: Apple's iPhone 17 tech specs page lists these sensors: Face ID, Barometer, High dynamic range gyro, High-g accelerometer, Proximity sensor, Dual ambient light sensors. It lists this wireless: Wi-Fi 7, Bluetooth 6, Thread, NFC with reader mode, and second-generation UWB. Neither "infrared" nor a standalone "IR" appears anywhere on the page. The evidence that iPhone has no IR transmitter is therefore only its absence from the spec list. Apple publishes no API for one either, but that was not re-verified this run.
  source: https://www.apple.com/iphone-17/specs/
  publisher: Apple
  pub_date: 2025-09 (iPhone 17 launch; page undated)
  accessed: 2026-11-12
  confidence: medium (absence-of-evidence only)
  class: device-availability

## Leads worth chasing
- Signify primary source for rate limits: the developers.meethue.com "Hue System Performance" / "Core concepts" pages (may require login). This is the only way to upgrade the 10/s and 1/s envelope above "low".
- The node-hue-api Changelog entry that swapped the light and group limits (60 ms per light, 1 s per group). Confirming it would give a second independent coded source for the 10/s and 1/s figures. Note that node-hue-api is v1-API and likely unmaintained.
- The openhue-api GitHub spec (openhue/openhue-api) for the light `color.gamut` / `gamut_type` (A/B/C/other) fields and the scene `recall.action` enum (active / dynamic_palette / static). This would also give a primary source for Gamut C coordinates, since v2 lights report their own gamut.
- The Hue Entertainment API (DTLS streaming to an entertainment area) as the true sync path for many-light effects. Its rate and licensing terms were not researched.
- HAP-NodeJS CharacteristicDefinitions.ts for Hue (0–360), Saturation (0–100) and ColorTemperature (believed 140–500 mireds) ranges.
- Google OnOff `commandOnlyOnOff` and Brightness `commandOnlyBrightness` attributes (believed to exist; not verified this run).
- HA `homeassistant/components/light/__init__.py`, which shows how HA converts a requested color into each entity's supported color mode. That is the reference for a "preset → per-device payload" layer.
- Manufacturer spec pages still to check: mi.com Redmi Note 15 Pro / Xiaomi 15T, honor.com Honor 400, and oneplus.com OnePlus 15 (only third-party sources so far).

## Looked for, not found
- Any official Signify statement of bridge request or command limits visible without login. None was retrieved, and both popular figures trace to library code or aggregators.
- A coded per-second throttle in aiohue. There is none: it uses a concurrency cap of 3 plus 429/503 backoff.
- Measured latency numbers for app-side fan-out to many mixed-transport lights, or for vendor groups versus app groups. No retrospective or benchmark was found.
- Govee group and LIFX multizone group semantics. Not reached within budget.
- Gamut C in Philips' own (archived) conversion note. It is not there.
- A publication date on the Google ColorSetting, HA Entity class and HAP-NodeJS sources. None is shown; they were fetched from current branches and pages.
