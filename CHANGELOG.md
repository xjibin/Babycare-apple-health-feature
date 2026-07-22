# Changelog

All notable changes to **Baby Care**. The in-app version number equals the release/commit
number. Format loosely follows [Keep a Changelog]; grouped as Added / Changed / Fixed.

> **Note on history:** detailed per-release notes below begin at **v0.59** (the point from
> which granular history is available). Everything before that is consolidated into a single
> **Foundations** section describing the feature set that existed by ~v0.58, reconstructed
> from the codebase rather than per-commit records.

---

## [1.16]
### Added
- Nappy prediction now also fires a **device notification** ("Mixed/Wet nappy likely due now — <baby>") when due, mirroring the in-app smart-stack hero card; de-duped once per feed cycle.
- Vomit/Spit-up card shows a green **"✓ All good for last 24 hrs"** line when no vomit/spit-up in the past 24 h.
### Notes
- A true iOS Live Activity / Dynamic Island Smart Stack needs a native app; the in-app hero card + device notification are the web equivalents.

## [1.15]
### Added
- Nursing sessions now record the **pause→resume gap**: each gap is stored (`data.gaps`, `data.pausedSec`) and the log detail appends "· paused M:SS". Only resumed gaps count; manual past-feeds unaffected.

## [1.14]
### Added
- **Baby-id safety net.** `babyGuard()` runs before every save, activity log, and sync: guarantees `S.activeBaby` resolves to a real record and self-heals a dangling pointer. Each entry is stamped with `bid` (audit tag); `syDoc` re-checks the id.

## [1.13]
### Changed
- Baby switcher rows: removed "Active now"/entry count; show **DOB** (renamed from "born") and the **baby id** on its own line; "+ Add another baby" centered.
### Added
- Switching/adding a profile is **blocked while an activity is running** ("Activity is running, can't switch profiles now"); allowed once stopped & saved.
- New baby data stays **local** until its endpoint + passphrase are configured; remote profile switching pauses/resumes sync automatically via per-baby endpoints.

## [1.12]
### Added
- **Force Recalculate** button in the Auto Mode card with an animated progress bar ("Recalculating…"). Result toasts: plain "No changes made. They are good now" or highlighted-border "Good catch. Timings are updated for Next Feed." Updates all Next Feed surfaces.

## [1.11]
### Changed
- Feed interval is now a **recency-weighted median over a rolling 7 days** (≈2-day half-life) instead of a flat median, so cluster-feeding/growth-spurt rhythms shorten it automatically and it lengthens again as gaps settle. Same-session re-latch cutoff lowered to 8 min.

## [1.10]
### Added
- Nightstand navbar shows a **Baby Profile** button (baby icon + name) in place of Chat when chat is disabled.

## [1.09]
### Fixed
- **Baby id auto-unifies across devices** on the same endpoint (smaller id wins, collision-safe) so two phones converge to one shared id after the legacy duplicate-profile mismatch.

## [1.08]
### Added
- Baby Profile shows the **baby id** right-aligned on the same row as the name (all devices).

## [1.07]
### Fixed
- **Auto-sync entry drift + missing remote-activity capsule.** Removed the obsolete `syMerge` baby-id guard that did an early `return false` on mismatched ids, silently blocking entries, live/capsule data, and chat between two devices. Isolation is now handled solely by per-baby endpoints.

## [1.06]
### Added
- Activity Log filter pills wrapped in a **"Filter by"** card.
- **Per-baby sync endpoints**: each baby stores its own on/url/passphrase; new babies start blank; each baby uses its own endpoint so remotes never mix.
### Fixed
- Baby profile sync omitted the **mother** section (caused incomplete profiles on the other device); now synced. Manual "Sync baby profile" auto-directs (newer/authoritative wins).

## [1.05]
### Fixed
- **Android Export-as-JSON + Send-to-partner.** Android Web Share rejects `application/json`; JSON now shares as `text/plain` (keeps `.json` name) with a robust `exportDl()` download fallback. iPhone/CSV unchanged.

## [1.04]
### Added
- **Manual sync card** in Sharing: "Sync activity entries" and "Sync baby profile" force an immediate reconcile (bypassing the per-baby id guard) with union-by-id merge and result counts. Auto-sync unchanged.

## [1.03]
### Added
- Per-baby **created** timestamp (back-filled on load); new profiles default DOB/TOB to today+now (never inherited) and open in edit mode.

## [1.02]
### Fixed
- **Cross-baby profile overwrite via sync.** A stale/bid-less remote could overwrite the active baby's profile and keep re-applying; hardened the guard and made records distinguishable (entries, born date, sex) for recovery.

## [1.01]
### Added
- **Multiple baby profiles.** Swap-based model: each baby has its own record (entries, sessions, meds, mother, feed settings, order, del). Long-press the navbar baby button for a switcher (switch / add). Settings → Baby Profiles lists all with a delete flow (confirm → type-the-exact-name → delete). Existing data migrates to baby #1.

## [1.00]
### Added
- **Device notifications** (foreground): next-feed-due, medicine reminders by dosage slot, "nappy o'clock" with hrs/mins since last nappy; de-duped, permission requested on first tab tap.
### Changed
- Activity Log filter pills wrap to the next line.
### Notes
- Closed-app push still needs a server (FCM/APNs) + service worker.

## [0.99]
### Added
- Settings **Chat enable/disable** toggle (off by default); when off, the navbar shows a Baby Profile button and the Summary baby card is hidden.
- Activity Log: Today/Yesterday expanded, older days collapsed (tap to expand, multi-expand), sorted latest-day-first even under filter; Log button scrolls to top.

## [0.98]
### Added
- Activity Log **filter pills** (Nursing, Nappy, Burp, Vomit, Medicine, Sleep, Car seat) + clear icon; multi-select; cleared on leaving the log.

## [0.97]
### Changed
- **Baby + mother profile redesign**: grouped cards (Baby details, Care team with multi-doctor, Mother) with a single DOB+TOB datetime picker; quick-call tel links.

## [0.96]
### Added
- Grid nursing suggested-side highlight; study-based intake-saturation feed-length factor.

## [0.95]
### Added
- **Auto Mode** toggle for feed prediction (greys manual steppers when on). Tomorrow's schedule projected above the Pattern card.

## [0.94]
### Added
- **List/Grid** layout toggle in Activity edit mode; auto-start cross-sleep capsule (start the other person's sleep now / in ~10 / ~30 min).

## [0.93]
### Changed
- Side-by-side compact Burp + Vomit cards (superseded by v0.94's grid).

## [0.92]
### Added
- **Vomit / Spit-up** full screen (time-of-vomit, past entries); forceful vomit shows a persistent "hold feeds" card + clamps the feed prediction. Baby profile hospital/ER/doctor quick-call (tel:).

## [0.73]–[0.91]
Granular per-release records for this range aren't preserved in this changelog. Themes across
the range include feed-prediction refinements, nightstand/landscape polish, sync/chat hardening,
medicine subsystem work, and the groundwork for the vomit screen and profile redesign that land at
0.92+.


## [0.72]
### Fixed
- **Auto-update now reliably prompts after a new deploy.** The version check existed but was
  defeated on real phones by iOS bfcache and caching.
### Changed
- Added `pageshow` + `focus` update triggers (iOS restores PWAs from bfcache without firing
  `visibilitychange`, so the check previously never re-ran on return).
- First update check now runs ~1.2 s after load; periodic check every 5 min (was 15).
- Version fetch hardened: absolute base URL + unique cache-busting query + `no-store` /
  `no-cache` headers so it always reads the freshly deployed file.
- **Refresh App** now navigates to a cache-busted URL so the new version actually loads.
- Dismissed updates still surface as a smartstack card as a fallback.

## [0.71]
### Added
- **Start date for every medication** — a "Started" stepper (Today / Yesterday / "Nd ago")
  in both the Add form and each Edit form; anchors course expiry.
- **+ Add past medication** — a focused form (pick medicine, pick date/time on the wheel) to
  log a dose given earlier as a past entry.
- **Portrait lock** — best-effort screen-orientation lock plus a landscape "rotate to
  portrait" overlay (the reliable path on iOS, where Safari/PWAs can't hard-lock).
### Changed
- Selected Log button glow increased ~20% (larger radius + slightly higher opacity).
### Fixed
- **Delete Entry** on edit screens moved above the fixed save bar so it's no longer hidden
  under the bottom gradient (a v0.69 side-effect) — fixes nappy and all edit screens.
- **Overscroll / rubber-band** — the document is now locked on mobile (fixed, overflow
  hidden) so dragging no longer pulls the whole app down and bounces back; only content
  scrolls.

## [0.70]
### Added
- **Edit medications** — an Edit button in the My Medications header opens every saved
  medicine in the full add-medication form format (name, dosage + unit, time of day,
  with-food, till-when, for-whom, Remove) in a scrollable section, with a fixed
  Exit / Save & Exit bar; edits are staged in a draft and committed together (preserving
  each medicine's original start date).
### Changed
- **Activity list core cards are never hidden:** Nursing, Nappy status, Burp, Baby sleep,
  Mother sleep, Medicine, **Growth**, Vaccine, Car seat (Growth was previously missing from
  the exempt set); all other cards, including user-added ones, still follow the hide rule.
### Fixed
- Nursing card no longer shows "Stop & Save" when nothing is running — it now keys off an
  actually-active session rather than the pre-created empty session shell.

## [0.69]
### Fixed
- **Bottom gradient covering action buttons.** Save-style buttons were rendered inside the
  scroll area where the gradient could cover them. They now render in a fixed bar pinned in
  the navbar zone (above the gradient), as a sibling of the scroll area, so the content
  scrolls underneath. The tab navbar is auto-hidden whenever a save bar is present. Applies
  everywhere the shared save-bar is used (add-past-entry, edit entry, baby profile, feed
  settings, growth, vaccines, expressed/formula, supplement, nappy, quantity, add-card).
  Implemented as a fixed sibling bar to avoid iOS's buggy `position:fixed`-in-a-scroller.

## [0.68]
### Fixed
- Nightstand "tap anywhere to exit" text no longer overlaps the session lines — it centers
  in the gap between the content and the bottom navbar.
### Changed
- Nightstand hides the Next Feed capsule whenever a nursing session is live (owner or remote).
- Removed the white outline on the selected Log button; added a soft ~10% glow in its own
  colour.
- Summary: Next Feed card now matches the Today's Output card height.
- Summary reordered → Next Feed → Output → Medicine Milestone, with Nightstand mode and
  Daily Dashboard as side-by-side cards after the milestone.
### Added
- **Chat device notifications** — incoming messages raise a native "Baby Care · sender:
  message" notification on other phones while their app is open/backgrounded-running; a
  permission prompt is requested when opening/sending in Chat.
  *(Limitation: closed-app delivery needs a push server — see roadmap.)*

## [0.67]
### Fixed
- Log screen no longer highlights Summary; the center Log button shows an active state instead.
### Changed
- Bottom black gradient now renders on all screens except Chat.
- Nightstand: Next Feed capsule always shown when a prediction exists; remote-running
  sessions show a blinking ▶ next to the timer; brighter background + brighter gender night
  colours; the bottom navbar is recoloured to match and made **functional** (tap any item to
  exit the nightstand and jump straight to that screen; Log → Activity Log).
- Add-medication card: DOSAGE promoted to a section title with quantity ± and unit dropdown
  on one line; added TIME OF DAY and WITH FOOD titles; more spacing between groups and above
  titles.
- Customise Settings: Exit/Save centered; Save renamed **Save and Exit**; Reset with no
  changes shows "No changes were made · already in default settings"; group cards tinted with
  their matching activity colours.
- Smartstack scroll dots moved to a vertical stack at the right-centre, beside the arrow.

## [0.66]
### Changed
- Bottom navbar: coloured Log button moved **between Activity and Sharing** (raised center
  button); bar spans full width; boy tint unified to **#6FA8FF**.
- Removed the dead Search action from the nav.
- Modal No/Yes button text centered.
- Footer "Customise Settings" gains a › chevron.
- Medicine milestone: streak line shows "· Baby/Mother"; tapping the card opens the Medicine
  screen (checkbox still logs); card pins above Next Feed when a dose is due or no medicine is
  added yet, else drops below.
- Medicine screen: 280 px bottom scroll; dosage unit is a separate dropdown (ml, mg, g, cap,
  tab), stored on new meds and backfilled to `ml` on old ones.
### Added
- Baby profile: "Hospital ID" → **Baby Hospital ID** plus a new **Mother Hospital ID** field.
- Growth › Checks: lists **all** developmental checks with future ones greyed out and
  disabled ("opens at N mo") until the baby reaches each window.
- Firebase security rules provided (1-year and permanent variants).

## [0.65]
### Added
- **Customise Settings engine** — footer entry → confirm modal (No auto-selects via a 3-second
  countdown) → a full settings screen: searchable (≥3 chars), grouped cards with subtext,
  ± steppers and option chips, red **Reset to Default**, and Exit / Save (with a save-confirm
  modal). **27 real tunables** wired end-to-end (feed intervals & day band, nursing autosave /
  side capsule / first side, sleep suggestion windows, medicine dose windows, resume window,
  card auto-hide, auto-nightstand, car-seat limits, sync poll / chat retention / presence /
  remote freshness & command expiry, vaccine window) and synced to the partner phone.
- **Medicine courses** — Before/After/With Food, Till-When (days/weeks/months/year, or SOS as
  "as needed"), and For Whom (Baby/Mother); stored on new meds, backfilled on old; courses
  expire out of Today/Due/milestone automatically; SOS meds never nag.
### Changed
- Log button raised to match the bottom navbar height (62 px), gender-tinted pill.
- Nightstand centers the exit hint and shows a dimmed navbar clone.
### Fixed
- Rubber-band pull-down locked at the top of every screen (`overscroll-behavior`).

## [0.64]
### Changed
- Log entry point restyled as a 52×62 rounded pill with the icon stacked over a "Log" label
  (both `#000`), on a gender-tinted background (pink for Girl, blue for Boy).

## [0.63]
### Added
- Scroll-to-top on every new screen; re-tapping the active tab smooth-scrolls to top (except
  Chat).
### Changed
- "All Health Data" renamed to "&lt;Baby&gt;'s Activity Log".
- Search circle replaced by the Log pill (→ Activity Log). *Note: the standalone search
  screen became unreachable from here and still needs re-homing.*

## [0.62]
### Added
- Time-aware Medicine Milestone — dose windows via `curSlot`; the milestone checkbox ticks
  the current window's slot specifically.
- Decorative bottom black gradient (`#bgrad`) on the Summary screen.

## [0.61]
### Added
- **Nursing auto-stops sleeps everywhere** — starting a nursing session (including a resume)
  auto-stops and saves any running baby/mother sleep across devices, attributed to the
  nursing starter.
- **Medicine Milestone card** — two columns, per-medicine day-streak with a tappable
  checkbox, and a "No medicines added" fallback.
### Changed
- Dual (nursing) saves now record the last-active side.

## [0.60]
### Added
- Vitamin D streak on the Summary screen.
- Output screen: Today's Nappies total with ▲▼ vs yesterday, plus Today's Nappy Log.
- **Remote control** — a pulse dot with remote pause/stop for the partner's live session,
  intent-based and attributed to the owner, with a "Saving on …'s phone" toast.
- **Resume model** — a 30-minute "Resume this session" capsule (dual/single views and the
  same-day entry editor) that rebuilds the session.
### Fixed
- Past-sleep picker initialisation (fell-asleep / woke-up wheels).

## [0.59]
### Added
- **Cross-phone live sessions** — the partner's in-progress activities appear live (read-only,
  with an owner initial badge); night/nightstand shows remote sessions too.
### Changed
- Removed the "GitHub project" footer link.

---

## Foundations (pre-0.59)

The core app was built up over the earliest ~58 commits. By this point Baby Care already had:

- **Activity tracking:** nursing (dual left/right timer with side suggestion), baby sleep and
  mother sleep timers, nappy status (Wet / Mixed / Soiled / Dry), burp, expressed / formula /
  supplement feeds, pumping, growth (weight / length / head), vaccines, car seat, medicine,
  and user-added custom cards with reorder/hide.
- **Summary dashboard:** next-feed prediction (day/night intervals with auto-learning), today's
  totals, a baby profile card (name, age, blood type, sex, hospital ID), and a smartstack of
  contextual hero cards.
- **Daily Dashboard & Trends:** per-activity totals vs yesterday and simple charts.
- **Activity Log / history:** full entry list, edit entry, and "add a past entry" (adaptive
  per activity) with date/time wheels.
- **Nightstand mode:** a dimmed, calm clock screen for night feeds.
- **Chat & Sharing:** in-app chat and the multi-device sync setup (encrypted Firebase RTDB).
- **PWA:** installable to the Home Screen (manifest + icons), portrait-oriented, with an
  in-app update check.
- **Data portability:** full JSON export / import; localStorage persistence that survives app
  close and restart.
- **Siri Shortcuts** note for quick logging.

[Keep a Changelog]: https://keepachangelog.com/
