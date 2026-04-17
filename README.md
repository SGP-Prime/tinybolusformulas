# TinyBolus — Formulary

Canonical pediatric reference data for the [TinyBolus](https://github.com/SGP-Prime/tinybolus) mobile app.

The file **`formulary.json`** is served raw to every installed instance of the TinyBolus app. When a correction is pushed here, every phone picks up the new values on next launch.

## Updating doses

1. Edit `formulary.json`.
2. Bump the `version` field (semver — `1.2.3` → `1.2.4` for dose fixes, `1.3.0` for new sections, `2.0.0` for schema changes).
3. Update the `updated` date.
4. Commit + push to `main`. That's it.

The app only updates its cache when the fetched `version` is **strictly greater** than the bundled one, so forgetting to bump the version means the update is silently ignored.

## Schema

See the `$schema_comment` field at the top of the JSON for the concise type reference. Canonical grammar:

| Row `type` | Requires | Meaning |
|---|---|---|
| `linear` | `factor`, `unit` | dose = factor × weight |
| `linear_range` | `low`, `high`, `unit` | dose shown as `low·w – high·w` |
| `linear_max` | `factor`, `max`, `unit` | `min(factor × weight, max)` with cap annotation |
| `fixed` | `value` | static string, no calculation |
| `computed` | `computer` | dispatches to a named function in the app's `lib/calculators/` |

Valid `computer` identifiers are defined in the app code. Changing them requires an app-store release, so stick to the existing set when editing here.

## Safety

This repository is the live source of truth for a medical reference tool. Treat every edit as you would a chart-wide dose correction.

- Double-check units (`µg` vs `mg`, `mL` vs `mL/h`).
- Prefer adding a new row with a clinically-precise name over modifying an existing one in-place when the clinical context changes.
- Only the repo owner (or collaborators explicitly added on GitHub) can push. All edits are version-controlled.
