# HabitLab

![Kotlin](https://img.shields.io/badge/Kotlin-2.2-7F52FF)
![Jetpack Compose](https://img.shields.io/badge/Jetpack%20Compose-Material%203-4285F4)
![Android](https://img.shields.io/badge/Android-minSdk%2024-3DDC84)
![Room](https://img.shields.io/badge/Room-2.7-blue)

A native, fully offline Android app for tracking daily habits, written in Kotlin with Jetpack Compose. Users create habits with a name, icon and color, tick them off once per day, and follow their progress through streaks, a daily and weekly completion rate, monthly statistics and an activity heatmap. The project demonstrates a clean **MVVM** architecture with the **Repository pattern**, an API-first design (every component is defined by an interface first), reactive data flow with Kotlin `Flow`/`StateFlow`, local persistence with Room (SQLite), manual dependency injection, and a three-level test suite (unit, integration, end-to-end).

HabitLab is structured like a production Gradle project so you can build, test and install it easily.


## Contents
- Overview
- Features
- Domain rules
- The app (Dashboard, Habits, Statistik)
- Architecture and data flow
- Database model
- Tech stack
- Project layout
- Getting started
  - Prerequisites
  - Build
  - Run on a device or emulator
  - Run unit tests
  - Run integration and end-to-end tests
- Tests
- Troubleshooting
- Contributing
- Author


## Overview
All code lives under the package `com.hudnah.habitlab`:
- `data.local` — The Room database `AppDatabase` (file `habitlab.db`), the entities `Habit`, `Entry` and `Streak` and their DAOs `HabitDao`, `EntryDao` and `StreakDao`.
- `data.repository` — One repository per component, each implementing a service interface: `HabitRepository`, `EntryRepository`, `StreakRepository` and `StatsRepository`.
- `domain` — The service interfaces (`IHabitService`, `IEntryService`, `IStreakService`, `IStatisticsService`), the stats models (`WeeklyStats`, `MonthlyStats`, `HeatmapDay`, `HabitWithProgress`) and the pure streak logic `StreakCalculator`.
- `di` — Manual dependency injection: `AppContainer` wires database → repositories → services; `HabitLabClock` is the app-wide, replaceable source of "today".
- `ui` — Jetpack Compose screens with their ViewModels (`DashboardViewModel`, `HabitViewModel`, `StatsViewModel`), bottom navigation (`HabitLabApp`), icons and colors (`HabitVisuals`), the theme and the test tags used by the UI tests.

`HabitLabApplication` creates the `AppContainer` once at start-up; `MainActivity` hands it to the Compose UI.


## Features
- Create, edit, archive and delete habits, each with a name, one of eight icons and one of ten colors
- Mark a habit as done for today with one tap, and undo it with another
- Streaks: current streak and best streak per habit, recalculated and saved after every check-in or undo
- Dashboard with today's progress (e.g. `2/3 erledigt`) and the current streak of every habit
- Statistics: completion rate of the current week, completions this month per habit, all-time completions per habit
- Activity heatmap of the last four weeks, shaded by the share of habits completed each day
- Fully offline: all data is stored locally in a Room (SQLite) database; the app needs no internet permission
- Reactive UI: screens observe the database through `Flow`/`StateFlow` and update immediately after every change
- Tests on three levels: JVM unit tests with MockK, integration tests against an in-memory Room database, and end-to-end tests that drive the real Compose UI


## Domain rules
A few rules the code enforces (and the tests check):
- **Habit name:** must not be empty or blank; creating or updating a habit with a blank name is rejected (`HabitRepository` throws `IllegalArgumentException`, the UI ignores the input).
- **One entry per habit and day:** the table `entry` has a `UNIQUE (habitId, date)` constraint, and `EntryRepository.createEntry` is idempotent: completing a habit twice on the same day returns the existing entry instead of creating a second one.
- **Current streak:** the number of consecutive days up to and including today. If today has no entry, the current streak is 0, so a single missed day breaks it.
- **Best streak:** the longest run of consecutive days ever reached. The stored best streak never decreases, even if entries are removed later.
- **Undo:** unchecking a habit deletes today's entry and recalculates the streak.
- **Archive:** an archived habit disappears from the dashboard, the habit list and all statistics, but its data stays in the database.
- **Delete:** deleting a habit also deletes its entries and its streak (`ON DELETE CASCADE`).
- **Rates:** today's progress is completed habits / active habits; the weekly rate is completions this week (Monday to Sunday) / (active habits × 7). With no active habits, every rate is 0 (no division by zero).
- **Days** are stored as epoch days (`LocalDate.toEpochDay()`), so a day is always a calendar day in the device's time zone.


## The app (Dashboard, Habits, Statistik)
The app has three screens, reached through the navigation bar at the bottom. The user interface is in German.

- **Dashboard** — the start screen, headed "Heute": today's progress as a counter and progress bar, and every active habit with its icon, its current streak (🔥) and a button to mark it **Erledigt** or set it back to **Offen**.
- **Habits** — the list of all active habits with check-in, **edit**, **archive** (A) and **delete** per habit. The **Neu** button opens a dialog to enter the name and pick an icon (reading, cycling, coding, running, sport, fitness, meditation, other) and a color.
- **Statistik** — the weekly completion rate and number of completions, completions this month per habit, all-time completions per habit, and the activity heatmap of the last four weeks.

Every switch between screens rebuilds the screen, so all numbers are recalculated for the current day.

**Data.** All data is saved automatically in the app's private storage (`habitlab.db`). Uninstalling the app or clearing its storage deletes all data. The database schema is exported to `app/schemas/` (version 1); if an installed database has an incompatible schema, it is rebuilt from scratch instead of crashing the app.


## Architecture and data flow
The app follows **Model-View-ViewModel (MVVM)** with the **Repository pattern**. Following the API-first principle, every component is defined by an interface in `domain.service` first; the ViewModels depend only on these interfaces, never on Room.

| Layer | Classes | Responsibility |
|---|---|---|
| View | `DashboardScreen`, `HabitsScreen`, `StatsScreen` | Display state, forward user actions |
| ViewModel | `DashboardViewModel`, `HabitViewModel`, `StatsViewModel` | Hold UI state as `StateFlow`, run use cases in `viewModelScope` |
| Service / Repository | `HabitRepository`, `EntryRepository`, `StreakRepository`, `StatsRepository` | Business logic and validation behind the `I…Service` interfaces |
| Data | `AppDatabase`, `HabitDao`, `EntryDao`, `StreakDao` | Room entities and queries on SQLite |

Reactive data flow:
```
User action ➔ ViewModel ➔ Service (Repository) ➔ DAO ➔ Room (SQLite)
                 ▲                                          │
                 └──── Flow / StateFlow ◄───────────────────┘ ➔ UI update
```

The current date comes from an injected `today: () -> LocalDate` function (`HabitLabClock` in the app). This keeps all date logic testable: tests can simulate several days in a row without waiting.


## Database model
The database `AppDatabase` (version 1) has three tables. Foreign keys use `ON DELETE CASCADE`, so removing a habit removes everything that belongs to it.

```
[ habit ] 1 ───────── N [ entry ]    at most one entry per habit and day
    │ 1
    │
    └──────────────── 1 [ streak ]   saved streak state, one per habit
```

**`habit`** — the habits defined by the user.

| Column | Type | Notes |
|---|---|---|
| `id` | Long | primary key, auto-generated |
| `name` | String | display name, not blank |
| `iconKey` | String | key of the icon, e.g. `reading` |
| `colorHex` | String | color, e.g. `#42A5F5` |
| `isArchived` | Boolean | default `false` |
| `createdAt` | Long | creation time (epoch ms) |

**`entry`** — one row per habit and completed day.

| Column | Type | Notes |
|---|---|---|
| `id` | Long | primary key, auto-generated |
| `habitId` | Long | foreign key → `habit(id)`, `ON DELETE CASCADE` |
| `date` | Long | calendar day (epoch day) |
| `completedAt` | Long | time of completion (epoch ms) |
| `status` | String | `DONE` |

Constraint: `UNIQUE (habitId, date)`.

**`streak`** — the saved streak of each habit (1:1).

| Column | Type | Notes |
|---|---|---|
| `id` | Long | primary key, auto-generated |
| `habitId` | Long | foreign key → `habit(id)`, `ON DELETE CASCADE`, unique |
| `currentStreak` | Int | current streak in days, default 0 |
| `bestStreak` | Int | best streak in days, default 0 |
| `lastActiveDate` | Long? | last day with an entry (epoch day) |
| `updatedAt` | Long | last update (epoch ms) |


## Tech stack
- Language: Kotlin 2.2 (JVM target 11)
- UI: Jetpack Compose with Material 3, Navigation Compose, Material Icons Extended
- Architecture: MVVM + Repository, AndroidX Lifecycle (ViewModel, `StateFlow`)
- Persistence: Room 2.7 (SQLite), annotation processing with KSP
- Concurrency: Kotlin Coroutines and `Flow`
- Build: Gradle 9.4 (Gradle wrapper included), Android Gradle Plugin 9.2, version catalog in `gradle/libs.versions.toml`
- Android: `minSdk` 24 (Android 7.0), `targetSdk` 36, `compileSdk` 37
- Testing: JUnit 4, MockK, kotlinx-coroutines-test, Turbine, AndroidX Test, Compose UI Test, Room Testing

See `gradle/libs.versions.toml` and `app/build.gradle.kts` for exact versions.


## Project layout
```
/ (project root)
├─ gradlew, gradlew.bat, gradle/wrapper/   # Gradle wrapper
├─ gradle/libs.versions.toml               # Version catalog (dependencies and plugins)
├─ build.gradle.kts, settings.gradle.kts, gradle.properties
├─ app
│  ├─ build.gradle.kts
│  ├─ schemas/                             # Exported Room schema (version 1)
│  └─ src
│     ├─ main
│     │  ├─ AndroidManifest.xml
│     │  ├─ java/com/hudnah/habitlab
│     │  │  ├─ MainActivity.kt, HabitLabApplication.kt
│     │  │  ├─ data/local/                 # AppDatabase, entities, DAOs
│     │  │  ├─ data/repository/            # Habit, Entry, Streak and Stats repositories
│     │  │  ├─ domain/                     # Service interfaces, stats models, StreakCalculator
│     │  │  ├─ di/                         # AppContainer, HabitLabClock
│     │  │  └─ ui/                         # Compose screens, ViewModels, navigation, theme
│     │  └─ res/                           # Launcher icons, strings, themes
│     ├─ test/java/com/hudnah/habitlab/    # JVM unit tests
│     └─ androidTest/java/com/hudnah/habitlab
│        ├─ integration/                   # Repositories + in-memory Room database
│        └─ e2e/                           # Full app through the Compose UI
└─ README.md
```


## Getting started

### Prerequisites
- JDK 21 (the Gradle daemon is configured for Java 21; Android Studio's bundled JDK works)
- Android SDK with platform 37 installed, e.g. via Android Studio
- No Gradle installation needed: the repository contains the Gradle wrapper (`gradlew` / `gradlew.bat`), which downloads the right Gradle version (9.4.1) on first use. The commands below use `./gradlew` (Windows: `gradlew.bat`).

The easiest way is to open the project folder in Android Studio, let it sync, and press **Run**. On the command line, Gradle needs to know where the SDK is: set `ANDROID_HOME`, or create a `local.properties` file with `sdk.dir=/path/to/Android/sdk` (Android Studio does this for you).

Verify your environment:
```
java -version
./gradlew --version
```

### Build
```
./gradlew assembleDebug
```
This compiles the app and creates the APK `app/build/outputs/apk/debug/app-debug.apk`. Use `./gradlew build` to also run the unit tests and lint.

### Run on a device or emulator
Start an emulator or connect a device with USB debugging enabled, then:
```
./gradlew installDebug
```
and open **Habitlab** from the launcher. In Android Studio, select the device and press **Run**.

### Run unit tests
The unit tests run on the JVM, no device needed:
```
./gradlew testDebugUnitTest
```
Run a single test class or method:
```
./gradlew testDebugUnitTest --tests "com.hudnah.habitlab.StreakCalculatorTest"
./gradlew testDebugUnitTest --tests "com.hudnah.habitlab.StreakCalculatorTest.StreakCalculator_bestStreak"
```
The HTML report is written to:
```
app/build/reports/tests/testDebugUnitTest/index.html
```

### Run integration and end-to-end tests
The integration and end-to-end tests are instrumented tests and need a running emulator or a connected device:
```
./gradlew connectedDebugAndroidTest
```
Run only one group:
```
./gradlew connectedDebugAndroidTest -Pandroid.testInstrumentationRunnerArguments.package=com.hudnah.habitlab.integration
./gradlew connectedDebugAndroidTest -Pandroid.testInstrumentationRunnerArguments.package=com.hudnah.habitlab.e2e
```
The HTML report is written to `app/build/reports/androidTests/connected/`.

> **Note:** the end-to-end tests use the app's real database and clear it before and after each test. Run them on an emulator or a test device, not on a phone with habits you want to keep.


## Tests
The project is tested on three levels, 53 tests in total.

- **Unit tests** (`app/src/test`, 37 tests) run on the JVM with JUnit 4, MockK and kotlinx-coroutines-test.
  - `StreakCalculatorTest` — consecutive days, a gap breaks the streak, today missing gives 0, best streak, no entries, duplicate days counted once.
  - `HabitRepositoryTest` — create, update, delete, archive, only active habits listed, lookup by id, blank names rejected.
  - `EntryRepositoryTest` — create and delete entries, "completed today", idempotent check-in, queries per habit and in total.
  - `StreakRepositoryTest` — the streak is calculated and saved.
  - `StatsRepositoryTest` — weekly rate, monthly stats, heatmap of four weeks, completions per habit, rates with no habits and with a fully completed day.
  - `DashboardViewModelTest` — today's and the weekly progress, archived habits excluded, the UI state updates after a new entry.
- **Integration tests** (`app/src/androidTest/.../integration`, 8 tests) wire the real repositories to an in-memory Room database and check the constraints and the interplay: idempotent entries, streak updates over several days and reset after a gap, stats and heatmap aggregation and `ON DELETE CASCADE`. `PersistenceTest` uses a real database file and checks that the data survives closing and reopening it.
- **End-to-end tests** (`app/src/androidTest/.../e2e`, 8 tests) start `MainActivity` and drive the real Compose UI: the full lifecycle (create a habit ➔ mark it done ➔ progress, streak and heatmap update), check-in and undo, a streak over several days, a streak broken by a gap, archiving, deleting, navigation, and data that survives an app restart.
- Dates never depend on the real clock: unit tests pass a fixed `today`, and the end-to-end tests move `HabitLabClock` from day to day to simulate multi-day streaks.
- Test names follow the pattern `Component_behaviour`, e.g. `StreakCalculator_brokenStreak` or `E2E_CompleteAndUndo`.


## Troubleshooting
- SDK location not found
  - Symptom: the build fails with "SDK location not found".
  - Fix: open the project once in Android Studio, or set `ANDROID_HOME`, or create `local.properties` with `sdk.dir=/path/to/Android/sdk`.
- Wrong Java version
  - Ensure JDK 21 is installed and active (`java -version`), or select Android Studio's bundled JDK under *Settings → Build, Execution, Deployment → Build Tools → Gradle → Gradle JDK*.
- `connectedDebugAndroidTest` fails with "No connected devices"
  - Start an emulator or connect a device first; `adb devices` should list it.
- A streak shows 0 although the habit was done yesterday
  - This is intended: the current streak counts only up to today. Once the habit is marked done today, the streak continues from yesterday.
- My habits are gone after running the tests
  - The end-to-end tests clear the app's database. Run them on an emulator or a separate test device.
- Start over with an empty app
  - Clear the app's storage in the Android settings (*Apps → Habitlab → Storage → Clear storage*) or uninstall and reinstall the app.


## Contributing
Contributions are welcome. Suggested areas:
- Restore archived habits (an "Archive" view)
- Reminders and notifications
- Weekly goals (e.g. 3× per week) in addition to daily habits
- Export and import of the data
- A longer, scrollable heatmap
- Documentation and comments (the code comments are in German)

Please open an issue or pull request.


## Author
**Mohameed Al-Harethi**
