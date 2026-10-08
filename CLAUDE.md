# No Work No Check — project guide for Claude

Workout tracking app for a strength coach (Heriberto) and his athletes. It works like TrainHeroic: the coach builds programs, athletes log workouts, and the coach tracks compliance and progress.

**Live site:** https://hjrivera15.github.io/noworknocheck/ (GitHub Pages, deployed from `main`, root folder)
**Backend:** Firebase project `no-work-no-check`, using Auth (Email/Password + Google) and Firestore.
**Coach email:** hjrivera1534@gmail.com (in `firebase-config.js`, inline in `index.html`, and in `firestore.rules`)

## Files
- `index.html`: the entire app (HTML + CSS + vanilla JS in one file, no build step). Firebase compat SDK v10.12.2 is loaded from gstatic. The Firebase config is also inlined as a fallback right after the `firebase-config.js` script tag.
- `firebase-config.js`: Firebase web config plus `coachEmails`. Not secret.
- `firestore.rules`: security rules. **This file does not deploy automatically.** After changing it, tell the coach to paste it into Firebase Console → Firestore → Rules → Publish.
- `sw.js`: service worker (network-first for app files, cache-first for the gstatic SDK and fonts). Bump `VERSION` when changing cached files.
- `manifest.webmanifest` and the icons (`icon-192.png`, `icon-512.png`, `apple-touch-icon.png`, `favicon-32.png`, `logo.webp`): PWA install and branding.

## Data model (Firestore)
- `library/{workoutId}`: workouts `{id, name, focus, items:[{type:'section',label} | {type:'ex',name,sets,reps,load,rest,notes}]}`. Coach writes.
- `programs/{id}`: `{name, weeks, days[7] (workoutId|null, Mon-first), phases:[{from,to,name,rpe,preset,main:{sets,reps,pct},note}], main:[exercise names], presetWorkouts:[ids], skipSections:[labels]}`. Coach writes.
- `assign/{uid}`: `{uid, programId, start (YYYY-MM-DD Monday), label}`. Coach writes. Athletes read only their own.
- `roster/{uid}`: `{id, name, email, lastSeen}`. Each user writes their own.
- `exinfo/{exKey}`: coach notes per exercise `{name, text}`.
- `users/{uid}/data/{doc}`: per-athlete data, split by `kind`:
  - no kind = workout log `{id, workoutId, name, date, week, rpe, notes, exercises:[{name, sets:[{weight,reps,rpe,done}]}]}`
  - `week` = manual check-offs, `body` = weight/BF%, `profile` = height, `max` = 1RM with history, `cnote` = coach's personal notes per workout.
  - The athlete and the coach can both read and write here.

## Key behaviors to preserve
- Week tab: Mon–Sun check-off; a day auto-checks when that workout is logged that week.
- 6-week phases: main lifts take the phase sets/reps/%; RPE preset applies to `presetWorkouts`, except `skipSections`.
- % loads: `@ 75%` or phase `main.pct` × athlete max, rounded to 5 lb.
- Smart paste importer (`smartParse`) turns pasted programs (DAY headers, "4×6", "3 sets of 8", tables, rounds, schedule, phases) into workouts and a program.
- No rest timer or workout clock. The coach asked for them to be removed.
- Coach-only tabs: Add, Coach (Athletes / Programs / Maxes / Info).

## Working rules
- Keep it a single static `index.html` with no frameworks or build tools. It's deployed by uploading or pushing to GitHub.
- Mobile-first: most users are on phones, so test at ~380px width. Support light and dark mode.
- Never break existing data shapes; athletes already have logs stored.
- After any change that touches Firestore paths or permissions, update `firestore.rules` **and** tell the coach to republish it.
- The coach is not a developer. Explain changes in plain language with short steps.
