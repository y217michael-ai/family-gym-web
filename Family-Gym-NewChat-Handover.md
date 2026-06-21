# FAMILY GYM — PROJECT HANDOVER (paste this into a new chat)

You are picking up an ongoing build of **Family Gym**, a single-file HTML fitness web app. This brief is everything you need to continue seamlessly. Read it fully before making changes.

---

## WHO / CONTEXT

- **User:** Mike (UK, Bridlington). Non-technical — prefers clear step-by-step guidance, picking from visual options, and confirming design decisions before big changes. Owns DM Design Fabrication.
- **Mike has a bilateral hip condition (cam lesion)** that limits hip flexion — "hip-safe" filtering is built in and matters throughout.
- **Three users of the app:** Mike (admin, green `#39FF14`), Michelle (orange `#FF8C42`), Amy (purple `#6C63FF`). Each person now has their own copy of the app with their own name/data; they share workouts to each other via a Share button.
- **Tone note:** Mike asked to avoid repeatedly saying "honestly"/"I have to be honest." Be direct but don't lean on that phrasing.

---

## ENVIRONMENT & WORKFLOW (critical — read carefully)

- The app is **ONE HTML file**. Live at: `https://y217michael-ai.github.io/family-gym-web` — repo `https://github.com/y217michael-ai/family-gym-web`.
- **You work in a sandboxed Linux container with NO network** to GitHub/Apple/Xcode/most APIs. You CANNOT push, deploy, wrap, or submit. Mike deploys manually on his Mac (MacBook Pro M5).
- **Working copy you edit:** `/home/claude/family-gym-app.html` (~9,200 lines).
- **Ship to:** `/mnt/user-data/outputs/family-gym-app.html`, then call `present_files`.
- **Mike's deploy command** (give after every ship):
  ```
  rm ~/Desktop/FamilyGym/index.html && cp ~/Downloads/family-gym-app.html ~/Desktop/FamilyGym/index.html && cd ~/Desktop/FamilyGym && git add . && git commit -m "..." && git push
  ```
  Then he hard-refreshes Safari with ⌘⇧R.

### Testing discipline (do this EVERY ship, no exceptions)
1. **JS syntax check:** extract `<script>` blocks, run `new Function(code)`.
   ```bash
   node -e "const fs=require('fs');const c=fs.readFileSync('family-gym-app.html','utf8');const m=c.match(/<script>([\s\S]*?)<\/script>/g);let code=m.map(s=>s.replace(/<\/?script>/g,'')).join('\n');try{new Function(code);console.log('JS OK');}catch(e){console.log('ERR:',e.message);}"
   ```
2. **Playwright headless test:** load `file:///home/claude/family-gym-app.html`, listen for `pageerror`, exercise the feature via `page.evaluate`.
3. **Regression:** run `/tmp/finaltest.js` (visits all screens + nutrition tabs). All must say "ok".
4. **Screenshot and view** the new UI before shipping.
5. Copy to outputs + `present_files` + give the deploy command.

### Edit gotchas (learned the hard way)
- `let` redeclaration throws — shared state must be declared ONCE in the early globals (~line 2086), not in functions that run at startup.
- Load-order: functions called during startup (`renderHome`, `renderPlanner`) need their globals declared early. `plannerView`, `EDB_CACHE`, `ROUTINE_FILTERS`, `activeEquipment` were all moved up after ReferenceErrors.
- When inserting a modal before an HTML comment anchor, the comment can get consumed — verify adjacent modals stay intact afterward with grep.
- Patches can accumulate syntax errors; if it gets messy, restore from a backup copy and reapply cleanly.

---

## ARCHITECTURE

- **Users:** `USERS` object (~line 1416) with color, initials, devices, hrMin/Max, calorieGoal, weight, age. `activeUser` global (~line 2086).
- **~528 exercises** in an `EXERCISES` array, fields `{name, muscle, sets, reps, weight, hipSafe, desc, video?, restSec?, equipment?}`.
- **localStorage keys** (all prefixed `fgym_`):
  `fgym_routines`, `fgym_history`, `fgym_sports`, `fgym_notes`, `fgym_photos`, `fgym_privacy`, `fgym_lastweights`, `fgym_custom_equip`, `fgym_equip_photos`, `fgym_videocfg`,
  and per-user (suffix `_<user>` or `_<user>_<date>`): `fgym_hipsafe_`, `fgym_tempo_`, `fgym_dayplans_`, `fgym_videogender_`, `fgym_water_`, `fgym_water_settings_`, `fgym_nutsettings_`, `fgym_meals_`, `fgym_savedmeals_`, `fgym_mealplan_`, `fgym_dateplan_`, `fgym_plan_`, `fgym_daylog_`, `fgym_bodystats_`.
- **Key globals (~line 2086):** `activeUser`, `plannerView`, `plannerMonthRef`, `planPickDay`, `ddDate`, `activeRoutineFilter`, `activeEquipment`, `EDB_CACHE`, `ROUTINE_FILTERS` (global list), `EQUIPMENT_LIST`, `COMMON_EQUIPMENT`, `EDB_EQUIPMENT_TYPES`, `activeRoutineView` (forced `'family'`).

### Styling (established)
CSS vars: `--bg:#08080E; --surface:#0F0F18; --card:#101018; --border:#3A3A48; --lime:#39FF14; --purple:#6C63FF; --red:#FF4D4D; --text:#FFFFFF; --muted:#C8C8D4; --soft:#16161F; --michelle:#FF8C42; --babyblue:#A9DCF5`. Exercise names = baby blue. Chips: unselected `#16161F` fill+border; selected = user-colour tint fill + colour border.

---

## EXTERNAL SERVICES

| Service | Use | Status | Where in code |
|---|---|---|---|
| **YMove Pro** | Nutrition DB, meal plans, AI food logging, videos | **PAID/active** | `YMOVE_API_KEY=''` (~line 6900) — EMPTY, needs Mike's `ym_...` key |
| **ExerciseDB (RapidAPI)** | Exercise GIFs + equipment tags | Free tier, working | `RAPIDAPI_KEY` (~line 7785), `fetchExerciseGif` |
| **Open Food Facts** | Free food DB (current fallback) | Working | `fetchFoods` |
| **Claude API** | Eating-out menu reader (vision) | Works on-device only | `analyseMenu` — model `claude-sonnet-4-6`, in-artifact API |
| **Gym-Animations** | Proper 3D videos (M+F) | NOT bought yet | local MP4 pipeline ready (`VIDEO_DEFAULTS`, `localVideoSrc`) |

- **Nutrition is wired to try YMove first, fall back to Open Food Facts.** `fetchFoods` (~line 6900s) tries `https://exercise-api.ymove.app/api/v2/foods?q=...` with header `X-API-Key`, normalises via `normaliseYMoveFood` (per-serving → per-100g), wraps with `packAsOFF`; `normaliseFood` passes through `__normalised` rows. **It only activates once Mike pastes his key into `YMOVE_API_KEY`.**

---

## FEATURES BUILT (all shipped, tested, zero errors)

- **Workout builder** (`openNewRoutine`/`saveNewRoutine`/`editRoutine`) — manual + auto-fill; routines start **private** (`sharedWith:[]`); **equipment tagging** in builder (`renderNrEquipmentTags`/`toggleNrEquipment`, saved as `r.equipment[]`).
- **Generator** (`openWorkoutGenerator`) — multi-group, difficulty, duration, antagonist pairing, hip-safe, warm-up/cool-down. Generated routines also start private.
- **Tempo circuits** — rounds, custom rest, collapsible circuits, tappable exercises → how-to modal.
- **Day Plan** ("super workout") — chains sport/tempo/routine items, each launched as its own session; saveable/reusable (`fgym_dayplans_`).
- **Guided phone runner** — warm-up/cool-down, 3-2-1 beep, video/GIF inline.
- **Sharing** — `openShareRoutine`/`confirmShareRoutine` modal; pick Michelle/Amy with ticks; lands in their "Shared With Me". `↗ Share` button on each routine card.
- **Equipment system** (above Routine Options): single **"Workout by Equipment Available"** button → picker modal grid (`openEquipmentPicker`/`renderEquipmentGrid`). `EQUIPMENT_LIST` (All/Technogym/FreeWeights/Cable/Machine/Bodyweight/Cardio(Wahoo)/Pool). **+ Add equipment** → `addCustomEquip` opens `modal-add-equip` with: built-in `COMMON_EQUIPMENT` list + `EDB_EQUIPMENT_TYPES` (ExerciseDB, ~28 types) + search + "type your own". Custom kit stored `fgym_custom_equip`; `allEquipment()` merges built-in+custom. Photos per item (`fgym_equip_photos`, `addEquipPhoto` downscales to dataURL). `matchesEquipment` filters using routine's own `equipment[]` tags + EXERCISES lib equipment + cached ExerciseDB `equipment` (EDB_CACHE) + keyword fallback.
- **Filter button** (above Routine Options): **"Filter Workouts"** → `openFilterPicker`/`renderFilterGrid`, uses global `ROUTINE_FILTERS`. `setRoutineFilter` updates button + grid + re-renders.
- **Log Sport** — Ready→Start→Pause→Finish. Single big **Pause** at top; **Finish & Save** + **Cancel session** at bottom (cancel confirms via `cancelSportSession`). GPS route tracking (`startSportGPS`, haversine distance, elevation), beta-labelled. HR/cals simulated, labelled "estimated — live watch data comes with the native app".
- **Planner** — Week/Month tabs (`setPlannerView`). Month calendar (`renderPlannerMonth`): Mon-Sun grid, today highlighted, dots (green=done, userColor=planned workout, michelle=planned meal), ‹›nav. Tap day → `openDayDetail`/`renderDayDetail` modal showing logged sessions w/ full stats + planned workout/meals + "+Plan workout"/"+Plan meal". Storage: `fgym_dateplan_` (planned workouts), `fgym_mealplan_` (planned meals).
- **Recent Workouts** — each card tappable → `openHistoryDetail` modal (stat tiles, muscles, time-in-zone chart, notes). Red × delete + Delete in modal → `deleteHistoryEntry` with confirm.
- **Nutrition** — food search (YMove→OFF), barcode, macros/micros, water, saved meals, meal plan, trends, body stats. **Eating-Out advisor** (`openMenuAdvisor`/`analyseMenu`): photo a menu → AI suggests dishes vs remaining targets (`todayTargetsRemaining`). On-device internet only.
- **Renames/cleanup:** "Family Routines"→"Routine Options" (tab + heading; internal `'family'` key kept). Removed redundant "👤 Mike" user tab — single combined Routine Options view (`tabsEl` hidden, `activeRoutineView` forced `'family'`).
- **Video pipeline** ready for Gym-Animations: `videos/male/<slug>.mp4` & `videos/female/`, graceful fallback to GIFs.

---

## DECISIONS ON RECORD

- **Video library:** Gym-Animations chosen (7,000+ 3D, male+female, lifetime commercial licence, $199–599). NOT bought yet — Mike deciding. Exercise Animatic = instant-buy alt (4K, male-only, ~$329). When files arrive: wire name-matching + align descriptions in one pass.
- **ExerciseDB:** DO NOT BUY. Already free for GIFs + equipment tags. Gives GIFs not videos.
- **No Technogym/Wahoo machine database exists** publicly; barcodes link to Technogym's private MyWellness. So branded-machine photos are user-added.
- **Launch path:** web app can't go to App Store as-is. Build/test on web (fast loop) → when solid, wrap once for **TestFlight** (Capacitor, ~half-day on Mac, Claude writes click-by-click guide) → then real HealthKit (Apple Health/Withings) + real GPS → App Store. Needs Apple Developer $99/yr + Xcode at that stage only. Real HR/Withings/background GPS are native-only (simulated on web now).

---

## PENDING / NEXT

1. **Mike to paste his YMove `ym_...` key** into `YMOVE_API_KEY` to switch nutrition to the paid library (then redeploy).
2. **Decide/buy the video library**; then wire videos in.
3. **Real-world testing** by Mike/Michelle/Amy; fix what surfaces; polish.
4. **TestFlight wrap guide** when features are solid; then HealthKit + Withings; then App Store.
5. Leftover minor "family" wording in a few toasts/admin spots — Mike hasn't asked to change these.
6. User switcher (Mike|Michelle|Amy at top of home) still present — flagged that each person now has own app so it may be redundant; Mike hasn't decided.

---

## HOW TO BEHAVE
- Before big/broad changes, ask 1–2 clarifying questions (Mike likes choosing from options and confirming).
- Flag limitations plainly (no network/GitHub/API/Apple in sandbox; web can't read real watch HR/Withings/reliable GPS; can't wrap/submit) — without leaning on the word "honestly".
- Always: syntax check → Playwright test → screenshot → regression → ship to outputs → present_files → give deploy command.
