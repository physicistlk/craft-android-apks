# Craft apps — Android 16 builds

Signed `arm64-v8a` APKs for the six [storytold](https://github.com/storytold) Craft apps,
built from the newest **stable** upstream releases, for Android 16 (16 KB page size).

Release: https://github.com/physicistlk/craft-android-apks/releases/tag/v2026.10.30

## APKs

| App | Upstream version | Upstream commit | Package | APK |
|---|---|---|---|---|
| WordCraft | v0.12.0 | `3de6ff95` | `ai.storyteller.wordcraft` | `wordcraft-0.12.0-arm64-v8a-20261030.apk` |
| DeckCraft | v0.12.0 | `29f0f777` | `ai.storyteller.deckcraft` | `deckcraft-0.12.0-arm64-v8a-20261030.apk` |
| GridCraft | v0.13.0 | `43284377` | `ai.storyteller.gridcraft` | `gridcraft-0.13.0-arm64-v8a-20261030.apk` |
| PdfCraft | v0.15.0 | `75360d1d` | `ai.storyteller.pdfcraft` | `pdfcraft-0.15.0-arm64-v8a-20261030.apk` |
| PhotoCraft | v0.17.0 | `b8c3f3aa8` | `ai.storyteller.photocraft` | `photocraft-0.17.0-arm64-v8a-20261030.apk` |
| VectorCraft | v0.18.0 | `68b68a30` | `ai.storyteller.vectorcraft` | `vectorcraft-0.18.0-arm64-v8a-20261030.apk` |

All six are `arm64-v8a` only, `minSdk 26`, `targetSdk 35`, signed with APK Signature Scheme
v2 + v3, and 16 KB page-size aligned (`0x4000` LOAD segments) for Android 16. They are **not**
official upstream builds — they are signed with local test keys (see "Signing" below).

Port repositories: [wordcraft-android](https://github.com/kanaaka/wordcraft-android),
[deckcraft-android](https://github.com/kanaaka/deckcraft-android),
[gridcraft-android](https://github.com/kanaaka/gridcraft-android),
[pdfcraft-android](https://github.com/kanaaka/pdfcraft-android),
[photocraft-android](https://github.com/kanaaka/photocraft-android),
[vectorcraft-android](https://github.com/kanaaka/vectorcraft-android).

## Tablet-compatibility changes

The ports are a shell crate around the unmodified upstream engine/UI plus a small set of
Android-specific additions. The following were implemented across the six apps (milestones 1–6):

### 1. System clipboard (text)
- winit's Android backend has no clipboard and never reports modifiers, so `egui-winit`'s
  `try_clipboard_shortcut` never fired and only the in-app simulated clipboard worked.
- `src/clipboard.rs` (per port) reads/writes `ClipboardManager`/`ClipData` over JNI, rewrites
  plain Ctrl+C/X/V into `Event::Copy`/`Cut`/`Paste` (consuming the key press as on desktop), and
  publishes `OutputCommand::CopyText` to the system clipboard.
- Wired into each app's own Paste paths: WordCraft `clipboard_text`, GridCraft `paste_text`,
  VectorCraft `clipboard_read` + a text-only `SystemClipboard`, PdfCraft
  `install_clipboard_text` (so *Create ▸ Clipboard* reads text).
- Android 10+ read restriction handled (exception described, logged, cleared; read answers
  "nothing"). Plain text only — pictures/HTML are not negotiated yet.

### 2. Open-with and share intents
- Incoming `ACTION_VIEW` and `ACTION_SEND` are read at startup; a `content://` URI is copied to a
  file in the app's cache dir through `ContentResolver.openInputStream` and opened on the first frame.
- Outgoing `ACTION_SEND` (text) via `intents::share_text` (chooser intent, no `FileProvider`
  needed for text). `share_file` is provided but documented as needing a `FileProvider` on
  Android 7+.
- Storage Access Framework pickers (`ACTION_OPEN_DOCUMENT`) are **not** available —
  `android-activity` 0.6.1 lacks `startActivityForResult` — so the in-app picker remains the way
  to open/save.

### 3. Flush preferences on suspend
- A lost-focus frame is detected in `raw_input_hook` (`raw.viewports` reporting
  `focused == Some(false)`) and preferences are written with `store_now` (or `save_prefs` for
  VectorCraft), so a swipe from Recents or a killed paused Activity no longer loses settings.
- PhotoCraft persists preferences as they change (via `Services::save_prefs`); it also gained
  crash-recovery autosave wiring (`recovery_services` → `RecoveryStore` in the app's data dir).

### 4. Back gesture closes the dialog, not the app
- `Key::BrowserBack` is rewritten to `Key::Escape` in every shell's `raw_input_hook`, so Back
  dismisses an open dialog/picker (which all these apps answer to Escape) instead of finishing the
  Activity. When nothing is open, Back does nothing.

### 5. Trackpad wheel and pen hover
- WordCraft and DeckCraft already vendored a patched winit; the patch was extended to GridCraft,
  PdfCraft, PhotoCraft and VectorCraft, so all six now forward `MotionAction::Scroll` (trackpad
  wheel / two-finger scroll → `MouseWheel`, with a preceding `CursorMoved` carrying the pointer
  position) and `MotionAction::HoverMove` (mouse/stylus hover → `CursorMoved`).

### 6. Stylus pressure for PhotoCraft
- The vendored winit now sets `Touch.force = Some(Force::Normalized(pointer.pressure()))`, which
  feeds PhotoCraft's `Stylus::update` (pressure curve, tablet-pressure preference) on Android,
  matching desktop behaviour.

### Other Android-specific work (carried from the ports)
- Two-finger scrolling translated into wheel events (`touch_scroll.rs`).
- Modifier tracking synthesized from key events (`modifiers.rs`), since the backend never emits
  `ModifiersChanged`.
- In-app file picker (no `rfd` on Android), `config_changes` so the Activity is not recreated,
  edge-to-edge theme variant B (Android 16 ignores the status-bar opt-out), file logger in the
  app's data dir, fonts folder under `Android/media/<package>/Fonts`.
- 16 KB ELF alignment via `-Wl,-z,max-page-size=16384` (NDK r27); `targetSdk 35` kept because
  `android-activity` 0.6.1 lacks `OnBackInvokedCallback`.

## Signing

Signed with **local test keys** (one per app, outside the repositories). Not official upstream
keys — these APKs cannot update an install made from an upstream/kanaaka key without an
uninstall first. The keys are in `craft_apps/.*-signing/` (mode 700/600) and are **not** in this
repo. See `output/SIGNING.md` in the source tree for fingerprints and rebuild commands.

## Verification

Each APK was checked with `aapt2` (package/version/ABI), `apksigner` (v2+v3), `llvm-readelf`
(ELF `LOAD` segments at `0x4000`), and `llvm-nm` (NativeActivity entry points). Per-APK reports
are in `output/verification/` of the source tree.

## Notes

- No Android device was connected during most of the builds; the clipboard and intent JNI paths are
  compile- and unit-test-verified. The final set was installed and smoke-tested on a Xiaomi Pad 5.
- Source tree: `/home/lk1/craft_apps` (upstream clones + ports + build trees + tools).
