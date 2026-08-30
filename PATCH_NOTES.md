# Patch Notes: Rotation Crash Fix

## Bug

`coerceIn(-maxOffset, maxOffset)` throws `IllegalArgumentException` when `maxOffset` is negative,
because that produces an inverted range (min > max). `maxOffset` is computed as
`(screenDimension / 2) - (handleDimension / 2) - safeMargin`, which goes negative in landscape when
the handle (especially with an expanded touch area) plus margin exceed half the shortened screen
dimension.

## Two independently reported crash traces, both traced to this same pattern

**Report 1** (OnePlus 15, OxygenOS 16, Android 16, rooted/KernelSU-Next):
```
java.lang.IllegalArgumentException: Cannot coerce value to an empty range: maximum -81 is less than minimum 81.
    at com.bumptech.glide.c.k(Unknown Source:38)
    at com.imi.smartedge.sidebar.panel.FloatingPanelService.addEdgeHandle(Unknown Source:258)
    at com.imi.smartedge.sidebar.panel.FloatingPanelService.addEdgeHandle$default(Unknown Source:5)
    at com.imi.smartedge.sidebar.panel.FloatingPanelService.onConfigurationChanged(Unknown Source:60)
    at android.app.ConfigurationController.performConfigurationChanged(ConfigurationController.java:316)
    ...
```
Trigger: any rotation while the sidebar handle is active (occurs regardless of "Show panel in
landscape" setting — that toggle only affects visibility, not whether `onConfigurationChanged`
attempts the calculation).

**Report 2** (Samsung Galaxy S22 Ultra, BeyondROM 8.3 / OneUI 8, Android 16, Magisk root):
```
java.lang.IllegalArgumentException: Cannot coerce value to an empty range: maximum -34 is less than minimum 34.
    at com.bumptech.glide.c.k(Unknown Source:38)
    at com.imi.smartedge.sidebar.panel.EdgeHandleView.setupImeListener$lambda$7(Unknown Source:169)
    at com.imi.smartedge.sidebar.panel.EdgeHandleView.g(Unknown Source:0)
    at com.imi.smartedge.sidebar.panel.t.r(Unknown Source:4)
    at J.P.onApplyWindowInsets(Unknown Source:36)
    ...
```
Trigger: handle set to "invisible" via the dedicated checkbox (not color/opacity transparency),
handle with expanded/wide touch area, rotate during video playback. Confirmed (by reading the actual
`setupImeListener` source) that this fires from the `else` branch of the IME-visibility check — i.e.
it runs on any `onApplyWindowInsets` call where the IME is *not* visible, which includes plain
rotation events, not just keyboard show/hide.

## Diffs applied (5 call sites, 2 files)

### `app/src/main/java/com/imi/smartedge/sidebar/panel/EdgeHandleView.kt`

**Line 216** (inside `setupImeListener`'s non-IME-visible branch — this is Report 2's exact crash site):
```diff
- val maxOffset = (screenHeight / 2) - (h / 2) - safeMargin
+ val maxOffset = ((screenHeight / 2) - (h / 2) - safeMargin).coerceAtLeast(0)
```

**Line 367** (inside the drag-reposition `ACTION_MOVE` handler; Float variant):
```diff
- val maxOffset = (screenH / 2f) - (height / 2f) - safeMargin
+ val maxOffset = ((screenH / 2f) - (height / 2f) - safeMargin).coerceAtLeast(0f)
```

**Line 601** (a third handle-positioning call site in the same file):
```diff
- val maxOffset = (screenH / 2) - (h / 2) - safeMargin
+ val maxOffset = ((screenH / 2) - (h / 2) - safeMargin).coerceAtLeast(0)
```

### `app/src/main/java/com/imi/smartedge/sidebar/panel/FloatingPanelService.kt`

**Line 580:**
```diff
- val maxOffset = (screenH / 2) - (h / 2) - safeMargin
+ val maxOffset = ((screenH / 2) - (h / 2) - safeMargin).coerceAtLeast(0)
```

**Line 647** (inside `addEdgeHandle`, called from `onConfigurationChanged` — this is Report 1's exact
crash site):
```diff
- val maxOffset = (screenH / 2) - (handleHeight / 2) - safeMargin
+ val maxOffset = ((screenH / 2) - (handleHeight / 2) - safeMargin).coerceAtLeast(0)
```

## Verification performed so far

- Each `sed` edit was applied individually and immediately re-printed with `sed -n '<line>p'` to
  confirm exact output before moving to the next.
- Final pass: `grep -n "coerceAtLeast"` across both files confirmed all five edits landed correctly,
  with correct `0` vs `0f` typing, and no corruption of surrounding parens/indentation.
- Source of `setupImeListener` (lines 183–230 of `EdgeHandleView.kt`) was read in full to confirm the
  patched line 216 is genuinely inside the code path Report 2's stack trace hit, not just a
  coincidentally-similar nearby line.

## Not yet done (this is what the build session is for)

- Actual compilation has **not** been attempted yet — no Gradle build has been run against these
  changes. There is a small chance of an unforeseen build issue, though the edits are minimal and
  syntactically conservative (parenthesization + a trailing method call, no logic restructuring).
- No on-device testing has been done yet — the fix is derived from careful reading of the crash
  traces and source code, not from a build-test-iterate loop.
