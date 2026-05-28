# CLAUDE.md

## Product Overview

DOUBLEBUMP is a real-world social connection app.
- Connect by physically bumping phones (double bump) — Apple/Google login required
- Memory-based social graph: Contacts = Discovery, Bump = Connection, Map = Memory

## System Pipeline

```
BLEProximityService   [advertisement broadcast + RSSI scan → nearby device_id list]
CMDeviceMotion (100Hz)
  → BumpDetector        [5-condition impact → double bump pattern]
  → MotionSignature     [150ms capture → 16-sample resample → SHA256]
  → SupabaseService     [upload + nearby_ids → BLE-filtered match → pair confirm]
  → BumpViewModel       [state machine → haptic → UI]
```

Security model: **BLE = proximity proof** (prevents remote faking) + **double bump = intent proof** (prevents accidental match)

State transitions: `idle → detecting → matching → bumped/failed → detecting`

## Hard Constraints

- **BLE required** — advertisement-only proximity gate (`BLEProximityService`). RSSI threshold -40dBm (~30cm). No device pairing.
- NO GPS / Wi-Fi / NFC — no alternative **pairing** methods (GPS is allowed post-bump for `bump_locations` memory only)
- Login required (Apple/Google) — `LoginView` is the entry point before `NicknameSetupView`
- NO third-party HTTP library — URLSession only
- NO group bump — 1:1 mutual match enforced
- NO simulator — motion logic requires a physical iPhone
- Score threshold ≥ 0.85 — never lower this value
- Double bump pattern retained — UX identity + intent proof (single tap insufficient even with BLE)

## MVP Scope

**Included:** BLE proximity gate, double bump detection, motion signature matching, 1:1 pairing, map memory, ads unlock, encounter memory (comment + 1 photo per bump)
**Excluded:** group bump, contact sync, public feed, likes, follow, DM, multiple photos per encounter

## Tech Stack

- Frontend: SwiftUI + CoreMotion + CoreBluetooth
- Backend: Supabase (PostgreSQL + PostgREST + Edge Functions)
- HTTP: URLSession (raw PostgREST) — no Supabase Swift SDK

## Development Philosophy

- Build minimal but correct — core must work on real devices before adding features
- A missed bump is acceptable; a false pair is not
- Avoid premature optimization — real-device testing reveals true bottlenecks

## Behavior Guidelines

**Think Before Coding**
- 요청이 모호하거나 해석이 여러 가지일 경우, 가정하고 진행하지 말고 먼저 질문할 것
- 수치 파라미터(score threshold, 시간 윈도우, 조건 개수 등)는 명시적 승인 없이 절대 수정 금지
- 더 단순한 접근법이 보이면 구현 전에 먼저 제안할 것

**Surgical Changes**
- 요청된 변경과 직접 연관된 코드만 수정 — 인접 코드 개선/정리 금지
- 기존 스타일, 주석, 포맷은 그대로 유지 (내 취향과 달라도)
- 내 변경으로 인해 생긴 dead code(orphan import, unused variable)만 제거
- 기존에 있던 dead code는 수정하지 말고 발견 시 언급만 할 것
- 모든 변경 줄은 사용자의 요청과 직접 연결되어야 함

## Build & Run

- Xcode project only (`DoubleBump.xcodeproj`) — no CLI build, no SPM
- Run on physical device (⌘R) · Tests: ⌘U
- SourceKit false positives (cross-file types not found) are harmless — compile in Xcode to verify

## Required Setup

**Supabase** — already configured in `DoubleBumpApp.swift`:
- URL: `https://dbzsuwxpxipoozeevwbi.supabase.co`
- All tables, RPCs, RLS policies, and triggers are live in the project.

**Info.plist** entries required:
- `NSMotionUsageDescription` — CoreMotion 권한
- `NSBluetoothAlwaysUsageDescription` — BLE proximity gate (bump 매칭 필수)
- `NSLocationWhenInUseUsageDescription` — bump 위치 기록 (bump 탭 진입 시 필수 요청 — 거부 시 감지 차단)
- `NSCameraUsageDescription` — Encounter Memory 사진 촬영
- `NSPhotoLibraryUsageDescription` — Encounter Memory 사진 라이브러리 접근
- `CFBundleURLTypes` → URL Scheme: `com.googleusercontent.apps.88437311800-sn2rerg8k6emh4ti63ift7vevjf7tqu6` — Google Sign-In 복귀
- `GADApplicationIdentifier` — AdMob App ID: `ca-app-pub-9618140053297810~5087378556` (production) / `ca-app-pub-3940256099942544~1458002511` (test, DEBUG 빌드)
- `SKAdNetworkItems` — AdMob 필수 (Google Mobile Ads SDK가 자동 추가 안 함 — 수동 추가)

**GoogleSignIn SDK** — Xcode Package Dependencies에 추가 필요:
`https://github.com/google/GoogleSignIn-iOS`

**Google Mobile Ads SDK** — Xcode Package Dependencies에 추가 필요:
`https://github.com/googleads/swift-package-manager-google-mobile-ads` (product: `GoogleMobileAds`)

New source files placed anywhere under `DoubleBump/` are **automatically included in the build** — the project uses `PBXFileSystemSynchronizedRootGroup`. No manual Navigator addition needed. New SPM packages still require editing `project.pbxproj` (XCRemoteSwiftPackageReference + XCSwiftPackageProductDependency + PBXBuildFile + packageProductDependencies sections).

## Confirmed Architectural Decisions

These were explicitly decided — do not reverse without user approval.

| Decision | Detail |
|---|---|
| **Login required** | Apple/Google login is mandatory on first launch. `RootView.mainContent`: `!onboarding_shown → OnboardingView` → `!isAuthenticated → LoginView` → `nickname.isEmpty → NicknameSetupView` → `MainTabView`. No guest mode. `DeviceIdentityService` still provides `device_id` (Keychain) for bump matching. |
| **SQL via dashboard** | All Supabase tables, RPCs, and RLS policies are run manually in the Supabase dashboard. No in-app migrations. `rules/network.md` SQL blocks are the canonical source. |
| **AdMob unlock** | Locked profile (non-relationship) → "Watch ad to unlock all profiles for 30 min" → `GADRewardedAd` → `didEarnReward` → `ProfileUnlockService.grantUnlock()`. Max 3 unlocks/day. Global pass (not per-profile). Same pass also unlocks encounter memories of non-connected users. `AdMobService` wraps GAD SDK. |
| **Encounter Memory** | bump 성공(winner) → `createEncounter` 호출 → `encounters` 행 생성. 각 사용자는 `encounter_memories`에 comment(100자) + photo(1장) 추가. 비연결 상대방 memory는 `ProfileUnlockService` 30분 패스로 unlock (별도 unlock 없음). `EncounterDetailView` + `EncounterMemoryEditor`로 접근. 공개 SNS 기능 없음. |
| **BLE proximity gate** | 매칭 보안 모델: BLE(근접 증명) + 더블범프(의도 증명) 이중 레이어. magnitude-only 모션 처리는 Newton's 3rd Law 방향 정보를 버리므로 원격 동시 조작을 모션만으로 차단 불가 — BLE가 필수. `BLEProximityService` (CoreBluetooth): Peripheral(브로드캐스트) + Central(스캔) 모드. RSSI > -40dBm (~30cm) 기기만 후보. 디바이스 페어링 없음. 권한 요청: 로그인 후 또는 첫 범프 시 (콜드 런치 즉시 금지). 권한 거부 시 모션 전용 fallback 동작. 크로스플랫폼(iOS↔Android) 호환. |
| **Firebase Analytics + Crashlytics** | Supabase = backend, Firebase = mobile analytics/monitoring 역할 분리. `AnalyticsService.shared` (`Services/AnalyticsService.swift`) 가 두 SDK를 래핑. `FirebaseApp.configure()`는 `DoubleBumpApp.init()` 최상단에서 호출. Crashlytics user ID는 `AuthViewModel.currentUser` `didSet`에서 자동 설정. 핵심 이벤트: `bump_started`, `bump_success`, `bump_failure_*`, `signup_completed`, `memory_created`. dSYM 업로드 빌드 스크립트는 `project.pbxproj` `PBXShellScriptBuildPhase`에 등록됨. `ENABLE_USER_SCRIPT_SANDBOXING = NO` 필수 (DoubleBump 타겟). `GoogleService-Info.plist` 필요 — `DoubleBump/` 폴더에 배치하면 자동 포함. |

## Known Issues

- `HomeView.swift`: Unreferenced after `MainTabView` became the root. Keep but do not add new logic there.
- `LoginView.swift`: Active entry point — Apple/Google sign-in. Keep this path; it is the first screen.
- `NotificationService.swift`: Push + local notification service. Requires Push Notifications capability in Xcode target.
- `AppSettings.swift`: User preferences singleton (`Services/`). UserDefaults-backed, `ObservableObject`.
- `SettingsView.swift`: Settings sheet (`Views/`). Opened via ⚙️ button in ProfileView toolbar.
- `LegalDocumentView.swift`: In-app Terms of Service and Privacy Policy viewer (`Views/`). Contact email: `bangcoderpro@gmail.com`. Shown as `.sheet` from `LoginView` (bottom links) and `SettingsView` (LEGAL section).
- `AnalyticsService.swift`: Firebase Analytics + Crashlytics wrapper (`Services/`). Requires `GoogleService-Info.plist` in `DoubleBump/` folder and Firebase SPM packages linked.
- Google Sign-In requires Supabase dashboard → Authentication → Providers → Google to be enabled with the OAuth client credentials.
- `send-push` Edge Function requires `APNS_KEY_ID`, `APNS_TEAM_ID`, `APNS_PRIVATE_KEY`, `APNS_BUNDLE_ID`, `APNS_PRODUCTION` secrets set via `supabase secrets set`.
- `CommentBubble.swift`: Reusable bump comment view (`Views/`). Shows 20px avatar + italic comment text. Used by `FeedCard`, `CDTimelineRow`, `MyMapTimelineRow`.

## Rules Index

| File | Domain |
|---|---|
| `rules/sensor.md` | CMDeviceMotion setup, 5-condition impact, double bump, signature capture, BLE security model |
| `rules/matching.md` | Scoring formula, server matching flow, Supabase tables, failure UX |
| `rules/network.md` | URLSession, REST endpoints, PostgREST syntax, SQL setup |
| `rules/anti-patterns.md` | Consolidated DO NOT list across all domains |
| `rules/product.rules.md` | Product scope, entry flow, social graph constraints |
| `rules/algorithm.rules.md` | Orientation invariance, attack prevention |
| `rules/backend.rules.md` | Server-side atomicity, TTL, validation |
| `rules/ios.rules.md` | Keychain, threading, APNs + local notifications |
| `rules/ux.rules.md` | Haptic feedback, onboarding, first-experience |
| `rules/devplan.md` | Week-by-week MVP build plan, dependency order |
| `rules/ui.md` | Tab structure, per-screen definition, level system, design tokens, prohibitions |
| `context/failure-cases.md` | Known failure/edge cases — debugging reference only, not rules |
