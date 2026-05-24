# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## About

Fleetbase Navigator is a React Native mobile app (iOS + Android) for drivers and agents to manage orders, track location, and navigate. It connects to a Fleetbase backend via `@fleetbase/sdk`.

## Commands

```bash
yarn install              # Install dependencies (uses Yarn Berry 3.6.4)
yarn start                # Start Metro bundler
yarn ios                  # Run on iOS simulator
yarn android              # Run on Android emulator
yarn test                 # Run all Jest tests
yarn test --testPathPattern="App-test"  # Run a single test file
yarn lint                 # ESLint
yarn format               # Prettier

# iOS-specific
yarn pod:install          # Install CocoaPods (RCT_NEW_ARCH_ENABLED=1)
yarn pod:reset            # Clean and reinstall Pods

# Android-specific
yarn android:clean        # Clean Gradle build
yarn adb:reverse          # ADB reverse port for Metro (tcp:8081)
yarn adb:devmenu          # Open Android dev menu
```

## Environment Setup

Copy `.env.example` to `.env` and fill in required keys:

```
APP_NAME=Navigator
APP_IDENTIFIER=io.fleetbase.navigator
APP_LINK_PREFIX=flbnavigator
FLEETBASE_HOST=https://api.fleetbase.io
FLEETBASE_KEY=          # From Fleetbase Console > Developers > API Keys
GOOGLE_MAPS_API_KEY=
DEFAULT_COORDINATES=1.369,103.8864
```

Additional keys used in CI (set as repo secrets for builds): `TRANSISTORSOFT_LICENSE_KEY`, `FACEBOOK_APP_ID`, `FACEBOOK_CLIENT_TOKEN`, `GOOGLE_CLIENT_ID`, `GOOGLE_IOS_CLIENT_ID`.

## Architecture

### Entry Point & Provider Tree

`App.tsx` composes all context providers in a specific nesting order. From outermost to innermost:

```
PortalProvider → ThemeProvider → TamaguiProvider
  → GestureHandlerRootView → SafeAreaProvider → BottomSheetModalProvider
    → ConfigProvider → NotificationProvider → LanguageProvider → AuthProvider
      → SocketClusterProvider → LocationProvider → TempStoreProvider
        → ChatProvider → OrderManagerProvider → AppNavigator
```

Order matters — inner contexts depend on outer ones (e.g., `AuthContext` uses `ConfigContext` and `LanguageContext`).

### Navigation

Defined in `src/navigation/` using React Navigation's static API (`createStaticNavigation`):

- `AppNavigator.tsx` — root stack: Boot → auth screens OR `DriverNavigator` (guarded by `useIsAuthenticated`)
- `DriverNavigator.tsx` — bottom tab navigator with 5 tabs, each being its own `NativeStackNavigator`:
  - **DriverDashboardTab** → `DriverDashboardScreen`
  - **DriverTaskTab** → `DriverOrderManagementScreen` + `OrderScreen`, `EntityScreen`, `ProofOfDeliveryScreen`
  - **DriverReportTab** → `DriverReportScreen` + fuel reports + issues
  - **DriverChatTab** → `ChatHomeScreen` + `ChatChannelScreen`
  - **DriverAccountTab** → `DriverProfileScreen` + `DriverAccountScreen`

Tab layout is configurable via `navigator.config.ts` (`driverNavigator.tabs`).

### Configuration System

Two layers of config, with instance-linked config taking precedence:

1. **Build-time** (`.env` via `react-native-config`): accessed via `config('KEY')` from `src/utils/index.js`
2. **Runtime** (`navigator.config.ts`): extends `config/default.js`, accessed via `navigatorConfig('key')`
3. **Instance-linked** (MMKV storage): set when a driver scans a QR code to connect to a custom Fleetbase instance; accessed via `resolveConnectionConfig('KEY')` from `ConfigContext`

`resolveConnectionConfig()` always checks instance-linked MMKV values first, then falls back to `.env`.

### State & Storage

- **MMKV** (`react-native-mmkv-storage`) is the persistence layer. Use `useStorage(key, defaultValue)` hook for reactive storage or the exported `storage` singleton for direct access.
- Driver session is stored as serialized JSON under `'driver'`; auth token under `'_driver_token'`.
- Orders are cached per-driver using keys like `${driver.id}_all_active_orders`.

### Key Contexts

| Context | Responsibility |
|---|---|
| `AuthContext` | Phone/OTP login, driver session, org switching, device sync |
| `ConfigContext` | Merges `.env` config with instance-linked overrides |
| `LocationContext` | Background geolocation via `react-native-background-geolocation`; starts/stops tracking based on driver online status |
| `OrderManagerContext` | Fetches and caches four order lists (active, recent, nearby, current); deduplicates in-flight requests via promise refs |
| `SocketClusterContext` | SocketCluster WebSocket connection for real-time events; `.native.tsx` and `.web.tsx` platform variants |
| `ChatContext` | Chat channels and unread counts |
| `TempStoreContext` | Ephemeral cross-screen state (e.g., passing an issue/fuel report to a modal screen's header) |

### Fleetbase SDK

`useFleetbase()` hook instantiates `@fleetbase/sdk` with the auth token (if logged in) or public API key. Returns `{ fleetbase, adapter }`. The `adapter` is used to make raw HTTP calls and to construct SDK resource instances (`new Order(json, adapter)`).

### Theming

Tamagui (`tamagui.config.ts`) handles theming. Theme name is persisted in MMKV via `APP_THEME_KEY`. Access theme tokens in components via `useTheme()`. Use `getTheme(key)` utility for non-component contexts (e.g., navigation options).

### Legacy Code

`src/legacy/` and the top-level `legacy/` directory contain the previous version of the app. These are kept for reference but should not be modified. New features belong in `src/`.

## CI/CD

GitHub Actions (`.github/workflows/react-native-ci.yml`) runs on PRs and pushes to `main`:
- **install_and_test**: installs dependencies and sets up `.env`
- **android_build**: builds debug APK (JDK 21, NDK 27.1.12297006, platform 35)
- **ios_build**: builds for iOS Simulator (macOS runner)
- **android_release / ios_release**: triggered by `v*` tags only

Release builds are triggered by pushing a `v*` tag.

## Yarn Patches

Three packages have local patches in `.yarn/patches/`:
- `react-native-i18n`
- `react-native-launch-navigator`
- `react-native-notifications`

Do not upgrade these packages without verifying the patches still apply cleanly.
