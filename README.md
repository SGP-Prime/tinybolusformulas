# TinyBolus — Formulary

Canonical pediatric reference data for the **TinyBolus** mobile app.

The file **`formulary.json`** is served raw to every installed instance of the TinyBolus app. When a correction is pushed here, every phone picks up the new values on next launch.

## Updating doses

1. Edit `formulary.json`.
2. Bump the `version` field (semver — `1.2.3` → `1.2.4` for dose fixes, `1.3.0` for new sections, `2.0.0` for schema changes).
3. Update the `updated` date.
4. **Mirror into the app repo and run its test suite before pushing** — copy the file byte-identical to `tinybolus_app/assets/formulary.json` and run `flutter test` there. That suite is where the mechanical guards live (id integrity, dose-magnitude bounds, note budgets, visibility decisions); this repo deliberately has no CI of its own.
5. Commit + push to `main`.

The app only updates its cache when the fetched `version` is **strictly greater** than the bundled one, so forgetting to bump the version means the update is silently ignored.

## Item identity (schema v2, since v0.0.69)

Every item carries three identity fields alongside its display `name`:

- **`id`** — stable lowercase slug, unique across the whole file. **Immutable forever: never rename one, never reuse one.** Installed apps key each user's row-visibility preferences on it, so it survives renames and section moves. When a row is deleted, move its id into the `$retired_ids` ledger at the bottom of the file — a retired id must never come back for a different row (the app repo's tests enforce disjointness).
- **`drug`** — display line one (the bare drug/equipment name; must equal the `name` prefix before the first `" ("`).
- **`qualifier`** — display line two, without parens and **without the route** — the `route` field is the row's only route surface. Omit it when the parenthetical is nothing but the route (e.g. `"Ketamine (IM)"`).

For a new row: author `name` as `"Drug (qualifier)"` per the existing convention, fill all three fields, and add the id to the app's `lib/defaults.dart` hidden set (new rows ship default-hidden) — or, by deliberate decision, to the pinned visible list in its `defaults_test.dart`. Every mistake in this list fails the app's test suite; an id-less or internally inconsistent payload is additionally rejected on-device (installed apps refuse it and keep their previous data, so the broken push is ignored, not dangerous — but also not delivered).

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
| `computed` | `computer` | dispatches to a named function in the app's `lib/calculators/` |

Valid `computer` identifiers are defined in the app code. Changing them requires an app-store release, so stick to the existing set when editing here.

## Safety

This repository is the live source of truth for a medical reference tool. Treat every edit as you would a chart-wide dose correction.

- Double-check units (`µg` vs `mg`, `mL` vs `mL/h`).
- Prefer adding a new row with a clinically-precise name over modifying an existing one in-place when the clinical context changes.
- Only the repo owner (or collaborators explicitly added on GitHub) can push. All edits are version-controlled.
