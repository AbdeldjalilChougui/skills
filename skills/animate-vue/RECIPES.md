# Vue Animation Recipes

Ready-to-build implementations for the cases that come up most in Vue and Nuxt. Start from the recipe, then adapt.

Curves are the `--ease-out`, `--ease-in-out` and `--ease-drawer` tokens defined in SKILL.md. Each recipe ships its reduced-motion variant; don't drop it.

---

## Button press

No Vue API. A CSS transition on the element.

```css
.button {
  transition: transform 160ms var(--ease-out);
}

.button:active {
  transform: scale(0.97);
}
```

`:active` is a real press on touch, so it needs no hover gating. Gate any `:hover` styling separately.

---

## Enter / leave with `<Transition>`

The base pattern every other recipe builds on.

```vue
<template>
  <Transition name="pop">
    <div v-if="open" class="panel">…</div>
  </Transition>
</template>

<style>
.pop-enter-active,
.pop-leave-active {
  transition:
    opacity 200ms var(--ease-out),
    transform 200ms var(--ease-out);
}

.pop-enter-from,
.pop-leave-to {
  opacity: 0;
  transform: scale(0.95);
}

@media (prefers-reduced-motion: reduce) {
  .pop-enter-from,
  .pop-leave-to {
    transform: none;
  }
}
</style>
```

Transitions, not `@keyframes`: if `open` flips back mid-enter, the transition retargets from wherever it is.

---

## Popover, dropdown, select (Reka UI)

Scales out of its trigger. Reka UI's own presence only waits for CSS *animations*; to use transitions, take over with `forceMount` and a `<Transition>`.

```vue
<script setup>
import { ref } from 'vue'
import { PopoverRoot, PopoverTrigger, PopoverPortal, PopoverContent } from 'reka-ui'

const open = ref(false)
</script>

<template>
  <PopoverRoot v-model:open="open">
    <PopoverTrigger class="button">Options</PopoverTrigger>
    <PopoverPortal>
      <Transition name="popover">
        <PopoverContent v-if="open" force-mount :side-offset="6" class="popover">…</PopoverContent>
      </Transition>
    </PopoverPortal>
  </PopoverRoot>
</template>

<style>
.popover {
  transform-origin: var(--reka-popover-content-transform-origin);
}

.popover-enter-active,
.popover-leave-active {
  transition:
    opacity 200ms var(--ease-out),
    transform 200ms var(--ease-out);
}

.popover-enter-from,
.popover-leave-to {
  opacity: 0;
  transform: scale(0.95);
}
</style>
```

Same shape for `DropdownMenuContent` (`--reka-dropdown-menu-content-transform-origin`) and `SelectContent` (`--reka-select-content-transform-origin`). A tooltip is the same at 125ms — and when the user moves between tooltips in a group, the second one opens instantly: skip the transition when Reka reports `data-state="instant-open"`.

The styles are unscoped on purpose: the content element is rendered inside Reka's component, out of reach of scoped CSS.

---

## Modal

Centered, not anchored — `transform-origin` stays `center`. `<Teleport>` outside, `<Transition>` inside, overlay and panel as siblings so each gets its own curve.

```vue
<template>
  <Teleport to="body">
    <Transition name="fade">
      <div v-if="open" class="overlay" @click="open = false" />
    </Transition>
    <Transition name="modal">
      <div v-if="open" class="modal" role="dialog" aria-modal="true">…</div>
    </Transition>
  </Teleport>
</template>

<style>
.fade-enter-active,
.fade-leave-active {
  transition: opacity 200ms var(--ease-out);
}
.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}

.modal {
  position: fixed;
  inset-block-start: 50%;
  inset-inline-start: 50%;
  transform: translate(-50%, -50%);
}

.modal-enter-active,
.modal-leave-active {
  transition:
    opacity 250ms var(--ease-out),
    transform 250ms var(--ease-out);
}
.modal-enter-from,
.modal-leave-to {
  opacity: 0;
  transform: translate(-50%, -50%) scale(0.96);
}
</style>
```

Prefer Reka UI's `DialogRoot` for focus trapping and escape handling; animate its `DialogOverlay` and `DialogContent` with the same classes, `forceMount` + `<Transition>` as in the popover recipe.

---

## Toast

Don't build one. `vue-sonner` is a port of Sonner: stacking, swipe-to-dismiss, pausing on hover and the enter/exit motion are done.

```vue
<script setup>
import { Toaster, toast } from 'vue-sonner'
import 'vue-sonner/style.css'
</script>

<template>
  <Toaster position="bottom-right" />
  <button class="button" @click="toast.success('Saved')">Save</button>
</template>
```

The API matches Sonner; the `ask-sonner` skill applies.

---

## List: enter, leave, reorder

`<TransitionGroup>` does FLIP for reorders through the `move` class. Stable keys, and leaving items out of the flow.

```vue
<template>
  <TransitionGroup tag="ul" name="list" class="list">
    <li v-for="item in items" :key="item.id" class="row">{{ item.name }}</li>
  </TransitionGroup>
</template>

<style>
.list {
  position: relative;
}

.list-move,
.list-enter-active,
.list-leave-active {
  transition:
    opacity 200ms var(--ease-out),
    transform 200ms var(--ease-in-out);
}

.list-enter-from,
.list-leave-to {
  opacity: 0;
  transform: translateY(6px);
}

/* Without this, siblings wait for the leaving row before moving */
.list-leave-active {
  position: absolute;
  inline-size: 100%;
}

@media (prefers-reduced-motion: reduce) {
  .list-move {
    transition: none;
  }
  .list-enter-from,
  .list-leave-to {
    transform: none;
  }
}
</style>
```

**Adding to a list the user fills fast** — lines on a till ticket, items in a cart — is a tens-of-times-a-day action. Drop the enter transform and keep a 120ms opacity at most, or nothing.

**Stagger** a first appearance with a variable set on each item, capped so the whole list lands within ~300ms:

```vue
<li v-for="(item, i) in items" :key="item.id" :style="{ '--i': Math.min(i, 8) }">
```

```css
.list-enter-active {
  transition-delay: calc(var(--i) * 35ms);
}
```

---

## Route and page transitions

Run the gate first. An admin panel, a dashboard or a POS changes routes constantly: no transition, or a 150ms fade. Content sites can afford more.

**Vue Router:**

```vue
<template>
  <RouterView v-slot="{ Component, route }">
    <Transition name="page" mode="out-in">
      <component :is="Component" :key="route.path" />
    </Transition>
  </RouterView>
</template>

<style>
.page-enter-active,
.page-leave-active {
  transition: opacity 150ms var(--ease-out);
}
.page-enter-from,
.page-leave-to {
  opacity: 0;
}
</style>
```

**Nuxt** — the same classes, declared once:

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  app: {
    pageTransition: { name: 'page', mode: 'out-in' },
  },
})
```

Every page needs a single root element, or the transition is skipped with a dev-only warning. A `mode="out-in"` route transition adds its leave time to every navigation — another reason to keep it at 150ms.

---

## Accordion

The one place a size animates. Animate `grid-template-rows` so no JS measures `scrollHeight`.

```vue
<template>
  <div class="accordion" :data-open="open">
    <button class="trigger" :aria-expanded="open" @click="open = !open">Details</button>
    <div class="body">
      <div class="inner">…</div>
    </div>
  </div>
</template>

<style>
.body {
  display: grid;
  grid-template-rows: 0fr;
  transition: grid-template-rows 250ms var(--ease-in-out);
}

.accordion[data-open='true'] .body {
  grid-template-rows: 1fr;
}

.inner {
  overflow: hidden;
}

@media (prefers-reduced-motion: reduce) {
  .body {
    transition: none;
  }
}
</style>
```

With Reka UI's `CollapsibleContent` / `AccordionContent`, use its `--reka-collapsible-content-height` / `--reka-accordion-content-height` variables instead.

---

## Tab indicator

A shared highlight that moves between tabs. `motion-v`'s `layoutId` animates it between positions with no measuring code.

```vue
<script setup>
import { motion } from 'motion-v'

defineProps({ tabs: Array })
const active = defineModel()
</script>

<template>
  <div class="tabs" role="tablist">
    <button
      v-for="tab in tabs"
      :key="tab.id"
      role="tab"
      :aria-selected="active === tab.id"
      class="tab"
      @click="active = tab.id"
    >
      <motion.span
        v-if="active === tab.id"
        layout-id="tab-indicator"
        class="indicator"
        :transition="{ type: 'spring', duration: 0.4, bounce: 0 }"
      />
      <span class="label">{{ tab.label }}</span>
    </button>
  </div>
</template>
```

The indicator moves; the content under the tabs does not slide — tabs are peers. Keyboard arrow navigation between tabs is a high-frequency action: Motion respects `MotionConfig reduced-motion="user"`, and the spring is short enough not to lag a held arrow key.

---

## Drag to dismiss

A sheet the user flicks away. Prefer `vaul-vue` for a full drawer; for a custom card, `motion-v`:

```vue
<script setup>
import { AnimatePresence, motion } from 'motion-v'

const props = defineProps({ open: Boolean })
const emit = defineEmits(['close'])

function onDragEnd(_event, info) {
  // A flick or a long drag both dismiss
  if (info.offset.y > 120 || info.velocity.y > 500) emit('close')
}
</script>

<template>
  <AnimatePresence>
    <motion.div
      v-if="props.open"
      class="sheet"
      drag="y"
      :drag-constraints="{ top: 0, bottom: 0 }"
      :drag-elastic="{ top: 0.05, bottom: 0.6 }"
      :initial="{ y: '100%' }"
      :animate="{ y: 0 }"
      :exit="{ y: '100%' }"
      :transition="{ type: 'spring', duration: 0.35, bounce: 0.15 }"
      @drag-end="onDragEnd"
    >
      …
    </motion.div>
  </AnimatePresence>
</template>
```

It exits the way it entered, the spring carries the flick's velocity, and dragging up past the top meets resistance (`dragElastic` near 0) instead of a hard stop. Test it with a real finger on a phone.

---

## Continuous pointer-driven value

A spotlight, tilt or follow effect. Never a `ref` written on `pointermove`.

```vue
<script setup>
import { motion, useMotionValue, useMotionTemplate } from 'motion-v'

const x = useMotionValue(0)
const y = useMotionValue(0)
const background = useMotionTemplate`radial-gradient(240px at ${x}px ${y}px, rgb(255 255 255 / 0.08), transparent)`

function onMove(event) {
  const rect = event.currentTarget.getBoundingClientRect()
  x.set(event.clientX - rect.left)
  y.set(event.clientY - rect.top)
}
</script>

<template>
  <motion.div class="card" :style="{ background }" @pointermove="onMove">…</motion.div>
</template>
```

Motion values write the style directly; the component never re-renders. Decorative only — never on a surface the user reads data from — and gate it behind `(hover: hover) and (pointer: fine)`.
