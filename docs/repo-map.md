# Navigator Flutter App — Repo Map

> Generated 2026-05-23 · Branch `chore/mobile/repo-intel`

---

## 1. Flutter / Dart SDK Version

| Setting | Value |
|---------|-------|
| Dart SDK constraint | `^3.8.1` |
| Flutter SDK | Resolved at runtime; Dart 3.8 requires Flutter 3.29+ |
| Platforms declared | Android, iOS, Windows, Linux (CMakeLists present) |
| Minimum target | Not explicitly pinned in pubspec; inferred from dependencies |

---

## 2. State Management

**Library:** `flutter_riverpod: ^2.6.1` + `riverpod_annotation: ^2.6.1`

**Pattern used:** `StateNotifierProvider` throughout — each feature domain owns a `StateNotifier` subclass and a plain state value-object.

| Provider | Notifier | State class | Purpose |
|----------|----------|-------------|---------|
| `authProvider` | `AuthNotifier` | `AuthState` | Authenticated driver, session restore, onboarding flag |
| `orderProvider` | `OrderNotifier` | `OrderState` | Active, recent, and current order |
| `locationProvider` | `LocationNotifier` | `LocationState` | GPS coordinates, speed, tracking flag |
| `chatProvider` | `ChatNotifier` | `ChatState` | Chat channels + messages |
| `earningsProvider` | `EarningsNotifier` | `EarningsState` | Wallet balance, earnings rows, payouts |
| `apiServiceProvider` | — | `ApiService` | Singleton HTTP client (plain `Provider`) |
| `storageServiceProvider` | — | `StorageService` | Singleton local storage (plain `Provider`) |
| `socketServiceProvider` | — | `SocketService` | Singleton WebSocket (plain `Provider`) |
| `routerProvider` | — | `GoRouter` | Singleton router (plain `Provider`) |

> Note: `riverpod_annotation` is listed as a runtime dependency but no `build_runner` / code-generation step is present in dev_dependencies. The annotation package is imported but the generated `@riverpod` macro style is not used; all providers are hand-written.

---

## 3. Routing Library

**Library:** `go_router: ^14.8.1`

**Pattern:** One `GoRouter` instance mounted via `routerProvider`. A `ShellRoute` wraps the five main tabs and renders `AppShell` (bottom navigation bar). Detail screens sit outside the shell so they push full-screen over the tab bar.

### Route Table

| Path | Widget | Notes |
|------|--------|-------|
| `/` | `BootScreen` | Splash + session restore; 2.5 s delay before redirect |
| `/instance-link` | `InstanceLinkScreen` | Connect to a custom Fleetbase host |
| `/auth/login` | `LoginScreen` | Email/password entry point |
| `/auth/phone-login` | `PhoneLoginScreen` | Phone number entry for SMS OTP |
| `/auth/phone-verify` | `PhoneVerifyScreen` | OTP verification; receives `phone` via `state.extra` |
| `/auth/register` | `RegisterScreen` | New driver registration |
| `/onboarding` | `OnboardingScreen` | 4-step post-registration setup |
| `/dashboard` *(shell)* | `DashboardScreen` | Home tab |
| `/orders` *(shell)* | `OrderManagementScreen` | Orders tab |
| `/earnings` *(shell)* | `EarningsScreen` | Earnings tab |
| `/chat` *(shell)* | `ChatHomeScreen` | Chat tab |
| `/account` *(shell)* | `ProfileScreen` | Account tab |
| `/orders/:id` | `OrderDetailScreen` | Full-screen order detail (outside shell) |
| `/orders/:id/navigate` | `NavigationScreen` | Full-screen in-app map (outside shell) |
| `/chat/channel/:id` | `ChatChannelScreen` | Individual channel conversation |
| `/chat/create` | `CreateChatChannelScreen` | New channel creation |
| `/account/settings` | `AccountSettingsScreen` | Edit profile fields |

**Auth redirect logic** (evaluated on every navigation):
- Unauthenticated → `/`
- Authenticated + `needsOnboarding` → `/onboarding`
- Authenticated accessing `/auth/*` → `/dashboard`

---

## 4. HTTP Client

**Library:** `dio: ^5.7.0`

**Wrapper class:** `lib/services/api_service.dart` (`ApiService`)

### Dual base-URL design

| Method group | Base URL | Auth header | Typical use |
|---|---|---|---|
| `get/post/put/patch/delete` | `{FLEETBASE_HOST}/int/v1` | Bearer JWT (driver token) | Authenticated internal routes |
| `publicGet/publicPost/publicPut/publicUpload` | `{FLEETBASE_HOST}` | Bearer API key (from `.env` or storage) | Public driver endpoints (`/v1/drivers/*`, `/v1/orders/*`) |

**Interceptors:**
- Injects `Authorization` header from token or API key on every request.
- On `401`: clears in-memory token and removes `auth_token` from local storage (auto-logout).

**File upload:** `FormData` / `MultipartFile` via `publicUpload` (used for document uploads in onboarding).

---

## 5. Storage

**Libraries:** `hive: ^2.2.3` + `hive_flutter: ^1.1.0`, `shared_preferences: ^2.3.4`

**Wrapper class:** `lib/services/storage_service.dart` (`StorageService`)

Used for:
- `auth_token` — JWT string
- `driver` — driver JSON map
- `documents_uploaded` — boolean string flag
- `active_orders` / `recent_orders` — JSON list cache
- `chat_channels` — JSON list cache
- `earnings_wallet` / `earnings_transactions` / `earnings_payouts` — JSON cache
- `last_known_position` — `[lat, lng]` list
- `fleetbase_key` — overrideable API key (set from instance-link screen)

---

## 6. WebSocket / Real-time

**Library:** `web_socket_channel: ^3.0.2`

**Service:** `lib/services/socket_service.dart` (`SocketService`)

Implements the **SocketCluster** wire protocol manually (JSON messages with `event`, `data`, `cid` fields; `#handshake`, `#subscribe`, `#unsubscribe`, `#publish`). Includes:
- Heartbeat ping every 20 seconds (empty string frame).
- Auto-reconnect with 3-second delay on disconnect; re-subscribes all existing channels.
- Channel listener map for routing incoming `#publish` payloads to registered callbacks.

---

## 7. Folder Layout (`lib/`)

```
lib/
├── main.dart                     Entry point; loads .env, inits storage, mounts ProviderScope
├── config/
│   ├── app_config.dart           Env-var driven config (host, API key, Maps key, etc.)
│   ├── router.dart               GoRouter definition with redirect guard
│   └── theme.dart                AppTheme (light only; black/white palette + Google Fonts)
├── models/
│   ├── driver.dart               Driver (auth subject)
│   ├── order.dart                Order + Payload + Place + Entity + TrackingNumber
│   ├── chat.dart                 ChatChannel + ChatMessage
│   ├── earning.dart              Earning + Wallet
│   └── payout.dart               Payout
├── providers/
│   ├── service_providers.dart    Singleton providers: storage, api, socket
│   ├── auth_provider.dart        AuthState + AuthNotifier
│   ├── order_provider.dart       OrderState + OrderNotifier
│   ├── location_provider.dart    LocationState + LocationNotifier
│   ├── chat_provider.dart        ChatState + ChatNotifier
│   └── earnings_provider.dart    EarningsState + EarningsNotifier
├── services/
│   ├── api_service.dart          Dio HTTP client wrapper
│   ├── socket_service.dart       SocketCluster WebSocket wrapper
│   └── storage_service.dart      Hive/SharedPreferences abstraction
├── screens/
│   ├── boot/
│   │   ├── boot_screen.dart      Splash + session restore
│   │   └── instance_link_screen.dart  Custom backend URL entry
│   ├── auth/
│   │   ├── login_screen.dart     Email/password login
│   │   ├── phone_login_screen.dart    Phone entry (SMS flow)
│   │   ├── phone_verify_screen.dart   OTP code entry
│   │   ├── register_screen.dart  Registration form
│   │   └── register_verify_screen.dart  (exists; role unclear — not wired in router)
│   ├── onboarding/
│   │   └── onboarding_screen.dart  4-step driver setup wizard
│   ├── dashboard/
│   │   └── dashboard_screen.dart   Home tab: greeting, stats, tracking toggle, active orders
│   ├── orders/
│   │   ├── order_management_screen.dart  Active / Recent tabs list
│   │   ├── order_detail_screen.dart      Full order info + action bar
│   │   └── navigation_screen.dart        Google Maps navigation view
│   ├── earnings/
│   │   └── earnings_screen.dart   Wallet balance hero + Earnings / Payouts tabs
│   ├── chat/
│   │   ├── chat_home_screen.dart  Channel list
│   │   ├── chat_channel_screen.dart   Per-channel conversation
│   │   └── create_chat_channel_screen.dart  New channel form
│   └── account/
│       ├── profile_screen.dart    Driver profile hero + menu groups
│       └── account_settings_screen.dart  Edit name/email/phone, clear cache
├── widgets/
│   ├── app_shell.dart            Bottom navigation shell (5 tabs)
│   ├── app_logo.dart             Brand logo (light/dark variant)
│   └── status_badge.dart         Order status chip
└── utils/
    └── format.dart               FormatUtils: dates, titleize, currency
```

---

## 8. Current Feature Set

### Boot & Instance Linking
- 2.5-second animated splash screen (black background, radial gradient).
- Session restore from local storage on every cold start.
- `InstanceLinkScreen` lets the driver point the app at a custom Fleetbase host (stored in `SharedPreferences`; updates `ApiService.baseUrl` at runtime).

### Authentication
- **Email/password login** — `POST /v1/drivers/login`
- **Phone + SMS OTP login** — `POST /v1/drivers/login-with-sms` → `POST /v1/drivers/verify-code`
- **Registration** — `POST /v1/drivers` then auto-login; sets status `inactive` until onboarding completes.
- Token stored in local storage; automatically injected into all subsequent requests.

### Driver Onboarding (4 steps)
1. Location — country code (2-char ISO) + city.
2. Vehicle — make, model, year, plate, type (car/motorcycle/truck/van/bus/bicycle); creates via `POST /v1/vehicles`.
3. Personal documents — ID + driver's licence (gallery or camera).
4. Vehicle documents — insurance certificate (gallery or camera).
- On completion: uploads files to `POST /v1/files`, then `PUT /v1/drivers/:id` with `status: pending-review`.

### Dashboard
- Personalised greeting by time of day.
- Quick-stat tiles: active order count, current speed, completed order count.
- Online/offline indicator (tied to `LocationState.isTracking`).
- Location tracking toggle — starts/stops GPS stream; calls `POST /v1/drivers/:id/toggle-online` and `POST /v1/drivers/:id/track`.
- Active orders preview (up to 3), with "View all" link.
- Pull-to-refresh.

### Order Management
- Two-tab list: **Active** (non-completed/non-canceled) and **Recent** (completed, last 20).
- Orders cached locally; served from cache on network failure.

### Order Detail
- Hero card with status badge, tracking number, creation date, inline customer summary.
- Route timeline (pickup → optional waypoints → dropoff) using GeoJSON `Place` coordinates.
- Customer card (name, phone call shortcut, email).
- Order details card (type, scheduled/dispatched timestamps, POD requirement).
- Packages card (entity list from `payload.entities`).
- **Action bar** (context-aware):
  - `dispatched` / `created` → **Start Order** button (calls `POST /v1/orders/:id/start`, then opens `NavigationScreen`).
  - `started` / `in_progress` → **Navigate** (opens `NavigationScreen`) + **Complete** (calls `POST /v1/orders/:id/complete`).

### In-app Navigation (Google Maps)
- Full-screen `GoogleMap` widget with live driver marker (azure), pickup marker (green), dropoff marker (red).
- Route polyline fetched from **Google Directions API** (`driving` mode, `departure_time=now` for traffic-aware ETA).
- Polyline decoded from encoded overview polyline string.
- Falls back to a dashed straight-line (haversine distance) when the Directions API is unavailable or the API key is missing.
- Route auto-refreshes every 10 seconds; also re-fetches when driver moves >50 m.
- Bottom card shows: destination label (Pickup/Dropoff), formatted address, distance, ETA, toggle destination button, and **Update Status** button.
- Status update sheet fetches valid next activities from `GET /v1/orders/:id/next-activity` and posts selected one to `POST /v1/orders/:id/update-activity`.
- Destination automatically switches from pickup phase to dropoff phase when order status advances past `driver_enroute`.

### Earnings
- Hero: wallet balance (from `GET /v1/drivers/:id/earnings`; fallback to summed earnings rows).
- Sub-stats: delivery count, payout count, last payout date.
- **Earnings tab**: per-delivery ledger rows (tracking number, customer name, date, amount).
- **Payouts tab**: payout history (description, date, status badge, amount).
- All data cached; pull-to-refresh.

### Chat
- `ChatHomeScreen`: channel list with unread counts.
- `ChatChannelScreen`: message thread with optimistic send (temporary ID replaced on server confirmation).
- `CreateChatChannelScreen`: new channel form (name + participant IDs).
- Incoming messages delivered via `SocketService` (`#publish` events on channel topics).
- Read receipts: `POST /chat-receipts`.

### Account / Profile
- Driver profile hero (avatar initial, name, status badge, company name).
- Contact card (email, phone).
- Menu groups: Account Settings, Organisation (stub), Support (stub), App version.
- Sign-out clears token, driver, and documents_uploaded from storage.
- `AccountSettingsScreen`: editable name, email, phone fields; **Clear Cache** action.

---

## 9. Clearly In-Progress / TODOs

| Location | Observation |
|----------|-------------|
| `test/widget_test.dart:4` | Placeholder smoke test: `"requires mocking services for full test"`. No real widget or integration tests exist. |
| `screens/auth/register_verify_screen.dart` | File exists in `lib/` but is **not wired into the router** (`config/router.dart`). Appears to be scaffolded but unused. |
| `screens/account/profile_screen.dart` — "Organisation" menu item | `onTap: () {}` (no-op). Organisation screen not implemented. |
| `screens/account/profile_screen.dart` — "Support", "Help" menu items | `onTap: () {}` (no-op). Support flows not implemented. |
| `screens/orders/order_detail_screen.dart` — `_showActivityPicker()` | Method defined but never called; superseded by the direct start/complete action bar. Dead code. |
| `config/app_config.dart` — `driverNavigatorTabs` constant | Lists `['dashboard', 'tasks', 'reports', 'chat', 'account']`; actual shell routes use `orders` and `earnings`, not `tasks`/`reports`. Stale constant. |
| `providers/service_providers.dart` — `socketServiceProvider` | Socket service is provided but never injected into any feature provider. SocketCluster real-time is wired manually inside screens (not through the provider graph). |
| `riverpod_annotation` in dependencies | Annotation package imported but no `build_runner` / `@riverpod` code generation is wired. Could cause confusion; the package is effectively unused at runtime. |
| Location restore | `restoreLastKnownPosition()` is defined on `LocationNotifier` but not called anywhere in the current screen lifecycle. |
| POD (Proof of Delivery) | `Order.podRequired` and `Order.podMethod` are modelled and displayed in Order Details, but no signature capture or photo-POD flow is implemented (the `signature` and `image_picker` packages in `pubspec.yaml` suggest this was planned). |
| `table_calendar` dependency | Declared in `pubspec.yaml` but no screen uses `TableCalendar`. Likely placeholder for a future schedule/calendar view. |

---

## 10. Test Setup

| Item | State |
|------|-------|
| Test directory | `test/widget_test.dart` (single file) |
| Test runner | `flutter_test` (SDK) |
| Linter | `flutter_lints: ^5.0.0` |
| Current coverage | Effectively zero — single placeholder `expect(true, isTrue)` test |
| Integration tests | None (`integration_test/` directory absent) |
| Mocking framework | None declared; `mockito` / `mocktail` not in dev_dependencies |
| CI configuration | Not present in the repo |

---

## 11. Key Dependencies Summary

| Category | Package | Version |
|----------|---------|---------|
| State management | `flutter_riverpod` | ^2.6.1 |
| Routing | `go_router` | ^14.8.1 |
| HTTP | `dio` | ^5.7.0 |
| WebSocket | `web_socket_channel` | ^3.0.2 |
| Local storage | `hive` + `hive_flutter` | ^2.2.3 / ^1.1.0 |
| Local storage (prefs) | `shared_preferences` | ^2.3.4 |
| Maps | `google_maps_flutter` | ^2.10.0 |
| Location / GPS | `geolocator` | ^13.0.2 |
| Permissions | `permission_handler` | ^11.3.1 |
| Geocoding | `geocoding` | ^3.0.0 |
| Camera / gallery | `image_picker` | ^1.1.2 |
| Signature capture | `signature` | ^5.5.0 (unused — POD planned) |
| Calendar widget | `table_calendar` | ^3.2.0 (unused — planned) |
| Fonts | `google_fonts` | ^6.2.1 |
| Images | `cached_network_image` | ^3.4.1 |
| SVG | `flutter_svg` | ^2.0.17 |
| Shimmer loading | `shimmer` | ^3.0.0 |
| i18n / date format | `intl` | ^0.20.0 |
| Env vars | `flutter_dotenv` | ^5.2.1 |
| UUID | `uuid` | ^4.5.1 |
| JSON annotation | `json_annotation` | ^4.9.0 |
| Equality | `equatable` | ^2.0.7 |
| URL launcher | `url_launcher` | ^6.3.1 |
