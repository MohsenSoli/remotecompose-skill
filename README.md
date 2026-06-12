# RemoteCompose — Claude Code Skill

A [Claude Code](https://claude.com/claude-code) skill that teaches the
assistant how to build and debug server-driven Android UIs with
**AndroidX RemoteCompose** (`androidx.compose.remote`).

RemoteCompose is experimental: APIs shift between alphas, several things only
fail on-device, and the official docs lag behind. This skill encodes the
hard-won knowledge from building a real end-to-end demo — so the assistant
gets it right the first time instead of rediscovering every pitfall.

## What's inside

- The server-side creation DSL (`createRcBuffer`, `RcScope`, `Modifier`) and
  the JVM/Android platform split
- The raw-pixel density model and the dp/density handshake that actually works
- Document state, expressions, host actions, and the persistence feedback loop
- Known alpha pitfalls: the themed-color crash, HostAction payload ids,
  triple-firing clicks, `StateLayout` positioning, and more
- A device-verification workflow (curl + adb + screenshots) — because an
  HTTP 200 does not mean the document renders

## Install

```bash
git clone https://github.com/<you>/remotecompose-skill ~/.claude/skills/remotecompose
```

Or copy the folder into a project as `.claude/skills/remotecompose/` to scope
it to that repo.

## See it in action

The skill was distilled from this working demo (Ktor server + Android player,
with state and persistence): https://github.com/<you>/RemoteCompose

## Caveat

Written against `1.0.0-alpha12` (June 2026). When targeting a newer alpha,
re-verify the pitfalls — some are upstream bugs that will get fixed.
