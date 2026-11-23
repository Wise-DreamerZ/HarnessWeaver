# Write an org baseline

An org baseline is the profile your platform or security team publishes for every app to extend. Its locks and bounds make sure no app that extends it removes or loosens a required control, and [step 8](#8-make-sure-every-app-extends-it) makes sure every app does. [Profiles and the org baseline](../tutorial/profiles.md) in the tutorial shows an app's side. This recipe is for the team that writes the baseline.

## 1. Start from a complete one

To write one from choices in a form, use the [Spec generator](../learn/spec-generator.md) and pick *Profile*. [`examples/_org/acme-baseline.yaml`](https://github.com/Wise-DreamerZ/HarnessWeaver/blob/main/examples/_org/acme-baseline.yaml) is the baseline the repository's examples extend, and its tests run in CI:

```yaml title="acme-baseline.yaml"
--8<-- "examples/_org/acme-baseline.yaml"
```

## 2. Decide what to lock

**Lock what must never change**, and leave the rest open. A baseline that locks everything gets forked.

| Usually locked | Why |
| --- | --- |
| `plugins.allow` | Apps may only use plugins the org approved (RULES S8) |
| `observability.capture_content` | No prompts or outputs in telemetry (R9) |
| `guardrails.input[base-pii]`, `guardrails.output[base-secrets]` | The baseline's own guards, by id, so apps can still add their own |
| `tools.require_manifest`, `tools.approval_by_risk` | No tool without a manifest (S2), and high-risk calls need a person |

A lock on a keyed-list item (`path[id]`) forbids changing that item, including `enabled: false`, but lets apps add items after it. A wildcard (`guardrails.input[*]`) also stops apps adding items to that list. If a locked guard is declared `where: sidecar`, lock `sidecar.url` too. [Locks](../advanced/layers.md#locks) lists every form.

## 3. Bound numbers instead of locking them

`bounds` let apps go lower, never higher: `budgets.per_run.max_usd: { max: 5.00 }`. A value outside a bound fails loading with `BoundViolation`. It's never clamped (R10). Bound task budgets with a wildcard, `tasks.*.budget.max_usd`.

## 4. Require an adoption level

`require_level` picks the level an app must reach by its `metadata.risk_tier`: `{ default: L1, tiers: { customer_facing: L2, takes_actions: L3 } }`. An app below it fails with `LevelTooLow` ([Adoption levels](../concepts/adoption-levels.md#requiring-a-level)).

## 5. Version it

Give it `metadata.version` and version it with SemVer. Loosening is a minor version. A new lock or a tighter bound is a **major** version (RULES V3), and apps pin a range (`org://acme-baseline@^1`), so a breaking change never reaches an app unannounced. `harnessweaver doctor` warns about unpinned profiles.

## 6. Test it

Write an overlay that tries to break each lock and bound, and expect each to fail to load. The example's [tests](https://github.com/Wise-DreamerZ/HarnessWeaver/tree/main/examples/_org/tests) do exactly that:

```python title="tests/test_acme_baseline.py (excerpt)"
--8<-- "examples/_org/tests/test_acme_baseline.py:13:19"


--8<-- "examples/_org/tests/test_acme_baseline.py:34:40"
```

## 7. Distribute it

Put it where every app finds it:

- a directory on `HARNESSWEAVER_PROFILE_PATH` (separated like `PATH`), which apps reference as `org://acme-baseline@^1`;
- `.harness/profiles/` next to the spec, only when the team that owns the app also owns the baseline: a team can edit files in its own repository, so this is no boundary ([Who owns a profile](../advanced/layers.md#who-owns-a-profile));
- inside a plugin package, as `hw.Plugin(profiles={"baseline": "profiles/baseline.yaml"})`, which apps reference as `profile://acme.baseline@^1` ([Plugins](../advanced/plugins.md#profiles-and-templates)).

Then review every app's changes against it with `harnessweaver spec diff git:main:harness.yaml harness.yaml` in pull requests: it shows changed values, locks, profile layers and the adoption level, and reads `git:` sides with the profiles committed at that ref.

## 8. Make sure every app extends it

Locks, bounds and `require_level` bind only a spec that names the baseline in `extends`. A spec that leaves it out loads with the framework defaults alone: none of the baseline's guards or locks, no plugin allow-list, and a required level of L0, whatever its `risk_tier`. HarnessWeaver can't tell an app that should extend your baseline from one that never needed to, so checking it is your job, in review and in CI:

- Make the platform team a required reviewer of every app's `harness.yaml`, with a `CODEOWNERS` entry, so a change to `extends` can't merge unseen. The `spec diff` in [step 7](#7-distribute-it) reports a dropped baseline as `Layer removed`, with every lock it took along.
- In each app's CI, load the spec and fail when the baseline isn't among its layers. `harnessweaver spec validate` prints them on its `layers:` line:

```bash
harnessweaver spec validate harness.yaml | grep -q "org://acme-baseline@" || { echo "harness.yaml must extend org://acme-baseline"; exit 1; }
```

Run it with the baseline on `HARNESSWEAVER_PROFILE_PATH`, as the app finds it in production. Once the spec extends the baseline, the same `spec validate` also fails on any broken lock, bound or required level.
