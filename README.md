# MyWorkSpace

A personal productivity dashboard for tracking daily work output, billing hours, and shift-based attendance. Delivered as a hybrid Android app: a single-page HTML/JS dashboard bundled in `assets/` and rendered inside a native `WebView`, backed by Firebase Auth + Firestore for cloud sync.

![platform](https://img.shields.io/badge/platform-Android-3DDC84) ![min-sdk](https://img.shields.io/badge/minSdk-24-blue) ![target-sdk](https://img.shields.io/badge/targetSdk-36-blue) ![lang](https://img.shields.io/badge/UI-HTML%2FJS-orange) ![backend](https://img.shields.io/badge/cloud-Firebase-FFCA28)

---

## Table of Contents
- [Overview](#overview)
- [Features](#features)
- [Architecture](#architecture)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
- [Configuration](#configuration)
- [Usage](#usage)
- [Data Model](#data-model)
- [Build & Run](#build--run)
- [Tech Stack](#tech-stack)
- [Roadmap](#roadmap)
- [License](#license)

---

## Overview

**MyWorkSpace** is a daily/weekly work-log tracker built for shift workers who need accurate accounting of:

- Production minutes, coaching, meetings, well-being, breaks/unavailable time
- Billing-grade hours and utilization %
- Job counts and accuracy
- Per-day attendance with leave types (PL, UL, HL, SL, WO)
- Weekly carry-forward against per-day targets

The UI is a single dense table per day-of-week with auto-recalculating totals, a "Today Bar" with real-time deficits, and an attendance calendar. The whole experience runs offline-first with periodic cloud sync.

## Features

### Time & Attendance
- **Weekly grid** (Sun → Thu) with editable durations in `HH:MM` format
- **Auto-totals** for production, aux, billing, utilization, accuracy, job count
- **Today Bar** showing real-time deficit vs daily target (prod / aux / billing)
- **Attendance calendar** with leave types: PL, UL, HL, SL, WO
- **Weekoff automation** with rotation end-date support

### Shift Management
- **Three shifts**: Morning (6:00 AM – 3:30 PM), Evening (4:00 PM – 1:30 AM), Night (9:00 PM – 6:30 AM)
- **Start Shift / End Shift** session — locks the calendar date as the working day; persists across midnight and app reloads
- **Active row highlight** — the day-row of the running shift glows red and remains pinned until End Shift
- **Shift Extending? popup** — auto-prompts at the scheduled shift end on cross-midnight shifts (suppressed while an explicit session is active)
- **Split-entry popup** (Evening & Night) — double-click a `PROD_` cell to open a 3-input calculator with START/END clock times that auto-handle cross-midnight (`if end < start, add 24h`)

### Productivity Analytics
- **Logout time estimate** — live badge that projects when you'll hit today's billing target
- **Weekly carry-forward** — week target ÷ remaining days, recomputed every minute
- **Accuracy check** — sample/error inputs with live accuracy & error rate
- **Push-to-database** flow with summary modal before commit

### Cloud & Account
- **Firebase Auth** (email/password) with onboarding tour
- **Firestore sync** (debounced 1.5s on input) of all working data
- **Refresh / Force-pull** controls
- **Profile management** — display name, username, email change, password change, account deletion
- **PDF export** of full database before deletion

### UX
- **Dark mode** toggle
- **Onboarding wizard** + in-app tour for first-time users
- **Mobile-first responsive** layout with a desktop-optimized top bar
- **Persistent local storage** as fallback when offline

## Architecture

```
┌──────────────────────────────────────────────┐
│  Android App (Kotlin / AppCompat)            │
│  ┌────────────────────────────────────────┐  │
│  │ MainActivity                           │  │
│  │   - WebView (JS + DOM storage enabled) │  │
│  │   - Splash → file:///android_asset/    │  │
│  │       splash.html → index.html         │  │
│  │   - Back-button bridges WebView history│  │
│  └────────────────────────────────────────┘  │
│                  │                            │
│                  ▼                            │
│  assets/index.html                            │
│   • Single-page HTML/CSS/JS dashboard         │
│   • Firebase JS SDK (CDN module imports)      │
│   • localStorage primary cache                │
│   • Firestore: doc('userData', uid)           │
└──────────────────────────────────────────────┘
                   │
                   ▼
           Firebase (Auth + Firestore)
```

The native Android shell is intentionally thin — it only hosts the WebView, manages splash → app handoff, and wires the hardware Back button to WebView history.

## Project Structure

```
MyWorkSpace/
├── app/
│   ├── build.gradle.kts          # Module config (minSdk 24, targetSdk 36)
│   └── src/main/
│       ├── AndroidManifest.xml
│       ├── java/com/t01w/myworkspace/
│       │   └── MainActivity.kt   # WebView host
│       ├── assets/
│       │   ├── splash.html       # Splash screen (shown for 2.5s)
│       │   ├── index.html        # Entire dashboard (HTML/CSS/JS)
│       │   └── icon.png
│       └── res/                  # Launcher icons, layout, themes
├── build.gradle.kts              # Root build script
├── settings.gradle.kts
├── gradle/libs.versions.toml     # Version catalog
└── README.md
```

## Getting Started

### Prerequisites
- **Android Studio** Hedgehog (2024.1.1) or newer
- **JDK 11**
- **Android SDK** with API level 36 installed
- A **Firebase project** (or use the bundled config for testing — see [Configuration](#configuration))

### Clone & open
```bash
git clone <your-repo-url>
cd MyWorkSpace
# Open the folder in Android Studio
```

Let Gradle sync, then run on a device or emulator (API 24+).

## Configuration

### Firebase
Firebase config currently lives inline in `app/src/main/assets/index.html` (around line 1071) so that the WebView can initialize the JS SDK directly:

```js
const firebaseConfig = {
  apiKey:        "...",
  authDomain:    "<project>.firebaseapp.com",
  projectId:     "<project>",
  storageBucket: "<project>.firebasestorage.app",
  messagingSenderId: "...",
  appId:         "..."
};
```

To point at your own Firebase project:
1. Create a new project in the [Firebase Console](https://console.firebase.google.com).
2. Enable **Authentication → Email/Password**.
3. Create a **Firestore** database (production or test mode).
4. Replace the `firebaseConfig` object in `index.html`.
5. Add Firestore security rules so each user can only read/write `userData/{uid}` and `users/{uid}`.

### Suggested Firestore rules
```
rules_version = '2';
service cloud.firestore {
  match /databases/{db}/documents {
    match /userData/{uid} {
      allow read, write: if request.auth != null && request.auth.uid == uid;
    }
    match /users/{uid} {
      allow read, write: if request.auth != null && request.auth.uid == uid;
    }
  }
}
```

## Usage

### First launch
1. Splash screen → onboarding modal.
2. Enter your **weekly targets** (Production, Coaching, Meeting, Well-being, Job count).
3. Pick **shift** (Morning / Evening / Night) and **week-offs** (max 2 days).
4. Set the **week-off validity date** (rotation end).
5. The in-app tour highlights the key dashboard areas.

### Daily workflow
1. Tap **▶ START SHIFT** in the header to lock today as your working day.
2. Enter durations into the day-row (or for Evening/Night, **double-click** a PROD cell to use the START/END clock-time split popup).
3. Totals, utilization, Today Bar, and the logout-time badge update live.
4. When done, tap **⏻ END SHIFT** → confirm → see the elapsed-time summary.
5. Data auto-syncs to Firestore (debounced 1.5s); manually press **☁ SYNC** or **⟳ REFRESH** any time.

### Attendance
- Click a calendar day → choose Half / Full → choose leave type (PL / UL / SL for full day; HL via half-day option).

### Settings
Profile, billing/count targets, shift change, password/email change, account deletion (with PDF export).

## Data Model

Stored at `userData/{uid}` in Firestore (mirrored to `localStorage`):

| Key prefix | Purpose |
|---|---|
| `prod_{day}`, `coach_{day}`, `meet_{day}`, `well_{day}`, `break_{day}`, `unav_{day}` | Per-day durations in `HH:MM` |
| `job_{day}`, `bill_{day}` | Per-day job count & billing hours |
| `q_prod`, `q_coach`, `q_meet`, `q_well`, `q_jobs` | Weekly targets |
| `att_data` | Attendance calendar JSON |
| `userShift` | `morning` / `evening` / `night` (localStorage) |
| `shiftSession` | Active Start-Shift session `{startMs, workDayKey, shift}` (localStorage) |

User profile (name, username) lives at `users/{uid}`.

## Build & Run

### Debug build
```bash
./gradlew assembleDebug
# APK output: app/build/outputs/apk/debug/app-debug.apk
```

### Release build (unsigned)
```bash
./gradlew assembleRelease
```

To sign the release APK, configure a signing config in `app/build.gradle.kts` or use Android Studio's **Build → Generate Signed Bundle / APK**.

### Iterating on the dashboard
Because the whole UI lives in `assets/index.html`, you can edit and reload without recompiling Kotlin:
1. Edit `app/src/main/assets/index.html`.
2. Run the app — Gradle repackages the asset and the WebView picks it up on next launch.

For faster iteration, open `index.html` directly in a desktop browser (some Firebase auth flows still work via the configured domain).

## Tech Stack

| Layer | Technology |
|---|---|
| **Native shell** | Kotlin, AndroidX AppCompat, ConstraintLayout, Material Components |
| **UI** | HTML5, CSS3 (vanilla, mobile-first), JavaScript (ES6 modules) |
| **Fonts** | Lexend |
| **Auth & DB** | Firebase Auth (email/password), Firestore (v10.12.0 via CDN) |
| **Local cache** | `localStorage`, `sessionStorage` |
| **Build** | Gradle 8.x (Kotlin DSL), Android Gradle Plugin |

## Roadmap

- [ ] Externalize Firebase config out of `index.html`
- [ ] Move from CDN Firebase SDK to bundled SDK for offline-first
- [ ] Multi-week history view
- [ ] Per-shift productivity reports / charts
- [ ] Notifications at shift start / end
- [ ] Biometric unlock

## License

Proprietary — internal use. Update this section if you intend to open-source the project.

---

_Maintained by_ [@Akram](https://github.com/) · Built with ☕ and a stubborn refusal to use spreadsheets.
