---
name: scroll-driven-video
description: Build a cinematic web experience where a video (or image sequence on mobile) is scrubbed frame by frame by the user's vertical scroll. Use when asked for scroll-driven / scroll-scrubbed video, scrollytelling with video, Apple-style scroll animation, "video controlled by scroll", or a cinematic hero whose text, zoom and opacity sync with scroll progress. Covers FFmpeg encoding, sticky 400vh layout, rAF-smoothed currentTime, WebP+Canvas mobile fallback, and performance/accessibility.
---

# Scroll-Driven Video

Act as a senior web developer (JavaScript/React, cinematic animation, media optimization). Build an experience where scroll position, not playback, drives the video. Never autoplay.

## 1. Prepare the video

Encode so every frame is a keyframe (fast random seeking), no audio, web-friendly size:

```bash
ffmpeg -i input.mp4 -an -c:v libx264 -crf 20 -preset medium -g 1 -keyint_min 1 -movflags +faststart -pix_fmt yuv420p output.mp4
```

- Scale to the display need (e.g. `-vf scale=1920:-2`); keep file weight modest (all-intra is heavy; raise `-crf` to 23-26 if > ~15 MB).
- Mobile frames (WebP sequence), 60-120 frames is usually enough:

```bash
ffmpeg -i input.mp4 -vf "fps=24,scale=720:-2" -c:v libwebp -quality 70 frames/frame_%04d.webp
```

## 2. Page structure

- Outer section: `height: 400vh`, `position: relative`.
- Inner container: `position: sticky; top: 0; height: 100vh; width: 100%; overflow: hidden`.
- Video: `width: 100%; height: 100%; object-fit: cover`.
- No horizontal scrollbar (`overflow-x: hidden` on the wrapper; avoid `100vw` on scrolling elements).
- Text overlays are absolutely positioned inside the sticky container.

## 3. Progress

```js
const rect = section.getBoundingClientRect();
const scrolled = -rect.top;
const progress = clamp(scrolled / (section.offsetHeight - window.innerHeight), 0, 1);
```

0 = first frame, 0.5 = middle, 1 = last frame.

## 4. Smoothing (required)

Do NOT set `video.currentTime` in the scroll handler. The scroll handler only stores the target progress (passive listener). A single `requestAnimationFrame` loop eases toward it:

```js
t += (progress * video.duration - t) * 0.1;
if (Math.abs(video.currentTime - t) > 1 / 120) video.currentTime = t;
```

Stop the loop when `|target - t|` is negligible and restart it on the next scroll, to save battery.

## 5. Video setup

- Attributes: `muted`, `playsInline`, `preload="auto"`, no `autoplay`, no `controls`.
- On `loadedmetadata`: call `video.pause()`, read `duration`, then enable the loop. Guard against `duration` being `NaN`/`Infinity`.
- Handle `error` and rejected promises (show the poster / fall back to the image sequence).
- iOS: seeking needs the video to be buffered; call `video.load()` and prime it with a muted `play().then(pause)` inside a try/catch after first user gesture if frames do not update.
- Never block the page while loading: show a poster and a lightweight progress indicator; load lazily (`IntersectionObserver`).

## 6. Scroll-synced animation

Drive overlays from the same `progress` (CSS variables or inline styles updated in the rAF loop; use `transform` and `opacity` only):

- Main headline fades/moves out between progress 0.15 and 0.25:
  `opacity = 1 - clamp((p - 0.15) / 0.10, 0, 1)`.
- Titles in/out in ranges (e.g. 0.30-0.45, 0.55-0.70, 0.80-0.95).
- Slow zoom on the media (`scale(1 + p * 0.15)`), opacity cross-fades, elements entering/leaving, text overlays, soft transitions to the next section.
- Helper: `rangeProgress(p, start, end) => clamp((p - start) / (end - start), 0, 1)`.

## 7. Mobile (< 768 px)

- Use the WebP sequence rendered on a `<canvas>` (`drawImage`, `cover` math, handle `devicePixelRatio`).
- Smart preload: load first frame immediately, then every Nth frame, then fill gaps; prioritize frames near the current index; keep a bounded cache (drop frames far from the viewport index).
- Draw only when the integer frame index changes. Use `ImageBitmap` (`createImageBitmap`) when available.
- Do not rely on video seeking on iOS.
- Decide with `matchMedia('(max-width: 767px)')` and react to changes.

## 8. Performance and accessibility

- 60 FPS target: one rAF loop, passive scroll listener, no layout reads in the loop except one cached measurement (refresh on `resize`).
- Cleanup on unmount: cancel rAF, remove scroll/resize/matchMedia listeners, disconnect observers, release bitmaps, `video.removeAttribute('src'); video.load()`.
- Adapt to constrained devices (`navigator.deviceMemory`, `saveData`): lower frame count/resolution.
- `prefers-reduced-motion`: disable zoom and smoothing, show static poster and plain text.
- Provide meaningful text in the DOM (not baked into the video), `aria-hidden` on decorative media, keyboard scrolling works by default.
- Cross-browser: Safari, Chrome, Firefox; test with throttled CPU.

## 9. Visual design

Modern, professional, cinematic: full-screen, balanced composition, elegant transitions, strong modern typography (variable font, tight tracking on large display text), restrained palette, legible overlays (gradient scrim behind text). For art direction and polish, also use `frontend-design`, `design-taste-frontend` or `impeccable` if available.

## Reference skeleton (React)

```jsx
import { useEffect, useRef } from 'react';

const clamp = (v, a = 0, b = 1) => Math.min(b, Math.max(a, v));
const range = (p, s, e) => clamp((p - s) / (e - s));

export default function ScrollVideo({ src, poster, frames, frameCount }) {
  const sectionRef = useRef(null);
  const videoRef = useRef(null);
  const canvasRef = useRef(null);
  const titleRef = useRef(null);

  useEffect(() => {
    const section = sectionRef.current;
    const mobile = window.matchMedia('(max-width: 767px)').matches;
    const reduce = window.matchMedia('(prefers-reduced-motion: reduce)').matches;
    let target = 0, t = 0, raf = 0, running = false, disposed = false;
    const images = [];

    const progressNow = () => {
      const r = section.getBoundingClientRect();
      return clamp(-r.top / (section.offsetHeight - window.innerHeight));
    };

    const apply = (p) => {
      if (titleRef.current) {
        titleRef.current.style.opacity = String(1 - range(p, 0.15, 0.25));
        titleRef.current.style.transform = `translateY(${range(p, 0.15, 0.25) * -40}px)`;
      }
      const media = mobile ? canvasRef.current : videoRef.current;
      if (media && !reduce) media.style.transform = `scale(${1 + p * 0.15})`;
    };

    const draw = (p) => {
      if (mobile) {
        const i = Math.round(p * (frameCount - 1));
        const img = images[i];
        const c = canvasRef.current;
        if (img?.complete && c) {
          const ctx = c.getContext('2d');
          const s = Math.max(c.width / img.naturalWidth, c.height / img.naturalHeight);
          const w = img.naturalWidth * s, h = img.naturalHeight * s;
          ctx.drawImage(img, (c.width - w) / 2, (c.height - h) / 2, w, h);
        }
      } else {
        const v = videoRef.current;
        if (v && Number.isFinite(v.duration)) {
          t += (p * v.duration - t) * (reduce ? 1 : 0.1);
          if (Math.abs(v.currentTime - t) > 1 / 120) v.currentTime = t;
        }
      }
    };

    const loop = () => {
      const p = target;
      draw(p);
      apply(p);
      raf = disposed ? 0 : requestAnimationFrame(loop);
    };
    const start = () => { if (!running) { running = true; raf = requestAnimationFrame(loop); } };
    const onScroll = () => { target = progressNow(); start(); };

    if (mobile) {
      const c = canvasRef.current;
      c.width = window.innerWidth * Math.min(devicePixelRatio, 2);
      c.height = window.innerHeight * Math.min(devicePixelRatio, 2);
      for (let i = 0; i < frameCount; i++) {
        const img = new Image();
        img.decoding = 'async';
        img.src = frames(i);
        images.push(img);
      }
    } else {
      const v = videoRef.current;
      v.addEventListener('loadedmetadata', () => v.pause());
    }

    window.addEventListener('scroll', onScroll, { passive: true });
    window.addEventListener('resize', onScroll);
    onScroll();
    return () => {
      disposed = true;
      cancelAnimationFrame(raf);
      window.removeEventListener('scroll', onScroll);
      window.removeEventListener('resize', onScroll);
    };
  }, [frames, frameCount]);

  return (
    <section ref={sectionRef} style={{ height: '400vh', position: 'relative' }}>
      <div style={{ position: 'sticky', top: 0, height: '100vh', width: '100%', overflow: 'hidden' }}>
        <video ref={videoRef} src={src} poster={poster} muted playsInline preload="auto"
               style={{ width: '100%', height: '100%', objectFit: 'cover' }} />
        <canvas ref={canvasRef} style={{ position: 'absolute', inset: 0, width: '100%', height: '100%' }} />
        <h1 ref={titleRef}>Headline</h1>
      </div>
    </section>
  );
}
```

(Show only the element for the active mode: hide the canvas on desktop and the video on mobile.)

## Checklist before finishing

- [ ] Scrolling forward/back scrubs smoothly, no jumps or flicker
- [ ] No autoplay; video paused after metadata
- [ ] Headline fades out across 0.15-0.25
- [ ] Mobile path uses WebP + canvas with bounded memory
- [ ] Listeners and rAF cleaned up on unmount
- [ ] Reduced-motion and load/error states handled
- [ ] No horizontal scroll at any width
