# Field Visit demo — change log

Mobile field-visit app for the ArcelorMittal Long SIC "Voice Visit Report" use case.

- **Live:** https://pwillemot.github.io/am-field-visit-demo/
- **Repo:** https://github.com/pwillemot/am-field-visit-demo (public)
- Self-contained: `index.html` + `assets/`.

## 1. Standalone site created
Copied from `InspirationSeptember/webdemos/fieldvisit/` into `Long SIC/fieldvisit/`
(index.html + assets, dropped `.sf/`). Published as its own GitHub repo / Pages URL,
separate from the other demos (`sic-feedback`, `sf-quiz-game`, `arcelormittal-demos`, …).

## 2. Feature — "ABM Project Types presented" + "Current Trend" popup
Hooked into the voice-report flow: **record → review → Save note** opens a bottom-sheet
(`#vrProjSheet`, built with the app's own `mfg-modal-sheet` + `openFill`/`closeFill`).

- **Step 1 – Project types:** 🤖 agent prompt, "Tap to speak" mic, and the full 9-item
  SSP checklist: Underground car parks & basement · Road & Railway infrastructure ·
  Dykes, dams and flood protection · Environmental protection · Maritime ports ·
  Inland waterways · Temporary works · Stock orders · Not specified / Others.
- **Step 2 – Current trend:** each selected type gets the 5-value scale
  (Decrease · No increase · Low increase · Steady increase · High increase).
  "Save to report" is disabled until every selected type has a trend.
- On save: selections are written into the **Previous Notes list** (`_notePrepend`) and
  injected as an **"ABM Project Types Presented" card in the completed-visit summary**
  (`#completionProjTypesCard`, above Notes) with colour-coded trend badges.
- Original follow-up-task creation (`c360AddTaskFromNote`) now runs after this step.

Key code: `vrOpenProjSheet` / `vrRenderStep` / `vrToggleType` / `vrSimVoice` /
`vrGoTrends` / `vrSetTrend` / `vrFinishProj` / `vrInjectSummary`, plus `VR_PROJECT_TYPES`
and `VR_TRENDS`. Hooked from `noteEditSave()`.

## 3. Voice-capture tuning
- **Listening window = 5 seconds** before transcription + chip selection (was ~2.2s).
- **Removed the on-screen transcript quote** — it just listens → "Transcribing…" →
  ticks the two matching chips (Underground car parks & basement, Maritime ports).
- Voice is simulated (demo-safe); matched chips set by `VR_SPOKEN_MATCH`.

## 4. Home page
- **Galère moved to Completed** (`GALERE_COMPLETED_CARD`); only **Meridian Infrastructure**
  and **BESIX Group** remain Upcoming.
- **Dynamic meeting times** (`homeDynamicTimes()` in `renderHomeDay`), computed live:
  - Meeting 1 (Meridian) = next full hour, 30 min.
  - Meeting 2 (BESIX) = +2 hours, 1 hour.
  - e.g. at 8 PM → 9:00–9:30 PM and 11:00 PM–12:00 AM; handles AM/PM + midnight rollover.
  - Recomputed on load / "Today" pill — reload before the demo if the clock has moved.
- **KPI card = `1/3` Visits completed** (static; does not auto-tick when completing a visit).

## 5. Related cleanup (outside this repo)
Reverted the "Generate structured report" button + Visit Report Agent code that had been
added by mistake to `Long SIC/SIC-usecases-site/index.html`. That page's UC03 is back to
its original "▶ Play visit report" talk track. (That folder is not a git repo — local only.)

## 6. Real Salesforce write on save
Completing a visit report now also creates a **real `LPE_Visit_Report__c`** record in the
`arcelormittal` org (`arcelor-mittal-ne9v6w`), in addition to the in-page simulation.

- **App:** `vrSyncToSalesforce()` (called from `vrFinishProj`) does a **fire-and-forget,
  fully guarded** `fetch` POST to a public endpoint. Any failure (offline / CORS / endpoint
  down) is silently swallowed — the demo UI behaves exactly as before. The write is fully
  **silent** (no toast/popup — the earlier "Synced to Salesforce" toast was removed because
  its close button was unreachable on the completion screen and the confirmation wasn't
  needed). Endpoint in `VR_SF_ENDPOINT`.
- **Payload:** `{name, notes, status:'Completed', projectTypes:[{type,trend}]}` — the 9 SSP
  project-type labels + 5 trend values map 1:1 onto the object's `SSP_*__c` booleans and
  `SSP_*_Trend__c` picklists.
- **Org side (in `Long SIC/sfdx/`):**
  - `VisitReportIntake` global `@RestResource` (`/services/apexrest/visitreport`), inserts in
    `AccessLevel.SYSTEM_MODE`. `Name` is Auto Number so it's never set; any title is prefixed
    into `Notes__c`.
  - Classic Force.com Site **`VisitIntake`** (`/visitintake`) exposes it unauthenticated.
  - Permission set **`Visit_Report_Intake_Guest`** grants the Site guest user Apex + Create.
  - `CorsWhitelistOrigin` **`pwillemot_github_io`** allowlists `https://pwillemot.github.io`.
- Open endpoint (no auth / no secret) by design — fine for this short-lived demo org.

### 6a. Linked to the Meridian opportunity + account
The created report is now linked to the **"Meridian - Kirchberg Underground Car Park"**
opportunity (`VR_SF_OPPORTUNITY_ID` = `006Kj000017zTmuIAE`) — the object has no direct
Account lookup, so Account is reached through `Opportunity__c`. The report shows on that
opp's related list (and thus under Meridian Infrastructure). Payload also sends
`typeOfVisit:'On Site'`. `VisitReportIntake` gained optional `opportunityId`, `typeOfVisit`,
and `purpose` handling. (Only fields the app genuinely has are sent — products/satisfaction
are left blank rather than fabricated.)

### 6b. Local testing + cleanup
Tested locally before publishing by serving the folder at `http://localhost:8000` and
temporarily adding that origin to the org CORS allowlist. After verifying, the
`localhost_8000` CORS entry was **removed** from both the org and `Long SIC/sfdx/` — only
`https://pwillemot.github.io` remains trusted. The local test server was stopped.

**Published** to `master` (GitHub Pages) — live at the URL above.
