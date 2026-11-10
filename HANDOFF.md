# BJJ Training App: Handoff

## What this is

A single-page, mobile-first web app (`index.html`, originally `bjj-workout.html`) for a BJJ white belt (1 stripe) getting back into strength training. The plan is **two kettlebell days a week using a single bell**. No framework, no build step, no backend. All data lives in the browser's `localStorage` on the user's phone. The owner explicitly does **not** want backups, export/import, or accounts.

It was built in claude.ai and published as a Claude artifact. This handoff moves it into a normal project so it can be hosted independently and installed on a phone.

## Files

- `index.html`: the entire app (HTML, CSS, and JS inline). About 700 lines. Renamed from
  `bjj-workout.html` so GitHub Pages serves it at the site root.
- `manifest.webmanifest`, `sw.js`: PWA manifest and service worker. Bump `CACHE` in `sw.js`
  on every deploy that changes any shell file, or phones keep the old copy.
- `fonts/`: self-hosted Barlow / Barlow Condensed woff2 (latin subset).
- `icons/`: `icon.svg` source plus 192, 512 and Apple touch PNGs.
- `bjj-workout.machines.html`: the previous gym-machine version, kept only as a reference
  snapshot from before the kettlebell rewrite. Nothing links to it; safe to delete.
- `HANDOFF.md`: this file.

## Features

1. **Day 1 / Day 2 tabs**: two single-kettlebell workouts (7 exercises each). Day 1 is swing, pull and grip (two-hand swing, goblet squat, single-arm row, suitcase deadlift, clean, suitcase carry, bottoms-up hold); Day 2 is get-up, press and hips (Turkish get-up, half-kneeling press, halo, single-leg deadlift, single-arm swing, windmill, neck work). Each exercise card has an animated SVG stick figure showing the movement (tap to pause), sets × reps, a short "why it helps BJJ" line, set circles to tap when done, a weight input (lbs), and collapsible form tips with a Google Images link.
2. **Rest timer**: fixed bar at the bottom, 60s or 90s. Starts automatically when a set is checked. Vibrates at zero where supported.
3. **Belt header**: shows the user's real rank (belt color + stripes on the rank bar). A stitching line fills in as today's sets get checked. "Edit belt" opens a panel to change color (white/blue/purple/brown/black) and stripes (0 to 4). Black belt gets a red bar.
4. **Finish workout**: saves a dated snapshot of the session to history, then clears today's set circles (weights stay filled in).
5. **Progress tab**: stats (total workouts, this week vs. goal of 2, weeks that hit the goal), a 12-week bar chart of workouts per week, a per-exercise weight trend line chart, and a recent workouts list with two-tap Remove.
6. **Game plan tab**: the user's own BJJ flowcharts, rendered vertically for mobile with their color coding (teal = position/goal, blue = your move, pink hexagon = key detail, yellow = read their reaction). Five charts: side control top, side control subs, side control defense (with interactive IF checklist that lights up Sweep vs. Guard), top half guard, and Reverse De La Riva.

## Architecture notes

- **Routing**: hash-based. `#day1`, `#day2`, `#flows`, `#progress`. `fromHash()` reads it; `render()` switches views by toggling `flows` / `progress` classes on `<body>`. Elements with `.workout-only` hide on the non-workout tabs.
- **Data**: `PLAN` (exercises per day), `FLOWS` (flowchart data), `ANIMS` (per-exercise animation definitions), `BELTS` (belt colors).
- **Animations**: a tiny forward-kinematics stick figure. Each entry in `ANIMS` gives two poses (`a`, `b`) as joint angles (`hx, hy` hip position; `t` torso; `ua, fa` arm; `th, sn` leg; `nk` neck; `rel` makes arm angles relative to torso). `lerp: true` interpolates joint positions instead of angles — use it whenever a planted foot would otherwise sink through the floor line mid-rep, which happens because the hip height lerps linearly but the leg's vertical extent doesn't (goblet squat and suitcase deadlift both need it). Some entries use a custom `draw(k)` instead (front views: halo, windmill). `eq(J, k)` and `top(J, k)` both receive the interpolation factor, which is how the bell swings from hanging to racked during the clean and the press. One `requestAnimationFrame` loop drives all figures; respects `prefers-reduced-motion` (starts paused).
- **The kettlebell** is drawn by `bell(p, a, s)`: a filled circle offset from point `p` in direction `a` (0 = straight down) with two handle lines back to `p`. `held` puts it in line with the forearm, `hang` straight down from the hand. Pointing it at `180` draws it bottoms-up.
- `/tmp/figcheck.js` (not checked in, easy to recreate) renders every `ANIMS` entry across `k` and prints bounding boxes — worth rerunning after pose edits to catch figures that clip the 200×180 viewBox.
- **Charts**: hand-written inline SVG (`weeklyChart`, `weightChart`). No chart library.
- **Theming**: CSS custom properties on `:root`, with dark-mode overrides under `prefers-color-scheme` and `[data-theme]`. Fonts: Barlow + Barlow Condensed from Google Fonts with system fallbacks.
- **Safe areas**: uses `viewport-fit=cover` and `env(safe-area-inset-*)` padding for iPhone notches.

## Data model (`localStorage` key: `bjj-plan-v1`)

One JSON object:

```json
{
  "day1-0": { "done": [0, 1], "w": "140" },
  "day2-3": { "done": [], "w": "60" },
  "history": [
    { "t": 1731170000000, "day": "day1",
      "ex": [ { "n": "Leg press", "d": 3, "of": 3, "w": 140 } ] }
  ],
  "rank": { "belt": "white", "stripes": 1 },
  "flow": "sc-def",
  "checks": { "sc-def-1": [0, 1] },
  "progEx": "Leg press"
}
```

- `day{N}-{i}`: today's checked set indices and the weight string for exercise `i`.
- `history[].ex[]`: `n` name, `d` sets done, `of` sets planned, `w` weight (number or null).
- Progress charts match exercises **by name**, so renaming an exercise in `PLAN` splits its history. Keep names stable or add a migration. The kettlebell rewrite renamed every exercise, so any history logged against the old machine plan still counts toward the workout totals and the weekly chart but will never appear in the weight-trend chart.

All reads and writes are wrapped in try/catch; the app works (without saving) if storage is unavailable.

## Important: data does not carry over

`localStorage` is per origin. Anything the user logged in the claude.ai artifact stays there. Once the app is hosted at a new URL, it starts fresh. The user chose no backups, so either accept the fresh start or (one time only, if they ask) write a small migration.

## Constraints and preferences

- Keep it a zero-build, single-file app unless there's a good reason to split it.
- No accounts, no backend, no backups, no export/import.
- Mobile-first; it's used one-handed at the gym.
- Keep the user's flowchart wording and color coding.
- The stick figures are illustrative, not precise form coaching.
- The plan assumes **one kettlebell**. Don't add exercises that need a matched pair or a
  barbell. Progression is reps and tempo first, a heavier bell second.

## Suggested next steps

Done: PWA (manifest, service worker, self-hosted fonts), `navigator.storage.persist()`,
and hosting on GitHub Pages.

1. **Rest timer in the background**: timers pause when the phone locks. Consider storing the end time (already done via `endAt`) and showing the remaining time on return, or using notifications.
2. Optional: let the user edit the plan (exercises, sets, reps) and the game plan charts from inside the app.

## Prompt to paste into Claude Code

> I have a single-file BJJ training web app (`bjj-workout.html`) and a `HANDOFF.md` describing it. Read the handoff first. Then turn the app into an installable, offline-capable PWA with no build step: add a web manifest, app icons, and a service worker that caches the app shell, and self-host the Google Fonts. Keep all data in localStorage under the existing key and schema, and don't add backups, accounts, or export. Then set it up for deployment to GitHub Pages and tell me the steps to install it on my phone.
