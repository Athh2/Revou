# Implementation Plan: Cat Clicker App

## Overview

Build the entire app as a single `index.html` file using HTML5, Tailwind CSS (CDN), and Vanilla JavaScript. Tasks progress from static structure → interactivity → animations → testing, with each step wired into the previous one.

## Tasks

- [x] 1. Create the HTML skeleton and page shell
  - Create `index.html` with `<!DOCTYPE html>`, `<head>` (charset, viewport, title), and Tailwind CDN `<script>` tag
  - Add a `<style>` block placeholder for custom keyframe animations
  - Add a `<body>` with a full-height flex layout centred vertically and horizontally using Tailwind utilities
  - _Requirements: 1.1, 6.1, 6.2_

- [~] 2. Build the video container and layered structure
  - [x] 2.1 Add the video container `<div id="video-container">` with `relative`, `max-w-4xl`, `w-full`, `mx-auto`, `rounded-2xl`, `overflow-hidden`, `shadow-2xl` classes
  - Add the `<video id="cat-video">` element with `autoplay`, `loop`, `muted`, `playsinline`, `poster`, `w-full h-auto block` classes, a `<source src="cat.mp4">` child, and an inner `<p id="video-fallback">` fallback paragraph
  - Add `<div id="effect-layer">` with `absolute inset-0 pointer-events-none overflow-hidden` and `aria-hidden="true"`
  - Add `<div id="click-zone">` with `absolute inset-0 cursor-pointer`, `role="button"`, and `aria-label`
  - _Requirements: 1.1, 1.2, 2.1, 5.2, 5.3_

  - [-] 2.2 Write unit tests for static HTML structure
    - Verify `<video>` has `autoplay`, `loop`, `muted`, `playsinline`, `poster` attributes
    - Verify fallback `<p>` exists inside `<video>`
    - Verify `#click-zone` has `absolute` and `inset-0` classes
    - _Requirements: 1.2, 2.1_

- [~] 3. Add the Reaction Counter HUD
  - Add `<div id="counter-container">` as a fixed/sticky element outside the video container, styled with Tailwind to be visually prominent (e.g., `fixed top-4 right-4`, contrasting background, rounded pill)
  - Add `<span id="counter-label">Reactions</span>` and `<span id="counter-value">0</span>` inside the container
  - _Requirements: 4.1, 4.2, 6.3_

  - [~] 3.1 Write unit test for counter initialisation
    - Verify `#counter-value` text content is `"0"` on page load
    - _Requirements: 4.2_

- [~] 4. Implement the JavaScript state and counter logic
  - Add an inline `<script>` block at the bottom of `<body>`
  - Define `const state = { reactionCount: 0 }`
  - Implement `incrementCounter()`: increments `state.reactionCount` and sets `#counter-value` `textContent` to the new value
  - _Requirements: 2.3, 4.3_

  - [~] 4.1 Write property test for counter accumulation (Property 2)
    - **Property 2: Counter value equals total click count**
    - Simulate N random calls to `incrementCounter()`, assert `#counter-value` equals N
    - Use `fc.integer({min: 0, max: 500})`
    - **Validates: Requirements 2.3, 4.3**

- [~] 5. Implement reaction type selection and particle creation
  - Define `const REACTION_TYPES = ['heart', 'star', 'paw']` and the emoji/animation mapping object
  - Implement `pickReactionType()`: returns a random element from `REACTION_TYPES`
  - Implement `createParticle(type, x, y)`: creates a `<span>` with `textContent` set to the emoji, class `reaction-particle reaction-<type>`, and inline styles for `left`, `top`, random `fontSize` (1.5–3 rem), and random `animationDuration` (0.6–1.2 s)
  - _Requirements: 3.1, 3.2, 3.3_

  - [~] 5.1 Write property test for reaction type reachability (Property 3)
    - **Property 3: All reaction types are reachable**
    - Call `pickReactionType()` 200 times, assert every type in `REACTION_TYPES` appears at least once
    - **Validates: Requirements 3.2, 3.3**

- [~] 6. Implement the CSS keyframe animations
  - Add `@keyframes floatUp` and `@keyframes burstOut` to the `<style>` block as defined in the design
  - Add `.reaction-particle` base styles: `position: absolute`, `pointer-events: none`, `user-select: none`, `animation-fill-mode: forwards`
  - Map `reaction-heart` and `reaction-paw` to `floatUp`; map `reaction-star` to `burstOut`
  - _Requirements: 3.1, 3.2_

- [~] 7. Implement `spawnReaction` and wire the click handler
  - Implement `spawnReaction(x, y)`: calls `pickReactionType()`, calls `createParticle(type, x, y)`, appends the particle to `#effect-layer`, adds an `animationend` listener that calls `particle.remove()`, and adds a 2000 ms `setTimeout` fallback that also calls `particle.remove()` if the element is still in the DOM
  - Implement `clickHandler(event)`: computes click coordinates relative to `#effect-layer` using `getBoundingClientRect()`, calls `incrementCounter()`, calls `spawnReaction(x, y)`
  - Wire `clickHandler` to `#click-zone`'s `click` event via `addEventListener`
  - _Requirements: 2.2, 2.3, 2.4, 3.1, 3.4, 3.5_

  - [~] 7.1 Write property test for reaction spawned at click coordinates (Property 1)
    - **Property 1: Reaction spawned at click coordinates**
    - For random (x, y), call `spawnReaction(x, y)`, assert one particle is appended to `#effect-layer` with matching `left` and `top` style values
    - Use `fc.integer({min: 0, max: 800})` for both axes
    - **Validates: Requirements 2.2, 3.1**

  - [~] 7.2 Write property test for particle removal after animationend (Property 4)
    - **Property 4: Completed reaction particles are removed from the DOM**
    - For a random reaction type, spawn a particle, dispatch a synthetic `animationend` event, assert the particle is no longer in `#effect-layer`
    - Use `fc.constantFrom(...REACTION_TYPES)`
    - **Validates: Requirements 3.4**

  - [~] 7.3 Write property test for simultaneous reactions (Property 5)
    - **Property 5: Multiple simultaneous reactions are supported**
    - Simulate N clicks before any animation ends, assert `#effect-layer` contains N particles and `#counter-value` equals N
    - Use `fc.integer({min: 2, max: 20})`
    - **Validates: Requirements 2.4, 3.5**

- [~] 8. Checkpoint — verify core interactivity
  - Ensure all tests pass, ask the user if questions arise.

- [~] 9. Apply responsive layout and visual polish
  - Ensure `<body>` uses Tailwind classes for a centred, full-viewport flex layout with a cohesive background colour
  - Verify `#video-container` uses `w-full max-w-4xl` so the video scales proportionally on all viewports (320 px – 1920 px)
  - Style `#counter-container` with a contrasting pill badge that does not overlap the video on small screens
  - Add a page title or heading with consistent typography using Tailwind text utilities
  - _Requirements: 5.1, 5.2, 5.3, 6.1, 6.2, 6.3_

- [~] 10. Add a placeholder cat asset and verify fallback behaviour
  - Add a `placeholder.jpg` reference in the `poster` attribute (or use a public placeholder URL) so the video container is not blank before the video loads
  - Confirm the `<p id="video-fallback">` text is visible when the `<video>` source is intentionally broken (manual smoke test)
  - _Requirements: 1.2_

- [~] 11. Final checkpoint — full test suite and smoke tests
  - Ensure all automated tests pass, ask the user if questions arise.
  - Manually verify layout at 320 px, 768 px, 1440 px, and 1920 px viewports
  - Manually verify counter prominence does not obscure the video
  - Manually verify consistent colour scheme and typography

## Notes

- Tasks marked with `*` are optional and can be skipped for a faster MVP
- Property tests use **fast-check** with jsdom (Jest or Vitest) — no browser required
- Each task references specific requirements for traceability
- All code lives in a single `index.html`; no build step is needed
