# Hex for Android

An unofficial Android port of [Hex](https://github.com/anomalyco/hex), the
local dictation app. A floating bubble records; Hex's own speech engine
transcribes on the phone; the text goes into whatever field you're typing in.
After a one-time model download, nothing you say leaves the phone.

This is a fork, not a release by Hex's authors. Everything Android lives in
`android/`; no file outside it changes, so `git merge upstream/dev` stays
clean.

## Install

Requirements: an arm64 Android phone on Android 11 or later, about 1.5 GB
free, and 2 GB of RAM to spare. Phones from 2021 on are fastest. Older ones
work, more slowly.

1. Download `hex-android-<version>.apk` and open it. Allow your browser or
   file manager to install apps when Android asks.
2. Open Hex and follow **Set up**: microphone, notifications, accessibility,
   battery, and the speech model (731 MB, use Wi-Fi).
3. **If the accessibility switch is greyed out:** Android restricts that
   setting for apps installed outside a store. Open App info for Hex, tap ⋮,
   then **Allow restricted settings**, and try again. Hex's setup row links
   there.
4. Tap **Turn on**. The bubble shows while a keyboard is open.

**Using the bubble**

- **Tap:** a tap starts recording; tap again to stop and insert. The rim turns
  coral while Hex is listening.
- **Hold:** hold to talk; release to insert. A held bubble grows and rises
  above your finger so you can see it listening.
- **Long-press:** cancels a tapped recording.
- **Drag:** moves the bubble; it snaps to the nearer edge.

**Customising:**

- **Bubble:** Shape and Size set the bubble's look (circle, square, hexagon,
  or desktop Hex's pill). Preview plays it without recording.
- **Dictation › Clean-up:** spoken formatting and filler-word removal are on
  by default; spoken symbols ("hashtag" → #) and word replacements are there
  to turn on.
- **Advanced:** desktop Hex's seven sound sets (off by default), how text is
  inserted, audio ducking, CPU threads, and the optional desktop sync.

To check the APK's signature before installing, compare the SHA-256
certificate digest published with the release against
`apksigner verify --print-certs hex-android-<version>.apk`.

## Dictation language

**Advanced › Dictation language** decides what the app tells the engine and the
text steps. `Auto` sends no language hint at all, so the engine uses the
model's own choice — the same as desktop Hex's `AUTO_LANGUAGE`. The text steps
fall back to English on `Auto`, because their filler-word list is English-only.
The setting is read at every dictation, so a change needs no restart.

With the single pinned English model, only English is useful today. This
setting is the plumbing for a model catalogue: once the app can load the
desktop's models, each one would offer its own supported languages, and a hint
would be sent only when the model accepts one (`accepts_language_hint` in
`src/transcription/models.rs`).

## What it reuses from Hex

| Part | Source in Hex | How |
|---|---|---|
| Speech engine | Hex's `transcribe.cpp` fork, the rev pinned in `../Cargo.toml` | `transcribe-cpp` crate, built with the NDK |
| Model | Parakeet Unified English 0.6B Q8_0 (`src/transcription/artifacts.rs`) | Same Hugging Face revision and SHA-256 |
| Vocabulary biasing | Parakeet rescoring in Hex's engine fork | Same term limit (64) and boost levels as `src/dictation/vocabulary.rs` |
| Text steps | `src/dictation/{spoken_formatting,spoken_symbols,filler_words,text_replacements}.rs` | Compiled directly with `#[path]`; their tests run here too |
| Look | Rune's Inter and IBM Plex Mono, colours sampled from `tests/golden/`, row icons from `src/rune_ui/icons.rs`, the mark from `app/AppIcon.icon` | `tools/icons-from-hex.py`; tokens and pieces in `Ui.kt` |
| Bubble | Dot scope (`src/rune_hud/dot_scope.rs`): braided traces, palette, Tuning defaults | `DotScope.kt` |

Not ported: LLM rewrite modes, live text, meetings, Hex's TypeScript
transformations, the Silero VAD.

## How it works

```
android/
├── core/              Rust cdylib over JNI: engine + text steps
├── app/               Kotlin, no AndroidX, no third-party dependencies
│   ├── HexAccessibilityService  owns the bubble (accessibility overlay) and inserts text
│   ├── Bubble, DotScope         gestures; the Dot scope drawing in four shapes
│   ├── DictationService         microphone foreground service; keeps the model loaded
│   ├── Dictation                record → transcribe → text steps → deliver
│   ├── Inserter                 edit the focused field at the caret, or paste
│   ├── ModelStore               download with resume, SHA-256 check
│   ├── Vocabulary               terms and sync with desktop Hex
│   └── MainActivity, Ui, Sheet  settings in desktop Hex's design
├── tools/             icons-from-hex.py, hex-vocab-sync (+ systemd unit)
├── build-core.sh      builds the core's two CPU variants
└── release.sh         signed release APK into dist/
```

- **Microphone:** Android lets an app use the microphone from the
  background only through a microphone foreground service started while the
  app is in front. Turning Hex on starts it. If Android stops it later, the
  next bubble press briefly opens an invisible activity to start it again.
- **Insertion:** on Android 13 and later Hex types the dictation through the
  accessibility service's input connection, as a keyboard would, so it works
  in browser address bars, web pages and Compose apps and keeps your
  clipboard. Older versions splice it into the field's text at the caret
  (`ACTION_SET_TEXT`). Anything that takes neither, or **Insert text ›
  Paste**, goes through the clipboard and `ACTION_PASTE`.
- **CPU variants:** ggml picks its kernels at build time, so the core ships
  twice: with dotprod and fp16 (most phones since 2018), and plain ARMv8-A
  (about 40% slower). `HexCore` loads the one that `/proc/cpuinfo` allows. On
  a Snapdragon 8 Gen 2, an 8 s dictation transcribes in about 1.4 s.

## Vocabulary sync with desktop Hex (optional)

Most people don't need this: the vocabulary works on the phone alone. It is
off until you pair, under Advanced › Desktop sync.

If you also use desktop Hex on Linux, the phone's vocabulary can sync with it
over [Tailscale](https://tailscale.com). `tools/hex-vocab-sync` runs on the
desktop. It merges three ways (the phone's list, the desktop's, and the list
both agreed on at the last sync), so additions and deletions on either side
carry over. It applies the result with `hex vocabulary add/remove`.

- **Network:** it listens only on the desktop's Tailscale addresses and needs
  a bearer token, kept in the GNOME Keyring. The app allows plain HTTP only to
  `*.ts.net` names, so pairing uses the desktop's MagicDNS name.
- **When:** Hex syncs when it turns on, when the app opens, after you edit the
  vocabulary, and from **Sync now**.
- **Safety:** a sync never deletes more than half of the desktop's terms at
  once.

Setup on the desktop, with the phone connected over USB and Hex opened once:

```sh
. android/env.sh                    # puts adb on PATH
android/tools/hex-vocab-sync self-test
android/tools/hex-vocab-sync pair   # creates the token and hands it to the phone
ln -s "$PWD/android/tools/hex-vocab-sync.service" ~/.config/systemd/user/
systemctl --user enable --now hex-vocab-sync
```

## Build

Prerequisites:

- Android SDK with platform 36, build-tools, platform-tools and NDK
  30.0.16248370. `env.sh` expects `~/Android/Sdk`; set `ANDROID_HOME`
  otherwise.
- Rust with the `aarch64-linux-android` target, `cargo-ndk`, CMake, Ninja,
  and JDK 17 or newer.
- Machine-specific paths (another SDK, a separate `RUSTUP_HOME`) go in
  `android/env.local.sh`, which `env.sh` reads and git ignores.

```sh
cd android
. ./env.sh
./gradlew assembleDebug        # builds the core first; -PskipCore reuses jniLibs/
adb install -r app/build/outputs/apk/debug/app-debug.apk
```

The core needs the whole fork checked out, because it compiles Hex's own
text-step files.

Tests and checks:

```sh
(cd core && cargo test --release)                 # includes Hex's own text-step tests
(cd core && cargo run --release --example smoke -- MODEL.gguf CLIP.wav "Term=sounds,like")
./gradlew lintDebug -PskipCore
```

To test insertion without speaking, focus a field on the phone and run:

```sh
adb shell "am broadcast -a app.hexforandroid.TEST_INSERT -n app.hexforandroid/.TestInsert --es text 'hello there'"
```

Only adb can send this: the receiver requires the `DUMP` permission, which
apps can't hold.

Release: `./release.sh` signs with the keystore named in `release.sh` and
copies the APK to `dist/`.

## Updating from upstream

```sh
git fetch upstream && git merge upstream/dev
```

If upstream moves the `transcribe.cpp` pin in `../Cargo.toml`, copy the new
rev into `core/Cargo.toml`, because a `[patch]` doesn't cross workspaces. If
one of the four text-step files starts depending on other Hex modules, the
core build fails, and the fix goes in `core/src/dictation.rs`.

## Licences

- **Code:** MIT, as Hex.
- **Speech engine:** transcribe.cpp, MIT.
- **Model:** NVIDIA Parakeet, CC-BY-4.0, downloaded from Hugging Face at
  first run.
- **Fonts:** Inter and IBM Plex Mono, SIL OFL 1.1 (`licenses/`).
- **Sounds:** desktop Hex's sound sets (`resources/audio/sounds/`), MIT.
