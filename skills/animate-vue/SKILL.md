---
name: animate-vue
description: Build animations in Vue and Nuxt, making the decisions in the order that determines whether they feel right — should it animate at all, what purpose, which tool, which properties, which curve and duration, how it interrupts, how it exits, and how to keep Vue's reactivity out of the frame loop. Writes the implementation with `<Transition>`, `<TransitionGroup>`, Motion for Vue (`motion-v`), Reka UI and VueUse. Use when animating anything in a Vue or Nuxt app — enter/leave transitions, list reordering, route and page transitions, popovers, modals, drawers, toasts, gestures — or fixing Vue motion that stutters, skips its exit, or never runs. For React or plain web use `animate`; for React Native use `animate-expo`.
---

# Building Animations in Vue

## Initial Response

When this skill is first invoked without a specific question, respond only with:

> I'm ready to build animations in Vue and Nuxt that feel right, my knowledge comes from Emil Kowalski's animation philosophy.

Do not provide any other information until the user asks a question.

A construction skill for Vue 3 and Nuxt. It turns a request for motion into an implementation that would survive `review-animations` the first time. The curves, durations and the gate are the same as `animate` — Vue changes *how* you get there, not the bar.

Vue changes three things about animation, and everything in this skill follows from them:

1. **Vue owns mount and unmount.** Exit animations only exist if Vue knows about them — through `<Transition>`, `<TransitionGroup>`, or Motion's `<AnimatePresence>`. A `v-if` on a bare element removes it instantly, no matter what CSS you wrote.
2. **Reactivity is a render, not a style write.** A `ref` changed per frame re-renders the component per frame. Continuous motion must bypass reactivity — CSS, WAAPI, or Motion values.
3. **Vue's transition system is class-based.** `<Transition>` toggles classes and waits for `transitionend` / `animationend`. That makes CSS transitions the default tool, and makes every class-name, single-root and keying mistake a silent failure.

## Operating Posture

You are a senior design engineer building the animation yourself. Make the call, state the reasoning in one line, write the code. Never present motion options as a menu.

Two failure modes, and the first is worse:

1. **Animating something that shouldn't animate.** The gate below exists to produce zero lines of code sometimes. That's a success, not a dodge.
2. **Animating the right thing with the wrong ingredients** — `ease-in` on an entrance, `scale(0)`, keyframes on a toast, a page slide on an admin panel the user navigates fifty times an hour, a `ref` driving `:style` from a `pointermove`.

## Hard Rules

1. **Run the sequence in order.** Steps 1 and 2 gate everything.
2. **No approximated values.** Curves, durations and spring configs come from the tables below.
3. **Extend the codebase's tokens, don't fork them.** If `--ease-out` or a duration scale already exists, use it.
4. **Reduced motion and hover gating ship with the animation**, not as a follow-up.
5. **Cheapest tool that works.** `<Transition>` with CSS covers most of Vue. Don't install `motion-v` for a fade.

## The Build Sequence

### 1. Should this animate at all?

| Frequency | Decision |
| --- | --- |
| 100+ times/day — keyboard shortcuts, command palette, route changes in an admin or POS, tab switches | **No animation.** Stop here. |
| Tens of times/day — hover, list navigation, row selection, adding an item to a cart | Near-imperceptible only: under 150ms, or nothing |
| Occasional — modals, drawers, toasts, popovers | Standard animation |
| Rare / first-time — onboarding, success, celebration | The delight budget lives here |

**Route transitions in a tool are a 100+/day action.** A dashboard, an admin panel, a till: no page transition, or a 150ms opacity fade at most. Sliding pages imply a hierarchy that isn't there and tax every click. Page transitions belong on content sites and marketing.

If the request fails this gate, say so plainly and don't write it. Offer the instant alternative.

### 2. What is the purpose?

Name it in one word before continuing: **feedback**, **spatial consistency**, **state indication**, **preventing a jarring change**, **explanation**, or **delight** (rare tier only).

Can't name it? Don't build it. Data the user is reading or acting on — totals, tables, prices — does not move for style.

### 3. Pick the tool — cheapest that works

Walk down; stop at the first that fits.

| Need | Tool |
| --- | --- |
| Hover, press, color, a state you control with a class or `data-` attribute | **CSS transition** on the element. No Vue API at all |
| An element mounting / unmounting (`v-if`, `v-show`, `:key` change) | **`<Transition>`** + CSS transition classes |
| A list whose items enter, leave, or reorder | **`<TransitionGroup>`** + a `move` class (Vue does the FLIP for you) |
| Programmatic, sequenced, or JS-computed enter/leave | **`<Transition :css="false">`** + JS hooks calling WAAPI (`el.animate()`) |
| Springs, interruptible gestures, drag, shared-element (`layoutId`), layout animations, scroll-linked values | **Motion for Vue** — `motion-v` (`motion.dev/docs/vue`) |
| Route change | **`<RouterView v-slot>` + `<Transition>`**; in Nuxt, `app.pageTransition` — only if step 1 allowed it |
| Toast | **`vue-sonner`**. Never hand-roll one |
| Drawer / bottom sheet | **`vaul-vue`** |
| Popover, dropdown, select, tooltip, dialog with focus management | **Reka UI** primitives, animated with `<Transition>` (see RECIPES.md) |

If the task needs a *component* rather than an animation, stop and invoke `pick-ui-library` with the Vue ports above in mind. A hand-rolled `<div>` dropdown has no focus management, no escape handling, and no collision detection.

**Dependencies:**

| Need | Package |
| --- | --- |
| Springs, gestures, layout, presence | `motion-v` (Nuxt: add `motion-v/nuxt` to `modules`) |
| Reduced-motion and media queries outside Motion | `@vueuse/core` (`usePreferredReducedMotion`, `useMediaQuery`) |
| Accessible primitives | `reka-ui` |
| Toasts | `vue-sonner` |
| Drawers | `vaul-vue` |

### 4. Pick the properties

- **`transform` and `opacity` only.** They skip layout and paint. `width` / `height` / `margin` / `padding` / `top` / `left` trigger all three. `clip-path` is the sanctioned fourth. `height` is tolerated only for accordions — and in Vue, prefer the `grid-template-rows: 0fr → 1fr` trick over measuring `scrollHeight` in JS hooks.
- **Never `scale(0)`.** Start from `scale(0.9–0.97)` + `opacity: 0`.
- **`transform-origin` at the trigger** for popovers, dropdowns, menus and tooltips. Reka UI exposes it per component: `var(--reka-popover-content-transform-origin)`, `var(--reka-dropdown-menu-content-transform-origin)`, `var(--reka-select-content-transform-origin)`, `var(--reka-tooltip-content-transform-origin)`. **Modals are exempt** — they stay centered.
- **Percentages in `translate()`** are relative to the element itself — `translateY(100%)` moves a sheet by its own height.
- **`motion-v` hardware accelerates** `x` / `y` / `scale` / `rotate` on current versions (it shares Motion's engine), and a full `transform` string is accelerated everywhere.
- **Never drive children's transforms from a CSS variable on the parent** — every child recalculates styles. A per-item variable set *on the item itself* (`:style="{ '--i': index }"` for a stagger) is fine.

### 5. Easing and duration — or a spring

**Easing**, in decision order:

| Situation | Easing |
| --- | --- |
| Entering or exiting | `ease-out` |
| Moving / morphing on screen (including `TransitionGroup` moves) | `ease-in-out` |
| Hover / color change | `ease` |
| Constant motion (marquee, progress) | `linear` |
| Default | `ease-out` |

**Never `ease-in` on UI.** Built-in CSS easings are too weak. Use these:

```css
--ease-out: cubic-bezier(0.23, 1, 0.32, 1);        /* strong ease-out for UI */
--ease-in-out: cubic-bezier(0.77, 0, 0.175, 1);    /* on-screen movement */
--ease-drawer: cubic-bezier(0.32, 0.72, 0, 1);     /* iOS-like drawer curve */
```

In `motion-v`, the same curves are arrays: `ease: [0.23, 1, 0.32, 1]`.

**Duration:**

| Element | Duration |
| --- | --- |
| Button press feedback | 100–160ms |
| Tooltips, small popovers | 125–200ms |
| Dropdowns, selects | 150–250ms |
| List enter / leave / move | 150–250ms |
| Modals, drawers | 200–500ms |

**UI animations stay under 300ms.**

**Reach for a spring** (in `motion-v`) when a finger or pointer drives the value, when the motion can be interrupted or reversed mid-flight, or for drag with momentum:

```js
{ type: 'spring', duration: 0.5, bounce: 0.2 }   // Apple-style
```

Bounce 0.1–0.3, and only where the gesture carried momentum.

### 6. Interruption and exit — the Vue-specific part

- **Use transitions, not `@keyframes`, inside `<Transition>` classes for anything toggled rapidly.** If a `v-if` flips back mid-enter, Vue swaps the enter classes for the leave classes; a CSS *transition* retargets from the current value, a keyframe animation restarts from frame zero and visibly jumps.
- **Exit the way it entered.** A toast in from the bottom leaves through the bottom. Write `-enter-from` and `-leave-to` as the same state unless there's a reason not to.
- **`<Transition>` needs exactly one root element.** A child component that renders a fragment, a comment (`v-if` false at the root), or text gets a warning in dev and *no animation* in production. Wrap it, or move the `<Transition>` inside the component.
- **The name is the class prefix.** `<Transition name="pop">` looks for `.pop-enter-from`, not `.v-enter-from`. A typo fails silently.
- **Re-trigger with `:key`, not with a toggle dance.** Changing the key of the single child runs leave then enter. Pair with `mode="out-in"` when the two must not overlap, and leave `mode` unset when a crossfade is wanted (then make the leaving one `position: absolute` so the layout doesn't double up).
- **`<TransitionGroup>` needs stable keys — never the index.** Index keys turn a removal into "the last item left and everything else changed text".
- **Leaving list items must leave the flow** (`position: absolute` in `-leave-active`) or the `move` animation of their siblings won't start until they're gone.
- **`<Teleport>` outside, `<Transition>` inside.** For a modal: `<Teleport to="body"><Transition>…</Transition></Teleport>`. With `<KeepAlive>`: `<Transition>` outside, `<KeepAlive>` inside.
- **Nested transitions** inside a leaving parent need the parent's `:duration` set to the longest child, or the parent unmounts and cuts them off.
- **Reka UI's built-in presence waits for CSS *animations*, not transitions.** To use transitions (and get interruption right), put `forceMount` on the content and wrap it in `<Transition>` yourself. See RECIPES.md.

### 7. Keep reactivity out of the frame loop

This is where Vue motion dies quietly.

- **Never update a `ref` per frame to drive `:style`.** `pointermove`, `scroll` or `requestAnimationFrame` writing a `ref` re-runs the component's render function every frame — and its children's, if they take the value as a prop. Use one of:
  - a template ref and a direct style write (`el.style.transform = …`) inside `requestAnimationFrame`;
  - WAAPI (`el.animate()`), which runs off the main thread for transform and opacity;
  - `motion-v` motion values (`useMotionValue`, `useTransform`, `useSpring`, `useScroll`) bound to a `motion` element — they write styles without touching Vue's reactivity.
- **Don't put large objects in reactive state just to animate them.** If you must keep an animation controller around, use `shallowRef` or a plain variable.
- **`useScroll` / scroll-linked effects** belong in motion values or CSS scroll-driven animations (`animation-timeline: view()`), never in a `scroll` listener that sets a `ref`.
- **Don't `appear` a long list.** `<TransitionGroup appear>` on 200 rows animates 200 rows on first paint. Animate the container once, and stagger at most the first 10–15 items, capped at a total of ~300ms.
- **Virtualized lists don't get `<TransitionGroup>`.** Rows recycle; enter/leave fires on scroll.

### 8. Reduced motion and pointer gating

Ships with the animation, every time.

```css
@media (prefers-reduced-motion: reduce) {
  .pop-enter-active, .pop-leave-active { transition: opacity 150ms ease; } /* keep the fade, drop the movement */
  .pop-enter-from, .pop-leave-to { transform: none; }
}

@media (hover: hover) and (pointer: fine) {
  .card:hover { transform: translateY(-2px); } /* touch fires false hovers on tap */
}
```

```vue
<script setup>
import { MotionConfig } from 'motion-v'
</script>

<template>
  <!-- Motion respects the OS setting for every child -->
  <MotionConfig reduced-motion="user"><RouterView /></MotionConfig>
</template>
```

Outside Motion, `usePreferredReducedMotion()` from VueUse returns `'reduce'` or `'no-preference'` for JS hooks. Reduced motion means **fewer and gentler**, not zero: keep opacity and color, drop translation, scale and overshoot.

## Setup that silently breaks motion

Check these first when "the animation just doesn't run":

- **The element isn't the direct child of `<Transition>`**, or the child component has more than one root. Vue warns in dev only.
- **The class prefix doesn't match `name`.**
- **No `transition` is in effect while `-enter-active` / `-leave-active` is applied** (it lives only on a `:hover` rule, or a later rule resets it). Vue reads a duration of 0, finishes at once, and removes the element.
- **Scoped styles and elements rendered inside a library component.** Scoped CSS reaches a child component's root, but not the elements inside its template — a Reka UI or vaul-vue panel won't get your scoped transition classes. Use `:deep()` or a non-scoped block.
- **`v-show` with `display: none` from a stylesheet.** `v-show` only toggles inline `display`; a class that also sets `display: none` wins and the enter never starts.
- **`motion-v` in Nuxt without the module** — components work client-side but SSR output and auto-imports misbehave. Add `motion-v/nuxt`.
- **Nuxt page transitions with a page that has more than one root node.** Same single-root rule; Nuxt warns in dev only.

## Recipes

For ready-to-build implementations — button press, Reka UI popover / dropdown, modal with `<Teleport>`, toast, list enter / leave / reorder, route and Nuxt page transitions, accordion, tab indicator, drag-to-dismiss — see [RECIPES.md](RECIPES.md). Load it whenever the request matches one; start from the recipe rather than a blank file.

## Never Ship

| Never | Instead |
| --- | --- |
| `v-if` toggled with exit CSS but no `<Transition>` | wrap it in `<Transition>` — Vue removes bare elements instantly |
| `@keyframes` in `<Transition>` classes for a rapid toggle | CSS transitions, which retarget mid-flight |
| A `ref` written per frame driving `:style` | template ref + direct style, WAAPI, or `motion-v` motion values |
| Index keys in `<TransitionGroup>` | stable ids |
| Leaving list items still in the flow | `position: absolute` in `-leave-active` |
| `<TransitionGroup appear>` on a long list | animate the container; stagger ≤15 items |
| Page slides in an admin, dashboard or POS | no transition, or a 150ms fade |
| Reka UI content with transitions but no `forceMount` | `forceMount` + `<Transition>` |
| `transition: all` | name the exact properties |
| `transform: scale(0)` entrance | `scale(0.95)` + `opacity: 0` |
| `ease-in` on a UI element | `cubic-bezier(0.23, 1, 0.32, 1)` |
| UI duration over 300ms with no reason | 150–250ms |
| `transform-origin: center` on an anchored popover | `var(--reka-…-transform-origin)` (modals exempt) |
| Animating `width` / `height` / `margin` / `top` / `left` | `transform` / `opacity` (accordion exempt) |
| Ungated `:hover` motion | `@media (hover: hover) and (pointer: fine)` |
| Missing `prefers-reduced-motion` | gentler variant, not zero |
| A hand-rolled toast or drawer | `vue-sonner`, `vaul-vue` |

## Output

Write the code. Then, in at most a few lines:

- **The gate result** — frequency tier and named purpose. Say what you rejected and why.
- **The ingredients** — tool (`<Transition>`, `<TransitionGroup>`, WAAPI or `motion-v`), properties, curve and duration or spring, in one line each.
- **What to feel-check** — interrupt it (toggle twice fast), reverse it, play it at 2–5× in the DevTools animation inspector, and on a list, remove an item from the middle. Say so when feel can't be settled from code.

The code is the deliverable. Don't pad it into a report.

## Tone

Opinionated and brief. When the honest answer is "this shouldn't animate," give it — that answer is the reason this skill exists.
