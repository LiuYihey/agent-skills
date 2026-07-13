---
name: hero-motion-sync-ux
description: Align landing hero text and molecule micro-animations into one rhythm for smoother UX. Use when users mention hero animation mismatch, timing drift, overlap, white flash, blank-gap flicker, float-in speed, or visual harmony between text and 3D viewer.
disable-model-invocation: true
---

# Hero Motion Sync UX

## Goal
Make landing hero feel coherent: text and molecule switch on the same beat, use
the same float-in feel, and avoid overlap/blank-frame/white-flash artifacts.

## Apply When
- User asks to sync hero text and molecule transitions.
- User reports visual issues like overlap, flicker, white frame, or blank gap.
- User asks to tune float-in speed/easing and keep both sides consistent.

## Implementation Pattern

1. **Use one shared motion config**
   - Keep cadence and motion tokens in one file (for example `heroMotionConfig.js`).
   - Put at least: interval, float duration, easing, offset, opacity/scale start.
   - Both text and molecule components must import these shared constants.

2. **Use one shared timeline source**
   - Drive switching with one index in the page container (for example `heroCycleIndex`).
   - Pass the same index to text and molecule components.
   - Avoid independent timers per component in controlled mode.

3. **Prefer single-viewer molecule swap**
   - Keep one 3D viewer/canvas; replace model in place after next data is ready.
   - Do not use layered cross-fade canvases unless required.
   - On fetch failure, keep current molecule visible and retry later.

4. **Use no-gap single-word text switch**
   - Avoid multi-word overlap rendering.
   - Avoid "old word disappears, then blank, then new word" gap.
   - At switch point, update word immediately, then run float-in on the new word.

5. **Tune by adjusting shared constants only**
   - Slow down/speed up by changing shared duration once.
   - Keep text and molecule motion parity by avoiding per-component magic numbers.

## UX Guardrails
- **No overlap:** old/new text should not be simultaneously readable.
- **No blank flash:** no empty frame between text swaps.
- **No white screen:** molecule area should not flash white during switch.
- **No drift:** text and molecule switch timestamps stay aligned over time.

## Validation Checklist
- [ ] Text and molecule switch on the same tick.
- [ ] Float-in duration/easing feel identical on both.
- [ ] Repeated cycles show no timing drift.
- [ ] Network slowdown does not blank the molecule area.
- [ ] Build and lints pass after edits.

## Fast Debug Heuristics
- **Overlap appears:** text likely rendering previous+current layers together.
- **Blank flicker appears:** text has explicit out-phase before in-phase.
- **White flash appears:** layered canvas transition or pre-render visibility issue.
- **Desync appears:** more than one active interval source exists.
