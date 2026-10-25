# PR: date runtime-context snapshots with "As of" (dedupe-preserving)

## Problem

The runtime-context snapshot lists environment facts (working directory, file policy,
approval policy, harness checkout, Web GUI URL) but carries **no timestamp**. A model cannot
tell how old the snapshot is, so after a long-lived session it may act on stale assumptions.

The snapshot is deduplicated by exact text: an audit of the shipped build
(`dsh-v0.2.1-rc.3`) measured **186 model steps producing only 1 snapshot**. Any fix that
stamps a fresh timestamp on every step would emit one snapshot per step instead, trading a
small correctness gap for a large context/cache regression.

## Change

- `packages/core/system-prompt/src/index.ts`
  - Export the fixed snapshot header as `RUNTIME_CONTEXT_HEADER` and use it when joining
    sections. Output is byte-identical.
- `packages/core/agent-loop/src/runtime-context.ts`
  - Stamp the emission time right after the header: `As of <ISO 8601>`.
  - Track the **unstamped body** separately for dedupe; the stamp never takes part in the
    equality check.
  - Restore from history strips the stamp (line immediately after the header only), so a
    resumed session compares bodies too.

## Behavior

| Situation | Result |
|---|---|
| Environment unchanged | No new snapshot (dedupe preserved) |
| Environment changed | Exactly one snapshot, stamped with the change time |
| Snapshot cleared | Existing `CLEARED` marker, unstamped |
| Restoring an older snapshot without a stamp | Works; strip is a no-op |

## Tests

- `system-prompt`: existing `joinContextSections` expectations remain byte-identical.
- `agent-loop` runtime-context unit:
  - unchanged body → no message;
  - changed body → one message whose text contains `As of ` after the header;
  - restore from a stamped message, then project the same body → no message;
  - clear → one `CLEARED` message, then nothing.
- Session-level replay: snapshot count equals content changes, not steps.

## Risks

- The snapshot is a user message, not part of the cacheable system-prompt prefix, so the
  stamp does not invalidate prompt caching.
- Providers whose own text contains an `As of` line cannot confuse dedupe: stripping is
  scoped to the line immediately after the header.
- Not in this PR: a structured `key: value` facts block (follow-up; a small plugin prototype
  already emits `As of` + `<env>` locally without touching the installation).

## Evidence

Installed-build audit: 186 steps / 1 snapshot / 839 chars. With a local prototype that
stamped on change only: 235 steps / 2 snapshots while the environment stayed constant.
