# Focus Manager — Handover Document

## 1. What this is

A single-page web app that combines a **daily habit/task tracker** with a
**football-manager game layer** as its reward system. Completing habits and
tasks earns XP, which levels up a Manager rank and unlocks stat points to
spend on an 11-player squad. A CPU league, season structure, and a
Google-Calendar-style daily schedule sit alongside the core tracker.

No build step, no framework, no bundler — it's plain HTML/CSS/JS designed to
be opened directly in a browser or dropped onto any static host.

---

## 2. Project structure

```
/
├── index.html            All markup, styling, and application logic (single file)
├── sw.js                 Service worker — enables real OS/browser notifications
├── manifest.webmanifest  Minimal PWA manifest (installable on Android/desktop)
└── HANDOVER.md            This document
```

All three runtime files must be **hosted together in the same folder** —
`index.html` references `sw.js` and `manifest.webmanifest` by relative path.

---

## 3. Feature map

| Area | What it does |
|---|---|
| **Setup** | One-time "Name your club" prompt. Input is permanently removed from the DOM after submit and never reappears once a club name exists in local storage (see §6). |
| **Summary tab** | Dashboard: today's habits/tasks (tickable inline) + a compact squad-rating strip. |
| **Tasks tab** | Red / Yellow / Blue priority tiers (Urgent&Important / Important / neither). Tasks support a scheduled start time, an optional countdown timer (minutes), and an optional numeric **target goal** with live progress logging. Sections **auto-sort**: tiers with active tasks float to the top (Red→Yellow→Blue order), empty/fully-done tiers sink to the bottom — this ordering is automatic and cannot be manually overridden. Individual tasks *within* a tier can still be nudged up/down. |
| **Calendar tab** | 24-hour vertical day view. Any task with a start time renders as a positioned block spanning its duration, color-coded by priority. The block for "right now" gets a gold highlight and a live mm:ss countdown. Goal-tracked tasks can be logged and adjusted (+/−) directly on the block; hitting the target auto-completes the task and awards XP. |
| **Habits tab** | Recurring daily habits. Three modes: plain tick, numeric goal (≥ / ≤ / = a target), or Yes/No "clarification" habits. |
| **Statistics tab** | Today's log + 7-day (or 7-week, for Yes/No habits) history charts per habit. |
| **Squad tab** | 11 players in strict formation order (GK, RB, CB, CB, LB, CM, CM, CM, RW, LW, ST). Outfield players use PAC/SHO/PAS/DRI/DEF/PHY; the GK uses DIV/HAN/KIC/REF/SPD/POS. Overall rating is an EA-style weighted average of position-relevant stats + a 0–3 reputation bonus. Any player can be selected at any time; XP-earned stat points are shared across the whole squad and spent 1-per-click in an upgrade modal. |
| **Weekly Matches tab** | One CPU match per real day, 38 games per season. The **Play Match** button stays locked until every habit and task for the day is complete, and shows a live progress readout while locked. Today's opponent (with a full generated 11-player roster, ratings clustered within ±2 of a random team baseline) is locked in for the day and displayed up front. Win probability: ratings within 7 points = 50/50 coin toss; otherwise the higher-rated team wins (plus a small independent draw chance). End-of-season recap shows W/D/L, goals, and top scorers. |
| **Notifications** | A persisted on/off toggle (not a one-shot button) drives real Web Notifications (via the service worker where available) for timer milestones — start, halfway, 5-minutes-left, and time's-up (with a Web Audio alarm). No custom in-app popups are used for these. |
| **Google Sign-In + Cloud Sync** | Optional. Wired against Firebase Auth + Firestore. Signing in checks for an existing cloud save and, if found, prompts to overwrite local-with-cloud or cloud-with-local; if none exists, it seeds the cloud from local data. Requires your own Firebase project keys (see §7). |

---

## 4. Data model (persisted as one JSON blob)

Stored under `localStorage` key `focus-manager-v6`, shape:

```js
{
  teamName, notificationsEnabled,
  tasks: [{ id, name, priority, time, durationMin, targetGoal, goalUnit,
             loggedValue, done, startedAt, notified, awaitingConfirm }],
  habits: [{ id, name, goalType, entryType, target, unit, done }],
  history: { "YYYY-MM-DD": { [habitId]: { value, done, failed } } },
  xp, points, selected, ngp, lastUpgrade,
  players: [{ id, pos, name, country, avatarSeed, rep, stats }],   // 11 entries
  cpuTeams: [{ id, name, rating, crestSeed, roster }],              // 20 entries
  season: { number, gamesPlayed, wins, draws, losses, goalsFor,
             goalsAgainst, scorers, lastPlayDate, todayOpponent },
  matches: [ /* last 15 match reports */ ]
}
```

A migration block runs on every load to backfill missing fields for anyone
upgrading from an older save, so this shape can keep evolving safely.

---

## 5. Setup instructions

1. Download all three files (`index.html`, `sw.js`, `manifest.webmanifest`) into one folder.
2. **Local testing:** serve them with any static server — service workers and
   some browser features refuse to run over `file://`. Easiest option:
   ```bash
   npx serve .
   # or
   python3 -m http.server 8080
   ```
   Then open `http://localhost:8080`.
3. **Real deployment:** push the folder to any static host (GitHub Pages,
   Netlify, Vercel, Firebase Hosting, S3+CloudFront, etc.) over HTTPS —
   required for the service worker and push notifications to work.
4. **Optional — enable cloud sync:** open `index.html`, find the
   `firebaseConfig` object near the top of the `<script>` block, and replace
   the placeholder values with your own Firebase project's SDK config
   (Firebase console → Project settings → Your apps). Also turn on the
   **Google** sign-in provider under Authentication, and create a Firestore
   database. Until this is filled in, the app runs fully offline.
5. **Optional — installable app feel:** once hosted over HTTPS, the
   `manifest.webmanifest` lets mobile Chrome offer "Add to Home Screen,"
   which gives push notifications a more native presentation on Android.

### Known platform limits (by design, not bugs)
- `localStorage` doesn't persist inside Claude.ai's own in-chat preview
  sandbox — it works normally once hosted or opened outside that sandbox.
- The service worker only registers on a real `https://` or `localhost`
  origin.
- True native Android (Kotlin/`NotificationCompat.Builder`) notifications
  aren't produced by this project — it's a web app. Installing it as a PWA on
  Android Chrome gets you real system notifications without writing native
  code; a fully native app would be a separate Android Studio project.

---

## 6. Task 3 — "Name Your Club" conditional logic

This was already implemented and is re-confirmed here:

- On load, `localStorage` is read **before** the first render.
- `render()` shows the setup screen only `if(!state.teamName)`; otherwise
  it's hidden and the main app renders immediately — no flash of the input.
- On submit, the club name is written to `localStorage` **synchronously**,
  and the `<form>` element is removed from the DOM outright
  (`e.target.remove()`), not just hidden — so it cannot reappear later in
  that session, and never appears again on any future load from that device.

---

## 7. Handoff notes / next steps for whoever picks this up

- Firebase keys are placeholders — cloud sync is inert until they're filled in.
- CPU rosters/team pool (20 clubs) are generated once and persisted; there's
  no "refresh league" action yet if you want new opponents to rotate in.
- All game-balance numbers (XP rewards, stat weights, win-probability
  thresholds, season length) live as named constants near the top of the
  `<script>` block for easy tuning.
