# Pechanga Mystery Shopper

A single-page web app that Pechanga Resort Casino uses to record, score and review anonymous "mystery shopper" visits to its restaurants, food court outlets and coffee houses. Shoppers fill in a Yes / No / N/A checklist after a visit; the app scores it live, saves it, and gives managers and directors history, trends, coaching insights and printable reports.

This file is the handover document for developers. It covers what the app does, how it is built, how scoring works, and what to watch out for. **Read [Known issues and recommendations](#known-issues-and-recommendations) first.**

## Contents

- [What the app does](#what-the-app-does)
- [Quick start](#quick-start)
- [Architecture](#architecture)
- [Roles and access](#roles-and-access)
- [Outlets and standards](#outlets-and-standards)
- [How scoring works](#how-scoring-works)
- [Data model](#data-model)
- [Code map](#code-map)
- [Common changes](#common-changes)
- [Legacy data](#legacy-data)
- [Testing](#testing)
- [Known issues and recommendations](#known-issues-and-recommendations)
- [Related material](#related-material)

## What the app does

1. **New Evaluation.** A shopper picks an outlet, enters visit details, and answers the checklist for that outlet. A live score bar shows the percentage, the follow-up band and the rating as they go. Saving writes the evaluation to Firestore.
2. **History and Trends.** A table of past evaluations with filters (outlet, band, date range), headline stats, a score trend chart, average by outlet, and an outcome chart. Clicking a row opens the full report, where a manager can mark it reviewed and signed and add action-plan notes. Each report downloads as a PDF.
3. **Soft Skills Insights.** Every question carries a soft-skill tag (for example Guest Engagement, Time Management). This view rolls answers up by skill across outlets and over time.
4. **Restaurant Insight.** Role roll-ups (host, server, bartender, management and so on) for the full-service outlets, as a radar chart, ranking and trend.
5. **Team Member Insight.** Names typed into "Team Member(s) Observed" and "Manager on Duty" are rolled up into a per-person scorecard, with tools to merge name variants (aliases).

## Quick start

There is no build step, package manager or server code. The whole app is one file, `index.html`.

```bash
git clone https://github.com/Arlo-B213/Secret-Shopper.git
cd Secret-Shopper
# open index.html in a browser, or serve it:
python3 -m http.server 8000      # then visit http://localhost:8000
```

- The page loads Firebase, Chart.js, jsPDF and Google Fonts from CDNs, so it needs internet access.
- It signs in to Firebase anonymously and reads and writes the Firestore collection `mysteryShopperEvals`. Running locally therefore talks to the **real project data** (see [Known issues](#known-issues-and-recommendations)). Use a separate Firebase project for development if you can.
- Deployment is not defined in this repo. The file is static and can be hosted anywhere (for example GitHub Pages, Firebase Hosting or an internal web server). Ask the owner where the live copy is served from.

## Architecture

| Piece | Details |
| --- | --- |
| App | One HTML file with inline CSS and JavaScript, about 3,800 lines. No framework, no modules, no bundler. Global functions and global state. |
| Data store | Firebase Firestore, project `soft-skills-f6ae0`, collection `mysteryShopperEvals`. The client config is in `index.html` (`firebaseConfig`). |
| Auth | Firebase Anonymous Auth is used only to get a Firestore session. Real access control is a passcode screen in the browser (see [Roles and access](#roles-and-access)). |
| Libraries | Firebase 9.23 (compat build), Chart.js 4.4 with the date-fns adapter, jsPDF 2.5.1, Google Fonts (Fraunces, Inter). All loaded by `<script>` / `@import` from CDNs. |
| Data flow | On load the app subscribes to the whole collection with `onSnapshot`, caches it in the global `allEvals`, and renders every view from that array. Filtering, charts and insights are all computed in the browser. |

State lives in a few globals: `currentRole`, `currentOutletScope`, `currentStandard`, `currentState` (the in-progress answers, keyed `sectionKey-index`), and `allEvals`.

## Roles and access

The gate screen offers three roles. Passcodes are constants in `index.html` (`SHOPPER_PASSCODE`, `DIRECTOR_PASSCODE`, `OUTLET_PASSCODES`).

| Role | Can do |
| --- | --- |
| Mystery Shopper | New Evaluation only. |
| Outlet Manager | Picks an outlet (or "Food Court, All Outlets") and sees History and the insight tabs for that outlet only. Cannot create evaluations. |
| Director | Everything, across all outlets. |

Role restrictions only hide tabs and lock filters in the UI (`applyRolePermissions`, `lockOutletFilters`). They do not limit what data the browser downloads. See the security notes below.

## Outlets and standards

`OUTLET_STANDARD` maps each outlet to a standard. A standard decides which questions appear, how they are scored, and which rating is shown. Every question is **Yes / No / N/A**.

| Standard key | Outlets | Questions | Rating |
| --- | --- | --- | --- |
| `fine_dining` | Pisano's Italiano | Fine dining checklist plus the AAA sections | AAA Diamond tier, target Four |
| `great_oak` | Great Oak Steakhouse | Fine dining plus Bar Service, plus AAA sections | AAA Diamond tier, target Four |
| `umi_sushi` | Umi Sushi | Fine dining plus Sushi Bar and Bar Service, plus AAA sections | AAA Diamond tier, target Four |
| `asian_dining` | Bamboo, Blazing Noodle | Casual checklist plus Service Style, plus AAA sections | AAA Diamond tier, target Three |
| `casual_dining`, `cantina`, `kelseys` | Journey's End, 1882 Cantina, Kelsey's | Casual checklist (Cantina adds a bar module, Kelsey's a sports bar module). No AAA sections. | Diamond rating on the absolute scale, no target cap |
| `food_court` | Pronto, Agave, American Classics, Pechanga Fried Chicken, Little Wok, Portable Smart Bar | Food Court standard (Smart Bar adds a bar module) | 1 to 5 Stars |
| `coffee_bar` | Café Cocoa's, Blends | Coffee Bar standard with product modules | Roast level (Reserve, Dark, Medium, Light, Below Standard) |
| `quick_service` | none (legacy) | Original quick-service template, kept for old records and as the fallback for unknown outlets | SOP action tiers |

The question banks are JavaScript constants: `FINE_DINING_SECTIONS`, `CASUAL_DINING_SECTIONS`, the `*_EXTRA_SECTIONS` modules, `AAA_REQ_SECTION` and `AAA_PILLAR_SECTIONS`, `COUNTER_STANDARDS`, and the legacy `SECTION_TEMPLATE`. Each section has a `key`, a `name` and `questions`; each question has `t` (text) and `skill` (soft-skill tag).

## How scoring works

Answers are percent-of-Yes (N/A is excluded). Every score is computed live in `recalcLiveScore` and again when saving in `submitEvaluation`.

### Overall percent and follow-up band

All outlets get an overall percent. Counter-service outlets (`food_court`, `coffee_bar`, and legacy `quick_service`) also get an SOP action band from `actionForScore`: 100% celebrate, 92% and above acceptable (log and sign), 85% and above verbal warning, below that written warning. Dining outlets are banded by Diamond tier instead (`diamondBand`).

### AAA Diamond (dining outlets)

Based on the AAA Restaurant Requirements and Diamond Guidelines.

- Pillar weights: **Food 50%, Service 35%, Vibe 15%** (`DIAMOND_WEIGHTS`). Pillars with no answers are dropped and the weights re-normalised.
- **Fine dining (Pisano's), Great Oak, Umi Sushi and Asian dining** carry extra sections (`aaa_req`, `aaa_food`, `aaa_service`, `aaa_vibe`). Each attribute question shows AAA's tier descriptor for the outlet's target tier (`OUTLET_TARGET_TIER`: Four for Great Oak, Pisano's, Umi; Three for Bamboo, Blazing Noodle). The Diamond tier comes only from these AAA sections.
- Banding against the target (`diamondFromItems`): 90% or more keeps the target tier; each step down at 80%, 65% and 50% drops one tier, floor Below Approved. The tier cannot exceed the target.
- `aaa_req` is a checklist of nine program requirements. Any No gives "Requirements Not Met" and no tier.
- **Casual dining** outlets have no AAA sections and no target, so the older legacy path applies: section keys are mapped to pillars by `PILLAR_BY_SECTION`, and the weighted percent maps to a tier on `DIAMOND_TIERS` (Five at 95, Four at 85, Three at 70, Approved at 55). The `overallexp` section is excluded.

The tier cut-offs and the 90/80/65/50 steps are **our own calibration**; AAA publishes no numeric thresholds. They are meant to be tuned.

### Counter-service ratings (Food Court and Coffee Bar)

Defined in `COUNTER_STANDARDS`; computed by `counterRatingFromItems`.

| Standard | Weights | Rating cut-offs |
| --- | --- | --- |
| `food_court` | Food 35%, Guest 30%, Speed 20%, Environment 15% | Stars: 5 at 95, 4 at 88, 3 at 80, 2 at 70, else 1 |
| `coffee_bar` | Guest 35%, Product 30%, Speed 20%, Environment 15% | Reserve 95, Dark 88, Medium 80, Light 70, else Below Standard |

A section whose key ends in `_crit` is the **Critical Items** gate (food safety, cash handling, ID, allergies, hand washing). Any No there forces the lowest rating. Coffee Bar product questions are modules (coffee, blended, cold, pastry, hot food) that the shopper marks N/A when not ordered.

### Wait times

For counter outlets the form records three times (joined line, paid, received) and shows the elapsed minutes. The wait standards themselves are written into the question text and answered by the shopper; the app does not auto-score them. Standards used: Food Court line 5 min (10 at rush), pre-assembled 5, made-to-order 12, total 15 (20 at rush); Coffee brewed 30 seconds, other drinks 4 minutes from payment, craft brew 6, hot food 8. These are starting benchmarks, to be tuned against point-of-sale data. "Rush" is judged by the shopper; there is no clock-based peak window.

## Data model

One document per evaluation in `mysteryShopperEvals`. One extra document (`_staffAliasConfig`, with `__type: "staffAliasConfig"`) stores team-member name merges and must be skipped when reading evaluations.

| Field | Notes |
| --- | --- |
| `standard` | Standard key from the table above. |
| `scoreType` | `"binary"` for current records. Absent on old 1 to 5 records (see Legacy data). |
| `outlet`, `date`, `day`, `timeIn`, `timeOut` | Visit details. |
| `shopperName`, `teamMembers`, `managerOnDuty` | Free text. `teamMembers` is comma separated. |
| `areaFocus` | Great Oak and Umi only (Full Visit, Dining Room, Sushi Bar, Bar). |
| `points`, `max`, `percent` | Raw Yes count, answered count, percent. |
| `actionKey` | Stored band key: `celebrate`, `acceptable`, `verbal`, `written`, or a `dia_*` Diamond key. |
| `sections[]` | `{ key, name, role, points, max, comment, questions[] }`, each question `{ text, skill, answer, note }` with `answer` of `"yes"`, `"no"` or `"na"`. |
| `diamond` | Dining outlets: `{ food, service, vibe, weighted, tier, label, target? }`. |
| `targetTier` | Dining outlets with AAA sections. |
| `rating` | Counter outlets: `{ scale, pillars, weighted, tier, label, critFailed }`. |
| `roleRollups[]`, `overallAvg` | Dining outlets: per-role Yes rate stored on a 0 to 5 scale (rate x 5) so insight charts read old and new records alike. |
| `lineTime`, `payTime`, `receivedTime`, `waitInLineMin`, `waitAfterPayMin`, `waitTotalMin` | Counter outlets only. |
| `overallComments`, `actionPlan { signed, signedDate, notes }`, `createdAt` | Review workflow and metadata. |

## Code map

Everything is in `index.html`; search for these names.

| Area | Where to look |
| --- | --- |
| Firebase and passcodes | `firebaseConfig`, `OUTLET_PASSCODES`, `SHOPPER_PASSCODE`, `DIRECTOR_PASSCODE`, `COLLECTION` |
| Question banks | `FINE_DINING_SECTIONS`, `CASUAL_DINING_SECTIONS`, `*_EXTRA_SECTIONS`, `AAA_*`, `COUNTER_STANDARDS`, `SECTION_TEMPLATE` |
| Outlet to standard | `OUTLET_STANDARD`, `STANDARDS`, `OUTLET_TARGET_TIER` |
| Scoring | `diamondFromItems`, `diamondFromState`, `diamondFromRecord`, `counterRatingFromItems`, `actionForScore`, `bandForScore`, `recalcLiveScore` |
| Form | `onOutletChange`, `buildSections`, `buildBinarySectionPanel`, `setAnswer`, `submitEvaluation`, `updateTiming` |
| Loading and re-banding | `loadEvaluations`, `normalizeEval`, `needsSignoff` |
| Views | `showView`, `renderHistory`, `renderSkillsInsights`, `renderRestaurantInsight`, `renderTeamInsight`, `renderTeamMergeTools` |
| Reports | `openModal`, `downloadPDF` (jsPDF) |
| Roles | `tryEnter`, `applyRolePermissions`, `lockOutletFilters` |

## Common changes

- **Add or edit a question.** Edit the relevant section array. Keep the `skill` value from `SKILLS`. Saved evaluations keep their own copy of the question text, so history is unaffected.
- **Add an outlet.** Add it to the gate dropdown and the outlet selects (`f-outlet` and the filter dropdowns), `OUTLET_STANDARD`, `OUTLET_PASSCODES`, and `OUTLET_TARGET_TIER` if it uses the AAA form. Add it to `FOOD_COURT_OUTLETS` if it belongs under the combined Food Court login.
- **Tune ratings.** Diamond cut-offs: `DIAMOND_TIERS` and the step thresholds in `diamondFromItems`. Counter ratings: `COUNTER_TIERS` and each standard's `weights`. SOP bands: `actionForScore`.
- **Change which pillar a section feeds.** `PILLAR_BY_SECTION` (legacy path) and the `pillar` property on counter sections.
- **Change a target tier.** `OUTLET_TARGET_TIER`.

## Legacy data

The app has been through several scoring changes, and old records stay readable.

- Early dining records used a **1 to 5 scale** per question and a pass/fail band at 80%. They have no `scoreType`; `isScaleRecord` detects them and the detail view and PDF show the 1 to 5 answers.
- On load, `normalizeEval` re-bands dining records by Diamond tier **in memory only**, derived from their saved sections. Nothing stored is changed.
- Older dining records still carry the four AAA-administrative requirement answers that were later removed from the checklist, and those answers still count when such a record is re-rated on screen.
- Earlier counter-service records use `standard: "quick_service"` with the original template and show no Stars or roast rating.
- Some outlet names have two spellings in old data ("Cafe Cocoa's" and "Café Cocoa's"); `OUTLET_STANDARD` handles both.

## Testing

There are no automated tests. The scoring was checked with headless-browser scripts (Playwright) that load `index.html` from disk, stub Chart.js, drive `setAnswer` on every question and read the live score pill, then call `submitEvaluation` against a stubbed `db` to inspect the saved record. A future developer should turn these into a real test suite, ideally after extracting the scoring functions from the page so they can be unit tested without a browser.

## Known issues and recommendations

1. **Passcodes are in the source code.** Every passcode (shopper, director, each outlet) is a plain string constant in `index.html`, and this repository is **public**, so they are readable by anyone and remain in git history. Treat them as compromised. Move authentication server-side (Firebase Auth with real users, or a Cloud Function that checks passcodes), then rotate them. Making the repository private does not remove them from history.
2. **Access control is cosmetic.** Role restrictions only hide parts of the UI. The browser subscribes to the whole `mysteryShopperEvals` collection regardless of role, so anyone who gets past (or around) the gate can read every evaluation. Firestore security rules must enforce access; the current rules are not stored in this repo and should be reviewed and committed.
3. **Anonymous Firebase Auth.** Anonymous sign-in gives every visitor a Firestore session. Combined with the points above, check that the rules do not allow open reads and writes, and consider App Check.
4. **Single 3,800-line file.** No modules, tests or linting. Scoring logic is entangled with DOM code. Splitting it (data, scoring, UI) would make it far easier to maintain.
5. **No dev/prod separation.** The Firebase project is hard-coded. Add environment config so developers do not write test data to production.
6. **Calibration values are assumptions.** Diamond cut-offs and steps, counter-service weights and cut-offs, and wait-time standards were chosen by the product owner and Claude, not taken from AAA or point-of-sale data. Expect to tune them.
7. **Rush hours are manual.** Peak versus off-peak wait standards rely on the shopper's judgement. A clock-based window per outlet would be more consistent.
8. **Long forms.** The Great Oak form has about 140 questions. Consider hiding the older operational sections for AAA outlets, since the Diamond tier uses only the AAA sections.
9. **Export.** There is no CSV or bulk export; only per-evaluation PDFs.

## Related material

Four printable manager guides (fine dining, casual dining, Food Court, coffee house) explain what the shopper checks and how managers can prepare for each visit. They were written alongside this app and mirror its current scoring settings; update them if the standards above change. Ask the product owner for the PDFs.
