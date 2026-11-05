# Flutter Animation Recipes

Ready-to-build implementations for the cases that come up most in a Flutter app. Start from the recipe, then adapt.

---

## What the recipes assume

Core Flutter only — no packages. Written against Flutter 3.32+ (`SpringDescription.withDurationAndBounce`, `showCupertinoSheet`).

Shared constants every recipe below uses. Put them in one file and import it everywhere:

```dart
import 'dart:math' as math;

import 'package:flutter/material.dart';
import 'package:flutter/physics.dart';
import 'package:flutter/services.dart';

const easeOut = Cubic(0.23, 1, 0.32, 1);      // strong ease-out for UI
const easeInOut = Cubic(0.77, 0, 0.175, 1);   // on-screen movement
const easeSheet = Cubic(0.32, 0.72, 0, 1);    // iOS sheet curve

final settle = SpringDescription.withDurationAndBounce(
  duration: const Duration(milliseconds: 400),
);
final snapBack = SpringDescription.withDurationAndBounce(
  duration: const Duration(milliseconds: 400),
  bounce: 0.2,
);
final sheetSpring = SpringDescription.withDurationAndBounce(
  duration: const Duration(milliseconds: 300),
  bounce: 0.2,
);
```

Three conventions, explained once here instead of in every recipe:

- **Gesture-driven values live in an `AnimationController.unbounded` whose value is in pixels.** The drag writes `controller.value` directly; release hands the finger's velocity to `animateWith(SpringSimulation(...))`. One object holds the value, is interruptible, and carries velocity — the role a shared value plays in Reanimated.
- **Only the moving part rebuilds.** Every recipe wraps the animated widget in `AnimatedBuilder` (or uses a transition widget) and passes the static subtree as `child`.
- **Controllers are created in `initState` and disposed in `dispose`.**

---

## Two functions you'll need everywhere

Momentum projection decides *where a flick was going*, so a fast short swipe commits and a slow long one doesn't. Rubber-banding makes a boundary resist instead of stopping dead.

```dart
/// Where the finger would come to rest if it kept decelerating.
/// Apple's exponential-decay form — not the v²/2a from physics class.
/// [velocity] in pixels per second.
double project(double velocity, {double decelerationRate = 0.998}) =>
    (velocity / 1000) * decelerationRate / (1 - decelerationRate);

/// The further past the edge, the less the element follows.
double rubberband(double overshoot, double dimension, {double constant = 0.55}) =>
    (overshoot * dimension * constant) / (dimension + constant * overshoot.abs());
```

---

## Press feedback

Every pressable in the app. This passes the frequency gate only because it's near-imperceptible: 120ms and a 3% scale is the ceiling for something touched this often — anything longer or larger belongs to rarer moments, per step 1 in SKILL.md. No controller — an implicit `AnimatedScale` is the whole implementation.

```dart
class PressableScale extends StatefulWidget {
  const PressableScale({super.key, required this.onPressed, required this.child});

  final VoidCallback? onPressed;
  final Widget child;

  @override
  State<PressableScale> createState() => _PressableScaleState();
}

class _PressableScaleState extends State<PressableScale> {
  bool _pressed = false;

  void _set(bool pressed) {
    if (_pressed != pressed) setState(() => _pressed = pressed);
  }

  @override
  Widget build(BuildContext context) {
    final enabled = widget.onPressed != null;
    return GestureDetector(
      behavior: HitTestBehavior.opaque,
      onTapDown: enabled ? (_) => _set(true) : null,
      onTapUp: enabled ? (_) => _set(false) : null,
      onTapCancel: enabled ? () => _set(false) : null,
      onTap: widget.onPressed,
      child: ConstrainedBox(
        constraints: const BoxConstraints(minWidth: 48, minHeight: 48),
        child: AnimatedScale(
          scale: _pressed ? 0.97 : 1,
          duration: const Duration(milliseconds: 120),
          curve: easeOut,
          child: widget.child,
        ),
      ),
    );
  }
}
```

`setState` is fine here — it fires twice per press, not per frame, and the `State` owns nothing but the scale. `HitTestBehavior.opaque` plus the 48dp minimum brings a small icon up to the touch target without growing the visual.

To dress an existing button without stealing its tap, swap the `GestureDetector` for a `Listener` (`onPointerDown` / `onPointerUp` / `onPointerCancel`) — a `Listener` doesn't enter the gesture arena, so the inner button still receives the tap. Give that wrapper its own `enabled` flag: a `Listener` scales a disabled button too, because it never learns the button is disabled. There is no theme-level way to give stock buttons a press scale — `ButtonStyle.backgroundBuilder` can't, since a local `backgroundColor` paints outside it — so wrap them.

---

## Bottom sheet you can drag to dismiss

Before writing this: if the sheet is its own destination, call `showModalBottomSheet` or `showCupertinoSheet` and get the framework's sheet — it already flings with the finger's velocity. Build this only when the sheet has to live inside an existing screen.

```dart
class DragSheet extends StatefulWidget {
  const DragSheet({super.key, required this.height, required this.onClose, required this.child});

  final double height;
  final VoidCallback onClose;
  final Widget child;

  @override
  State<DragSheet> createState() => _DragSheetState();
}

class _DragSheetState extends State<DragSheet> with SingleTickerProviderStateMixin {
  // Value = how far the sheet is pushed down, in pixels. 0 = fully open.
  late final AnimationController _y = AnimationController.unbounded(vsync: this);

  @override
  void dispose() {
    _y.dispose();
    super.dispose();
  }

  void _onDragStart(DragStartDetails _) {
    _y.stop(); // grab it mid-flight: continue from where the eye last saw it
  }

  void _onDragUpdate(DragUpdateDetails d) {
    final next = _y.value + d.delta.dy;
    // downward is free; upward past the top resists
    _y.value = next >= 0 ? next : -rubberband(-next, widget.height);
  }

  void _onDragEnd(DragEndDetails d) {
    final v = d.velocity.pixelsPerSecond.dy;
    final projected = _y.value + project(v);
    if (projected > widget.height * 0.4) {
      _y
          .animateWith(SpringSimulation(settle, _y.value, widget.height, v))
          .whenCompleteOrCancel(widget.onClose);
    } else {
      HapticFeedback.lightImpact(); // it snapped home — same moment as the visual
      _y.animateWith(SpringSimulation(sheetSpring, _y.value, 0, v));
    }
  }

  @override
  Widget build(BuildContext context) {
    final sheet = GestureDetector(
      onVerticalDragStart: _onDragStart,
      onVerticalDragUpdate: _onDragUpdate,
      onVerticalDragEnd: _onDragEnd,
      child: SizedBox(height: widget.height, child: widget.child),
    );

    return Stack(
      children: [
        // The backdrop derives from the same value, so it's always in sync.
        Positioned.fill(
          child: AnimatedBuilder(
            animation: _y,
            builder: (context, _) => ColoredBox(
              color: Colors.black.withValues(
                alpha: 0.4 * (1 - (_y.value / widget.height)).clamp(0.0, 1.0),
              ),
            ),
          ),
        ),
        Align(
          alignment: Alignment.bottomCenter,
          child: AnimatedBuilder(
            animation: _y,
            child: sheet, // built once, not per frame
            builder: (context, child) =>
                Transform.translate(offset: Offset(0, _y.value), child: child),
          ),
        ),
      ],
    );
  }
}
```

The four details that separate this from a bad drag:

- **`_y.stop()` on drag start.** Without it, grabbing a sheet mid-spring fights the simulation — the finger and the spring both write the value.
- **Velocity decides, not distance.** `project()` means a quick flick dismisses even a few pixels down. Requiring 40% travel makes the sheet feel heavy.
- **Velocity is handed to the spring**, so there's no seam between the finger releasing and the animation continuing. This is the single detail that most separates "fluid" from "fine".
- **Critically damped on dismissal** (`settle`, bounce 0) — otherwise the sheet springs past the bottom of the screen and flashes a gap.

`Transform.translate` moves the painted sheet without relaying it out; the `child` argument keeps its contents from rebuilding while it moves.

---

## Swipe to delete a row

Before writing this: `Dismissible` already does swipe-to-dismiss with a velocity threshold, a resize animation for the gap, and `confirmDismiss` for undo flows. Use it unless the interaction is different in kind.

```dart
Dismissible(
  key: ValueKey(item.id),
  direction: DismissDirection.endToStart,
  dismissThresholds: const {DismissDirection.endToStart: 0.4},
  movementDuration: const Duration(milliseconds: 200),
  resizeDuration: const Duration(milliseconds: 200),
  background: const ColoredBox(color: Colors.red),
  onDismissed: (_) {
    HapticFeedback.mediumImpact(); // destructive action fired
    onDelete(item.id);
  },
  child: ItemRow(item: item),
)
```

A fling past 700px/s dismisses regardless of distance — `Dismissible` already honors "velocity **or** distance". `resizeDuration` is the gap closing: the list's reflow, done for you.

The key must be the item's identity, never its index. An index key makes Flutter reuse the dismissed row's `State` for its neighbor, and the next row appears already swiped.

---

## Collapsing header on scroll

Use the slivers — they collapse without relaying out the list:

```dart
CustomScrollView(
  slivers: [
    SliverAppBar.large(title: const Text('Orders'), pinned: true),
    SliverList.builder(
      itemCount: orders.length,
      itemBuilder: (context, i) => OrderRow(order: orders[i]),
    ),
  ],
)
```

On iOS-styled screens, `CupertinoSliverNavigationBar` gives the native large title.

For a custom header that fades its content as the list scrolls, read the scroll offset as a `Listenable` and touch only the header:

```dart
final _scroll = ScrollController(); // created in initState, disposed in dispose

AnimatedBuilder(
  animation: _scroll,
  child: const HeaderContent(),
  builder: (context, child) {
    final offset = _scroll.hasClients ? _scroll.offset : 0.0;
    final t = (offset / 60).clamp(0.0, 1.0); // clamp is not optional
    return Opacity(
      opacity: 1 - t,
      child: Transform.translate(offset: Offset(0, -12 * t), child: child),
    );
  },
)
```

**Never resize the header's box to collapse it.** That runs a layout pass on the header and everything below it on every scroll frame — the one animation guaranteed to stutter, because it competes with the scroll itself. Give the header a fixed height and translate its content inside a `ClipRect`.

The clamp is not optional: without it, scrolling past 60 drives opacity negative and Flutter throws an assertion in debug — and in release the header misbehaves at the bottom of a long list.

---

## List entrances

One controller for the group, staggered with `Interval`s — not one controller per row:

```dart
class StaggeredColumn extends StatefulWidget {
  const StaggeredColumn({super.key, required this.children});

  final List<Widget> children;

  @override
  State<StaggeredColumn> createState() => _StaggeredColumnState();
}

class _StaggeredColumnState extends State<StaggeredColumn> with SingleTickerProviderStateMixin {
  static const _item = 250;    // ms each item takes
  static const _stagger = 40;  // ms between items
  static const _maxSlots = 8;  // after eight, the eye stops following

  late final int _slots = math.min(widget.children.length, _maxSlots);
  late final AnimationController _c = AnimationController(
    vsync: this,
    duration: Duration(milliseconds: _item + _stagger * math.max(0, _slots - 1)),
  );
  // Built once, not in build(): one CurvedAnimation per slot.
  late final List<Animation<double>> _slotAnims = [
    for (var s = 0; s < _slots; s++)
      CurvedAnimation(
        parent: _c,
        curve: Interval(
          s * _stagger / _c.duration!.inMilliseconds,
          (s * _stagger + _item) / _c.duration!.inMilliseconds,
          curve: easeOut,
        ),
      ),
  ];
  static final _offset = Tween(begin: const Offset(0, 0.15), end: Offset.zero);

  @override
  void didChangeDependencies() {
    super.didChangeDependencies();
    if (MediaQuery.disableAnimationsOf(context)) {
      _c.value = 1;
    } else if (!_c.isAnimating && _c.value == 0) {
      _c.forward();
    }
  }

  @override
  void dispose() {
    _c.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        for (var i = 0; i < widget.children.length; i++)
          FadeTransition(
            opacity: _slotAnims[math.min(i, _slots - 1)],
            child: SlideTransition(
              position: _offset.animate(_slotAnims[math.min(i, _slots - 1)]),
              child: widget.children[i],
            ),
          ),
      ],
    );
  }
}
```

Stagger 30–80ms. Longer feels slow, shorter reads as simultaneous. Items past the eighth share the last slot: the eye stops following a cascade that long, and the group would otherwise take over half a second to land.

**Never put an entrance animation on a row of `ListView.builder`, `SliverList` or `GridView.builder`.** Rows are disposed when they scroll out and rebuilt when they scroll back, so the animation replays every time — the list flickers while the user scrolls. Animate the list container once on mount, or use `AnimatedList` for inserts and removals only.

Entrance animations are for content the user asked for and is waiting on. A list they scroll past all day should already be there.

---

## Keyboard-following UI

Flutter updates `MediaQuery.viewInsets` on every frame of the keyboard's own animation, on both platforms. Follow it; never animate toward it.

```dart
class KeyboardFollowingBar extends StatelessWidget {
  const KeyboardFollowingBar({super.key, required this.child});

  final Widget child;

  @override
  Widget build(BuildContext context) {
    // Aspect-specific: only this widget rebuilds while the keyboard moves.
    final inset = MediaQuery.viewInsetsOf(context).bottom;
    return Transform.translate(offset: Offset(0, -inset), child: child);
  }
}
```

Put it in a `Stack` over a `Scaffold(resizeToAvoidBottomInset: false)` when the content behind shouldn't relayout. Keep `resizeToAvoidBottomInset: true` (the default) for a form that must scroll its focused field into view — that resize is a relayout per keyboard frame, so keep the body behind it light.

Never build this from a focus listener plus a timed animation. The keyboard rides a private system curve, and any duration you pick visibly lags or leads it.

---

## Segmented / tab indicator

The pill is a childless, positioned element — the sanctioned case for animating a size. Measure the segments once with a `LayoutBuilder` (equal widths here), then let `AnimatedPositioned` move it:

```dart
class Segmented extends StatelessWidget {
  const Segmented({super.key, required this.labels, required this.index, required this.onChanged});

  final List<String> labels;
  final int index;
  final ValueChanged<int> onChanged;

  @override
  Widget build(BuildContext context) {
    return LayoutBuilder(builder: (context, box) {
      final w = box.maxWidth / labels.length;
      return SizedBox(
        height: 40,
        child: Stack(
          children: [
            AnimatedPositioned(
              duration: const Duration(milliseconds: 250),
              curve: easeInOut,
              left: index * w,
              top: 0,
              bottom: 0,
              width: w,
              child: const DecoratedBox(
                decoration: BoxDecoration(
                  color: Colors.white,
                  borderRadius: BorderRadius.all(Radius.circular(10)),
                ),
              ),
            ),
            Row(
              children: [
                for (var i = 0; i < labels.length; i++)
                  Expanded(
                    child: GestureDetector(
                      behavior: HitTestBehavior.opaque,
                      onTap: () {
                        if (i == index) return;
                        HapticFeedback.selectionClick(); // on the press, not when the pill lands
                        onChanged(i);
                      },
                      child: Center(child: Text(labels[i])),
                    ),
                  ),
              ],
            ),
          ],
        ),
      );
    });
  }
}
```

Ease-in-out, because the pill is moving across the screen rather than entering or leaving it. On an iOS-styled screen, `CupertinoSlidingSegmentedControl` is this, already done.

---

## Screen transitions

Configure the theme; never rebuild a screen transition with `PageRouteBuilder` or go_router's `CustomTransitionPage`. A custom page route loses the iOS edge-swipe back gesture and Android's predictive back, and dragging back stops matching every other app on the device.

```dart
MaterialApp.router(
  theme: ThemeData(
    pageTransitionsTheme: const PageTransitionsTheme(
      builders: {
        TargetPlatform.android: PredictiveBackPageTransitionsBuilder(),
        TargetPlatform.iOS: CupertinoPageTransitionsBuilder(),
      },
    ),
  ),
  routerConfig: router,
)
```

| Navigation | Option |
| --- | --- |
| Deeper into a hierarchy | the platform push — `MaterialPage` / `MaterialPageRoute`, unmodified |
| A self-contained task the user can abandon | `fullscreenDialog: true` |
| A short interruption: picker, filter, share | `showModalBottomSheet` / `showCupertinoSheet` |
| Between tabs | no transition — `StatefulShellRoute.indexedStack`, or `NoTransitionPage` |
| Reduced motion | a plain fade — the wrapping builder below |

Not `FadeForwardsPageTransitionsBuilder` for reduced motion: despite the name it also slides pages 25% sideways. Wrap the platform builder instead — a `PageTransitionsBuilder` (unlike a custom route) keeps the back gestures whenever motion is on:

```dart
class ReducedMotionAware extends PageTransitionsBuilder {
  const ReducedMotionAware(this.platform);

  final PageTransitionsBuilder platform;
  static final _fade = CurveTween(curve: easeOut); // static: no allocation per frame

  @override
  Widget buildTransitions<T>(PageRoute<T> route, BuildContext context, Animation<double> animation,
      Animation<double> secondaryAnimation, Widget child) {
    if (MediaQuery.disableAnimationsOf(context)) {
      return FadeTransition(opacity: animation.drive(_fade), child: child);
    }
    return platform.buildTransitions(route, context, animation, secondaryAnimation, child);
  }
}

// builders: {TargetPlatform.iOS: ReducedMotionAware(CupertinoPageTransitionsBuilder()), ...}
```

`PredictiveBackPageTransitionsBuilder` needs `android:enableOnBackInvokedCallback="true"` on the `<application>` in `AndroidManifest.xml`; without it, Android 14+ shows no back preview.

`showModalBottomSheet` takes `sheetAnimationStyle: AnimationStyle(duration: ..., reverseDuration: ..., curve: easeSheet)` when the default timing needs tuning. Its drag-to-dismiss already uses the release velocity.

---

## Snackbar / toast

The framework's floating `SnackBar` already enters and exits as a pair and respects safe areas:

```dart
ScaffoldMessenger.of(context).showSnackBar(
  const SnackBar(
    content: Text('Saved'),
    behavior: SnackBarBehavior.floating,
    duration: Duration(seconds: 3),
  ),
);
```

If the design calls for a custom toast in an `Overlay`, the same rules hold:

- **The 300ms cap holds here too.** A toast is uninvited, so if anything it should be quicker and quieter than motion the user asked for.
- **It exits the way it entered.** Entering from the bottom and leaving to the side reads as two unrelated elements.
- **Exit ~20% faster than entry.** Pass a shorter `reverseDuration` to the controller; with an implicit animation, pick the duration from the target state: `duration: visible ? 260ms : 210ms`.
- **Safe area, always.** Position from `MediaQuery.paddingOf(context).bottom + 16`, not `16`, or it sits under the home indicator.

---

## Firing something once at a threshold

When a crossing point matters — a detent, a snap, a pull-to-refresh arming — don't fire on every frame. Keep the last state and act only on the change:

```dart
bool _armed = false;

void _onPull() {
  final armed = _pull.value > refreshThreshold; // _pull: the AnimationController driving the pull
  if (armed != _armed) {
    _armed = armed;
    HapticFeedback.lightImpact();
  }
}

// initState: _pull.addListener(_onPull);   dispose: _pull.removeListener(_onPull);
```

The comparison runs every frame; the haptic fires twice per pull. That's the pattern for every "do something when the animation reaches X". If the crossing also changes what's drawn, flip a `ValueNotifier<bool>` here and let a `ValueListenableBuilder` rebuild just that widget.
