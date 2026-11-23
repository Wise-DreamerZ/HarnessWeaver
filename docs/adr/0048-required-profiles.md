# ADR-0048: A platform can require profiles at load

**Status:** Proposed. It would close the gap [ADR-0043](0043-profile-ownership.md) leaves open: a baseline isn't mandatory.

## Context

An org baseline's locks, bounds and `require_level` bind only a spec that names it in `extends` (DESIGNS No. 11). A spec that names nothing loads with `profile://defaults` alone. A `risk_tier: takes_actions` spec with no `extends` passes `harnessweaver spec validate` at L0, required L0, while the baseline sits unused on `HARNESSWEAVER_PROFILE_PATH`. `require_level` can't catch it, because it comes from the profile that is missing. Today the platform team closes the gap by review and CI (*Write an org baseline*, step 8): a `CODEOWNERS` entry on `harness.yaml`, and a CI step that greps the `layers:` line of `spec validate`.

**Who this binds.** R1 and design goal 1 constrain what the app's team writes as configuration: the spec, its overlays and, since ADR-0043, any file in the app's repository. The app's code runs with the app's authority. It can call a model without HarnessWeaver, or build a spec in code with resolvers of its own, and no check inside its process binds it. A platform that needs every model call governed, whatever the code does, runs a gateway that every call must pass. So the question here is narrower: can the platform make "this app extends the baseline" hold at load, so that nothing the app's team writes as configuration opts out of it?

**Where the setting lives decides that.** It must be somewhere the platform writes and the app's team doesn't. One channel already works that way: `HARNESSWEAVER_PROFILE_PATH`. ADR-0043 counts its directories as the platform's, and whoever sets it chooses which `acme-baseline` an app finds. If the app's team sets the runtime environment, it can point the path at a copy of the baseline with no locks and pass any check of names, today's grep included. A requirement in that same channel adds no trust assumption the baseline doesn't already make. A requirement anywhere the app's team writes (the spec, its repository, its own CI file, its own deploy files) is no boundary.

## Options

| | Where the setting lives | When it binds | Contract impact (R11) | Fails closed (R4) |
| --- | --- | --- | --- | --- |
| A. `HARNESSWEAVER_REQUIRED_PROFILES`, read by the loader | The runtime environment, beside `HARNESSWEAVER_PROFILE_PATH` | Every load in a process that has it set | Python and TypeScript loaders, one new fixture suite; harnessd inherits Python's; no schema change | Yes, on every path that loads a spec file |
| B. `--require-profile` on `spec validate` and `doctor` | A CI job | At merge, if the job runs | The Python CLI only | Only for what CI loads |
| C. A `doctor` finding for a spec that extends no org or plugin profile | Nowhere: no setting | When someone runs `doctor` | None | No: a warning, failing only with `--strict` |
| D. A profile declares itself required | The profile file on the path | Every load that searches its directory | A new `Profile` key: schemas, fixtures, both loaders | Yes, for `org://` profiles only |

### A. An environment variable the loader reads

The platform sets `HARNESSWEAVER_REQUIRED_PROFILES=org://acme-baseline` on the runtime, and a spec that doesn't extend that profile fails to load. The details are in the decision below.

- **Where it lives.** It's a boundary only where the platform sets the environment and the app's team can't change it: an admission policy that injects it into every workload in the platform's clusters, a deployment template or runtime configuration the platform owns, a required CI workflow the platform owns. It's no boundary when it's set in a file the app's team writes: its Dockerfile, Helm values, compose file, `.env` or CI file, or a base image's `ENV`, which the app's own Dockerfile can reset. That is exactly where `HARNESSWEAVER_PROFILE_PATH` is or isn't a boundary, so the two hold or fail together.
- **Contract.** No document, schema or wire payload changes. What the loader does with the variable is behavior both languages must share: how the list is read, what counts as a match, and which owners count. That goes in a new conformance suite that Python and TypeScript both run (T4). harnessd loads through Python's `load_spec`, so it gets the check without a route change.
- **V2.** Additive within `harnessweaver/v1`. No document gains or changes a key, and with the variable unset every spec loads as it does today. A new `SpecError` subclass only adds a value of error-info's `type`, under the existing `spec_invalid` code.
- **R4.** A malformed entry, a missing profile, and a profile found only as a copy the app's team owns all fail loading, naming the entry. One gap remains that code can't close: a release older than this change ignores the variable. The platform needs a version floor on the runtimes it sets it on, and `spec validate` and `doctor` print what was required, so a log without that line shows the check didn't run.

### B. A flag on `spec validate` and `doctor`

- **Where it lives.** In a CI job. In the app's own CI file, the app's team can delete it as easily as today's grep; it's a boundary only in a required workflow the platform owns.
- **Contract.** The Python CLI only. TypeScript has no CLI and needs nothing; no fixtures are needed beyond the CLI's tests.
- **V2.** An additive flag.
- **R4.** It checks the spec that CI loads, with CI's profile path, not the one that runs. A runtime with another profile path, another spec file named in code, or a deploy that skips the pipeline isn't covered. In its favor, an older CLI fails on an unknown flag rather than ignoring it.

It's a sturdier spelling of today's grep, not enforcement at load. With A in place, `HARNESSWEAVER_REQUIRED_PROFILES=org://acme-baseline harnessweaver spec validate` is the same check in CI, so B would be a second spelling of it.

### C. A `doctor` finding

- **Where it lives.** Nowhere, which is its appeal: it needs no platform setup.
- **Contract.** None. **V2.** Nothing changes.
- **R4.** It reports the gap and closes none: `doctor` exits 1 on a warning only with `--strict`, and nothing that loads a spec runs `doctor`.

Without a setting it can't know which profile is required. A spec that extends any org profile passes, even one the app's team wrote, while every project made from a template (each extends a built-in profile, such as `profile://chat`) and every app that has no org baseline by design gets a warning. That is noise for most users and no protection for the platform.

### D. A profile that declares itself required

It lives in the same channel as A, and binds as A does. But a new `Profile` key is a schema change in both languages, and the loader would have to list every directory on the path instead of resolving names. A plugin's profile can't use it, because installing a plugin enables nothing (S8). And it ties a deployment rule to the baseline's content, so the same file can't be required in production and optional on a laptop.

## Decision (proposed)

Option A.

- **The variable.** `HARNESSWEAVER_REQUIRED_PROFILES` holds a comma-separated list of profile names, such as `org://acme-baseline,profile://acme.baseline`. Spaces around an entry are ignored; unset or empty, nothing is required. The separator is a comma, not `os.pathsep`, because every entry contains `:`.
- **Names, not ranges.** An entry with `@` fails loading. Ranges can hold commas and spaces (`>=1.2, <2`), and the platform already decides versions: by what it puts in the directory that owns the name, or by the plugin version it installs. The app's own range still picks among those, and `doctor` still warns when it's unpinned.
- **Only org and plugin profiles.** A file path or a built-in profile (`profile://rag`) fails loading: a file path is no platform channel, and a built-in profile is no org's to require.
- **What satisfies it.** A layer of the effective spec with the same scheme and name that the platform owns in ADR-0043's sense: one found in a `--profile-dir`, `profile_dirs=` or `HARNESSWEAVER_PROFILE_PATH` directory, or in a plugin. A profile reached through another profile's `extends` counts, so an app that extends `org://acme-payments`, which extends `org://acme-baseline`, meets a requirement for the latter. A copy in the app's own `.harness/profiles`, or a file the app names, doesn't count; otherwise unsetting the profile path and committing a copy would pass.
- **The failure.** A new `ProfileRequired(SpecError)`, code `spec_invalid`, with `details.missing` listing the names. It's raised as soon as `extends` is resolved, before the merge, so it's reported ahead of any lock, bound or level error. Its message names the missing profile and the variable that requires it and, when only an app-owned copy was found, says that copies in the app's repository don't count.
- **Where it's read.** `load_spec` (`loadSpec` in TypeScript) reads it, beside `HARNESSWEAVER_ENV` and the profile path, and passes it to a new `required_profiles` argument of `spec_from_documents` (`specFromDocuments`), empty by default. `load_spec` takes no argument for it, so `Harness.load`, every CLI command, harnessd and `harness_for_test(path)` all apply it, and nothing that loads a file turns it off. `spec_from_documents` keeps reading no environment: a spec built in code is the app's code choosing its inputs, as with `resolvers=`.
- **Reporting.** When the variable is set, `spec validate` prints a `required:` line (and `required_profiles` with `--format json`), and `doctor` an `ok` finding naming them. Unset, both print what they print today.
- **Docs.** Step 8 of *Write an org baseline* tells the platform to set the variable where the app's team can't, keeps the `CODEOWNERS` entry, and replaces the grep with `spec validate` run with the variable set.

## Why

A is the only option that binds at load, and its setting is exactly as hard for an app's team to change as the baseline itself. B and C leave enforcement where it is today, in a pipeline the app's team may own. D binds as A does, but costs a schema change, can't cover plugin profiles, and fuses deployment policy with the baseline's content. A costs one variable, one check after `extends` is resolved, and one fixture suite, with no change to any document or wire payload.

It also answers the consequence ADR-0043 left open: once the baseline is required, its `require_level` binds every app in that runtime, so a `takes_actions` app can't load at L0.

## Consequences

- Enforcement is as strong as the platform's hold on the runtime environment, and no stronger. The docs must say plainly where the variable is a boundary and where it isn't, as they do for the profile path.
- A runtime on a release older than this change ignores the variable. The platform sets a version floor on the runtimes it sets the variable on.
- Every command that loads a spec applies it, `spec diff` included. In CI, a pull request that drops the baseline then fails on its new side with `ProfileRequired` instead of showing `Layer removed`; either way it fails.
- Exempting an app is the platform's decision, made where the platform is: it leaves the variable off that app's runtime. Nothing in the spec can exempt an app or a risk tier, since an exemption the app writes would undo the requirement.
- Tests stay offline and deterministic (R14): nothing sets the variable by default, and the repository's root `conftest.py` clears it so a developer's shell can't change results.
- Implementing it (R11, D1) touches: the Python and TypeScript loaders and their error classes; a `required_profiles` conformance suite in `contracts/fixtures/conformance/`, whose cases give the variable's text and the layers with their kind and owner and expect `ok` or `{error: "profile_required" | "spec_invalid", missing}`, written by hand and listed in `contracts/README.md`; a harnessd test that it won't start without a required profile; the output of `spec validate` and `doctor`; DESIGNS No. 11, whose layers diagram gains `ProfileRequired`; `docs/how-to/org-baseline.md`, `docs/deployment/configuration.md`, `docs/advanced/layers.md` and `docs/how-to/troubleshooting.md`; and the changelogs of `harnessweaver`, `@harnessweaver/core` and `harnessweaver-cli`, each a minor version.
