# No Work No Check — project guide for Claude

Workout tracking app for a strength coach (Heriberto) and his athletes. It works like TrainHeroic: the coach builds programs, athletes log workouts, and the coach tracks compliance and progress.

**Live site:** https://hjrivera15.github.io/noworknocheck/ (GitHub Pages, deployed from `main`, root folder)
**Backend:** Firebase project `no-work-no-check`, using Auth (Email/Password + Google) and Firestore.
**Coach email:** hjrivera1534@gmail.com (in `firebase-config.js`, inline in `index.html`, and in `firestore.rules`)

## Files
- `index.html`: the entire app (HTML + CSS + vanilla JS in one file, no build step). Firebase compat SDK v10.12.2 is loaded from gstatic. The Firebase config is also inlined as a fallback right after the `firebase-config.js` script tag.
- `firebase-config.js`: Firebase web config plus `coachEmails`. Not secret.
- `firestore.rules`: security rules. **This file does not deploy automatically.** After changing it, tell the coach to paste it into Firebase Console → Firestore → Rules → Publish.
- `sw.js`: service worker (network-first for app files, cache-first for the gstatic SDK and fonts).
- **Versioning:** on every change that will be uploaded, bump BOTH `APP_VERSION` in `index.html` (shown as "Version N" at the bottom of the app) and `VERSION = 'nwnc-vN'` in `sw.js`, keeping the numbers equal. Each user's version is saved to `roster/{uid}.appVersion` on open; Coach → Athletes flags anyone behind as "needs to reload".
- `manifest.webmanifest` and the icons (`icon-192.png`, `icon-512.png`, `apple-touch-icon.png`, `favicon-32.png`, `logo.webp`): PWA install and branding.

## Data model (Firestore)
- `library/{workoutId}`: workouts `{id, name, focus, items:[{type:'section',label} | {type:'ex',name,sets,reps,load,rest,notes}]}`. Coach writes.
- `programs/{id}`: `{name, weeks, days[7] (workoutId|null, Mon-first), phases:[{from,to,name,rpe,preset,main:{sets,reps,pct},note}], main:[exercise names], presetWorkouts:[ids], skipSections:[labels], warmup:[exercise names]}`. Coach writes.
- `assign/{uid}`: `{uid, programId, start (YYYY-MM-DD Monday), label, wids:[workout ids in that program]}`. Coach writes. Athletes read only their own.
- `roster/{uid}`: `{id, name, email, lastSeen, appVersion}`. Each user writes their own.
- `exinfo/{exKey}`: coach notes per exercise `{name, text, video?}`. `video` = a YouTube / Shorts, Instagram Reel, TikTok, Vimeo or .mp4 link, or a clip file name in the repo's `videos/` folder (`videoEmbed`). Plays inside the ⓘ pop-up everywhere (sandboxed iframe: no popups or navigation; own clips use a plain `<video playsinline>`). Video links pasted in the notes text also play inline. `sw.js` never caches videos or range requests.
- `teamfoods/{id}`: foods shared with the whole team `{id, name, brand?, serving, per (one serving), by (uid), byName, created}`. Any signed-in user reads/creates (by = self); only the creator or coach deletes. Shown in Food → Saved and first in Search.
- `msgs/{id}`: messages `{id, uid (athlete thread, or 'all' = announcement), from, coach (bool), name, text, date, workout?, logId?}`. Coach reads all (latest 500); athletes read their own thread + `all`. Read markers: `users/{uid}/data/msgread` (`kind: 'msgread'`; athletes `{last}`, coach `{at:{uid:time}}`).
- `users/{uid}/data/{doc}`: per-athlete data, split by `kind` (also `moves`, see below; week check-off docs may carry `perfect: true`):
  - no kind = workout log `{id, workoutId, name, date, week, rpe, notes, pros, cons, exercises:[{name, sets:[{weight,reps,rpe,done}]}]}`. Cardio sets use other fields, chosen by `exKind` / `SET_COLS`: treadmill → `{speed, incline}`, assault/air/echo bike → `{cal}`, timed run (run/jog/sprint + a time plan like "20 min") → `{dist, time}`, treadmill timed run → all four. Logs before Oct 2026 may have treadmill sets saved as weight/reps; `setTxt` still displays those.
  - `food` (id `food-YYYY-MM-DD`) = that day's food log `{date, items:[{id, meal b|l|d|s, name, brand?, unit g|srv, amt, sdesc?, liq?, per:{p,c,f,kcal}}]}` (unit g: amt is grams, or ml when liq; always shown/typed in oz / fl oz); `myfood` = saved foods `{name, serving, per}`; `macros` (id `macros`) = the athlete's own targets `{p,c,f,goal}`. The coach sees it read-only on the athlete's page (Coach → Athletes → name → Food), via `foodDayHtml(src, ro)`.
  - `week` = manual check-offs, `body` = weight/BF%, `profile` = height, `max` = 1RM with history, `cnote` = coach's personal notes per workout.
  - The athlete and the coach can both read and write here.

## Access control (paid clients)
- Athletes can read ONLY their assigned program (`programs/{programId}`) and the workouts listed in `assign/{uid}.wids`. Unassigned accounts see nothing. Enforced in `firestore.rules`, not just the UI (the repo is public).
- The coach's app keeps `wids` in sync automatically (`syncAssignWids`, plus `saveProgram` updates assignments before saving the program).
- Athlete clients never query whole collections for `library`/`programs`; they listen to single docs (`watchProgram` / `watchWorkouts`). Don't add collection listeners for athletes — the rules will reject them.

## Key behaviors to preserve
- Week tab: Mon–Sun check-off; a day auto-checks when that workout is logged that week.
- 6-week phases: main lifts take the phase sets/reps/%; RPE preset applies to `presetWorkouts`, except `skipSections`.
- % loads: `@ 75%` or phase `main.pct` × athlete max, rounded to 5 lb.
- Smart paste importer (`smartParse`) turns pasted programs (DAY headers, "4×6", "3 sets of 8", tables, rounds, schedule, phases) into workouts and a program.
- Rest timer (`startRest`, `restFor`, `restsIn`, `buildBlocks`): starts when a set is checked off. Length comes from the cue ("rest 90 sec", "rest 2 to 3 min" → low end), else the coach default `exinfo/_settings.restDefault` (90). Supersets / "N rounds" sections rest only after the last exercise; warm-up/prep/mobility/cool-down sections and cardio sets (treadmill, bike, timed runs) never. Big countdown in the bottom dock with −15 / +15 / Skip, 3-2-1 beeps, Go. (The coach removed an older timer in 2026; this one was requested Oct 2026.)
- Interval timer (`ivOpen`/`ivStart`/`ivTick`): a rounds block where every move is a time set and a rest ≤ 30 sec is written gets a Start card. Full-screen work/rest/lead-in phases (A, B, A, B…), reps box during each rest, pause/resume/end, session-only work/rest/rounds edits (`session.ivcfg`). Finishing starts the block rest from the cue ("rest 90 sec after the block"). Timing used is saved as `log.timing` (array of strings) and shown in History and the coach view.
- All timers run off end timestamps (one 200 ms clock), so they survive app switching. Per-phone prefs in localStorage `nwnc-prefs` {sound, vib, wake}, set in Profile → Workout settings. Audio unlocks on first tap.
- Moving workouts: Week tab Move button swaps a workout with another day (or moves it to a rest day) for this week only. Stored as `users/{uid}/data/mv-YYYY-MM-DD` {kind:'moves', week, days[7], origin[7]}; `effDays` applies it everywhere (Week tab, grid via `gridDone`, coach `athWeek`), ignoring it if the program changed underneath. Manual ✓ travels with the workout. Risky-move warnings (`moveWarning`, word list `HARD_DAY`) warn but never block; starting another day's workout warns if it makes two workouts today. Coach sees "Fri → Mon".
- Perfect Week (`weekPerfect`, `celebrateIfPerfect`): every scheduled workout done Mon–Sun shows a full-screen card once (flag `perfect: true` on the week's check-off doc, preserved by `saveChecks`), a ★ Perfect Week badge on the Week tab and coach Athletes list, and a ★ on that week's row in the progress grid.
- Athletes see Week, Program, History, Profile, Messages (they start workouts from Week/Program).
- Coach tab sub-sections (`S.csub`): Athletes / Programs / Workouts (`library`) / Add (`build`) / Maxes / Notes (`info`: start-of-workout message settings + exercise notes) / Backup.
- Warm-up sets: exercises in `program.warmup` get one extra set flagged `warm: true` (shown as "W", ~50% of working weight or last warm-up weight). Warm sets are excluded from volume, bests and progress charts.
- Messages tab (everyone, with unread badge): two-way coach↔athlete threads + coach announcements to all; pop-up of unread messages once per sign-in (`maybeMsgPopup`). Finishing a workout asks "What went well / What didn't" and sends it to the coach as a message.
- Screen stays on during a workout: `syncWake` (called from `render`) holds a Screen Wake Lock while `S.session` exists and re-requests it when the app returns to the foreground; a "Screen stays on" note shows only while the lock is held.
- Food tab (macro tracking, pilot): shown only to emails in `FOOD_USERS` (index.html). Search = USDA FoodData Central (Foundation + SR Legacy, which includes older menus from big chains like McDonald's, Chick-fil-A, Taco Bell, Wendy's, KFC, Subway; plus Branded). Picking a result loads its USDA serving sizes (`loadPortions`) as tap-to-pick buttons (`USDA_KEY` = the coach's free api.data.gov key, 1,000 searches/hour), plus My foods, Quick add, copy day/meal, athlete-set targets with a body-weight suggestion, and a 7-day summary vs body weight.
- Update notice: `watchForUpdates` HEAD-checks `index.html`'s ETag on open, on return to foreground and every 15 min; if it changed, a "App updated · Reload" bar shows at the top (draft workouts survive the reload).
- Start-of-workout message: `showMotivation` pops a Bible verse and/or coach message when a workout starts; settings live in `exinfo/_motivation`.

## Working rules
- Keep it a single static `index.html` with no frameworks or build tools. It's deployed by uploading or pushing to GitHub.
- Mobile-first: most users are on phones, so test at ~380px width. Support light and dark mode.
- Never break existing data shapes; athletes already have logs stored.
- After any change that touches Firestore paths or permissions, update `firestore.rules` **and** tell the coach to republish it.
- The coach is not a developer. Explain changes in plain language with short steps.

## Before every change
The coach sometimes edits the app elsewhere (e.g. in Claude chat) and uploads it straight to GitHub. Before editing, compare the local `index.html`/`sw.js` with the live site (`curl` the GitHub Pages URL). If live has changes the local copy lacks, start from the live version so nothing gets overwritten.
