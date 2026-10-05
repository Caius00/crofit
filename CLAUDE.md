# CLAUDE.md — Crofit

This file is the canonical, living source of project-specific rules for
anyone (human or AI) working on this codebase. It is read at the start of
every session and should guide every change made to this repo.

> This file grows over time as the project grows. See "How This File
> Evolves" at the bottom before editing it.

## 1. Project Overview

Crofit is a cross-platform Flutter fitness/training-tracking app targeting
Android, iOS, web, macOS, Linux, and Windows. The project is currently in
early skeleton stage (a single "Hello World" screen) — everything below
applies starting with the first real feature, not just once the app is
"finished."

The app is expected to eventually handle **sensitive personal health and
fitness data** (workouts, body metrics, possibly heart rate/sleep/nutrition
data). Treat this as true for every new feature from day one, not as a
future concern — see [Security Rules](#4-security-rules) and
[Health-Data Privacy Rules](#10-health-data-privacy-rules-gdpr).

Licensed under Apache-2.0 (see `LICENSE`). Repo: `Caius00/crofit` on GitHub.

## 2. Tech Stack & Versions

- **Flutter/Dart SDK:** `^3.11.4` (from `pubspec.yaml` — keep this file in
  sync if that changes).
- **Linting:** `flutter_lints: ^6.0.0`, to be extended with stricter rules
  (see [Testing Requirements](#8-testing-requirements)).
- **State management:** **Riverpod**, code-generation style
  (`flutter_riverpod` + `riverpod_annotation` + `riverpod_generator` +
  `build_runner`, using `@riverpod` annotations — not legacy manual
  `StateNotifierProvider`/`Provider` boilerplate). Add `riverpod_lint` +
  `custom_lint` as dev dependencies once Riverpod is introduced.
- **Routing:** not yet decided. Leading candidate: `go_router` (official,
  actively maintained, pairs well with Riverpod). Record the final choice
  in the [Decisions Log](#12-decisions-log).
- **Local persistence:** not yet decided. Must support encryption at rest
  once it stores any user data (see Security Rules). Leading candidates:
  `drift` + `sqlcipher_flutter_libs`, or encrypted `Isar`. Record the final
  choice in the Decisions Log.
- **Secrets/config:** `--dart-define` / `--dart-define-from-file` for
  build-time config, `flutter_secure_storage` for runtime secrets/tokens.
  Never commit `.env`-style files.
- **Supported platforms:** Android, iOS, web, macOS, Linux, Windows. Check
  every new package's platform support table before adding it — many
  packages silently drop Linux/Windows support.

## 3. Security Rules

Security is a top priority on this project. Treat every change — not just
features that obviously touch sensitive data — as a potential security
surface.

**Core rule:** if you find a security vulnerability while reading or
writing code, fix it immediately as part of the current task whenever
feasible. If a fix is genuinely out of scope right now (requires a product
decision, breaking API change, credentials, etc.), do **not** just mention
it in chat — flag it inline in the code with this exact marker:

```dart
// ! TODO(security): <short description of the vulnerability and why it
// wasn't fixed now> — flagged <YYYY-MM-DD>
```

- Always use `! TODO(security):` verbatim — the `!` makes it greppable and
  distinguishes it from ordinary TODOs.
- Always include the date in ISO format so staleness is visible.
- If a GitHub issue exists for it, append `— see #<issue-number>`.
- This marker is only for genuinely deferred fixes, never a substitute for
  an easy fix.

**Concrete rules:**

- No secrets, API keys, tokens, or credentials in source code,
  `pubspec.yaml`, or any committed config file. Use `--dart-define` for
  build-time values and `flutter_secure_storage` for anything persisted at
  runtime.
- All network calls use TLS (`https://` only). Any endpoint handling health
  data should use certificate pinning once a backend exists.
- Any locally persisted user data must be encrypted at rest. Plain,
  unencrypted storage (`sqflite`, `SharedPreferences`) is fine only for
  trivial, non-personal UI prefs (e.g. theme mode) — never for health or
  account data.
- **Never log personal or health data in plaintext** — not via `print`,
  not via the app logger, not via crash reporting. Any logging wrapper
  must redact fields tagged as sensitive. Configure PII scrubbing before
  enabling any crash-reporting SDK (Sentry/Crashlytics).
- Dependencies are not blindly trusted — see
  [Dependency / Package Policy](#9-dependency--package-policy); an
  abandoned or compromised package is a security issue too.
- Validate input at every trust boundary (user input, deep links, platform
  channels, file imports) — assume all external input is hostile.
- No `WebView`/`dart:html` rendering of remote content without explicit
  sanitization.
- Consider biometric/local-auth gating for health-data screens once they
  exist (candidate for the Decisions Log, not required on day one).

## 4. Architecture — Riverpod + Feature-First

- **Feature-first**, not layer-first, at the top level. Shared/cross-cutting
  code lives in `core/`; app-wide wiring lives in `app/`.
- Each feature is internally layered: `data` → `domain` → `presentation`.
  Dependencies flow one way: `presentation` depends on `domain`, `data`
  implements `domain`'s abstractions — never the reverse.
- No feature imports another feature's `data` or `presentation` layer
  directly. Cross-feature communication goes through `domain` contracts or
  shared `core/` services.
- A feature that only needs UI (no persistence/business logic) may skip
  `data`/`domain` and just have `presentation/` — don't force empty layers.

Target folder structure (build it incrementally as real features land —
don't scaffold it empty up front):

```
lib/
  app/
    app.dart                  # MaterialApp/MaterialApp.router root widget
    router/
      app_router.dart
    theme/
      app_theme.dart
      app_colors.dart
  core/
    constants/
    errors/
      failures.dart           # sealed Failure types
      exceptions.dart
    extensions/
    logging/
      app_logger.dart         # single wrapper, PII-safe
    network/
      api_client.dart
    storage/
      secure_storage_service.dart
      local_database.dart     # encrypted drift/Isar instance
    providers/
      shared_providers.dart
    utils/
    widgets/                  # shared *dumb* widgets only
  features/
    auth/
      data/
        datasources/
        models/
        repositories/
      domain/
        entities/
        repositories/        # abstract interfaces
        usecases/
      presentation/
        providers/
        screens/
        widgets/
    workouts/
      ...
    health_data/
      ...
    profile/
      ...
  bootstrap.dart              # env-specific init
  main.dart                   # thin entrypoint, calls bootstrap()
```

## 5. Coding Standards

- **Naming:** `lower_snake_case` files, `UpperCamelCase` classes/widgets,
  `lowerCamelCase` members. Riverpod providers follow the generator
  convention: `@riverpod class Xyz` → `xyzProvider`.
- **Formatting:** run `dart format` before every commit. Use trailing
  commas for multiline constructors.
- **Widget size:** a single `Widget build()` should not exceed roughly
  150 lines. If it does, extract private widget *classes* (not just
  methods — methods don't get `const` rebuild isolation). Screens should
  compose smaller widgets, not contain business logic.
- **No god-classes:** a class should not mix data-fetching, business
  logic, and UI state. A `Notifier`/`AsyncNotifier` orchestrates calls to
  `domain` usecases/repositories — it must not make raw HTTP/SQL calls
  itself.
- **Avoid `StatefulWidget`** except for pure, ephemeral UI-local state with
  no domain meaning (e.g. a `TextEditingController`, scroll position,
  local expand/collapse toggle). Anything representing app/domain state
  belongs in a Riverpod provider.
- **No business logic in `build()`** — no API calls, no raw parsing, no
  `Future`/`Stream` construction inside `build()`.
- **Immutability:** domain entities and models are immutable (`final`
  fields, `copyWith`).
- **Error handling:** no silent `catch (_) {}`. Use typed `Failure`
  classes from `core/errors/` and surface errors via `AsyncValue.error`,
  never via raw `print` or swallowed exceptions.
- **No `print()`** in committed code — use the `app_logger.dart` wrapper.

## 6. State Management Conventions (Riverpod)

- **`Provider`** (`@riverpod` function): derived/computed, read-only
  values with no mutation — formatted strings, filtered lists, stateless
  service/repository DI.
- **`Notifier`**: synchronous, in-memory mutable UI/app state with no
  async I/O — a form's current step, a filter selection, a toggle.
- **`AsyncNotifier`**: state loaded asynchronously (DB/network) that can
  also be mutated — the default for most feature state (e.g. "today's
  workouts", "current user profile"). Use `build()` for the initial load
  and `AsyncValue.guard` for mutations.
- **`StreamNotifier`**: for push-based sources (e.g. wearable sync, live
  timers).
- Providers depend on **domain repository interfaces**, never on `data`
  implementations directly — bind implementations via provider overrides
  so tests can substitute fakes/mocks.
- `ref.watch` for anything a widget should rebuild on; `ref.read` only
  inside callbacks/event handlers, never inside `build()`.
- Default to `autoDispose` (the generator default); if a provider must
  survive outside its usage scope, document why in a comment.
- Riverpod is the single state-management standard for this project — do
  not introduce `ChangeNotifier`/the `provider` package, `Bloc`, or
  `GetX` alongside it.

## 7. Testing Requirements

Mandatory, not aspirational:

- Every `domain/usecases/*` — unit tests, no I/O, no Flutter dependency.
- Every `data/repositories/*` — unit tests with faked/mocked data sources
  (`mocktail`, not `mockito` — null-safety-first, no code generation).
- Every `Notifier`/`AsyncNotifier` in `presentation/providers/*` — unit
  tests via `ProviderContainer` + overrides, asserting state transitions
  (`loading` → `data`/`error`).
- Non-trivial reusable widgets with conditional rendering — widget tests.
- Critical end-to-end flows (auth, logging a workout, deleting
  account/data — anything touching health data or irreversible actions) —
  `integration_test`, run at least before release builds.
- Test files mirror `lib/`: `test/features/workouts/domain/usecases/...`,
  same filename + `_test.dart`.
- Golden tests are optional — only for visually critical shared
  design-system widgets, since they're brittle across platforms.

`analysis_options.yaml` currently only includes `flutter_lints`. Once the
team is ready, strengthen it with: `avoid_print`,
`always_declare_return_types`, `cancel_subscriptions`, `close_sinks`,
`unnecessary_lambdas`, `use_super_parameters`, and
`analyzer: language: strict-casts/strict-inference/strict-raw-types:
true`. Record adoption in the Decisions Log.

CI is not yet set up. Once a `test/` folder exists, add a GitHub Actions
workflow running `flutter analyze` + `flutter test` on every PR.

## 8. Dependency / Package Policy

Before suggesting or adding any `flutter pub add <package>`, check:

1. **pub.dev score** — pub points should be near-max, reasonable
   popularity/likes. Low scores are a yellow flag, not an automatic
   rejection, but must be justified.
2. **Maintenance recency** — last published version ideally within 6–12
   months; if older, check the GitHub repo's recent activity and open
   issues before trusting it.
3. **Null-safety & SDK compatibility** — must declare sound null safety
   and an `sdk:` constraint compatible with `^3.11.4`.
4. **Known advisories** — cross-check for CVEs/security advisories and
   "discontinued"/"unlisted" flags on pub.dev before adding.
5. **License compatibility** — must be compatible with Apache-2.0 (MIT,
   BSD, Apache-2.0 are fine; GPL/AGPL/"source available" are not — flag
   explicitly if encountered).
6. **Platform support** — must support every platform the feature
   targets; verify on the package's pub.dev platform table.
7. **Transitive dependency weight** — prefer packages with few, well-known
   transitive dependencies over heavy ones for small features.
8. **Prefer official/first-party packages** (`flutter_riverpod`,
   `go_router`, packages under the `dart-lang`/`flutter` orgs) over
   community alternatives when functionality is equivalent.

Record the choice and rationale in the Decisions Log whenever a
significant dependency (state mgmt, DB, networking, auth) is added —
not needed for trivial UI-only packages.

## 9. Health-Data Privacy Rules (GDPR)

Given the EU/German context:

- **Data minimization** — only collect fields strictly necessary for the
  feature at hand; no "collect now, might use later" health fields.
- **Explicit consent** — collecting any health/fitness data (weight, heart
  rate, sleep, nutrition, run location, etc.) requires an explicit,
  specific opt-in — not a generic bundled ToS checkbox. Consent state
  itself must be stored and auditable.
- **Right to access/export** — users must be able to export their own
  data (JSON/CSV is enough for MVP); design data models with this in mind.
- **Right to deletion** — account/data deletion must be a real, complete
  cascade (local DB, secure storage, and any server-side/cloud copies and
  backups once they exist). Document any retention window if a hard
  delete isn't immediate.
- **Local-first by default** — prefer encrypted local storage over
  syncing to a backend until a backend has a proper legal basis and DPA.
  Any feature introducing server-side storage of health data needs a
  Decisions Log entry and a note that a DPIA (GDPR Art. 35) may be
  legally required before shipping.
- **No health data to third-party analytics/tracking SDKs.** If analytics
  are added later, only anonymous usage metrics may be tracked.
- **Anonymize/pseudonymize** before sending health data anywhere, even for
  debugging.
- **Retention** — define and document a retention policy once real
  backend storage exists.
- **Minors** — if the app is ever opened to minors, additional
  GDPR/age-verification rules apply; out of scope until a product
  decision is made.

## 10. Git / Commit Conventions

- Commit messages in English, Conventional Commits style: `feat:`,
  `fix:`, `refactor:`, `test:`, `docs:`, `chore:`, `security:`.
  Example: `feat(workouts): add workout logging screen`.
- Use the `security:` prefix specifically for commits that fix a flagged
  `! TODO(security)` item or otherwise address a vulnerability — keeps
  them greppable in `git log`.
- Branch naming: `feature/<short-desc>`, `fix/<short-desc>`,
  `security/<short-desc>`.
- No direct commits to `main` once collaborators exist (currently a
  solo/early-stage repo — revisit once a PR workflow/CI is set up).
- Don't commit generated files (`*.g.dart`, `*.freezed.dart`) once
  `build_runner` is introduced — run codegen in CI instead. Record the
  final choice in the Decisions Log.
- Never commit `.env*` files, keystores, or any secret material — check
  `.gitignore` whenever a new secret file pattern is introduced.

## 11. Decisions Log

| Date | Decision | Rationale | Alternatives considered |
|------|----------|-----------|--------------------------|
| 2026-10-05 | Riverpod (code-gen) is the state management standard | Compile-safe DI, testability, strong Flutter ecosystem support | Provider, Bloc, GetX |
| 2026-10-05 | Feature-first folder structure under `lib/features` | Scales better than layer-first as the app grows | Layer-first (`lib/screens`, `lib/models`, ...) |

## 12. How This File Evolves

This file is intentionally incomplete — it grows as the project grows.

- When a new architectural or tooling decision is made (new package, new
  pattern, new convention), add it to the Decisions Log **and** update
  the relevant section above in the same change.
- Prefer appending new subsections over rewriting existing ones — keep
  headings stable so this file stays a reliable reference across
  sessions.
- If a rule here turns out to be wrong or outdated, replace it and note
  the change in the Decisions Log with a one-line "why" — don't silently
  delete it.
- Proactively propose additions to this file when a recurring pattern, a
  newly adopted dependency, or a security/privacy consideration shows up
  that isn't documented here yet.
