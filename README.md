# Why 
Dutch Butterfly conservation is a non-profit aimed at preserving butterflies and other insects. Why did we spend time updating an app for a laser game toy? We had a team building exercise where we turned our office into a laser game battleground for one evening. We used the Recoil guns as they were cheap. We had great fun, we are now sharing this code hoping that other people might also have the joy of pretend shooting each other. Also we support using devices as long as possible, so even when official support has ended, these guns are now still usable. 


# SimpleCoil

An Android app for the Recoil laser-tag system. Lets you connect to a Recoil
"blaster" over Bluetooth LE and run multiplayer games against other players
either ad-hoc or coordinated by a dedicated-server phone over WiFi.

## About this fork

This is a fork of [Dees-Troy/SimpleCoil](https://github.com/Dees-Troy/SimpleCoil),
which appears to be inactive. The upstream code targeted Android 10 (API 29) and
no longer builds or runs on modern devices: Google Play now requires `targetSdk`
35, jcenter (which hosted some of the dependencies) is gone, and several
behavioural changes in Android 12 / 13 / 14 break the upstream code at runtime
even when it does compile.

This fork brings the project up to current Android standards while preserving
the original gameplay loop and the wire-level protocol so it stays compatible
with anyone still running the upstream build.

The license remains Apache 2.0; the upstream `LICENSE` file and per-file
copyright headers are preserved unchanged.

## What changed

### Build / tooling
- Gradle 5.4.1 → 8.9, Android Gradle Plugin 3.5.1 → 8.7.3
- `jcenter()` replaced with `mavenCentral()` (jcenter is read-only as of 2022)
- `compileSdk` / `targetSdk` 29 → 35
- Java 11 source / target compatibility
- `namespace` declared in `build.gradle`; `package` removed from manifest
- AndroidX dependencies bumped to current versions (appcompat 1.7, material
  1.12, constraintlayout 2.2, fragment 1.8.5)
- Legacy `legacy-support-v4` dependency dropped

### Android 12+ compatibility
- New `BLUETOOTH_SCAN` (with `neverForLocation`) and `BLUETOOTH_CONNECT`
  permissions added; old `BLUETOOTH` / `BLUETOOTH_ADMIN` / location permissions
  capped at `maxSdkVersion=30`
- Runtime permission requests in `FullscreenActivity.connectWeapon()` updated to
  request the new permissions on API 31+ and fall back to `ACCESS_FINE_LOCATION`
  on older devices

### Android 14+ compatibility (the two crash fixes)
- All runtime-registered `BroadcastReceiver`s now use
  `ContextCompat.registerReceiver(..., RECEIVER_NOT_EXPORTED)`. Without this
  the app crashes on launch with `SecurityException` on Android 14+.
- All internal `sendBroadcast` calls now set the package on the intent
  (`intent.setPackage(getPackageName())`). Android 14+ silently drops implicit
  broadcasts to your own app, which previously left the connect flow stuck
  forever at "Connecting…" because the BLE service's `ACTION_GATT_*` events
  never reached the activity. Centralized in `NetMsg.sendInternal(...)`.

### Activity / service hardening
- All four services (`BluetoothLeService`, `UDPListenerService`, `TcpClient`,
  `TcpServer`) explicitly declared `exported="false"` in the manifest
- The launcher activity has `android:exported="true"` (required Android 12+)

### New features
- **Connect by code.** After connecting to a blaster, a "Show Code" button on
  the play screen displays the blaster's BLE MAC formatted as
  `XXXX-XXXX-XXXX`, with a Copy button. On the connect screen, a "Connect Using
  Code" button accepts the same string (lenient: hyphens, colons, spaces, and
  case are all ignored) and connects directly to that blaster without
  scanning. This replaces the QR scanner the upstream had — see below.
- **Team scoreboard on the dedicated server.** The dedicated-server activity
  now shows live team totals (T1 / T2 / T3 / T4, hidden in FFA mode) above the
  player list. Refreshes on every kill via the existing
  `NETMSG_PLAYERDATAUPDATE` broadcast.
- **Score limit auto-end.** When the score limit is configured, the server
  ends the game automatically when any team total (or any individual score in
  FFA) reaches the limit. Same shutdown path as the existing time limit.

## What was removed or disabled

These features existed upstream but were tied to dependencies that no longer
work. Don't go looking for them:

- **In-game map view.** The upstream app rendered a tile-based map with player
  position markers using the [Maply](https://github.com/mousebird/WhirlyGlobe)
  library (`com.mousebirdconsulting.maply:Android:2.5`). That artifact lived
  only on jcenter and was never republished. `MapFragment.java` is now a no-op
  `Fragment` stub. The server-side GPS data plumbing (`NETMSG_GPSLOCUPDATE`,
  GPS-mode dropdown, etc.) is intact and waiting for a new mapping library to
  be wired in.
- **QR code scanner.** Upstream invoked the Google "Barcode Scanner" app via an
  `Intent` with action `com.google.zxing.client.android.SCAN`. That app has
  been delisted from the Play Store, so the feature was already broken before
  this fork. Replaced by the typeable code described above.

If you want maps back, the natural drop-ins are
[osmdroid](https://github.com/osmdroid/osmdroid) (open-source, OpenStreetMap
tiles) or the Google Maps SDK. The data plumbing is preserved.

## Building

Standard Android Studio project. To build a debug APK from the command line:

```bash
./gradlew assembleDebug
```

Output: `app/build/outputs/apk/debug/app-debug.apk`

To install onto a connected device with USB debugging:

```bash
./gradlew installDebug
```

JDK 17+ is required by AGP 8. Android Studio's bundled JBR (Java 21) works
out of the box.

## Using the app

### Connecting to a blaster

1. Power on the blaster.
2. Open the app. On first launch you'll be prompted for **Nearby devices**
   permission (Android 12+) or **Location** (Android 11 and below).
3. Tap **Connect Weapon** to scan for any nearby `SRG1*` device, or
   **Connect Using Code** to connect to a specific blaster you have the code
   for. After a successful connection the address is remembered, so
   **Reconnect Same Weapon Only** picks up where you left off next time.

### Network play

1. On each player phone: connect to a blaster, then tap **Use Networking**.
2. Either pick **Create Server** (one phone hosts) or **Join** (broadcasts to
   discover an existing server on the same WiFi).

### Dedicated server

A phone running in dedicated-server mode acts as a referee. It doesn't connect
to a blaster, doesn't appear as a player, and offers central control over game
mode, limits, GPS settings, allow-join, and per-player settings.

1. From the connect screen, tap **Dedicated Server** instead of connecting to
   a blaster.
2. The IP is shown at the top. Players join from their phones via
   **Use Networking → Join**.
3. Set game mode, limits, etc. — these are pushed to all clients on every
   change, so the lobby stays in sync.
4. **Start Game**.

The dedicated server enforces the time and score limits automatically. The
**End Game** button is always available as a manual override.

## Compatibility

- Android 5.0 (API 21) minimum, Android 15 target
- Tested by the fork maintainer on Android 14+. The pre-Android-12 permission
  fallback path is in place but has not been re-tested on the upstream's
  original API 21 floor.

## License

Apache License 2.0, same as upstream. See `LICENSE`.
