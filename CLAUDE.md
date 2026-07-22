# CLAUDE.md — Baby Care development guide

Guidance for Claude Code (or any LLM/human) continuing work on the **Baby Care** app.
Read this fully before making changes. The app is a single self-contained file with a
lot of hard-won, non-obvious structure — small careless edits break it easily.

---

## 1. What this is

**Baby Care** is a portfolio mockup presented as an "Apple Health feature" concept — a
newborn-tracking PWA. It was started on day one of a real baby's life and is used daily by
two real people:

- **Dad** (primary user) — iPhone, adds it to the Home Screen as a standalone PWA.
- **Mum** — Android PWA.
- Baby: **Iza** (girl, blood type B−, DOB 8 Jul 2026 12:17). Multiple baby profiles are now supported (see §13).

It is **not** an Apple product; every screen carries a "portfolio mockup, not an Apple
product" disclaimer. Design language deliberately mimics iOS / Apple Health.

**Live:** https://xjibin.github.io/Babycare-apple-health-feature/
**Repo file served by GitHub Pages:** `index.html` (at repo root).

> **Convention that matters:** the in-app version number **equals the commit/release
> number**. Current release is **v1.16**. Keep this 1:1 (in-app `APPVER`, the `<!-- Baby Care vX.XX -->` marker, and `Test build vX.XX` must all match).

---

## 2. Architecture at a glance

- **One file.** The whole app is a single HTML document — HTML + CSS + one big `<script>`.
  No framework, no bundler, no npm runtime deps. **Plain ES5-flavoured vanilla JS** (use
  `var`, `function`, string concatenation — no arrow functions / template literals / `let`
  / optional chaining in shipped code, for maximum old-Safari safety).
- **Rendering** is a hand-rolled string-templating loop: state → `render()` builds an HTML
  string → assigned to `#scroll.innerHTML`. No virtual DOM.
- **Events** use a single delegated click handler: every interactive element has
  `data-a="actionName"` (and optional `data-x="param"`). The handler finds
  `e.target.closest("[data-a]")` and calls `A[actionName](dataX)`. This is why nesting a
  button inside a tappable card "just works" (closest wins).
- **Persistence:** `localStorage["babycare.v1"]` holds the full state blob.
- **Multi-device sync:** Firebase Realtime Database, end-to-end encrypted client-side
  (see §9).
- **Two personas of data:** owner (this device) vs remote (partner phone), surfaced live.

---

## 3. Build & deploy pipeline

The shipped `index.html` is **generated**, not hand-edited, so the source stays readable
(the huge base64 asset blobs live outside the template).

### Source/build files (keep these together; commit them to the repo, ideally under `/src`)

| File | Purpose |
|---|---|
| `tpl.html` | **Master template.** All real editing happens here. Contains placeholders `__ICONS__`, `__TOUCHICON__`, `__MANIFEST__`. |
| `icons.json` | ~57 Lucide inner-SVG path strings, keyed by name, used by `ic(name,...)`. |
| `icon.b64` | Base64 PNG for the PWA/touch icon. |
| `manifest.b64` | Base64 of the PWA manifest JSON (already has `"orientation":"portrait"`). |

> **If you only have `index.html`:** it is fully self-contained and can be edited directly,
> but it inlines the base64 blobs (icons/manifest/touch icon) which makes it unwieldy. Prefer
> reconstructing the split (extract the three blobs back out into `icons.json` / `icon.b64` /
> `manifest.b64` and replace them with the placeholders) so you can use the pipeline below.

### Build steps

```bash
# 1) inject assets into the template
python3 - <<'PY'
import re
icons = open('icons.json').read()
touch = open('icon.b64').read().strip()
mani  = open('manifest.b64').read().strip()
src   = open('tpl.html').read()
out   = src.replace('__ICONS__', icons).replace('__TOUCHICON__', touch).replace('__MANIFEST__', mani)
# assert the three version markers agree BEFORE writing
assert re.search(r'Test build v([0-9.]+)', out).group(1)  # sanity
open('index.html','w').write(out)          # <-- deploy artifact
m = re.search(r'<script>\n(.*)\n</script>', out, re.S)
open('/tmp/app.js','w').write(m.group(1))  # <-- for syntax check
print('built')
PY

# 2) syntax-check the extracted script (fast fail before shipping)
node --check /tmp/app.js && echo "OK"
```

Only ship if `node --check` passes.

### Three-way version markers — ALL must match on every release
> ⚠️ Environment quirk seen repeatedly: writes to the big template occasionally **revert the
> three version markers** (content edits persist). Always **re-verify the markers in the BUILT
> file** after building; if wrong, force them with `re.sub(...,count=1)` and rebuild + re-verify.


1. `<!-- Baby Care vX.XX -->` (top HTML comment)
2. `var APPVER="X.XX";` (used for the in-app "App Version" label + update check)
3. `Test build vX.XX` (in the export/about copy — **this is the string the auto-updater
   greps for**, so it must be correct or updates won't be detected)

A bump helper:

```python
for old,new in [('<!-- Baby Care v0.72 -->','<!-- Baby Care v0.73 -->'),
                ('var APPVER="0.72";','var APPVER="0.73";'),
                ('Test build v0.72','Test build v0.73')]:
    assert s.count(old)==1, old
    s = s.replace(old, new)
```

### Deploy (GitHub Pages)

1. Build → produces `index.html`.
2. Commit `index.html` to the repo root and push (GitHub Pages serves it).
   - **Common failure:** downloading the file but never committing it. If the live "App
     Version" label doesn't change, the file was never published.
3. GitHub Pages takes ~1–2 min to build + purge its CDN (watch for the green check).
4. Users on **v0.72+** get an in-app "Refresh App" modal automatically within a few minutes
   / on next foreground. Earlier versions may need one manual hard refresh
   (`…/?r=<anything-new>`) to reach a version that has the fixed updater.

---

## 4. The safe patch workflow (how to actually make code changes)

Never free-hand large rewrites. Edit `tpl.html` with **anchored, counted string
replacements** so a moved anchor fails loudly instead of silently corrupting the file.

```python
s = open('tpl.html').read()
def R(name, old, new, n=1):
    # idempotent: if already applied, skip
    if new in s and old not in s:
        print("skip:", name); return
    c = s.count(old)
    if c != n:
        print("FAIL:", name, "count=", c, "exp", n); raise SystemExit(1)  # abort, don't write
    s = s.replace(old, new)
# ... many R(...) calls ...
open('tpl.html','w').write(s)   # write ONLY at the very end
```

Rules that prevent pain:
- **Count-guard every replace** (`n=`). If the count is wrong, abort and re-read the file —
  the anchor moved.
- **Write only at the end.** A mid-script `raise` then leaves `tpl.html` clean.
- **Prefer function-name / unique-marker anchors over line numbers** (line numbers drift
  constantly).
- For an ambiguous anchor, slice the function: find `function X(` then the next
  `\nfunction ` to bound your edit region.
- After a big feature, extract a helper (as done with `medAddCardHtml`) rather than making
  one return statement enormous.
- Escape carefully: the script is full of nested single/double quotes and `\u00B7` (·)
  style unicode escapes. Match them exactly.

---

## 5. Testing (do this every release)

There is a jsdom harness discipline that has caught dozens of regressions. Two layers:

### a) Behavioural suite (per feature)

Load the built HTML in jsdom with these stubs (critical — the app touches all of them):

```js
const dom = new JSDOM(html, {url:'https://test.local/', runScripts:'dangerously',
  pretendToBeVisual:true, beforeParse(w){
    // make scrollTo FUNCTIONAL so scroll-to-top logic can be asserted
    w.Element.prototype.scrollTo = function(o){ if(o&&typeof o==='object') this.scrollTop=o.top||0; };
    w.scrollTo = ()=>{};
    w.confirm = ()=>true; w.alert = ()=>{};
    w.HTMLCanvasElement.prototype.getContext = ()=>null;      // canvas charts
    // for sync tests only:
    // Object.defineProperty(w,'crypto',{value:require('crypto').webcrypto});
    // w.fetch = <mock>;   // for update-check / sync tests
  }});
```

Then drive `w.A.<action>()`, set `w.S` / `w.ui`, call `w.render()`, and assert on
`#scroll.innerHTML`, the fixed bars (`#tb`, `#svbar`, `#csbar`, `#medbar`), and state.

### b) 14-screen smoke test (always run before shipping)

Render each screen and assert it produces non-empty content:
summary, dashboard (`sub=dash`), output (`sub=output`), activity list, nursing, sleep,
car seat, medicine, vaccines, growth, sharing, chat, history (`sub=hist`), cset (`sub=cset`).

Navigation cheat-sheet for tests:
- Summary root: `S.tab='summary'; S.sub=''; S.route='list'`
- Summary sub-screens: `S.tab='summary'; S.sub='dash'|'output'|'hist'|'trends'|'baby'|'feed'|'cset'`
- Activity detail (e.g. medicine): `S.tab='activity'; A.open('medicine')` (sets `S.route`)
- Edit an entry: `S.eidx = <index>` → renders `vEdit()`
- Past entry: `S.pmode = true` on an activity route
- Nightstand: `A.night()` (overlay in `#ov`), exit `A.nightoff()`

---

## 6. Code map (find things by name, not line number)

- `render()` — the master render. Resets `pendingSaveBar`, builds `h` via the dispatch
  chain, sets `#scroll.innerHTML`, then toggles the fixed bottom bars and runs any wheel
  (date/time) picker init for the current screen.
- Dispatch chain (inside render): `S.eidx!=null → vEdit()`, else `S.tab==="summary"` →
  sub-router (`vCset/vHistory/vTrends/vBaby/vFeed/vDash/vOutput/vSummary`), else activity /
  pmode / new-card views.
- **Views:** `vSummary`, `vDash`, `vOutput`, `vHistory`, `vTrends`, `vBaby`/`vBabyView`,
  `vFeed`, `vMed`, `vGrowth`, `vVax`, `vDual`/`vSingle` (timers), `vNew`, `vCset`, plus the
  "add a past entry" adaptive view.
- **Actions object `A`** — every `data-a` maps to `A.<name>`. This is the entry point for
  all interactivity.
- **Icons:** `ic(name, size, color, {sw, fill})` reads from `icons.json`.
- **Cards (Activity list):** `cardChips(a)` renders the per-card quick buttons; the
  never-hide set is `CORE` and the auto-hide test is `usedIn4d()` driven by `cfgGet("hideDays")`.
- **Fixed bottom bars (all siblings of `#scroll`, z-index 20, above the gradient z-12):**
  - `#tb` — the tab navbar, built by `tabbarHtml()` (Summary · Activity · **Log** · Sharing
    · Chat, with the coloured Log button docked in the middle).
  - `#svbar` — single-button save bar, driven by the `pendingSaveBar` descriptor that
    `saveBar(color,enabled,action,label)` records (returns "" and defers rendering to
    render()). This dodges the iOS `position:fixed`-inside-a-scroller bug.
  - `#csbar` — Customise Settings Exit / Save & Exit bar.
  - `#medbar` — Edit-medications Exit / Save & Exit bar.
  - `#bgrad` — decorative bottom fade (shown on all screens except Chat / nightstand).
  - `#rotate` — landscape "please rotate to portrait" overlay (CSS media query).
- **Date/time wheels:** `dtPicker(pid)` renders the columns; `initDT(pid, defDate, onch)`
  must be called after render (see the init block in `render()`); `dtValue(pid)` reads it.
  `DMAX` caps days-back. `initDOB`/`dobValue` are the birthday variant.

---

## 7. State model

- **`S`** — the persisted app state (saved to localStorage + synced). Key fields:
  `entries` (the activity log; sorted newest-first), `sess` (live session shells per
  activity), `order`/`custom`/`hiddenCards` (activity card layout), `baby` (profile incl.
  `uhid`, `muhid`, `sex`, `dob`, `blood`), `meds` (medicine list — see §8), `feedIvD`/
  `feedIvN` (feed intervals), `cfg` ({} of Customise-Settings overrides), `who` (this
  device's person name), `chat`, `reads`/`seen` (presence), `rsess` (remote live sessions),
  `cmd`/`cmdAp` (remote command channel), `del`/`setUp`/tombstones (sync bookkeeping).
- **`ui`** — transient UI state (NOT persisted): current form inputs, draft objects
  (`csDraft`, `medDraft`), picker state `dts`, flags (`sheet`, `showAll`, `bedit`,
  `medEdit`, `medPast`, `upVer`, `upDismissed`), smartstack index, etc.
- `save()` persists `S` and schedules a sync. `bumpSet()` bumps the "settings bundle"
  version so profile/meds/cfg/order changes propagate to the partner phone.

---

## 8. Medicine subsystem (the most-edited area)

A medicine object:

```js
{ id, name, ml, unit,               // unit ∈ ml|mg|g|cap|tab
  times:{m,n,ni},                    // morning / noon / night slots
  food,                             // "Before Food" | "After Food" | "With Food"
  tillN, tillU,                     // course length; tillU ∈ d|w|mo|y|sos
  whom,                             // "Baby" | "Mother"
  ts0 }                             // course START timestamp (anchor for expiry)
```

- `medMigrate()` backfills legacy meds (called on load/import and before any med UI).
- Course maths: `medEnd(m)` / `medExpired(m)` / `medSched(m)` (scheduled = not expired &&
  not SOS). Expired & SOS meds are excluded from Today/Due; SOS is excluded from Due but
  allowed in the milestone.
- Slot windows are cfg-driven via `curSlot(hour)` (morA/morB/noonA/noonB/niA).
- Screens: `vMed` shows Today (tap to log via `medDone`), a **Medicine Milestone** hero
  (streak + `· Baby/Mother`, taps through to the screen; the checkbox logs), **My
  Medications** list with an **Edit** button, an **Add** form, and **+ Add past
  medication** (`ui.medPast` → focused form with a time wheel → `savePastMed`).
- **Edit mode** (`ui.medEdit`, `ui.medDraft`): per-medicine forms (`medFormHtml`,
  handlers `emT/emF/emW/emU/emTd/emml/emStart/emDel`), staged in a draft; `medEditSave`
  commits, `medEditExit` discards; leaving the screen auto-cancels.

---

## 9. Sync (Firebase RTDB) & security rules

- **Per-baby endpoints (v1.06).** Sync config (`on`/`url`/`pass`) is stored **per baby**
  (`baby.sync`) and swapped into `SY` by `babyLoad`/out by `babyStash`. Iza uses her own
  `…/iza.json`; a new baby starts **blank** (no sync) until an endpoint + passphrase are set,
  so its data stays local until "subscribed". Each baby must use its **own** node/URL — the
  URL is now the isolation boundary. `syLoadCfg()` runs **before** `load()` so migration has SY.
- Client-side crypto: PBKDF2 (150k iters, salt `babycare-sync-v1`) → AES-GCM. Wire format `{c:<ciphertext>}`.
- Poll loop: `syTick` (~pollS s) + `save()` (debounced) + `visibilitychange`.
- Doc shape (`syDoc()`, guarded by `babyGuard()`): `bid` (**active baby id**), `entries`
  (LWW by `up`), `del` tombstones, `live` (remote sessions → `S.rsess`, drives the remote-activity
  capsule), a `set{}` bundle (`baby`, **`mother`**, `cfg`, `meds`, `order`, `hidden`, `custom`,
  `feedIvD/N`) versioned by `setUp`, `chat`, `cmd`, `reads`/`seen`.
- **`syMerge` no longer blocks by baby id (v1.07).** The old bid guard did an early `return false`
  on mismatched ids and silently blocked entries + live/capsule + chat between two devices; removed
  because per-baby endpoints now isolate. Isolation = the endpoint, not the id.
- **Baby-id auto-unify (v1.09):** if two devices on the same endpoint have different ids, they
  converge to the smaller id (collision-safe) — legacy from an early duplicate-profile bug.
- **Manual sync (v1.04):** Sharing → "Sync activity entries" / "Sync baby profile" call
  `manualSync(mode)` which force-merges (`SY.force`) bypassing any guard; entries = union by id
  (skips profile), profile = auto-direction (newer/authoritative wins), never destructive.
- **Rules (publish in the RTDB console).** Lock everything except the `iza` node:

```json
{
  "rules": {
    ".read": false,
    ".write": false,
    "iza": {
      ".read": true,
      ".write": true,
      ".validate": "newData.hasChildren(['c']) && newData.child('c').isString() && newData.child('c').val().length < 2000000"
    }
  }
}
```

(A time-boxed variant swaps `true` for `"now < <ms-timestamp>"`.) With **per-baby endpoints**, each
baby uses its own node (e.g. `iza`, `romero`) — add a rule per node (or a shared parent) so every
baby’s URL is writable. Verify a bare `…/.json` is denied while each baby node works. **Test-mode
rules expire — publish permanent rules or sync silently dies.**

---

## 10. Non-obvious gotchas (learned the hard way)

- **Empty session shells:** `render()` pre-creates `S.sess[k]` as an empty object, so
  `if(S.sess.nursing)` is truthy even when nothing runs. **Always test liveness with
  `sessActive(k)`** (running OR accumulated > 0.5 s). (This caused a "Stop & Save" ghost.)
- **Duplicate `function` declarations:** last one wins. Watch for name collisions
  (`fmtMin` vs a new `fmtM`).
- **`position:fixed` inside `.scroll` is buggy on iOS** (momentum scrolling). Fixed bottom
  bars must be **siblings of `#scroll`**, toggled by `render()` (that's the whole point of
  `#svbar`/`#csbar`/`#medbar` + `pendingSaveBar`).
- **Hiding the navbar for save bars:** when a save bar is present, `render()` blanks `#tb`
  so the two never fight for the bottom slot.
- **jsdom normalises inline styles** — assert via the style object (`el.style.zIndex`) not
  a raw-attribute regex; and beware that some numbers render as `0.22` not `.22`.
- **Base64 blobs & manifests are embedded**, so raw-HTML string checks for manifest fields
  fail — decode the base64 (mime is `application/manifest+json`).
- **Update marker:** the auto-updater greps the fetched HTML for `Test build vX.XX`. If you
  ever reword the export copy, keep that exact token.
- **iOS PWA update detection** needs `pageshow` + `focus` listeners (bfcache restores don't
  fire `visibilitychange`) plus an absolute, uniquely-cache-busted fetch. This is already
  in `upCheck()`; don't weaken it.
- **Child-of-`.scroll` date wheels** re-init on every render via the init block — read
  current input values into the draft before re-rendering (see `medEditSync`,
  `byId("medname")` preservation) or typed text is lost.
- **Ternary dispatch chains** (render router, summary hero placement) — insert new branches
  at the correct position; order matters.

---

## 11. Conventions

- **Vanilla ES5 only** in shipped code (see §2). Keep it framework-free and dependency-free.
- **Minimal formatting on the phone.** The UI targets a small screen; keep prose/labels tight.
- **Every release:** bump the 3 version markers, run the behavioural suite + 14-screen
  smoke test, `node --check`, then build `index.html`.
- **Release notes format** used historically — a short headline (`vN.NN: summary`, < ~72
  chars) plus a bulleted description, and an honest caveats section (iOS limits, latency,
  known-unreachable features).
- Be honest in replies about what can't be done client-side (e.g. closed-app push).

---

## 12. Open items / roadmap

- **Search screen unreachable since v0.63** — the search entry point was replaced by the
  Log pill and never re-homed. Either delete the dead `vSearch`/action or add an entry point.
- **Closed-app push (chat + feed/medicine/nappy notifications)** — foreground device
  notifications exist (v1.00: feed-due, medicine-by-slot, nappy o'clock/prediction). True
  closed-app delivery needs a Firebase Cloud Function + service-worker push handler + FCM.
  Not possible in the single client file alone. **On hold** per product decision.
- **iOS Live Activity / Smart Stack** — not web-doable from a PWA; the in-app hero smart-stack
  card + device notifications are the equivalents.
- **Editing a Vomit log entry** — deferred (tap-to-edit all vomit fields not built).
- **Firebase permanent rules** — confirm they're published (see §9).
- **No service worker** currently — offline support is limited to what the browser caches;
  a SW would enable true offline + reliable install, but adds cache-invalidation
  complexity (would interact with the update flow — design carefully).

---

## 13. Multiple baby profiles & data isolation (v1.01+)

**Swap-based model.** The **active** baby's data lives in the live `S.*` fields as usual.
A registry `S.babies=[{id, created, sync{}, ...PERBABY...}]` + `S.activeBaby` holds every baby.

- `PERBABY = ["baby","entries","sess","meds","order","custom","feedIvD","feedIvN","hiddenCards","mother","del","setUp","rsess","autoSleep"]`
- `mkBaby()` → fresh record: new `id` (`"b"+Date.now()+rand`), `created`, blank `sync{on:false,url:"",pass:""}`, `baby:{dob:now}` (DOB/TOB default to today; user edits), PERBABY defaults.
- `babyStash()` writes live `S[f]` → active record (+ `sync` from `SY`).
- `babyLoad(id, skipStash)` stashes current, copies target record → live `S[f]`, sets `S.activeBaby`, swaps `SY` from `baby.sync`, resets `SY.key/pushSig/err`, `sySaveCfg()`.
- **Persistence v4:** `save()` calls `babyStash()` then persists `{v:4, babies, activeBaby, + globals(who,chat,cfg,lang,grid,feedAuto,chatOn,notifSeen,…)}`. Load: if `d.babies` → load + back-fill `created`/`sync` (active inherits global SY, others blank) + `babyLoad(active)`; else **migrate** the old flat blob into one baby (Iza).

**Safety net (v1.14).** `babyGuard()` runs before every **save, activity log (`addEntryAt`), and sync (`syDoc`)**: ensures `S.babies` is non-empty and `S.activeBaby` resolves to a real record; self-heals a dangling pointer (re-points to `babies[0]`, or recreates one). Every entry is stamped with `bid` (audit tag).

**UI.** Long-press the navbar Baby button → `babySheet` switcher (rows show name, DOB, id; centered "+ Add another baby"). Settings → **Baby Profiles** lists all with delete. **Delete flow:** confirm modal (Yes, Delete / No, Exit) → type-the-exact-name (Delete / Exit) → `babyDelDo` compares to *that* profile's name (a different baby's name is rejected: "Wrong name, Baby Profile not deleted") → removes the record + all its data.

**Guards.** Switching/adding is **blocked while an activity is running** (`anyRunning()`), toast "Activity is running, can't switch profiles now". Remote profile switching pauses/resumes sync automatically (per-baby endpoint swap).

**Never-mix invariant.** Two things guarantee isolation: (1) separate records swapped cleanly; (2) **per-baby endpoints** — never point two babies at the same URL. The removed bid guard is *not* the isolation mechanism anymore.

## 14. Feed prediction (recency-weighted, v1.11)

`feedModel()` learns day/night intervals as a **recency-weighted median** of feed-start gaps over a rolling **7 days** (weight `exp(-age/2d)`), so cluster-feeding shortens the interval and it lengthens as gaps settle. `feedAdjust()` still applies per-feed factors (intake-saturation feed-length curve, sleep, nappy output, burp, growth spurt, vomit hold). Prediction is **derived live** on every render/log/tick — no stored state. Auto Mode ignores manual overrides; **Force Recalculate** (v1.12) reseeds the feed-screen intervals and toasts whether timings changed. Manual override `S.feedIvD/N` gated on `!S.feedAuto`.


*Keep this file in sync with the code. If you change the build pipeline, sync format,
version markers, or a core subsystem, update the relevant section here.*
