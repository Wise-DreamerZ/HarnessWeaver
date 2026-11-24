# ADR-0049: A governor block beats a tool.before Suspend

**Status:** Accepted. It amends [ADR-0025](0025-governor-placement.md).

## Context

The governor's steps 1 to 7 run after the `tool.before` middleware, on the arguments it leaves (ADR-0025). When a middleware answers `Suspend`, the run stops there and waits for a person, before the governor has looked at the call. If the governor would block the call, the approver is asked about a call that can't run: once they approve, the run resumes, the governor blocks it, and the approval was wasted.

Example 03 showed it. On a Saturday, `acme.no_weekend_refunds` asks finance about a 1500 USD refund that the `refund-cap` rule denies; on a Monday the same call is blocked at once. The verdict was the same, but on a weekend a person was asked first.

## Decision

When the `tool.before` middleware answers `Suspend` for a call, the governor decides that call as a dry run (`authorize(..., dry_run=True)`) before the run suspends. If the verdict is a block, the call is blocked with the governor's reason and code, and nobody is asked. Otherwise the run suspends as the middleware asked, and on resume the governor runs as before, with the approval.

- The dry run uses nothing up: it gives back the rate-limit place it took, and it's traced like any governor decision (R13).
- It decides on the arguments the middleware leaves, the same ones the governor would see after approval.
- Python and TypeScript do the same.

## Why

The most severe outcome wins (DESIGNS No. 8), and inside the governor a deny already beats an approval (ADR-0025). Before this, which of the two a run met first depended on a middleware's answer: a call the policy denies was blocked at once on a weekday, and only after a person's approval on a weekend. Now it is blocked at once either way, and approvers see only calls they can actually allow.

## Consequences

- An approver can no longer edit a denied call into one the policy allows, when a middleware was the one that asked. The governor's own approvals never allowed that either: a deny there blocks before anyone is asked.
- A `tools.judge` runs once more for a call a middleware suspends: in the dry run, then again after approval.
- Only `Suspend` is affected, since it is the one outcome after which the call can still run. A middleware's `Block` or `Escalate` ends the run without the call.
