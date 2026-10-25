# env-facts — DSH plugin: structured environment facts with an as-of stamp

A DeepSeek Harness (DSH) Host plugin that adds a machine-readable fact block with its
capture time to the runtime-context snapshot:

```
As of 2026-10-25T12:34:56.789Z
<env>
cwd: "F:\\workspace"
file_policy: danger-full-access
approval_policy: never
gui_url: http://127.0.0.1:3442
session: 019...
</env>
```

The built-in snapshot lists these facts as prose and carries no timestamp; this plugin adds
both a structured form and an `As of` time without touching the DSH installation.

## Why the as-of stamp is computed this way

The snapshot text participates in DSH's dedupe: while the environment is stable the snapshot
is **not re-sent** (an audit of `dsh-v0.2.1-rc.3` measured 186 model steps producing a single
snapshot). Writing a fresh timestamp every step would defeat that dedupe and emit one message
per step.

This plugin caches per session and only recomputes the text when the facts change. `As of` is
the moment those facts last changed (or were first captured), so the text stays byte-identical
while the environment is stable and the snapshot dedupe keeps working.

## Facts and their sources

| Field | Source |
|---|---|
| `cwd` | `workingDirectory.get(session)` |
| `file_policy` | `sandboxPolicy.resolve({ session }).mode` |
| `approval_policy` | `approval.overrideOf(session)` (omitted when unset) |
| `gui_url` | `webApp.getAdvertisedUrl()` |
| `session` | `session.id` |

A missing service or fact is simply omitted; it never fails assembly.

## Install

Install the package directory as a local bundle with `plugin_manager` (`install_bundle`).
The package declares `dsh.bundle.patch`, which inserts the `env-facts` row.

## `upstream-patch/`

This directory contains two patches against `deepseek-ai/deepseek-harness` tag
`dsh-v0.2.1-rc.3` that move the same idea into the core (snapshot header constant + stamp at
emission, dedupe by body). They are provided for reference and for the upstream project; see
`upstream-patch/PR.md`.

## License

MIT — see `LICENSE`.
