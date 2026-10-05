# Crofit

A cross-platform Flutter app.

[![License: Apache 2.0](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)

## Getting Started

This project uses [Flutter](https://flutter.dev). Make sure you have the
Flutter SDK installed, then:

```bash
flutter pub get
flutter run
```

Supported platforms: Android, iOS, web, macOS, Linux, Windows.

## Editor Setup (VS Code)

This repo ships a `.vscode/` config (settings, recommended extensions,
debug launch configs, and tasks) so the project behaves the same for
everyone. Open the folder in VS Code and install the recommended
extensions when prompted (`Dart-Code.flutter`, `Dart-Code.dart-code`).

Useful tasks (Command Palette → "Run Task"):

- `flutter: analyze` — run the linter/static analysis.
- `dart: format (check only)` / `dart: format (apply)` — check or fix
  formatting.
- `flutter: test` — run the test suite.
- `git: enable local pre-push checks` — one-time setup, see below.

## Local Pre-Push Checks

A git hook under `.githooks/pre-push` runs `dart format` and
`flutter analyze` automatically before every `git push`, so formatting
or lint issues are caught locally instead of failing CI. Since git
doesn't track hooks by default, enable it once per clone:

```bash
git config core.hooksPath .githooks
```

(or run the `git: enable local pre-push checks` VS Code task above).
If either check fails, the push is aborted until it's fixed.

## CI

Every pull request against `main` runs a GitHub Actions pipeline
(`.github/workflows/ci.yml`) that checks formatting, runs
`flutter analyze`, and runs `flutter test`. A pull request can only be
merged once this pipeline passes.

## License

This project is licensed under the [Apache License 2.0](LICENSE).
