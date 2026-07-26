# Renovate profiles

Shared Renovate config for my repos. `default.json` is the baseline; the rest are
opt-in profiles you compose on top of it.

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>MihaiBojin/renovate"]
}
```

## Profiles

| Profile | Reference | What it does |
| --- | --- | --- |
| baseline | `github>MihaiBojin/renovate` | `config:recommended`, no lock file maintenance, plus `release-age` and `security-alerts` |
| release age | `github>MihaiBojin/renovate:release-age` | Holds an update until its release is 3 days old, and hides updates that have not aged yet |
| security alerts | `github>MihaiBojin/renovate:security-alerts` | Vulnerability fixes bypass the other filters; unresolved OSV advisories stay on the dashboard |
| grouping | `github>MihaiBojin/renovate:group-non-major` | Collects every non-major update into one PR |
| automerge | `github>MihaiBojin/renovate:automerge-non-major` | Merges non-major updates once CI is green |
| mise | `github>MihaiBojin/renovate:mise-manual` | Keeps mise tool pins out of automerge |
| astro | `github>MihaiBojin/renovate:astro` | Groups Astro, its integrations and Starlight plugins into one PR |
| psycopg | `github>MihaiBojin/renovate:group-psycopg` | Groups psycopg, psycopg-binary and psycopg_pool into one PR |
| GitHub Actions | `github>MihaiBojin/renovate:group-github-actions` | Groups non-major updates to actions used in workflows into one PR |
| Python tooling | `github>MihaiBojin/renovate:group-python-tooling` | Groups non-major updates to build/pip/pre-commit/pytest/toml/twine/wheel/wheel-inspect into one PR |

The baseline already pulls in `release-age` and `security-alerts`, so you only
reference those directly if you are composing a baseline of your own.

## Composing

Batch updates into one PR and merge it once the checks pass, weekly:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "github>MihaiBojin/renovate",
    "github>MihaiBojin/renovate:group-non-major",
    "github>MihaiBojin/renovate:automerge-non-major",
    "github>MihaiBojin/renovate:mise-manual",
    "schedule:earlyMondays"
  ]
}
```

Grouping and automerge are separate on purpose. Adopt `group-non-major` on its
own to go from one PR per dependency to one PR per batch, then add
`automerge-non-major` once you trust the CI gate.

Add `schedule:*` when you want batching to mean something. Without a schedule
Renovate opens the group PR as soon as the first update is ready, so there is
little for the batch to collect.

## Order matters

Later entries in `extends` win where they conflict, and `packageRules` from
every profile are concatenated rather than replaced. Two consequences:

- `mise-manual` has to come after `automerge-non-major`. The automerge rule
  matches non-major mise updates too, so reversing them quietly turns automerge
  back on for tool pins.
- `astro`, `group-psycopg`, `group-github-actions` and `group-python-tooling`
  all have to come after `group-non-major` if you compose them together. The
  catch-all otherwise claims those non-major updates for the general batch
  first, splitting peer-locked packages from the others they need to travel
  with.
- Anything repo-specific belongs in the repo's own `packageRules`, which are
  applied after everything pulled in through `extends`.

## Pinning

Consumers track this repo's default branch, so a change here reaches every repo
on Renovate's next run. That is usually what you want. When it is not, pin a tag
and upgrade deliberately:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>MihaiBojin/renovate#v1"]
}
```

`v1` moves forward as profiles change in compatible ways, the way action tags
do. A change that alters what an existing profile does to a repo gets `v2`
instead, so pinned repos stay put until they opt in.

## Why automerge is safe without branch protection

None of these repos protect their default branch — mihaibojin/website cannot on
a Free plan, and the others just do not. That matters because two different
things can do the merging:

- **GitHub native auto-merge** waits only for *required* status checks. With no
  branch protection there are none, so it merges immediately. `automerge-non-major`
  sets `platformAutomerge: false` to keep this switched off.
- **Renovate's own automerge** waits for the checks it observes on the branch.
  `ignoreTests` defaults to `false`, and internal checks such as
  `minimumReleaseAge` are deliberately not counted toward branch success, so a
  repo with no real CI does not start merging on internal checks alone.

Majors are never automerged. They are the updates that need a human, and they
are also where grouping earns its keep: peer-locked packages only pass CI when
they move together, which is what the `astro` profile is for.

## Validating changes

```sh
npx --yes --package renovate@latest -- renovate-config-validator *.json
```

Pin `@latest`. An older Renovate validates against an older schema and will miss
migrations the hosted app has already applied.
