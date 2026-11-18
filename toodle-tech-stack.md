# Toodle: tech stack, libraries and why

The whole stack of Toodle (a baby app for the whole family: trackers, a Journey of milestones, a Learn path and one shared family record), read from `andepants/toodle` at `main` (`a581036`, "Phase 1: the daily app") on 2026-11-18, with the reasons taken from its architecture docs and from the decision records in `andepants/company` (`apps/toodle/decisions/`).

Where a reason was not written down anywhere, this file says so and marks the reason given as *inferred*.

---

## 1. The constraints every choice follows

From `docs/architecture/layers.md`. Most library choices below trace back to one of these.

1. **Logging never waits on the network.** A log is written locally first and synced later. Competitors' low reviews are 25 to 39 percent bugs, data loss and sync. → local SQLite as the source of truth, an outbox, offline-first sync.
2. **The app runs in a browser with no vendor keys.** Agents verify work by driving the web build. → every vendor sits behind a port with a fake adapter; react-native-web; Playwright.
3. **No baby or health data in analytics, crash reports or logs**, enforced by the type system, not review. → a typed analytics catalog, a crash scrubber.
4. **The mascot never blocks a parent.** No save, navigation or event waits on an animation.
5. **Health content is YMYL.** No health sentence ships unless it is sourced and signed in a claims registry.
6. **No sentence lives in code.** Every word comes from a language file (lint enforced).
7. **Nothing depends on which phone.** A family can be half iPhone, half Android. Platform-only surfaces sit behind one interface.

---

## 2. Platform and targets

| Piece | Choice | Why |
| --- | --- | --- |
| Framework | **Expo SDK 58** (`expo ~58.0.25`), **React Native 0.88**, **React 19.3** | Decision `2026-11-15-stay-on-expo`: Expo keeps over-the-air updates, browser testing and cloud-agent testing; Swift would need Xcode builds on the Mac mini for everything. A later switch would mean rewriting screens only; backend, data, Rive files and bundle ID carry over. Revisit only if a real feature is blocked after a genuine Expo attempt. |
| Routing | **Expo Router** (`~58.0.31`) with `experiments.typedRoutes` | File routes in `app/`, kept thin. `pnpm typecheck` writes route types first, so a link to a missing screen fails the type check. |
| Language | **TypeScript ~6.0, strict** | Types are the first check (see "Checks before prose"). Typed i18n keys, typed routes, typed analytics. |
| Web | **react-native-web ~0.21**, **react-dom** | Not shipped. It exists so agents can drive and screenshot the real app in a browser with fake vendors. |
| Package manager | **pnpm 10.28**, one package (no monorepo) | Pinned via `packageManager`; `pnpm install --frozen-lockfile` in 1 to 11 s. |
| Runtime | **Node ≥ 22.18** (`.nvmrc` 22) | Scripts are `.ts` run directly by Node (type stripping); tests use `node:sqlite`. |
| Targets | **iOS** (iPhone, folding iPhone, iPad) is the product. **Android** is tentative. **Web** is for verification only. | Decision `2026-11-16-full-app-scope` (superseded the iOS-only v1). No Apple Watch app for now. |

---

## 3. Architecture: layers and ports

```
app/            Expo Router routes. Thin. Compose features, no logic.
src/domain/     Pure TypeScript. Types, reducers, selectors, registries. No React, no IO.
src/services/   Ports (interfaces) and adapters (real, fake). The only place a vendor SDK is imported.
src/state/      The app store. Wraps the domain, persists through the record store, feeds sync.
src/features/   One folder per feature: screens, hooks, components.
src/ui/         Design tokens, primitives, the mascot, the i18n runtime.
src/locales/    Language files. Data only.
content/        Learn lessons and the claims snapshot. Data only.
supabase/       Migrations, policies, edge functions, pgTAP tests.
e2e/            Playwright flows that drive the web build.
```

- **Why ports:** constraint 2. Every vendor has a real and a fake adapter. `parseConfig` validates the environment, `createServices` picks each adapter once at startup. A dev or web build with a missing key gets the fake; a production build with a missing key fails at config time. No code outside `src/services` knows which adapter is live.
- **Import rules are lint, not prose** (custom ESLint plugin `lint/plugin.js`): domain imports only domain; a vendor SDK only inside `src/services/<port>/adapters/`; features never import each other; Rive only inside `src/ui/mascot/`; i18n runtime only inside `src/ui/i18n/`; no raw colors outside `src/ui/tokens`; no window reads outside `src/ui`; no sentence literals in code. Rule of the repo: when an agent repeats a mistake, the fix is a lint rule or a type, not another paragraph in `AGENTS.md`.
- **One action list** (`src/domain/actions`) is the only way a feature changes or reads the record. Each input is described once as plain data, and that one description is the validator, the JSON Schema for the AI assistant's tools, the stored-row reader and the database boundary parser.

---

## 4. State and data

| Concern | Library | Why |
| --- | --- | --- |
| App state | **Zustand 5** — one store of slices (`src/state/appStore.ts`) | Andrew, 18 Nov: one store, selectors in screens, no provider or context holding state. Slices: records, onboarding, account, prefs, pro, learn, timers, family, assistant. |
| Server state | **TanStack Query 5** — one `QueryClient` at the root | Only for real server calls (the Pro entitlement, family membership). Records do not go through it; they live in SQLite. |
| Forms | **react-hook-form 7** + **zod 4** (`@hookform/resolvers`) | For any form of more than one field. |
| Local database | **expo-sqlite ~58** on iOS, Android and web | Constraint 1: SQLite is the source of truth; a reload rebuilds every slice from it. On the web expo-sqlite runs wa-sqlite in a worker over the origin private file system. The store uses only the async API, so the page needs no cross-origin isolation. |
| Schema and migrations | **Drizzle ORM 0.45** + **drizzle-kit 0.31** (`pnpm db:generate`) | Drizzle is used for the **schema only**: `tables.ts` is the one description of the tables, and drizzle-kit writes the migrations. Its expo-sqlite driver runs only the synchronous API, which on the web needs cross-origin isolation, so the store keeps its own async driver and SQL. A test fails when the committed migration module differs. Never edit a shipped migration. |
| Preferences and flags | **react-native-mmkv 4** (native), `localStorage` (web), behind the `KeyValueStore` port | Fast synchronous key-value for small device keys. Zustand's `persist` stores the prefs slice under `toodle.prefs`. |
| Secrets | **expo-secure-store** | Secrets only. It holds just the key that seals the Supabase session, because SecureStore takes at most 2048 bytes. |
| Session storage | **expo-sqlite/localStorage** + **expo-crypto** (AES-256-GCM) | The Supabase session is sealed and kept in expo-sqlite's localStorage, as Supabase recommends; the key lives in SecureStore. |
| IDs | UUID v7 (time-ordered) | Two meals at one instant list newest-made first; ids are made on the device so offline writes never collide. |
| Native modules glue | **react-native-nitro-modules 0.36.5** | Required by MMKV 4 and Rive. Held on 0.36.x until Rive allows 0.37. |

**Storage roles, in one line:** SecureStore for secrets, MMKV for preferences, SQLite for records and the session.

**Sync model (own code, not a sync library):** local commits write an outbox row in the same transaction; a coordinator pushes and pulls through two Postgres functions (`sync_push`, at most 50 rows; `sync_pull`, at most 300 changes per page after a transaction-id cursor). Each entry and segment merges by itself on `updatedAt`; the server keeps its own row on a tie so every device converges. Timed sessions are stored as facts (run, pause, stop at an instant), not intervals, so every arrival order reads the same.

---

## 5. Backend and vendors

Decision `2026-11-15-backend-stack` and `2026-11-14-superwall-only`.

| Port | Real adapter | Fake adapter | Why this vendor |
| --- | --- | --- | --- |
| Backend, database, auth | **Supabase** (`@supabase/supabase-js` 2): Postgres with row-level security, Auth, edge functions | In-memory server / in-memory user | One Supabase project per app (portfolio policy). RLS mirrors the app's `can()` rule in SQL (`private.can`), tested with pgTAP. |
| Auth | Supabase Auth: email magic link, **Sign in with Apple** (`expo-apple-authentication`, native, SHA-256 nonce), Google (`expo-auth-session`, `expo-web-browser`) | Instant sign-in | Account required. Email-only at launch is *proposed*, not accepted (`2026-11-15-email-only-sign-in-at-launch`). |
| Email | **Resend** as Supabase's custom SMTP | — | Transactional and magic-link email from a verified Toodle domain. Blocked on the domain. |
| Paywall and subscriptions | **Superwall** (`expo-superwall` 1.7, Apple IAP underneath) | Entitled or not; a stand-in paywall | Superwall only, **no RevenueCat**, no web checkout or Stripe at launch. Prices and paywalls change in Superwall's dashboard, not in code. |
| Product analytics | **PostHog** (`posthog-react-native`, `posthog-js` on web), autocapture off | Records events | Full funnel; also the feature-flag system (`2026-11-15-ship-to-production-behind-flags`: after launch, features ship off and open for Andrew first). An account is identified by a random `analytics_id`, never its auth id. |
| Crash reporting | **Sentry** (`@sentry/react-native` 8), only when the DSN is set | Records | Every event goes through `scrubCrash`, held by a test (constraint 3). |
| Notifications | **expo-notifications** (+ `expo-task-manager`, `expo-background-task`) | Records scheduled items | Daily reminder and one "streak saver" reminder. Asks the system only while it has not answered. |
| Photos | **expo-image-picker**, resized to 2048 px by **expo-image-manipulator**, saved with **expo-media-library** | Nothing to pick | — |
| Store review | **expo-store-review**, asked once per device | Records the first ask | — |
| Connectivity | **@react-native-community/netinfo** | — | Sync knows when it can run. |
| AI assistant ("Ask Toodle") | A Supabase edge function (`supabase/functions/assistant`, Deno, `@anthropic-ai/sdk`), streamed to the app with `expo/fetch` | Scripted turns, tool calls included | The model's tools are generated from the same action list the app uses; only what a card may show reaches the model, undo values never do. The model is set by `ASSISTANT_MODEL`. |
| Account deletion | Edge function `delete-account` | — | Runs on the server; also asks PostHog to delete the analytics id. |
| OTA updates | **expo-updates**, channel per app environment, `runtimeVersion: appVersion` | Off until an EAS project exists | Over-the-air updates were a main reason to stay on Expo. |

**Vendor SDKs are imported only inside `src/services/<port>/adapters/`** (lint rule `toodle/vendor-sdk`).

---

## 6. UI

| Concern | Library | Why |
| --- | --- | --- |
| Styling | **Uniwind 1.14** (Tailwind 4 classes in `className`), `tailwindcss` 4 | Andrew, 18 Nov. `pnpm theme:css` generates `theme.css` from the TypeScript tokens in `src/ui/tokens`, so every value has one source; a test fails when the committed file differs. Light and dark are Uniwind themes. Tailwind preflight is left out so the browser's own controls are not restyled. |
| Colors | Tokens only, read with `useTheme()` when a value is needed | Lint fails a hex, `rgb()`, an arbitrary color class or a color in `StyleSheet.create` outside the tokens. Text tokens clear WCAG AA in both palettes. |
| Font | **Allo** 1.0.1 (SIL OFL), via **expo-font** | One face per weight and no `fontWeight`, since Android picks no face by weight. |
| Icons | **lucide-react-native**, through `src/ui/Icon` | One icon family, wrapped so it can change in one place. |
| Images | **expo-image** | — |
| Motion | **react-native-reanimated 4.7** + **react-native-worklets** | All animation that is not the mascot. |
| Mascot | **Rive** (`@rive-app/react-native` on device, `@rive-app/canvas` with bundled wasm on web, no CDN) | Decision `2026-11-15-stay-on-expo`: Expo + Reanimated + Rive (Duolingo uses Rive too). The `.riv` file is **generated from code** (`scripts/rive/`, `pnpm mascot:build`), never edited by hand; a test fails when the committed file differs from what the generator writes. Static art moved by Reanimated stands in under Reduce Motion, while loading, on failure and in Jest. |
| Gestures | **react-native-gesture-handler 3** | Root view `GestureRoot` at the app root. |
| Screens and safe area | **react-native-screens**, **react-native-safe-area-context** | Native navigation stacks. |
| Sheets | Expo Router's **`formSheet`** on iOS and Android, an own web sheet (`src/ui/Sheet`) | The system dismisses native sheets by drag. Gorhom's bottom sheet was removed; add it back only for a persistent snapping sheet. |
| Lists | **@shopify/flash-list 2**, through `Screen`'s `list` | Recycled rows for Home's timeline, a food's history, Statistics. |
| Keyboard | **react-native-keyboard-controller** | `KeyboardAwareScrollView` keeps the focused field above the keyboard on both platforms. |
| Toasts | **sonner-native**, through `toast()` | One toast at a time, above the floating tab bar, with Undo. |
| Haptics | **expo-haptics**, through `haptic()` | Skipped on web; off when the parent turns haptics off. |
| Glass and blur | **expo-glass-effect** (Liquid Glass on iOS 26), **expo-blur** before it, **expo-linear-gradient** | The floating tab bar and Log button. |
| Charts | **react-native-gifted-charts** on **react-native-svg** | Chart primitives in `src/ui/charts` (bars, a line, a day grid), drawn with tokens and read by VoiceOver as a sentence. |
| Date picker | **@react-native-community/datetimepicker** | System dialog on Android, the browser's date input on web. |
| Markdown | **react-native-enriched-markdown** | The assistant's replies (`src/ui/Markdown`). |
| QR codes | **react-native-qrcode-svg** | Family invite codes. |
| Share images and export | **react-native-view-shot**, **expo-sharing**, **expo-file-system** | The share card a parent posts; data export. `expo-print` is the chosen tool for PDF reports. |
| System chrome | **expo-status-bar**, **expo-system-ui**, **expo-splash-screen** (light and dark launch images), **expo-screen-orientation** | Phones locked to portrait at runtime; iPad and folds run at their own width. |
| Layout | Own `useScreenLayout()`: `phone` < 600, `fold` 600–839, `tablet` ≥ 840 points | 600 and 840 are Android's window classes, so a device lands in the same class on both platforms. Lint stops any file outside `src/ui` reading the window. |

---

## 7. Words, dates and languages

| Concern | Library | Why |
| --- | --- | --- |
| i18n | **i18next 26** + **react-i18next 17**, **expo-localization** | Constraint 6. One namespace per feature. Keys are typed from the English files, so a missing key fails the type check. A pseudo-language finds sentences hiding in code. |
| Dates | **date-fns 4** + **@date-fns/tz**, through `useFormat()` | Dates, times, money, counts on `Intl`; words follow the app's language, formats follow the phone's region. The day boundary is its own pure code (`src/domain/entries/day.ts`): an entry stores its instant and IANA zone, never a day, so a clock change or a flight never moves it. |
| Health content | Own Learn engine + claims registry (`content/claims.json` generated from the company repo's `health-claims.yml` with **yaml**) | Constraint 5. A lesson shows only when every claim it names is signed; a production build refuses while any draft remains. |

---

## 8. Testing and verification

The rules (AGENTS.md, "How we work", 18 Nov): **checks before prose, proof over review.** A verifier runs the one path a change is for, in the running app, on the exact commit, and returns a receipt.

| Layer | Tool | What it covers |
| --- | --- | --- |
| Lint | **ESLint 9** + `eslint-config-expo` + the custom `lint/plugin.js` | Layers, vendor SDKs, i18n, no literal copy, window size, raw color, no `any`. |
| Types | `tsc --noEmit`, strict, after route types | Routes, i18n keys, analytics payloads, action schemas. |
| Unit | **Jest 30** + **jest-expo**, `test-renderer` | A deliberately small core (cut to 15 files on 18 Nov: "tokens matter more than coverage while there are no users"): `can()` rights, hidden-equals-absent, the action door, the day boundary, the record-store contract on memory and on real SQLite (`node:sqlite`), the solids summary, locale keys, the analytics catalog, the generated mascot file, the lit-lesson gate. |
| Browser end to end | **Playwright 1.63** (`pnpm e2e`, the `verify-toodle` skill) | Exports the web app with every vendor key stripped, runs flows `boot`, `onboarding-happy-path`, `first-log`, `shell` at 393×852 with Reduce Motion on, at phone, fold or tablet size, light or dark. Screenshots and `result.json` per flow. |
| Database | **pgTAP** via `pg_prove` on plain local Postgres (`scripts/run-rls-tests.sh`) | One family cannot read or write another's rows; `private.can` answers exactly as `can()` does (a generated parity file). No Supabase CLI, no Docker, 2.5 s. |
| Device | **Maestro** (`.maestro/`, on the Mac) | One flow per Phase 1 feature on the iOS simulator. Planned as the default regression suite; not run yet. |

**The gate is local** (no hosted CI on every push; GitHub Actions cost too much, Andrew 17 Nov): `pnpm check`, `pnpm e2e`, `pnpm build:android`, and the RLS tests when `supabase/` or `can()` changes. GitHub workflows exist but start only by hand.

---

## 9. Build and release

| Piece | Choice | Why |
| --- | --- | --- |
| iOS builds | **The Mac mini**, `scripts/ios-preview.sh` (Xcode, CocoaPods) → internal **TestFlight** as "Toodle Preview" | Decision `2026-11-15-build-locally-first`: no cloud build service. Preview installs beside the real app with a PREVIEW band on its icon. Public App Review needs Andrew's own words. |
| Build config | **EAS config** (`eas.json`: `preview`, `production`) and `app.config.ts` with `APP_VARIANT` / `EXPO_PUBLIC_APP_ENV` | An EAS build without an app environment fails at config time; production refuses fakes, test controls and draft content. |
| Android | `pnpm build:android` exports the Hermes bundle in the gate; a debug-APK workflow by hand | Closed testing waits on a Google Play account. |
| Metro | `expo/metro-config` + `uniwind/metro`; `.riv` and `.wasm` as assets | Rive files and the web Rive runtime ship inside the bundle. |
| Generated assets | `pnpm mascot:build`, `theme:css`, `icon:android`, `icon:preview`, `claims`, `db:generate` | Every generated file has one source and, mostly, a test that fails when the committed copy drifts. Never edited by hand. |
| Shipping after launch | PostHog feature flags, off by default and when PostHog is unreachable | Ship to production behind flags; Andrew sees new features first. |

---

## 10. How the code gets written

Toodle is built by AI agents, so the stack includes the agent setup.

| Role | Model | Does |
| --- | --- | --- |
| Planner | Opus, high effort | Scope, data shape, sequencing, briefs, merging. No feature code. |
| Builder | Opus, high effort | All code; one unit per branch and pull request. |
| Grunt | Sonnet | Mechanical work with a fully specified result. |
| Verifier | A fresh session | Runs one path on the exact commit and returns a receipt. |

- **pstack** (Lauren Tan, MIT) is vendored under `.claude/` and pinned; coding runs in `/poteto-mode`.
- Rules born from the 17 Nov audit: a fresh builder for every fix round, at most two fix rounds per unit, only P0 (loses or leaks data, blocks a parent) and P1 (a visible bug, a wrong number) block a merge, at most three heavy agents at once on a 4-core machine.

---

## 11. Deliberately avoided

Listed in `AGENTS.md` (Andrew, 18 Nov). Reasons are written down only where noted; the rest are *inferred* from the architecture.

| Avoid | Reason |
| --- | --- |
| PowerSync, Legend-State, TinyBase | *Inferred:* sync and local state are already the app's own SQLite store, outbox and merge rules, with server rules mirrored in Postgres; a sync or reactive-store library would be a second source of truth. |
| Lottie | *Inferred:* Rive is the one animation format for the mascot, Reanimated for everything else. |
| `victory-native`, Skia | *Inferred:* charts are built on `react-native-gifted-charts` over SVG, which also runs in the web build agents verify. |
| `zeego` | Not written down. |
| `react-native-vision-camera` | *Inferred:* photos come from the system picker; no camera permission is asked (`cameraPermission: false`). |
| `sentry-expo` | *Inferred:* deprecated in favor of `@sentry/react-native`, which is used. |
| A URL polyfill | Written down: Expo already provides `URL`. |
| RevenueCat | Decision `2026-11-14-superwall-only`: Superwall only. |
| Gorhom bottom sheet | Removed; native `formSheet` instead. Back only for a persistent snapping sheet. |

---

## 12. Every dependency, at a glance

Versions from `package.json` at `a581036`. Install Expo packages with `EXPO_OFFLINE=1 npx expo install <pkg>` so versions match SDK 58, and run `npx expo install --check`.

### Runtime

| Package | Version | Used for |
| --- | --- | --- |
| `expo` | ~58.0.25 | Framework |
| `react`, `react-dom` | 19.3.0 | UI runtime |
| `react-native` | 0.88.1 | Native runtime |
| `react-native-web` | ~0.21.4 | Web build for verification |
| `expo-router` | ~58.0.31 | Routing |
| `@expo/metro-runtime`, `@expo/log-box` | ~58.0 | Metro runtime, error overlay |
| `zustand` | ^5.1.0 | App store |
| `@tanstack/react-query` | ^5.108.0 | Server state |
| `react-hook-form`, `@hookform/resolvers` | ^7.92.0, ^7.0.7 | Forms |
| `zod` | ^4.6.7 | Form schemas |
| `drizzle-orm` | 0.45.4 | Table schema |
| `expo-sqlite` | ~58.0.17 | Records, sealed session |
| `react-native-mmkv` | ^4.3.2 | Preferences |
| `react-native-nitro-modules` | 0.36.5 | Native module bridge (hold < 0.37) |
| `expo-secure-store` | ~58.0.3 | Session key |
| `expo-crypto` | ~58.0.8 | Session sealing, nonces |
| `@supabase/supabase-js` | ^2.119.0 | Backend, auth, sync |
| `expo-apple-authentication` | ~58.0.3 | Sign in with Apple |
| `expo-auth-session`, `expo-web-browser` | 58.0.18, ~58.0.8 | OAuth, in-app browser |
| `expo-linking` | ~58.0.17 | Deep links (email sign-in) |
| `expo-superwall` | 1.7.0 | Paywall, subscriptions |
| `posthog-react-native`, `posthog-js` | ^4.87.0, ^1.450.0 | Analytics, feature flags |
| `@sentry/react-native` | ~8.28.1 | Crash reporting |
| `expo-notifications` | ~58.1.9 | Reminders |
| `expo-task-manager`, `expo-background-task` | ~58.0 | Background work |
| `expo-image-picker`, `expo-image-manipulator`, `expo-media-library` | ~58.0 | Photos |
| `expo-store-review` | ~58.0.3 | Review prompt |
| `@react-native-community/netinfo` | 12.0.1 | Connectivity |
| `expo-updates` | ~58.0.22 | Over-the-air updates |
| `expo-application`, `expo-device`, `expo-constants` | ~58 | Build and device info for analytics and config |
| `uniwind` | 1.14.0 | Tailwind styling |
| `lucide-react-native` | ^1.74.0 | Icons |
| `expo-image` | ~58.0.18 | Images |
| `expo-font` | ~58.0.12 | Allo font |
| `expo-asset` | ~58.0.21 | Bundled assets (`.riv`, `.wasm`) |
| `react-native-reanimated`, `react-native-worklets` | 4.7.1, 0.13.0 | Motion |
| `@rive-app/react-native`, `@rive-app/canvas` | 0.5.15, 2.50.0 | Mascot on device and on web |
| `react-native-gesture-handler` | ~3.3.0 | Gestures |
| `react-native-screens`, `react-native-safe-area-context` | ~4.30.1, ~5.11.1 | Native stacks, insets |
| `@shopify/flash-list` | 2.3.2 | Long lists |
| `react-native-keyboard-controller` | 1.22.7 | Keyboard |
| `sonner-native` | ^0.27.0 | Toasts |
| `expo-haptics` | ~58.0.8 | Haptics |
| `expo-glass-effect`, `expo-blur`, `expo-linear-gradient` | ~58.0 | Glass tab bar, gradients |
| `react-native-svg` | 15.16.0 | SVG (charts, QR) |
| `react-native-gifted-charts` | ^1.4.83 | Charts |
| `@react-native-community/datetimepicker` | 9.2.1 | Date and time picker |
| `react-native-enriched-markdown` | 1.1.1 | Assistant replies |
| `react-native-qrcode-svg` | ^6.3.54 | Invite QR codes |
| `react-native-view-shot` | 6.1.1 | Share card image |
| `expo-sharing`, `expo-file-system` | ~58.0 | Export and share |
| `expo-print` | ~58.0.9 | PDF reports |
| `i18next`, `react-i18next` | 26.6.0, 17.1.10 | Languages |
| `expo-localization` | ~58.0.5 | Device language and region |
| `date-fns`, `@date-fns/tz` | ^4.4.0, ^1.5.0 | Date formatting, zones |
| `expo-status-bar`, `expo-system-ui`, `expo-splash-screen`, `expo-screen-orientation` | ~58.0 | System chrome, launch, orientation |

### Development

| Package | Version | Used for |
| --- | --- | --- |
| `typescript` | ~6.0.3 | Strict types |
| `eslint`, `eslint-config-expo` | ^9.39.5, ~58.0.4 | Lint (plus the custom plugin) |
| `jest`, `jest-expo`, `@types/jest`, `test-renderer` | ^30.5.2, ~58.0.14 | Unit tests |
| `@playwright/test` | 1.63.0 | Web end-to-end flows |
| `drizzle-kit` | 0.31.11 | Migrations |
| `tailwindcss` | ^4.3.3 | Uniwind's Tailwind 4 |
| `yaml` | 2.8.1 | Reading the claims registry |
| `@types/react`, `@types/node` | ~19.3.0, ^22.20.5 | Types |

### Outside npm

| Tool | Used for |
| --- | --- |
| Supabase (Postgres, RLS, Auth, Deno edge functions) | Server |
| `@anthropic-ai/sdk` (in the `assistant` edge function) | Ask Toodle |
| Resend | Email via Supabase SMTP |
| PostgreSQL + pgTAP + pgcrypto, `pg_prove` | RLS tests |
| Xcode, CocoaPods, Java 17, Maestro (on the Mac mini) | iOS builds, TestFlight, device flows |
| EAS (config only so far) | Build profiles, update channels |
| pstack (vendored Claude Code skills) | Agent workflow |
