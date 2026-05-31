# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Markor is an Android text editor app supporting Markdown, todo.txt, Zim Wiki, AsciiDoc, CSV, Org-Mode, and other plaintext formats. Works completely offline. Created files are interoperable with any plaintext software.

## Build & Development Commands

### Prerequisites
- Android SDK (set `ANDROID_SDK_ROOT` environment variable)
- Java 21 (Temurin distribution preferred for CI alignment)

### Makefile targets (common development workflow)

```bash
make all             # spellcheck, lint, deptree, test, build, aapt_dump_badging
make build           # Build APK (Debug), output in dist/
make test            # Run unit tests (output in dist/tests/)
make lint            # Run Android lint (output in dist/lint/)
make deptree         # Print dependency tree
make install         # Install APK to connected device via adb
make run             # Launch app on connected device
make clean           # Full clean
```

### Gradle commands

```bash
# Build
./gradlew assembleFlavorDefaultDebug

# Run all unit tests
./gradlew testFlavorDefaultDebugUnitTest

# Run a specific test class
./gradlew testFlavorDefaultDebugUnitTest --tests "net.gsantner.markor.format.markdown.MarkdownHighlighterPatternBoldTest"

# Run a single test method
./gradlew testFlavorDefaultDebugUnitTest --tests "net.gsantner.markor.format.markdown.MarkdownHighlighterPatternBoldTest.someTestMethod"

# Lint
./gradlew lintFlavorDefaultDebug

# Build release (minified with ProGuard)
./gradlew assembleFlavorDefaultRelease
```

Tests use JUnit 4 and AssertJ. Test results go to `app/build/test-results/testFlavorDefaultDebugUnitTest/`.

## Architecture

### Two-layer package structure

The codebase has two Java package layers:

1. **`net.gsantner.opoc`** — Reusable Android library (shared across projects). Provides base classes for activities, fragments, file browsing, settings, WebView helpers, and utility classes. Licensed CC0/Public Domain. Everything here is generic — no Markor-specific logic.

2. **`net.gsantner.markor`** — Markor application code. Extends `opoc` base classes. Licensed Apache 2.0.

### Activity hierarchy

```
GsActivityBase (opoc)
  └── MarkorBaseActivity<AppSettings, MarkorContextUtils> (markor)
        ├── MainActivity          — Main screen with ViewPager2: Notebook, QuickNote, ToDo, More
        ├── DocumentActivity      — Single-document edit/view host
        ├── SettingsActivity      — Preferences
        └── ActionButtonSettingsActivity
```

### Fragment hierarchy

```
GsFragmentBase (opoc)
  └── MarkorBaseFragment<AppSettings, MarkorContextUtils> (markor)
        ├── DocumentEditAndViewFragment  — Core editor + preview for a single document
        ├── DocumentShareIntoFragment    — Handles shared-to-Markor content
        ├── MoreFragment / MoreInfoFragment
        └── GsFileBrowserFragment (opoc) — Used directly as Notebook browser
```

`DocumentEditAndViewFragment` is the heart of the app — it manages the editor (`HighlightingEditor`), the preview (`WebView`), and toggles between them. It is hosted both in `DocumentActivity` (standalone) and in `MainActivity`'s ViewPager (for QuickNote/ToDo).

### Format plugin system

The `FormatRegistry` maps each supported format to three components:

| Component | Base Class | Purpose |
|-----------|-----------|---------|
| **TextConverter** | `TextConverterBase` | Converts source text to HTML for preview (uses flexmark for Markdown, custom implementations for others) |
| **SyntaxHighlighter** | `SyntaxHighlighterBase` | Computes spans for editor syntax highlighting using Android's `Spannable` API. Only highlights visible viewport for performance |
| **ActionButtons** | `ActionButtonBase` | Provides the format-specific toolbar actions (e.g., insert bold/italic for Markdown, context/project insertion for todo.txt) |

Each format has its own package under `net.gsantner.markor.format.<formatname>/`:
- `markdown/` — Uses flexmark-java library for parsing
- `todotxt/` — Custom parser, supports contexts (@) and projects (+)
- `wikitext/` — Zim Wiki format, custom transpiler to Markdown
- `asciidoc/`, `orgmode/`, `csv/`, `keyvalue/`, `plaintext/`, `binary/`

`FormatRegistry.getFormat(int formatId, Context, Document)` is a factory that wires up all three components for a given format. `FormatRegistry.FORMATS` list determines format detection order by file extension and content heading.

### Key classes

- **`Document`** (`model/Document`) — Serializable model for a file. Holds path, title, extension, format ID, modification tracking, and AES encryption support (via jpencconverter).
- **`AppSettings`** (`model/AppSettings`) — All user preferences, extends `GsSharedPreferencesPropertyBackend`. Single instance via `AppSettings.get(context)`.
- **`ApplicationObject`** — Application singleton (`MultiDexApplication`), initializes `AppSettings` and pre-warms WebView.
- **`HighlightingEditor`** (`frontend/textview/`) — Custom `EditText` with syntax highlighting support.
- **`MarkorWebViewClient`** / `DraggableScrollbarWebView` — WebView customizations for preview mode.

### Product flavors

Three flavors defined in `app/build.gradle`:
- **`flavorDefault`** — Standard debug/release builds
- **`flavorAtest`** — Test build with `net.gsantner.markor_test` application ID (can be installed alongside production)
- **`flavorGplay`** — Google Play store build (`IS_GPLAY_BUILD = true`)

### Third-party code

`app/src/main/java/other/` contains vendored third-party code:
- `other.writeily` — Widget providers from predecessor project
- `other.de.stanetz.jpencconverter` — AES-256 file encryption
- `other.so.AndroidBug5497Workaround` — Fix for Android keyboard resize bug

### Code style

Follow AOSP Java Code Style. Use Android Studio's auto-reformat before committing. Min SDK is 17/18 (varies by flavor), compile SDK 35.
