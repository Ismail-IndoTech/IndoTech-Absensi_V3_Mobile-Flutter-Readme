> 📚 **Documentation:** [Full ID/EN index](DOCS.md) · [English index](DOCS.en.md)

# GSG-MoBas — IndoTech Absensi V3 Mobile


A Flutter mobile app for attendance, location verification, attendance photos, leave requests, overtime, and shift swaps.

- **App name:** GSG-MoBas
- **Flutter package:** `gsg_presensi`
- **App version:** `1.30.32+1332`
- **Primary platforms:** Android and iOS

## Requirements

- Flutter `3.44.6` or a compatible version
- Dart `3.12.2` or a version that satisfies the `sdk: ^3.12.2` constraint
- Android Studio, Android SDK, and JDK 17 for Android builds
- Xcode and CocoaPods for iOS builds
- A configured Android device/emulator or iPhone/simulator

## Installation and running

1. Clone this private repository.
2. Enter the project directory.
3. Fetch the Flutter dependencies:

   ```bash
   flutter pub get
   ```

4. Make sure the target device is detected, then run the app:

   ```bash
   flutter devices
   flutter run
   ```

5. Run the pre-commit checks:

   ```bash
   flutter analyze
   flutter test
   ```

Dependencies can be updated with `flutter pub upgrade --major-versions`, but every dependency change must be re-verified using analyze and test.

## Build release APK

```bash
flutter build apk --release
```

The APK output is located in `build/app/outputs/flutter-apk/`. For production builds, prepare the following signing secrets on the builder machine:

- `android/key.properties`
- release keystore (`.jks` or `.keystore` format)

These secret files must not be committed to the repository. The Flutter Gradle Plugin reads `versionName` and `versionCode` from the version in `pubspec.yaml`, so for a release with a different build number use the Flutter options, for example:

```bash
flutter build apk --release --build-name 1.30.32 --build-number 1332
```

## API Configuration

The API base URL is currently read from two constants in the source code:

- `lib/services/api_service.dart` — `ApiService.baseUrl`
- `lib/services/widget_background_service.dart` — `WidgetBackgroundService._baseUrl`

The production value used is:

```text
https://presensi.grahasaranagresik.com/api/v3
```

This project does not yet read the API URL from `.env` or an external configuration file. If the API host changes, update both constants and use HTTPS. The Android manifest allows cleartext `false`, while iOS ATS is made strict; do not add HTTP exceptions without a clear need.

## Root/jailbreak detection

The device security check is run at the splash screen by `SplashScreen._checkJailbreak()` using the `flutter_jailbreak_detection` package.

- An unavailable plugin or a platform that does not support detection is treated as undetected so the app does not crash.
- `PlatformException` or an unexpected error in other checks is treated as an unsafe condition and the app stops with a blocking screen.
- `PackageInfo.fromPlatform()` has a version fallback, while `device_info_plus`/`AndroidId` reads have a fallback so device requests do not cause startup crashes.

This behavior is not a substitute for MDM or server integrity verification; the endpoint must still validate the timestamp, location, device, and access rights on the server.

## Folder structure

```text
.
├── android/                 # Gradle, manifest, permissions, and Android configuration
├── assets/                  # Local app assets (logo, images, animations)
├── ios/                     # iOS project and Info.plist
├── lib/
│   ├── main.dart            # Entry point, theme, and app routing
│   ├── screens/             # Splash, login, dashboard, attendance, leave, and other screens
│   ├── services/            # API, cache, auth, notifications, sync, and update
│   ├── widgets/             # Reusable UI components
│   └── utils/               # Database helpers, modals, and app utilities
├── test/                    # Widget tests and app tests
├── pubspec.yaml             # Dependencies and app version
├── analysis_options.yaml    # Analyzer/lint configuration
└── README.md
```

## Platform security

- Android declares `INTERNET`, `allowBackup=false`, `fullBackupContent=false`, and `usesCleartextTraffic=false`.
- iOS has permission descriptions for location, camera, gallery, and Face ID.
- All app assets are encoded or taken from the local bundle; runtime API and map tiles use HTTPS.
- `android/local.properties`, `android/key.properties`, keystores, `.env`, build output, IDE metadata, and debug screenshots must not be committed.

## Test and release verification

```bash
flutter analyze
flutter test
flutter build apk --release
```

`flutter test` must pass before a release build. A full Android build also depends on the availability of the Android SDK/NDK and the signing keystore on the builder machine.

## Firebase & Push Notification

Project `presensi-online-pt-gsg` (number `108247760935`).

| Platform | Package / Bundle ID | App ID |
|----------|--------------------|--------|
| Android | `com.ptgrahasaranagresik.gsg_presensi` | `1:108247760935:android:1b118bd5aae215a7e00e02` |
| iOS | `com.ptgrahasaranagresik.gsgPresensi` | `1:108247760935:ios:7debdaf28d6fb2f5e00e02` |

**Initialization.** `lib/main.dart` uses `DefaultFirebaseOptions.currentPlatform`
from `lib/firebase_options.dart`, then falls back to the native configuration if
the options are unavailable. This means FCM still runs **without** an additional
file; if `android/app/google-services.json` is provided, the Google Services
plugin (`android/settings.gradle.kts`) is automatically applied as well. Verify:

```bash
cd android && ./gradlew :app:processDebugGoogleServices
# resource google_app_id must contain 1b118bd5aae215a7e00e02
```

**Device token.** `ApiService.registerDeviceToken()` sends the FCM token to
`POST /api/v3/device-tokens` using the already logged-in auth token, and only
resends when the token changes (rotation is handled via `onTokenRefresh`).

**Local reminders (alarms).** `NotificationSchedulerService`:

- channel `alarm_channel` (used by server FCM payloads), `default`,
  `schedule_reminders` (attendance alarms), `success_notifications`,
  `error_notifications`;
- `SCHEDULE_EXACT_ALARM` permission + runtime request — without this, Android 12+
  rejects all reminders;
- mode `exactAllowWhileIdle` with an `inexactAllowWhileIdle` fallback;
- clock-out reminders respect shifts crossing midnight (`next_day`).

The timing rules are the same as the web version (see `resources/js/reminders.js` in the web
repo): clock-in −30/−20/−10 minutes, at start, +10/+20/+30; clock-out +15/+30
only if clock-in has been recorded and clock-out has not.

## Release verification

```bash
flutter analyze --no-fatal-infos   # must be 0 error / 0 warning
flutter test                       # 5 tests
cd android && ./gradlew :app:processDebugGoogleServices
```

Status at the last push: **analyze 0 error/0 warning, 5 tests passing,
Gradle Firebase config BUILD SUCCESSFUL.**

---

## Access & License

This repository is **my own work**, **Mochamad Ismail Soleh** (Ismail-IndoTech), and is specifically licensed under the **[MIT License](LICENSE)**.

To obtain access to the **private source repository** for this project, contact me as the **Repository Creator** via WhatsApp using the number registered to my DANA account: **08121600058** — [wa.me/628121600058](https://wa.me/628121600058). Access is granted by agreement.
## 💖 Contribution & Donation

If this application makes IT operations, DevOps, and server management easier for your company or personal project, support its development by becoming a contributor or a **Special Donor**.
*(We will list the company logos of official Donors and Sponsors in this repository).*

Support ongoing maintenance, research into new panel bypass modules, and system updates by donating through:

<div align="center">

**DANA — 08121600058** (a.n. Mochamad Ismail Soleh)

<img src="docs/qris.jpg" alt="IndoTech DANA QRIS" width="280">

*Scan with the DANA, BCA, Mandiri, GoPay, OVO app, or any QRIS payment app in Indonesia.*

</div>

All funds are allocated for server research infrastructure costs, testing tool licenses, and the development of IndoTech open-source modules. If you or your institution would like to become an official partner or sponsor, please contact us. Thank you for your support!