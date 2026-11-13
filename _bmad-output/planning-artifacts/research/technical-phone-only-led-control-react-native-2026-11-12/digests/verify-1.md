# Verification batch 1

## C1
- outcome: unverified
- independent_source: none found within budget (closest checked: https://habr.com/ru/articles/781118 — NOT independent)
- publisher: Habr (retelling by denis-19 of Will Cook / 8none1's own write-up)
- pub_date: 2023-12
- accessed: 2026-11-12
- note: The only extra write-up I fetched is a Habr retelling of 8none1's own blog post, so it is not independent. It gives no GATT UUIDs (FFFF/FF01/FF02) and no 0x31 command. It also says a second capture looked AES-encrypted, though it is unclear whether that device is in the LEDnetWF family. Two searches for FF01/FF02 plus 0x31 found nothing independent. Untested lead: the-sz/BluetoothLedStrip, a separate JS library that handles devices named "LEDnetWF" (not fetched). An elektroda.com thread timed out. Treat the UUIDs and "unencrypted" as single-source until checked on real hardware (nRF Connect).

## C2
- outcome: verified
- independent_source: https://reactnative.dev/blog/2025/10/08/react-native-0.82
- publisher: React Native team (Meta) — reactnative.dev
- pub_date: 2025-10
- accessed: 2026-11-12
- note: The RN 0.82 release post calls it "the first React Native that runs entirely on the New Architecture". It tells apps to move to the New Architecture on 0.81 first, because "React Native 0.82 ... prevents enabling the Legacy Architecture" (RFC0929). That confirms the RN half. For the Expo SDK 55 half, the non-Expo confirmation is weaker: only third-party guide titles turned up (e.g., reactnativerelay.com "Expo SDK 55 ... New Architecture必須化" = "made mandatory"), not fetched. Expo's own changelog pins SDK 55 to RN 0.83, so being New-Arch-only follows from the RN 0.82+ rule.

## C3
- outcome: verified
- independent_source: https://developer.apple.com/videos/play/wwdc2024/10203/
- publisher: Apple (WWDC24 session 10203 "Meet AccessorySetupKit" transcript)
- pub_date: 2024-06
- accessed: 2026-11-12
- note: The transcript says: "Since we declared AccessorySetupKit support in our Info.plist ... people will not be asked for permission to use Bluetooth". It also says "The CBCentralManager state will only become poweredOn when your app has paired accessories", and that for the scan API "Only accessories paired with your app will be returned in the scan results." Nuance: Apple says "paired [via AccessorySetupKit]" and that the state never becomes poweredOn. It does not literally say the state reads "poweredOff". Same parent publisher (Apple) as the forum, but this is Apple's own session content, not the forum post.

## C4a
- outcome: verified
- independent_source: https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/java/android/hardware/ConsumerIrManager.java
- publisher: Android Open Source Project (AOSP source, main branch)
- pub_date: unknown (file header Copyright 2013; main branch as fetched 2026-11-12)
- accessed: 2026-11-12
- note: In current AOSP main source, `public final class ConsumerIrManager` carries only `@SystemService(Context.CONSUMER_IR_SERVICE)` and `@RequiresFeature(PackageManager.FEATURE_CONSUMER_IR)`. There is no @Deprecated or Javadoc deprecation on the class or on hasIrEmitter/transmit/getCarrierFrequencies. Caveat: this is source, not an API-level diff. The main branch is the newest code, but I did not check each API level (35/36/37) one by one. The API also still needs device hardware (FEATURE_CONSUMER_IR), which few current phones have.

## C4b
- outcome: verified
- independent_source: https://mobilityarena.com/how-to-get-an-ir-blaster-on-my-phone/
- publisher: MobilityArena
- pub_date: unknown (mentions USB-C iPhone 15+ dongles, so 2023 or later)
- accessed: 2026-11-12
- note: The guide states "No iPhone has ever shipped with a built-in IR blaster". It says the only way to use an iPhone as an IR remote is an external USB-C IR dongle for iPhone 15 and newer, with Lightning dongles largely discontinued. Caveat: a direct fetch returned HTTP 403, so this text comes from the search engine's index of the page, not a full page read. A secondary App Store listing likewise says iOS devices have no IR hardware and need an audio-to-IR accessory.
