# Design Document: Cat Clicker App

## Overview

The Cat Clicker App is a single-page web application built with plain HTML5, Tailwind CSS (via CDN), and Vanilla JavaScript. No build tools, bundlers, or frameworks are used — the entire app ships as a single `index.html` file.

The core experience is simple: a looping cat video fills the page, the user clicks it, animated emoji-style reactions (hearts, stars, paw prints) burst out at the click position, and a counter tracks total clicks. The design prioritises immediate interactivity, smooth animations, and a fully responsive layout from 320 px to 1920 px.

### Key Design Decisions

- **Single-file delivery** — all HTML, CSS (Tailwind utilities + a small `<style>` block for keyframe animations), and JS live in `index.html`. This keeps deployment trivial and removes any dependency on a server-side language or build pipeline.
- **CSS keyframe animations over JS animation loops** — reaction particles are driven by CSS `@keyframes` so the browser's compositor thread handles rendering, keeping the main thread free for click events.
- **DOM-based particle system** — each reaction is a short-lived `<span>` injected into the Effect Layer and removed via an `animationend` listener, avoiding manual timers and memory leaks.
- **Tailwind CDN** — loaded from the official CDN so no local installation is needed. Custom keyframes that Tailwind cannot express are written in a `<style>` block.

---

## Architecture

The app is a single HTML document with three logical layers stacked via CSS positioning:

```
┌─────────────────────────────────────┐
│           Page Shell                │  ← <body> with Tailwind flex layout
│  ┌───────────────────────────────┐  │
│  │       Video Container         │  │  ← relative-positioned wrapper
│  │  ┌─────────────────────────┐  │  │
│  │  │     <video> element     │  │  │  ← Layer 0: Cat_Video
│  │  └─────────────────────────┘  │  │
│  │  ┌─────────────────────────┐  │  │
│  │  │     Effect Layer        │  │  │  ← Layer 1: absolute overlay, pointer-events:none
│  │  │   (reaction particles)  │  │  │     except when receiving forwarded coords
│  │  └─────────────────────────┘  │  │
│  │  ┌─────────────────────────┐  │  │
│  │  │     Click Zone          │  │  │  ← Layer 2: absolute overlay, captures clicks
│  │  └─────────────────────────┘  │  │
│  └───────────────────────────────┘  │
│  ┌───────────────────────────────┐  │
│  │      Reaction Counter         │  │  ← fixed/sticky HUD element
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
```

### Data Flow

```
User Click on Click_Zone
        │
        ▼
  clickHandler(event)
        │
        ├─► incrementCounter()   → updates counter state → renders counter DOM
        │
        └─► spawnReaction(x, y)  → picks random type
                                  → creates <span> particle
                                  → appends to Effect_Layer
                                  → CSS animation runs
                                  → animationend → removes <span>
```

---

## Components and Interfaces

### 1. Video Container

An HTML `<div>` that acts as the positioning context for all overlaid layers.

```html
<div id="video-container" class="relative w-full max-w-4xl mx-auto rounded-2xl overflow-hidden shadow-2xl">
  <!-- video, effect layer, click zone -->
</div>
```

- `relative` positioning so child `absolute` layers stack correctly.
- `max-w-4xl` caps width on large screens; `w-full` fills smaller viewports.
- `overflow-hidden` clips reaction particles that drift outside the container bounds.

### 2. Cat Video (`<video>`)

```html
<video id="cat-video"
       class="w-full h-auto block"
       autoplay loop muted playsinline
       poster="placeholder.jpg">
  <source src="cat.mp4" type="video/mp4">
  <p id="video-fallback">Sorry, your browser can't play this video.</p>
</video>
```

- `autoplay muted` is required by browsers to allow autoplay without user gesture.
- `loop` satisfies Requirement 1.2.
- `playsinline` prevents iOS from going full-screen.
- `poster` attribute provides the loading placeholder (Requirement 1.4).
- The inner `<p>` fallback satisfies Requirement 1.5.

### 3. Effect Layer

```html
<div id="effect-layer"
     class="absolute inset-0 pointer-events-none overflow-hidden"
     aria-hidden="true">
</div>
```

- `pointer-events-none` ensures clicks pass through to the Click Zone.
- Reaction `<span>` elements are dynamically appended here by JS.

### 4. Click Zone

```html
<div id="click-zone"
     class="absolute inset-0 cursor-pointer"
     role="button"
     aria-label="Click to trigger a reaction">
</div>
```

- `inset-0` makes it cover the full video area (Requirement 2.1).
- Sits above the Effect Layer in z-order so it always receives clicks.
- `cursor-pointer` provides visual affordance.

### 5. Reaction Counter

```html
<div id="counter-container" class="...">
  <span id="counter-label">Reactions</span>
  <span id="counter-value">0</span>
</div>
```

- Positioned as a fixed HUD element so it remains visible during scrolling on small screens.
- Updated by JS whenever the counter state changes.

### 6. JavaScript Module (inline `<script>`)

All logic lives in a single `<script>` block at the bottom of `<body>`. The public interface is:

| Function | Signature | Description |
|---|---|---|
| `incrementCounter` | `() → void` | Increments internal count, updates DOM |
| `spawnReaction` | `(x: number, y: number) → void` | Creates and animates a reaction particle |
| `pickReactionType` | `() → ReactionType` | Randomly selects one of the 3 reaction types |
| `createParticle` | `(type: ReactionType, x: number, y: number) → HTMLElement` | Builds the particle `<span>` |
| `clickHandler` | `(event: MouseEvent) → void` | Wired to Click Zone's `click` event |

---

## Data Models

### ReactionType

An enumeration of the three supported reaction types:

```javascript
const REACTION_TYPES = ['heart', 'star', 'paw'];
// ReactionType = 'heart' | 'star' | 'paw'
```

Each type maps to an emoji character and a named CSS animation:

| Type | Emoji | CSS Animation |
|---|---|---|
| `heart` | ❤️ | `floatUp` |
| `star` | ⭐ | `burstOut` |
| `paw` | 🐾 | `floatUp` |

### AppState

A plain JavaScript object holding mutable runtime state:

```javascript
const state = {
  reactionCount: 0,   // number — total clicks since page load
};
```

No persistence is required; state resets on page reload.

### Particle Element Schema

Each reaction particle is a `<span>` with the following attributes set by `createParticle`:

```
{
  textContent: <emoji string>,
  className:   "reaction-particle reaction-<type>",
  style: {
    left:   "<x>px",   // click x relative to Effect Layer
    top:    "<y>px",   // click y relative to Effect Layer
    fontSize: "<random 1.5–3rem>",
    animationDuration: "<random 0.6–1.2s>",
  }
}
```

The particle is appended to `#effect-layer` and removed when its `animationend` event fires.

### CSS Animation Definitions

Two keyframe animations cover all three reaction types:

```css
@keyframes floatUp {
  0%   { transform: translate(-50%, -50%) scale(0.5); opacity: 1; }
  100% { transform: translate(-50%, -50%) translateY(-80px) scale(1.2); opacity: 0; }
}

@keyframes burstOut {
  0%   { transform: translate(-50%, -50%) scale(0); opacity: 1; }
  60%  { transform: translate(-50%, -50%) scale(1.4); opacity: 1; }
  100% { transform: translate(-50%, -50%) scale(1.8); opacity: 0; }
}
```


---

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system — essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: Reaction spawned at click coordinates

*For any* (x, y) click coordinates within the Click Zone, calling `spawnReaction(x, y)` shall append exactly one particle element to the Effect Layer whose `left` and `top` style values correspond to those coordinates.

**Validates: Requirements 2.2, 3.1**

---

### Property 2: Counter value equals total click count

*For any* sequence of N clicks on the Click Zone (N ≥ 0), the value displayed by the Reaction Counter shall equal N after all clicks have been processed.

**Validates: Requirements 2.3, 4.3**

---

### Property 3: All reaction types are reachable

*For any* sufficiently large number of calls to `pickReactionType()` (e.g., 200 iterations), every type defined in `REACTION_TYPES` shall appear at least once in the results.

**Validates: Requirements 3.2, 3.3**

---

### Property 4: Completed reaction particles are removed from the DOM

*For any* reaction type, after the `animationend` event fires on a spawned particle element, that element shall no longer be present in the Effect Layer's child list.

**Validates: Requirements 3.4**

---

### Property 5: Multiple simultaneous reactions are supported

*For any* N rapid successive clicks (N ≥ 2) before any animation completes, the Effect Layer shall contain N particle elements simultaneously, and the Reaction Counter shall equal N.

**Validates: Requirements 2.4, 3.5**

---

## Error Handling

### Video Load Failure

- The `<video>` element's inner `<p>` fallback is displayed natively by the browser when the video source cannot be loaded (Requirement 1.5).
- No JavaScript error handling is needed for this case — it is handled declaratively by HTML.

### Missing Video Source

- If `cat.mp4` is absent, the `poster` image remains visible as a static placeholder (Requirement 1.4).
- A `video.addEventListener('error', ...)` handler logs a console warning in development but does not break the click interaction — the Click Zone and counter remain fully functional even without a video.

### Animation Cleanup

- Each particle's `animationend` listener calls `particle.remove()`. If `animationend` never fires (e.g., `animation-duration: 0` in a test environment), a fallback `setTimeout` of 2000 ms removes the particle to prevent DOM leaks.

### Rapid Clicking / Event Flooding

- No debouncing or throttling is applied — every click event is processed independently. The DOM particle system is lightweight enough to handle hundreds of simultaneous particles without performance issues on modern devices.

### Counter Overflow

- The counter is a plain JavaScript `number`. At realistic click rates it will never approach `Number.MAX_SAFE_INTEGER`. No overflow guard is needed.

---

## Testing Strategy

### Approach

Because this feature involves DOM manipulation, CSS animations, and event handling — rather than pure data transformation — the testing strategy combines:

- **Unit / example-based tests** for specific behaviors (counter initialisation, DOM structure)
- **Property-based tests** for universal behaviors that hold across all inputs (click coordinates, counter accumulation, reaction type distribution, particle cleanup)
- **Smoke / manual tests** for visual and layout requirements that require a browser rendering engine

### Property-Based Testing Library

**[fast-check](https://github.com/dubzzz/fast-check)** (JavaScript) is used for property-based tests. It runs in Node.js with jsdom (via Jest or Vitest) to simulate the DOM.

Each property test runs a minimum of **100 iterations**.

Tag format: `// Feature: cat-clicker-app, Property <N>: <property_text>`

### Property Tests

| Property | Test Description | Arbitraries |
|---|---|---|
| P1: Reaction at coordinates | Simulate click at random (x, y), verify particle `left`/`top` | `fc.integer({min:0, max:800})` × 2 |
| P2: Counter equals click count | Simulate N random clicks, verify counter DOM value equals N | `fc.integer({min:0, max:500})` |
| P3: All types reachable | Call `pickReactionType()` 200 times, verify all 3 types appear | (no arbitrary needed — fixed 200 iterations) |
| P4: Particle removed after animationend | Spawn particle of random type, dispatch `animationend`, verify removal | `fc.constantFrom(...REACTION_TYPES)` |
| P5: Simultaneous reactions | Simulate N clicks before any animation ends, verify N particles in DOM | `fc.integer({min:2, max:20})` |

### Unit / Example Tests

- Counter initialises to 0 on page load (Requirement 4.2)
- Video element has `autoplay`, `loop`, `muted`, `playsinline` attributes (Requirements 1.2, 1.3)
- Video element has `poster` attribute set (Requirement 1.4)
- Fallback element exists inside `<video>` (Requirement 1.5)
- `REACTION_TYPES` array has length ≥ 3 (Requirement 3.2)
- Click Zone has `inset-0` and `absolute` positioning (Requirement 2.1)

### Smoke / Manual Tests

- Visual layout at 320 px, 768 px, 1440 px, 1920 px viewports (Requirement 5.1)
- Video scales without overflow on resize (Requirement 5.2)
- Counter is visually prominent and does not obscure the video (Requirement 6.3)
- Consistent colour scheme and typography (Requirement 6.2)
