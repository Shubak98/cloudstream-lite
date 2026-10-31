# CloudStream TV-Lite (v4.8.0 base)

A trimmed-down build of CloudStream for low-end Android TV devices.
Base: upstream commit `5024ace2922a148d38df2a816b483b529789cf58` (v4.8.0).

## What was removed / changed

| # | Change | Effect |
|---|--------|--------|
| 1 | **Built-in torrent server removed** (TorrServer / `libgojni.so`) | −12.4 MB native lib per ABI, no background torrent service. Magnet links show an info message; "Open with → external torrent app" (BiglyBT/LibreTorrent) still works |
| 2 | **x86 / x86_64 native libs removed** (ARM-only ABI splits) | −50 MB uncompressed; CI produces `arm64-v8a`, `armeabi-v7a` and `universal` (ARM) APKs |
| 3 | **Chromecast removed** (`media3-cast`, play-services-cast, cast UI/sessions/mini-controller, Chromecast subtitle settings) | −3–4 MB code, no Google Play Services cast discovery running in background |
| 4 | **Video downloads removed** (download services, queue service, restart receiver, download episode actions) | No download services can ever start. Playing/deleting already-downloaded files still works |
| 5 | **Trailers disabled + NewPipe Extractor removed** | No trailer fetching (network/CPU) on result pages |
| 6 | **Tracking unregistered** (AniList, MAL, Kitsu, Simkl) | No sync providers in settings, no sync network calls. Local watchlist + progress tracking fully kept |
| 7 | **Seekbar preview removed** (`previewseekbar`, frame-decoding preview generator) | Less RAM/CPU in player. Live-edge seekbar behavior kept |
| 8 | **Biometric app-lock permanently disabled** | Was already hidden on TV; now can never activate (e.g. from a restored backup) |
| 9 | **All auto-update checks default to OFF** (app updates + plugin auto-updates) | No background update checks/downloads. Manual checking still available in Settings → Updates |
| 10 | **App ID changed to `com.lagradost.cloudstream3.tvlite`**, version suffix `-TVLITE` | Installs **alongside** the official app — no uninstall needed, easy rollback |

### Kept intact
- Extension/plugin system (repositories, providers, extractors) — **the `library` module is untouched, so all existing extensions keep working**
- Subtitle providers (OpenSubtitles, Addic7ed, SubSource, Subdl) + subtitle downloading + anime-skip
- Local watchlist / watch history / accounts (local)
- In-app updater code (manual "check for update" possible, but see note below)
- WorkManager jobs (backups, plugin subscriptions) — but auto-update defaults are OFF
- Android TV "Watch Next" integration, fcast (Web Video Caster / external cast actions)

### Expected results
- APK download: **~72 MB → ~33 MB** (arm64) / ~38 MB (universal ARM)
- Install footprint: **~167 MB → ~76 MB**
- Runtime: fewer background services, no cast/GMS discovery, no trailer fetching, no preview decoding, no auto-update checks → less RAM pressure and fewer stutters on 1–2 GB TVs

## How to build (no PC needed — GitHub Codespaces + Actions)

1. **Fork** `recloudstream/cloudstream` on GitHub.
2. In your fork page: green **Code** button → **Codespaces** tab → **Create codespace on master** (free tier is enough).
3. Download `cloudstream-tv-lite.patch` (provided alongside this file) and **drag & drop it into the Codespaces file explorer** (into the repo root folder).
4. In the Codespaces terminal:
   ```bash
   git checkout -b tv-lite 5024ace2922a148d38df2a816b483b529789cf58
   git apply --whitespace=nowarn cloudstream-tv-lite.patch
   git add -A
   git commit -m "CloudStream TV-Lite v4.8.0"
   git push -u origin tv-lite
   ```
5. Go to your fork on GitHub → **Actions** tab (enable workflows if asked) → **TV-Lite Build** → **Run workflow** → branch `tv-lite`.
6. After ~15–30 min, open the workflow run → **Artifacts** → download `cloudstream-tv-lite`.
7. Inside the zip:
   - `app-stable-arm64-v8a-release.apk` ← **use this on virtually any Android TV made since ~2017**
   - `app-stable-armeabi-v7a-release.apk` ← only if the arm64 one refuses to install (very old device)
   - `app-stable-universal-release.apk` ← works on any ARM device (bigger)

Transfer the APK to your TV (USB stick, "Send files to TV" app, or `adb install`) and install.

### Alternative: build on your own PC
Requirements: JDK 17 + Android SDK (compileSdk 37) + ~8 GB RAM.
```bash
git clone https://github.com/<you>/cloudstream && cd cloudstream
git checkout -b tv-lite 5024ace2922a148d38df2a816b483b529789cf58
git apply --whitespace=nowarn /path/to/cloudstream-tv-lite.patch
./gradlew assembleStableRelease
# APKs: app/build/outputs/apk/stable/release/
```

## Install & migration notes

- **Both apps coexist** (different app IDs). To move your data: in the official app go to
  *Settings → Backup & Restore → create a backup*, then in TV-Lite *restore* that backup.
  (Restore skips removed features automatically; downloads/tracking data is simply not restored.)
- Once TV-Lite runs well, you can uninstall the official app to free RAM/storage.
- **Updating:** the in-app updater points at official releases, whose signature differs — do **not**
  use it to update TV-Lite (install would fail). To update: re-run the **TV-Lite Build** workflow on
  an updated base and install the new APK over the old one (same signing key = seamless update).

## Security note about the signing key

`tv-lite-keystore.p12` (password/alias: `tv-lite`) is committed so that every CI build produces
APKs that can update each other. Since your fork is public, **anyone could sign an APK that your
device would accept as an update to TV-Lite**. Only install TV-Lite APKs that **your own** Actions
run produced. If that bothers you, delete the `.p12` from the repo and instead store it as a
GitHub Actions secret and adapt the workflow + `signingConfigs` accordingly.

## Troubleshooting

- **Build fails on the `splits { abi { ... } }` block** (very future AGP versions): replace that
  block in `app/build.gradle.kts` with `ndk { abiFilters += listOf("armeabi-v7a", "arm64-v8a") }`
  inside `defaultConfig` — you then get a single universal ARM APK.
- **`git apply` reports conflicts:** upstream moved past the base commit. Check out the exact base
  SHA shown above first (step 4 does this). If it still fails, the patch needs a rebase.
- **Only run `assembleStableRelease`** — unit tests were not updated for the removals, so
  `./gradlew build` / `check` will fail on test compilation; this does not affect the APK.
- TV still stutters? Check *Settings → Extensions*: many installed repositories/extensions are the
  most common cause of lag on low-RAM TVs. Keep only the ones you use.
