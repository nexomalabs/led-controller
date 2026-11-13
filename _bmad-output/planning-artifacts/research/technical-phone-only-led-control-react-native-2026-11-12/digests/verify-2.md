# Verification batch 2

## C5
- outcome: disputed
- independent_source: https://github.com/make-all/tuya-local (README + DEVICE_DETAILS.md); corroborating: https://community.home-assistant.io/t/tuya-iot-account-expired-unstable-operation/369202
- publisher: make-all (Tuya-Local Home Assistant project); Home Assistant Community forum
- pub_date: unknown (README, living doc); forum thread 2021-12
- accessed: 2026-11-12
- note: Tuya-Local agrees that local control needs the device ID + local key, that the manual route is the Tuya IoT developer portal (open a cloud project, linked Smart Life/Tuya Smart account, API Explorer "Query Device Details"), and that Tuya "started time limiting access ... in the IOT developer portal to only a month", with the trial refreshable "only every 6 months". The HA forum thread reports extensions on application (up to a year, or 5 months). BUT the same README says its cloud-assisted setup (sign in through the Tuya/Smart Life app) pre-fills the local key "without needing to create a Tuya IOT developer account". So the developer project is one way to get the key, not the only way.

## C6
- outcome: verified
- independent_source: https://www.nabto.com/docs/developer/platforms/ios/ios145.html
- publisher: Nabto (IoT SDK vendor docs)
- pub_date: unknown (content mentions iOS 14.5 to iOS 16 and Q1 2023, so likely 2023)
- accessed: 2026-11-12
- note: Nabto's iOS notes say "Apps using low level BSD based broadcast communication need to send a formal request to Apple to be allowed to do so", that such access "requires the com.apple.developer.networking.multicast restricted entitlement", and that developers "must request the multicast entitlement for each app". Nuance: the page is written for broadcast discovery in its legacy SDK and does not cover approval timelines. Apple forum threads found in the same search add that the entitlement is necessary but not enough: the user's local-network permission is still required.

## C7
- outcome: verified
- independent_source: https://www.helpnetsecurity.com/?p=359446 (Android 17 second beta expands privacy controls for contacts, SMS and local networks)
- publisher: Help Net Security (Anamarija Pogorelec)
- pub_date: 2026-02
- accessed: 2026-11-12
- note: Describes ACCESS_LOCAL_NETWORK as a new runtime permission for Android 17: "Apps must declare and request this permission to discover or connect with local devices" (smart home, casting). It quotes Google's Matthew McCullough: apps targeting Android 17 or higher can either use system-mediated device pickers, which avoid the prompt, or request the permission at runtime. Nuance: the article never mentions TCP/UDP or ties API level 37 to the permission by name. The picker/NSD-resolve exemptions mean the permission is not the only route.

## C8
- outcome: verified
- independent_source: https://developer.apple.com/documentation/mattersupport/adding-matter-support-to-your-ecosystem ; https://developer.apple.com/documentation/mattersupport ; https://developer.apple.com/documentation/bundleresources/entitlements/com.apple.developer.matter.allow-setup-payload
- publisher: Apple Developer Documentation
- pub_date: unknown (living docs)
- accessed: 2026-11-12
- note: Apple says to adopt MatterSupport "to add compatible devices to your ecosystem, then use Matter [framework] to commission and control them". This uses a MatterAddDeviceRequest with your own ecosystem topology plus a MatterAddDeviceExtensionRequestHandler extension (extension point com.apple.matter.support.extension.device-setup). Corrections: Apple lists MatterSupport as iOS 16.1+ (the entitlement key as iOS 16.0+), not 16.4. allow-setup-payload is described as optional. It is needed only when the app supplies the setup payload itself: a device without a Matter QR code, moving an already-paired device to Apple Home, or an in-app scanner that also reads non-Matter QR codes. The standard system QR flow does not need it.

## C9
- outcome: verified
- independent_source: https://www.home-assistant.io/integrations/tuya/
- publisher: Home Assistant (Open Home Foundation)
- pub_date: unknown (living doc)
- accessed: 2026-11-12
- note: The integration "relies on the official Python SDK provided by Tuya" (link goes to tuya-device-sharing-sdk). Setup takes the User Code from the Smart Life / Tuya Smart app, then you "Scan the QR code displayed on your Home Assistant screen" with the app. No Tuya IoT developer project is required. iot.tuya.com comes up only in a re-auth troubleshooting tip ("accept the terms and conditions").
