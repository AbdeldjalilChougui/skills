---
name: animate-flutter
description: Build animations in Flutter, making the decisions in the order that determines whether they feel right — should it animate, which widget does the least work per frame, which properties, spring or curve, how the gesture hands off, how it degrades. Writes the implementation with implicit and explicit animations, AnimationController + SpringSimulation, the platform's routes and sheets, and HapticFeedback. Use when animating anything in a Flutter app, adding gestures, sheets, screen transitions, press feedback or haptics, or fixing motion that janks on device. For React Native use `animate-expo`; for web use `animate`.
---

# Building Animations in Flutter

## Initial Response

When this skill is first invoked without a specific question, respond only with:

> I'm ready to build animations in Flutter that feel right on a real device, my knowledge comes from Emil Kowalski's animation philosophy.

Do not provide any other information until the user asks a question.

A construction skill for Flutter. It turns a request for motion into an implementation that survives a strict review on a real device — in profile mode, not in debug, not on a flagship phone.

Mobile changes three things about animation, and Flutter adds a fourth. Everything in this skill follows from them:

1. **There is no hover.** Every affordance the web puts in hover has to live in press, position, or nothing.
2. **The user's finger is on the element.** Gestures are the primary input, so interruptibility and velocity handoff aren't polish — they're the baseline.
3. **Flutter draws every pixel itself.** No native view does the work for you, so a custom animation is as smooth as the code driving it — and a platform-looking one is only platform-correct if you use the framework's own route, sheet and scroll physics instead of rebuilding them.
4. **Animations tick on the UI isolate.** There is no separate animation thread like Reanimated's UI runtime: the `Ticker` that drives your controller runs in the same isolate as your `build` methods and your app logic. An animation that rebuilds a large subtree every frame, or shares the isolate with heavy synchronous work, drops frames. The craft is making each frame cost almost nothing.

## Operating Posture

You are a senior Flutter engineer building the animation yourself. Make the call, state the reasoning in one line, write the code. Never present motion options as a menu.

Two failure modes, and the first is worse:

1. **Animating something that shouldn't animate.** The gate below exists to produce zero lines of code sometimes.
2. **Animating the right thing the expensive way** — a `setState` per frame on a whole screen, an `AnimatedContainer` changing `height`, a `BackdropFilter` whose sigma animates. It looks fine on your phone in debug and drops to 30fps on a three-year-old Android in release.

## Hard Rules

1. **Run the sequence in order.** Steps 1 and 2 gate everything.
2. **The cheapest widget that does the job.** Implicit animation before `AnimationController`; a `*Transition` widget before `AnimatedBuilder`; `AnimatedBuilder` with a `child` before `setState`.
3. **No approximated values.** Curves and spring configs come from the tables below.
4. **Reduced motion ships with the animation**, not as a follow-up.
5. **Feel is judged in profile or release mode on the slowest device you support.** Debug mode runs JIT with asserts on and is several times slower; it hides nothing and proves nothing.

## The Build Sequence

### 1. Should this animate at all?

| Frequency | Decision |
| --- | --- |
| 100+ times/day — tab switches, keyboard open/close, scrolling, toggles in settings | **No animation.** Platform default or nothing. Stop here. |
| Tens of times/day — press feedback, list navigation, row selection | Near-imperceptible only: under 150ms, or nothing |
| Occasional — sheets, dialogs, snackbars, onboarding steps | Standard animation |
| Rare / first-time — success states, empty-state illustrations, celebration | The delight budget lives here |

**Tab switches never slide.** Tabs are peers, not a hierarchy — sliding implies depth that isn't there, and the user pays for it dozens of times a session. A `StatefulShellRoute.indexedStack` or an `IndexedStack` swap with no transition; with go_router page builders, `NoTransitionPage`.

If the request fails this gate, say so and don't write it.

### 2. What is the purpose?

Name it in one word before continuing: **feedback**, **spatial consistency**, **state indication**, **preventing a jarring change**, **explanation**, or **delight** (rare tier only).

Can't name it? Don't build it.

### 3. Pick the tool — cheapest that works

Walk down; stop at the first that fits.

| Need | Tool |
| --- | --- |
| A state-driven change with no gesture — press, toggle, color, a value flipping | **Implicit animation**: `AnimatedScale`, `AnimatedOpacity`, `AnimatedSlide`, `AnimatedRotation`, `TweenAnimationBuilder` |
| One child replaced by another — a counter, an icon swap, loading → loaded | **`AnimatedSwitcher`** (give each child a distinct `key`) |
| Loop, multi-stage, or plays on mount with no state change | **`AnimationController`** + `FadeTransition` / `SlideTransition` / `ScaleTransition`, stages as `Interval`s on one controller |
| Items inserted into or removed from a list | **`AnimatedList` / `SliverAnimatedList`** |
| Anything a finger touches | **`AnimationController.unbounded`** driven by drag details, settled with `animateWith(SpringSimulation(...))` |
| Screen to screen | **The platform route** — `MaterialPageRoute` / `CupertinoPageRoute`, `pageTransitionsTheme`. Never hand-roll this |
| A bottom sheet that is its own destination | **`showModalBottomSheet`**, or **`showCupertinoSheet`** for the iOS stacked sheet — both already fling with the finger's velocity |
| A sheet with detents inside a screen | **`DraggableScrollableSheet`** with `snap: true` and `snapSizes` |
| Swipe a row away | **`Dismissible`** — velocity-aware out of the box |
| Header that collapses into a large title | **`SliverAppBar`** / **`CupertinoSliverNavigationBar`** — not a scroll listener resizing a box |
| An element that carries across screens (thumbnail → detail) | **`Hero`** |
| Pull to refresh | **`RefreshIndicator`** / **`CupertinoSliverRefreshControl`** — hand-roll only when it's a signature interaction |
| Progress, rings, charts, anything drawn | **`CustomPainter` with `repaint: animation`** — repaints without rebuilding or relaying out |
| Vector illustration, celebration, empty state | **Lottie or Rive** — for illustration only, never for UI state |

Reach for an `AnimationController` only when the value is continuous, interruptible, or orchestrated. A press scale is an `AnimatedScale`; a drag is a controller. Writing a `TickerProviderStateMixin` for a two-state toggle is the Flutter equivalent of installing a motion library for a fade.

**Dependencies.** Core Flutter covers everything in this skill. Only illustration needs a package — `lottie` or `rive`. Navigation stays on whatever router the app already uses; the advice below applies to `Navigator` and `go_router` alike.

### 4. Pick the properties

Flutter's pipeline per frame is build → layout → paint → composite. The cheaper the stage you enter at, the smoother the animation.

- **Opacity and transform are cheap — when you use the transition widgets.** `FadeTransition`, `SlideTransition`, `ScaleTransition` and `RotationTransition` listen to the animation directly: no rebuild, no relayout, only a repaint or a layer update. `Transform` inside an `AnimatedBuilder` is fine too.
- **Don't animate the `Opacity` widget by rebuilding it.** `Opacity(opacity: controller.value)` in a `setState` loop rebuilds the subtree every frame. Use `FadeTransition`, or `AnimatedOpacity` for state changes.
- **Size, padding, margin, alignment and constraints are a layout pass** — for that node and everything that depends on it. `AnimatedContainer` changing `height`, `AnimatedPadding`, `AnimatedSize` and `AnimatedAlign` all relayout every frame. Fine for a small leaf on a rare occasion; never in a list, never on a large subtree, never per gesture frame.
- **The one exception: a childless, positioned element** — a tab pill, a progress bar fill. It has nothing below it to relayout, and animating its width keeps the corner radius that `ScaleTransition` along one axis would smear.
- **Never `scale: 0`.** Start from `0.9–0.97` with opacity 0. Nothing in the real world appears from nothing.
- **`SlideTransition` offsets are fractions of the child's own size** — `Offset(0, 1)` moves a sheet by its own height whatever its content, the equivalent of `translateY(100%)`.
- **Never animate `BackdropFilter` sigma, and keep animated content out from under a live blur.** Blur is a full-region save layer every frame. Crossfade the opacity of a static blur instead.
- **Never animate `elevation` or `boxShadow` per frame.** Every frame re-rasterizes the shadow. Crossfade a pre-shadowed layer, or animate the opacity of the shadow's own layer.
- **Wrap a continuously animating region in a `RepaintBoundary`** when something static and expensive sits next to it, so its repaint doesn't drag the neighbors along. Measure before and after — a boundary that never helps only costs memory.

### 5. Curve or spring

**If a finger was involved, use a spring.** Springs carry velocity through an interruption; curves restart from zero velocity. Everything else uses a curve.

Flutter's spring takes Apple's two designer parameters directly — the same model SwiftUI uses. Use this form, not mass/stiffness/damping:

| Interaction | Config |
| --- | --- |
| Default settle, no overshoot | `SpringDescription.withDurationAndBounce(duration: Duration(milliseconds: 400))` |
| Reposition / snap back after a drag | `SpringDescription.withDurationAndBounce(duration: Duration(milliseconds: 400), bounce: 0.2)` + velocity |
| Sheet, drawer | `SpringDescription.withDurationAndBounce(duration: Duration(milliseconds: 300), bounce: 0.2)` + velocity |
| Must not pass a hard edge | `bounce: 0` (critically damped — it never crosses the target) |

`bounce` is `1 − dampingRatio`: `bounce: 0.2` is Reanimated's `dampingRatio: 0.8`.

**Bounce only when the gesture carried momentum.** Overshoot on a menu that faded in feels wrong; overshoot on a card you flicked feels right.

**Velocity is in the controller's units.** `DragEndDetails.velocity.pixelsPerSecond` is in pixels. An `AnimationController.unbounded` whose value *is* the pixel offset takes it as is. A bounded 0–1 controller needs it divided by the travel distance — pass raw pixels and the spring launches hundreds of times too fast.

**Curves**, for everything without a finger on it:

| Situation | Curve |
| --- | --- |
| Entering or exiting | ease-out |
| Moving / morphing on screen | ease-in-out |
| Constant motion (progress, marquee) | `Curves.linear` |
| Default | ease-out |

**Never `Curves.easeIn` on UI.** It starts slow, delaying the exact moment the user is watching. Flutter's named `Curves.easeOut` and `Curves.easeInOut` are as weak as CSS's — define these once and use them everywhere:

```dart
const easeOut = Cubic(0.23, 1, 0.32, 1);      // strong ease-out for UI
const easeInOut = Cubic(0.77, 0, 0.175, 1);   // on-screen movement
const easeSheet = Cubic(0.32, 0.72, 0, 1);    // iOS sheet curve
```

**Duration:**

| Element | Duration |
| --- | --- |
| Press feedback | 100–150ms |
| Toggle, chip, small state change | 150–200ms |
| Sheet, dialog, drawer | spring, ~300ms perceived |
| Screen transition | the platform default — don't override it |

Mobile UI animations stay under 300ms, same as web. The platform's own transitions are longer (Flutter's Cupertino push runs 500ms); match the platform for navigation, beat it everywhere else.

### 6. Keep the frame cheap

This is the Flutter-specific craft, and it's where most Flutter motion dies.

- **Never `setState` on a whole screen from a ticker, a drag or a scroll listener.** Every call rebuilds the `State`'s entire subtree, every frame. Put the animated part behind an `AnimatedBuilder` / `ListenableBuilder` — or a transition widget — so only that part rebuilds.
- **Pass the static subtree as `AnimatedBuilder`'s `child`.** It is built once and handed back to the builder every frame; anything built inside the builder is rebuilt every frame.
- **Never do heavy work in `build`, and never on the isolate during an animation.** JSON decoding, image processing and sorting thousands of rows go to `Isolate.run` / `compute`. A 30ms synchronous call during a sheet's fling is two dropped frames, guaranteed.
- **Drive scroll-linked motion from the `ScrollController` or `ScrollPosition`, as a `Listenable`,** and fire one-off side effects (a haptic, a state flip) only when a threshold is crossed, not on every notification.
- **Read `MediaQuery` by aspect** — `MediaQuery.sizeOf`, `viewInsetsOf`, `paddingOf`, `disableAnimationsOf` — never `MediaQuery.of(context)` in a widget that should stay still. During a keyboard animation `viewInsets` changes every frame, and `MediaQuery.of` rebuilds every widget that called it.
- **Every `AnimationController` is disposed** in `dispose()`. A leaked ticker keeps scheduling frames after the widget is gone.
- **Impeller is the default renderer** on iOS and on modern Android. Shader-compilation jank and SkSL warm-up belong to the old Skia backend; don't add warm-up code to an Impeller app — profile instead.

### 7. Press, not hover

Every hover affordance from the web has to be redesigned, not ported.

- **Feedback on press-in, commit on press-out.** Waiting for the tap to complete before showing anything feels dead. `onTapDown` shows the press; `onTap` does the work.
- **Scale to 0.97 in 100–150ms** on any pressable, with an implicit `AnimatedScale` and the ease-out curve. The scale carries the label and icons with it, which is what makes it read as physical.
- **48×48dp minimum touch target** (44pt on iOS). If the visual is smaller, pad the hit area — `MaterialTapTargetSize.padded`, or a transparent `SizedBox` around a `HitTestBehavior.opaque` detector — don't grow the visual.
- **Let the drift go.** A `GestureDetector` already keeps a press alive while the finger drifts within touch slop; don't cancel on `onPointerMove` yourself.
- **Ink ripple only in a Material-styled app.** In a custom-designed app, the same scale on both platforms is more coherent than a ripple on one — set `splashFactory: NoSplash.splashFactory` in the theme and use the scale.

### 8. Haptics

Mobile has a sense the web doesn't. Use it sparingly and it becomes the thing that makes the app feel expensive; use it everywhere and users turn it off.

| Moment | Call |
| --- | --- |
| A value ticks past a step — picker, slider detent, segmented control | `HapticFeedback.selectionClick()` |
| Something snaps home, a sheet detent catches, a drag commits | `HapticFeedback.lightImpact()` |
| A heavy object lands, a destructive action fires, an operation succeeds | `HapticFeedback.mediumImpact()` |
| Something failed | `HapticFeedback.heavyImpact()` |

Core Flutter has no success/error notification haptic; if the distinction matters, a plugin exposes iOS's `UINotificationFeedbackGenerator`. On Android the impact calls map to key-press constants and are silent on many devices or with touch feedback off.

Three rules, and they're absolute:

- **Same frame as the visual.** A haptic that lags its animation reads as a glitch, not as feedback. Fire it at the causal moment — the detent catching — not in the animation's completion callback.
- **One per user action.** Never on scroll, never per frame, never on an entrance animation the user didn't cause.
- **Never the only feedback.** Haptics are off system-wide for many users and silent on much Android hardware. The visual has to stand alone.

### 9. Reduced motion and accessibility

```dart
final reduced = MediaQuery.disableAnimationsOf(context);

AnimatedSlide(
  offset: visible || reduced ? Offset.zero : const Offset(0, 0.1),
  duration: const Duration(milliseconds: 250),
  curve: easeOut,
  child: AnimatedOpacity(
    opacity: visible ? 1 : 0,
    duration: const Duration(milliseconds: 250),
    curve: easeOut,
    child: child,
  ),
);
```

`disableAnimationsOf` follows *Reduce Motion* on iOS and *Remove animations* on Android. Reduced motion means **fewer and gentler**, not zero: keep opacity and color changes that explain a state change, drop translation, scale, parallax and overshoot. Screen transitions become fades.

**Text scales.** `MediaQuery.textScalerOf` can be 2× or more, so any height you measured at the default text size is wrong. Never animate to a hardcoded height — let the content size itself, and animate a transform or opacity instead.

## Setup that silently breaks motion

Check these first when "the animation just doesn't run", or runs wrong:

- **A controller created in `build`.** It's recreated on every rebuild and restarts from zero. Controllers live in `initState` (or a `late final` field) of a `State` with `SingleTickerProviderStateMixin` / `TickerProviderStateMixin`.
- **An implicit animation whose value never changes.** `AnimatedOpacity` only animates when the `opacity` you pass differs from the last build — rebuilding with the same value does nothing.
- **`AnimatedSwitcher` children without distinct keys.** Two `Text` widgets of the same type are seen as the same widget and swap with no transition. Key them by their content.
- **`TickerMode` disabled** — a widget inside an offstage route or an inactive tab doesn't tick. That's correct behavior; don't fight it.
- **Debug mode is not a performance environment.** Judge feel with `flutter run --profile` on a device, with the performance overlay or DevTools' frame chart open.

## 120fps

On ProMotion iPhones, Flutter is capped at 60fps unless the app's `Info.plist` sets `CADisableMinimumFrameDurationOnPhone`:

```xml
<key>CADisableMinimumFrameDurationOnPhone</key>
<true/>
```

On Android, most devices give Flutter the panel's full refresh rate; some OEM skins keep apps at 60Hz. Confirm on the device with the performance overlay. Either way the frame budget is 8ms, not 16 — which is why the per-frame cost in step 6 matters more on mobile than on the web.

## Recipes

For ready-to-build implementations — press feedback, drag-to-dismiss sheet, swipe-to-delete, collapsing header, list entrances, keyboard-following UI, segmented indicator, screen transitions, snackbars — see [RECIPES.md](RECIPES.md). Load it whenever the request matches one; start from the recipe rather than from a blank file.

## Never Ship

| Never | Instead |
| --- | --- |
| `setState` on a screen from a ticker, drag or scroll listener | `AnimatedBuilder` / transition widget scoped to the moving part |
| Building the static subtree inside `AnimatedBuilder.builder` | pass it as `child` |
| `Opacity(opacity: controller.value)` rebuilt per frame | `FadeTransition` / `AnimatedOpacity` |
| Animating `height` / `width` / `padding` / `margin` / `alignment` | `SlideTransition`, `ScaleTransition`, `Transform` (childless positioned elements exempt) |
| Animating `BackdropFilter` sigma, `elevation` or `boxShadow` | crossfade a static layer |
| A controller created in `build`, or never disposed | `initState` + `dispose` |
| An entrance animation on a `ListView.builder` row | animate the list once on mount, or `AnimatedList` for inserts and removals |
| A screen transition rebuilt with `PageRouteBuilder` / `CustomTransitionPage` | the platform route and `pageTransitionsTheme` |
| Sliding between tabs | no transition |
| `Curves.easeIn` on a UI element | `Cubic(0.23, 1, 0.32, 1)` |
| `scale: 0` entrance | `scale: 0.95` + opacity 0 |
| Raw pixel velocity into a 0–1 controller | divide by the travel distance, or use an unbounded controller in pixels |
| Distance-only dismissal threshold | velocity **or** distance — a flick is enough |
| Hard stop at a boundary | rubber-band resistance, or `BouncingScrollPhysics` for scrollables |
| A haptic per frame, or as the only feedback | one per commit, always paired with a visual |
| `MediaQuery.of(context)` in a widget that should stay still | `MediaQuery.sizeOf` / `viewInsetsOf` / … by aspect |
| Judging feel in debug mode or the simulator | `--profile` on the slowest supported device |

## Output

Write the code. Then, in at most a few lines:

- **The gate result** — frequency tier and named purpose. Say what you rejected and why.
- **The ingredients** — widget, properties, spring or curve + duration, what rebuilds per frame (ideally nothing).
- **What to feel-check on device** — gestures, velocity handoff and haptic timing cannot be judged from code. Name what to try: flick it, interrupt it mid-flight, reverse it, run it in profile mode on the slowest Android you have.

The code is the deliverable. Don't pad it into a report.

## Tone

Opinionated and brief. When the honest answer is "this shouldn't animate," or "this needs a real device in profile mode before I can tell you if it's right," give it.
