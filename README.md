# Sukkot Sidur (Android)

A Kotlin/Jetpack Compose Android app: the full Ashkenaz Sukkot liturgy (Hebrew, right-to-left),
location-based zmanim (halachic times), and reminders.

## How to open and build

1. Install **Android Studio** (Koala/2024.1 or newer).
2. Open this folder as a project — File → Open → select `SukkotSidur/`.
3. Android Studio will detect there's no Gradle wrapper jar and offer to create one automatically
   ("Import Project" / "Sync Now" will prompt you). Accept it, or run `gradle wrapper --gradle-version 8.7`
   once from a terminal if you have a system-wide Gradle install. (The wrapper *jar* is a binary that
   couldn't be generated in this environment — everything else needed to build is here.)
4. Let Gradle sync — it will download Compose, Retrofit, Room, KosherJava's zmanim library, and
   Play Services Location from the repositories declared in `settings.gradle.kts`.
5. Run on a device or emulator (minSdk 26 / Android 8.0+).

## Architecture

- **Liturgy text** (`data/`): rather than hardcoding the Hebrew text of every Sukkot prayer (which
  risks transcription errors and drifts if Sefaria reorganizes its library), `SiddurRepository`
  fetches Sefaria's **schema/index** for "Siddur Ashkenaz" at runtime, walks down through
  `Festivals → Sukkot`, and treats every leaf node it finds (Ushpizin, the Sukkot Kiddush, each
  day's Hoshanot, the Sukkot Amidah insertions, etc.) as one section. It then fetches each
  section's Hebrew text from Sefaria's `/api/v3/texts/{ref}` endpoint and caches it in a local
  Room database, so the siddur works fully offline after the first successful sync. The Hebrew
  liturgical text itself is in the public domain; Sefaria's own database export is
  distributed under a mix of public-domain and CC-BY-SA terms depending on the specific edition
  used — check **sefaria.org/about** and add an in-app "Text courtesy of Sefaria.org" attribution
  before publishing, which is good practice regardless of license.
- **Zmanim** (`zmanim/ZmanimRepository.kt`): wraps the open-source **KosherJava Zmanim** library.
  Given latitude/longitude (and optionally elevation) it returns sunrise, sunset, candle
  lighting (18 minutes before sunset), tzeit hakochavim, chatzot, and the Shema/Amidah cutoff
  times. **Double-check the exact method names against the KosherJava version you land on** —
  the library was going through a naming migration; the names used here (`sunrise`, `sunset`,
  `sofZmanShmaGRA`, `sofZmanTfilaGRA`, `chatzos`, `candleLighting`, `alos72`,
  `tzaisGeonim72Min`) match the stable 2.x API (pinned to `2.5.0` here).
- **Location** (`location/LocationHelper.kt`): one-shot current location via
  `FusedLocationProviderClient`, used to feed the zmanim calculation.
- **Reminders** (`notifications/`): `ReminderScheduler` sets exact alarms via `AlarmManager`
  (falling back to inexact alarms if the user hasn't granted the Android 12+ exact-alarm
  permission) and persists the reminder list in DataStore so `BootReceiver` can restore them
  after a phone reboot, since Android clears alarms on reboot.
- **UI** (`ui/`): `HomeScreen` shows today's zmanim plus the list of Sukkot sections;
  `PrayerScreen` displays a section's Hebrew text right-to-left.

## What's scaffolded vs. what to finish

This is a complete, working skeleton — not a finished, store-ready app. Before you rely on it:

- **Test the Sefaria index-walking logic against the live API.** I confirmed via search that
  Sefaria's "Siddur Ashkenaz" work has a `Festivals → Sukkot` branch containing sections like
  the Hoshanot for each day, but I did not execute live API calls against it, so the exact node
  titles and JSON shape should be verified once you can run the app with network access.
  `SiddurRepository.kt` is written defensively (falls back to cache, `runCatching` around each
  fetch) but you may need to adjust `findChild`/`collectLeaves` once you see the real payload.
- **Add a real app icon** — the placeholder in `res/drawable/ic_launcher_foreground.xml` is a
  rough hourglass shape, not real artwork. Use Android Studio's Image Asset tool
  (right-click `res` → New → Image Asset).
- **Decide on a Torah-reading module.** Sukkot davening includes a Torah reading; that's a
  separate Sefaria work (Tanakh, not Siddur) and isn't wired up yet.
- **Sukkot-specific "date awareness."** Right now the section list always shows every Sukkot
  leaf found (all 7 days of Hoshanot, etc.) rather than just today's. Consider computing the
  current Hebrew date (KosherJava's `JewishCalendar` class does this) and filtering/highlighting
  the section for today.
- **Reminder UI.** The Home screen only wires up one example reminder button ("30 min before
  candle lighting"). A dedicated settings screen for arbitrary reminders (per zman, custom
  offsets) is a natural next step — `ReminderScheduler`/`ReminderStore` already support any
  arbitrary label + trigger time.
