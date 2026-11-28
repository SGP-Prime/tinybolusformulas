# TinyBolus — Formulary

Canonical pediatric reference data for the **TinyBolus** mobile app.

The file **`formulary.json`** is served raw to every installed instance of the TinyBolus app. When a correction is pushed here, every phone picks up the new values on next launch.

## Updating doses

1. Edit `formulary.json`.
2. Bump the `version` field (semver, patch bumps for everything in practice — the app compares versions numerically and doesn't attach meaning to the segments; what matters is that every push is strictly higher than the last).
3. Update the `updated` date.
4. **Mirror into the app repo and run its test suite before pushing** — copy the file byte-identical to `tinybolus_app/assets/formulary.json` and run `flutter test` there. That suite is where the mechanical guards live (id integrity, dose-magnitude bounds, note budgets, visibility decisions); this repo deliberately has no CI of its own.
5. Commit + push to `main`.

The app only updates its cache when the fetched `version` is **strictly greater** than the version it is currently running (cached or bundled, whichever won at launch), so forgetting to bump the version means the update is silently ignored. Recovering from a push whose version was accidentally too HIGH requires an app release whose bundled version matches or exceeds it (bundled wins ties and evicts the cache).

## Item identity (schema v2, since v0.0.69)

Every item carries three identity fields alongside its display `name`:

- **`id`** — stable lowercase slug, unique across the whole file. **Immutable forever: never rename one, never reuse one.** Installed apps key each user's row-visibility preferences on it, so it survives renames and section moves. When a row is deleted, move its id into the `$retired_ids` ledger at the bottom of the file — a retired id must never come back for a different row (the app repo's tests enforce disjointness).
- **`drug`** — display line one (the bare drug/equipment name; must equal the `name` prefix before the first `" ("`).
- **`qualifier`** — display line two, without parens and **without the route** — the `route` field is the row's only route surface. Omit it when the parenthetical is nothing but the route (e.g. `"Ketamine (IM)"`).

**Naming.** `name` is the drug plus, in parentheses, only what the row's header does not already say. A route or regimen token stays (`"Ketamine (IM)"`, `"Lipid emulsion 20% (infusion)"`, `"Sugammadex (profound block)"`). An indication appears only when it differs from the section title or group label above the row — the header counts (ruled 25 November 2026, applied in v0.0.85): `"Dexamethasone"` under PONV but `"Dexamethasone (airway oedema)"` under Emergency Drugs › Respiratory; `"Adrenaline (IM)"`, `"(IV)"`, `"(infusion)"` under Emergency Drugs › Anaphylaxis; `"Fentanyl"` under both Induction and Analgesia. The same plain name in two sections is fine (`"Atropine"` has been that way since the first public version); two rows in one section never share a name. Every surface that lists names also prints the header above them — section title and group label — (result sheet, search results and the hidden-rows hint, Settings, the About references list, clipboard export), and the app's test suite enforces the rule mechanically.

For a new row: author `name` as `"Drug (qualifier)"` per the convention above, fill all three fields, and add the id to the app's `lib/defaults.dart` hidden set (new rows ship default-hidden) — or, by deliberate decision, to the pinned visible list in its `defaults_test.dart`. Every mistake in this list fails the app's test suite; an id-less or internally inconsistent payload is additionally rejected on-device (installed apps refuse it and keep their previous data, so the broken push is ignored, not dangerous — but also not delivered).

## Schema

See the `$schema_comment` field at the top of the JSON for the concise type reference. Canonical grammar:

| Row `type` | Requires | Meaning |
|---|---|---|
| `linear` | `factor`, `unit` | dose = factor × weight |
| `linear_range` | `low`, `high`, `unit` | dose shown as `low·w – high·w` |
| `linear_max` | `factor`, `max`, `unit` | `min(factor × weight, max)` with cap annotation |
| `fixed` | `value` | static string, no calculation |
| `fixed_age` | `brackets` (`max_months`/`max_years` + `value`) | static string chosen by age bracket (first match wins; terminal bracket `max_years: 999`) |
| `linear_age` | `brackets` (`max_months`/`max_years` + `low`, `high`) | per-kg dose whose factors are chosen by age bracket; optional per-bracket `max`/`note` |
| `linear_weight` | `brackets` (`max_kg` + `low`, `high`) | per-kg dose whose factors are chosen by body-weight band (terminal `max_kg: 9999`) |
| `fixed_weight` | `brackets` (`max_kg` + `value`) | static string chosen by body-weight band (first match wins; terminal `max_kg: 9999`) |
| `steps` | `steps` (list of `label`, `factor`, optional `max`/`max_unit`) + `unit` | one drug given as a dose SEQUENCE (adenosine 1st/2nd, amiodarone after the 3rd/5th shock); each step is weight-scaled and capped on its own; the app prints one ordinal-labelled line per step in the value column (`1st: 2 mg` / `2nd: 6 mg`) and `label: factor unit/kg (max …)` per step on the formula line |
| `computed` | `computer` | dispatches to a named function in the app's `lib/calculators/` |

Optional on every row type: `note` (always visible) and `formula_note` (v0.0.87, shown only while the app's *Show formulas* toggle is on). The split (ruled 26 November 2026): `note` carries only what makes the number safe — a condition that changes the number, a safety qualifier (companion drug, 'not routine in arrest', a cumulative ceiling, what the number *is*), the concentration or strength that gives a volume/rate/puff value its meaning, and the four administration speeds where a rapid push is itself the hazard (magnesium, calcium, tranexamic acid, IV salbutamol). `formula_note` carries the formulaic rest — derivations and equivalences, concentrations and preparations, other speeds, intervals, repeats, escalation arithmetic, follow-on regimens, monitoring, titration advice. Neither may contain indication prose, positioning, drug-class reminders, source tags or eponyms as attribution; both are limited to 90 characters (test-enforced). Binaries older than v0.0.87 ignore `formula_note`.

`group` (optional, v0.0.82) is a short sub-heading inside a section — Emergency Drugs carries Anaphylaxis, Respiratory, Cardiovascular, Metabolic, Seizures (labelled Status epilepticus until v0.0.89), Malignant hyperthermia and Intracranial pressure, in that order (airway and breathing after the multi-system block, then circulation). Rows sharing a group are adjacent; the label is presentational only — every row keeps its own id, visibility switch and citations. v0.0.88 folded the former Respiratory section into Emergency Drugs this way; its row ids did not change.

The ETT row is the one exception to item-level notes: its size comes from one of three paths (neonatal weight table, infant age table, child formula), so its notes live per path in `ett_rules.notes` (`neonatal_table` / `infant_age_table` / `child_formula`, each with optional `note` and `formula_note`) and the app shows the pair for the path that produced the value — 'uncuffed' for a neonate on the weight table, 'cuffed' otherwise.

A note built from age clauses is authored as `note_by_age` instead of one string: ordinary age brackets (`max_months` / `max_years`, first match wins, terminal `max_years: 999`) carrying only `note` and/or `formula_note`; the bracket matching the patient replaces the item-level field (an empty string clears it; a bracket with neither field keeps the item's text). The app then prints only the clause for the age at hand — the nasal ETT adjustment, the adrenaline auto-injector equivalent, the neonatal maintenance and tidal-volume caveats.

`note_by_weight` (v0.0.89) is the same device on the kilogram axis — brackets bounded by `max_kg` (first bracket whose `max_kg` is at or above the patient's weight wins, terminal `max_kg: 999`) — for a clause that is a fact about the computed number rather than the age: the defibrillation ceiling 'max 120–200 J (adult energy)' prints only above 30 kg, where 4 J/kg passes 120 J, and never on a newborn. The clause also stays at item level, cleared by the ≤30 kg bracket, so a binary older than v0.0.89 (which ignores the field) prints it for every patient, as v0.0.88 did, instead of losing it.

Inside `note` and `formula_note` the only clause separator is ` · ` (v0.0.90). From v0.0.89 the app breaks an annotation line that would wrap at those separators, one clause per line, so the separator is a layout promise rather than punctuation; dashes and semicolons are not used between clauses.

Valid `computer` identifiers are defined in the app code. Changing them requires an app-store release, so stick to the existing set when editing here.

## Safety

This repository is the live source of truth for a medical reference tool. Treat every edit as you would a chart-wide dose correction.

- Double-check units (`µg` vs `mg`, `mL` vs `mL/h`).
- Prefer adding a new row with a clinically-precise name over modifying an existing one in-place when the clinical context changes.
- Only the repo owner (or collaborators explicitly added on GitHub) can push. All edits are version-controlled.
