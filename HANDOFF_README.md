# Smart Edge — Rotation Crash Fix: Build & Verify Handoff

## Context

This is a fork of [Imtiaz-Official/Smart-Edge](https://github.com/Imtiaz-Official/Smart-Edge)
(`com.imi.smartedge.sidebar.panel`), an open-source (MIT) Android sidebar/edge-panel app.

The app crashes on screen rotation with:

```
java.lang.IllegalArgumentException: Cannot coerce value to an empty range: maximum -81 is less than minimum 81.
    at com.bumptech.glide.c.k(Unknown Source:38)
    at com.imi.smartedge.sidebar.panel.FloatingPanelService.addEdgeHandle(Unknown Source:258)
    at com.imi.smartedge.sidebar.panel.FloatingPanelService.onConfigurationChanged(Unknown Source:60)
```

A second, independently-reported crash (different device, different ROM — see `PATCH_NOTES.md`)
hits the same root cause through a different call site:
`EdgeHandleView.setupImeListener` → `onApplyWindowInsets` (which also fires on rotation, not just IME
visibility changes).

**Root cause (confirmed via source review, not just decompiled stack traces):** several places compute

```kotlin
val maxOffset = (screenH / 2) - (handleHeight / 2) - safeMargin
...
params.y = requestedOffset.coerceIn(-maxOffset, maxOffset)
```

In landscape, `screenH` (the short physical dimension) can be small enough that `maxOffset` goes
**negative** once handle height and margin are subtracted — especially with an expanded/wide touch
area configured. `coerceIn(-maxOffset, maxOffset)` then becomes `coerceIn(positiveNumber,
negativeNumber)`, an inverted range, which Kotlin's stdlib throws on immediately.

## What's already been done (do NOT redo this)

Five call sites across two files have been patched to clamp `maxOffset` to a non-negative value.
Full diffs are in `PATCH_NOTES.md`. Summary: every `val maxOffset = (...)` line now ends with
`.coerceAtLeast(0)` (or `.coerceAtLeast(0f)` where the surrounding code is Float), so when there's
truly no room to offset the handle, it centers (offset `0`) instead of crashing.

Both known crash signatures (the `FloatingPanelService.addEdgeHandle` one and the
`EdgeHandleView.setupImeListener` one) go through call sites that are now patched — confirmed by
reading the actual function bodies, not just guessing from stack traces.

## What THIS session needs to do

1. **Verify the patch compiles.** Run a Gradle build (`./gradlew assembleDebug` or the release
   variant if the project needs a signed release-config test — debug is fine for a first pass) and
   fix any incidental syntax issues if they show up (there shouldn't be any — the diffs were applied
   surgically to five single lines and verified byte-for-byte after each edit, see `PATCH_NOTES.md`).

2. **Sanity-check no other call site was missed.** Before finalizing, re-run:
   ```
   grep -rn "coerceIn" --include="*.kt" .
   ```
   and confirm every `coerceIn(-X, X)` / `coerceIn(-X.toInt(), X.toInt())` pattern where `X` is a
   computed subtraction (not a fixed constant) has a `.coerceAtLeast(0)` upstream of it. The five
   known sites are listed in `PATCH_NOTES.md` — if this grep turns up something new (e.g. the
   codebase changed since this handoff was written), evaluate whether it needs the same treatment.

3. **Build and sign a debug (or self-signed release) APK** suitable for sideloading via `adb install`
   on a rooted device for manual testing. A debug-signed APK is sufficient for the human's own
   testing; it does not need to be a Play-Store-grade release build.

4. **Produce the APK as a downloadable/handoff artifact** so the human can `adb push`/`adb install`
   it from their Termux session on their phone. They do not have a build toolchain (JDK/Android SDK/
   Gradle) set up in Termux — the build must happen in this Claude Code session's environment, not
   on-device.

5. **Do not modify the five patched lines further** unless the build actually fails because of them
   — they've been verified correct against the source, byte-for-byte, before this handoff.

## Testing the human will do after receiving the APK

- Install over the existing app (`adb install -r <apk>`), so settings/prefs are preserved
- Reproduce the original repro: fullscreen video in a browser or gallery app, then rotate to
  landscape and back — should no longer crash
- Also test with the sidebar handle set to "invisible" via the dedicated checkbox (the other
  reporter's specific trigger condition) plus rotation, to confirm both known trigger paths are fixed

## If it works

The intent is to submit this as a PR back to the upstream repo
(`Imtiaz-Official/Smart-Edge`) once verified on-device, since it's a real, reproducible crash fix
with two independent corroborating reports. A clean, minimal diff (just the five `.coerceAtLeast(0)`
additions, no unrelated changes) makes for an easy upstream review — please don't refactor,
reformat, or "improve" anything beyond the scope of this fix in the process.
