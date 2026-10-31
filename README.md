> 📚 **Dokumentasi:** [Indeks lengkap ID/EN](DOCS.md) · [English documentation index](DOCS.en.md)

# GSG-MoBas — IndoTech Absensi V3 Mobile


Aplikasi mobile Flutter untuk absensi, verifikasi lokasi, foto absensi, pengajuan izin, lembur, dan tukar shift.

- **Nama aplikasi:** GSG-MoBas
- **Package Flutter:** `gsg_presensi`
- **Versi aplikasi:** `1.30.32+1332`
- **Platform utama:** Android dan iOS

## Kebutuhan

- Flutter `3.44.6` atau versi yang kompatibel
- Dart `3.12.2` atau versi yang memenuhi constraint `sdk: ^3.12.2`
- Android Studio, Android SDK, dan JDK 17 untuk build Android
- Xcode dan CocoaPods untuk build iOS
- Perangkat Android/emulator atau iPhone/simulator yang sudah dikonfigurasi

## Instalasi dan menjalankan

1. Clone repository private ini.
2. Masuk ke direktori project.
3. Ambil dependency Flutter:

   ```bash
   flutter pub get
   ```

4. Pastikan perangkat tujuan sudah terdeteksi, kemudian jalankan aplikasi:

   ```bash
   flutter devices
   flutter run
   ```

5. Menjalankan pemeriksaan sebelum commit:

   ```bash
   flutter analyze
   flutter test
   ```

Dependency dapat diperbarui dengan `flutter pub upgrade --major-versions`, tetapi setiap perubahan dependency harus diverifikasi ulang menggunakan analyze dan test.

## Build APK release

```bash
flutter build apk --release
```

Output APK berada di `build/app/outputs/flutter-apk/`. Untuk build production, siapkan secret signing berikut di mesin builder:

- `android/key.properties`
- keystore release (format `.jks` atau `.keystore`)

File secret tersebut tidak boleh dimasukkan ke repository. Flutter Gradle Plugin membaca `versionName` dan `versionCode` dari versi pada `pubspec.yaml`, sehingga untuk rilis dengan build number berbeda gunakan opsi Flutter, misalnya:

```bash
flutter build apk --release --build-name 1.30.32 --build-number 1332
```

## Konfigurasi API

Base URL API saat ini dibaca dari dua konstanta di source code:

- `lib/services/api_service.dart` — `ApiService.baseUrl`
- `lib/services/widget_background_service.dart` — `WidgetBackgroundService._baseUrl`

Nilai produksi yang digunakan adalah:

```text
https://presensi.grahasaranagresik.com/api/v3
```

Project ini belum membaca API URL dari `.env` atau file konfigurasi eksternal. Jika host API diganti, perbarui kedua konstanta tersebut dan gunakan HTTPS. Manifest Android mengizinkan cleartext `false`, sedangkan ATS iOS dibuat strict; jangan menambahkan exception HTTP tanpa kebutuhan yang jelas.

## Deteksi root/jailbreak

Pemeriksaan keamanan perangkat dijalankan saat splash screen oleh `SplashScreen._checkJailbreak()` menggunakan package `flutter_jailbreak_detection`.

- Plugin yang tidak tersedia atau platform yang tidak mendukung deteksi dianggap tidak terdeteksi agar aplikasi tidak crash.
- `PlatformException` atau error tak terduga pada pemeriksaan lain diperlakukan sebagai kondisi tidak aman dan aplikasi berhenti dengan layar blokir.
- `PackageInfo.fromPlatform()` memiliki fallback versi, sedangkan pembacaan `device_info_plus`/`AndroidId` memiliki fallback agar request perangkat tidak membuat startup crash.

Perilaku ini bukan pengganti MDM atau verifikasi integritas server; endpoint tetap harus memvalidasi timestamp, lokasi, perangkat, dan hak akses di server.

## Struktur folder

```text
.
├── android/                 # Gradle, manifest, permission, dan konfigurasi Android
├── assets/                  # Aset lokal aplikasi (logo, gambar, animasi)
├── ios/                     # Project iOS dan Info.plist
├── lib/
│   ├── main.dart            # Entry point, theme, dan routing aplikasi
│   ├── screens/             # Splash, login, dashboard, absensi, izin, dan screen lainnya
│   ├── services/            # API, cache, auth, notifikasi, sinkronisasi, dan update
│   ├── widgets/             # Komponen UI reusable
│   └── utils/               # Helper database, modal, dan utility aplikasi
├── test/                    # Widget test dan test aplikasi
├── pubspec.yaml             # Dependency dan versi aplikasi
├── analysis_options.yaml    # Konfigurasi analyzer/lint
└── README.md
```

## Platform security

- Android mendeklarasikan `INTERNET`, `allowBackup=false`, `fullBackupContent=false`, dan `usesCleartextTraffic=false`.
- iOS memiliki deskripsi permission lokasi, kamera, galeri, dan Face ID.
- Semua asset aplikasi dienkode atau diambil dari bundle lokal; runtime API dan tile map menggunakan HTTPS.
- `android/local.properties`, `android/key.properties`, keystore, `.env`, build output, IDE metadata, dan screenshot debug tidak boleh di-commit.

## Test dan verifikasi rilis

```bash
flutter analyze
flutter test
flutter build apk --release
```

`flutter test` harus lulus sebelum build release. Build Android penuh juga bergantung pada availability Android SDK/NDK dan signing keystore pada mesin builder.

## Firebase & Push Notification

Project `presensi-online-pt-gsg` (nomor `108247760935`).

| Platform | Package / Bundle ID | App ID |
|----------|--------------------|--------|
| Android | `com.ptgrahasaranagresik.gsg_presensi` | `1:108247760935:android:1b118bd5aae215a7e00e02` |
| iOS | `com.ptgrahasaranagresik.gsgPresensi` | `1:108247760935:ios:7debdaf28d6fb2f5e00e02` |

**Inisialisasi.** `lib/main.dart` memakai `DefaultFirebaseOptions.currentPlatform`
dari `lib/firebase_options.dart`, lalu jatuh ke konfigurasi native bila options
tidak tersedia. Artinya FCM tetap berjalan **tanpa** file tambahan; bila
`android/app/google-services.json` diletakkan, plugin Google Services
(`android/settings.gradle.kts`) otomatis ikut diterapkan. Verifikasi:

```bash
cd android && ./gradlew :app:processDebugGoogleServices
# resource google_app_id harus berisi 1b118bd5aae215a7e00e02
```

**Token device.** `ApiService.registerDeviceToken()` mengirim token FCM ke
`POST /api/v3/device-tokens` memakai auth token yang sudah login, dan hanya
mengirim ulang bila token berubah (rotasi ditangani lewat `onTokenRefresh`).

**Pengingat lokal (alarm).** `NotificationSchedulerService`:

- channel `alarm_channel` (dipakai payload FCM server), `default`,
  `schedule_reminders` (alarm absen), `success_notifications`,
  `error_notifications`;
- izin `SCHEDULE_EXACT_ALARM` + permintaan runtime — tanpa ini Android 12+
  menolak semua pengingat;
- mode `exactAllowWhileIdle` dengan fallback `inexactAllowWhileIdle`;
- pengingat pulang menghormati shift lintas tengah malam (`next_day`).

Aturan waktunya sama dengan versi web (lihat `resources/js/reminders.js` di
repo web): masuk −30/−20/−10 menit, tepat mulai, +10/+20/+30; pulang +15/+30
hanya bila sudah absen masuk dan belum absen pulang.

## Verifikasi rilis

```bash
flutter analyze --no-fatal-infos   # harus 0 error / 0 warning
flutter test                       # 5 test
cd android && ./gradlew :app:processDebugGoogleServices
```

Status saat push terakhir: **analyze 0 error/0 warning, 5 test lulus,
Gradle Firebase config BUILD SUCCESSFUL.**

---

## Akses & Lisensi

Repositori ini adalah **karya saya sendiri**, **Mochamad Ismail Soleh** (Ismail-IndoTech), dan dilisensikan secara khusus di bawah **[MIT License](LICENSE)**.

Untuk memperoleh akses ke **repositori sumber (privat)** dari proyek ini, silakan hubungi saya sebagai **Creator Repo** melalui WhatsApp di nomor yang terdaftar pada akun DANA saya: **08121600058** — [wa.me/628121600058](https://wa.me/628121600058). Akses diberikan sesuai kesepakatan.
## 💖 Kontribusi & Donasi

Jika aplikasi ini mempermudah operasional IT, DevOps, dan pengelolaan server di perusahaan atau proyek pribadi Anda, dukung pengembangannya dengan menjadi kontributor atau **Donatur Spesial**.  
*(Donatur dan Sponsor resmi akan kami cantumkan logo perusahaannya di repositori ini).*

Dukung pemeliharaan berkelanjutan, riset modul bypass panel baru, dan pembaruan sistem dengan berdonasi melalui:

<div align="center">

**DANA — 08121600058** (a.n. Mochamad Ismail Soleh)

<img src="docs/qris.jpg" alt="QRIS DANA IndoTech" width="280">

*Pindai dengan aplikasi DANA, BCA, Mandiri, GoPay, OVO, atau aplikasi pembayaran QRIS apa pun di Indonesia.*

</div>

Semua dana dialokasikan untuk biaya infrastruktur riset server, lisensi alat pengujian, dan pengembangan modul open-source IndoTech. Jika Anda atau institusi Anda ingin menjadi mitra resmi atau sponsor, silakan hubungi kami. Terima kasih atas dukungan Anda!