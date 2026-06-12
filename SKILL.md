---
name: remotecompose
description: Build and debug server-driven Android UIs with AndroidX RemoteCompose (androidx.compose.remote). Use when writing or fixing RemoteCompose documents (server-side creation DSL), wiring a RemoteDocumentPlayer client, upgrading the alpha version, or diagnosing player crashes/mis-rendering. Covers the JVM creation DSL, density model, document state, host actions, and known alpha pitfalls.
---

# RemoteCompose

AndroidX RemoteCompose is server-driven UI: a creation library writes a binary
*document* (layout + text + colors + state variables + expressions + actions);
an Android player (`RemoteDocumentPlayer`) renders it and runs all state and
animation locally. The library is **experimental** — APIs and behavior change
between alphas, and several things only fail on-device.

If the project contains a `REMOTECOMPOSE.md` field guide, read it first; it
may be newer than this skill.

## Ground rules

1. **Read the real sources, not memory.** For the exact alpha in use, download
   and unzip the sources jar:
   `https://dl.google.com/android/maven2/androidx/compose/remote/<artifact>/<version>/<artifact>-<version>-sources.jar`
   List versions via `.../androidx/compose/remote/group-index.xml`. Versions
   sort oddly (`alpha010` < `alpha11`).
2. **Platform split**: `remote-creation`, `remote-creation-core`,
   `remote-creation-jvm`, `remote-core` run on plain JVM (servers OK).
   `remote-creation-compose` (the `@Composable` authoring API) and all players
   are Android-only.
3. **Verify on a device/emulator, never just by HTTP 200.** Document
   generation succeeding does not mean it renders: several pitfalls produce
   valid-looking bytes that crash or mis-render only in the player.
   Player errors don't crash the app — look for a red triangle + message, and
   read `adb logcat -d | grep -B2 -A15 "System.err"`.

## Server-side authoring (alpha12-era DSL)

Use `androidx.compose.remote.creation.dsl`: `createRcBuffer` +
`RcScope` (`Box/Column/Row/Text/Spacer/StateLayout`) + `Modifier` chains
(`padding/size/background/clip/onClick/ripple/verticalScroll/weight`). It is
`@RestrictTo(LIBRARY_GROUP)` — fine from a JVM module (no Android lint), but
expect churn per alpha.

- Profile: `RcProfile(Profile(CoreDocument.DOCUMENT_API_LEVEL, RcProfiles.PROFILE_ANDROIDX, JvmRcPlatformServices(), factory), experimental = true)`.
- Header tags carry doc metadata: `hTag(Header.DOC_WIDTH, w)`, `DOC_HEIGHT`,
  `DOC_CONTENT_DESCRIPTION`, and **always** `hTag(Header.FEATURE_CLICK_VERSION, 1)`
  (see pitfall list).
- Ops are validated at write time against the declared profile; unsupported
  ops throw `Operation N is not supported for this version` server-side.
- State: `remoteFloat/remoteInteger/remoteBool/remoteText`; expressions with
  operators; time sources `hour()/minutes()/animationTime()`; live text via
  `createTextFromFloat` + `RcText` concatenation.
- Actions: `Modifier.onClick { setValue(v, expr); hostAction("name") }` —
  multiple actions per block are fine.
- `StateLayout(intVar) { stateA; stateB }` = tabs/toggles/segmented controls.
- Theming: `remoteThemedColor` pairs resolve light/dark on the player.

## The density model (biggest foot-gun)

The alpha12 player lays out **everything in raw pixels** — modifier dims,
radii, AND font sizes. Pattern that works: client sends viewport
`?w=<dp>&h=<dp>&d=<density>`; server multiplies every dp/sp by density when
writing. Do NOT rely on `DOC_DENSITY_BEHAVIOR` (partial coverage) or
`ROOT_CONTENT_BEHAVIOR`/`SIZING_SCALE` (deprecated profile, wrong scale
factor) — both tested broken at alpha12.

## Host ↔ document state (persistence pattern)

Document state resets on every document load. To persist:
1. Document mutates its own state on tap (instant UI) **and** fires
   `hostAction("name")` in the same onClick block.
2. The host mirrors the value (memory or disk) in `onNamedAction`.
3. The host sends the value back as a query param; the server bakes it in as
   the initial value of the next document.

For host actions **with a payload**: `HostAction(name, INT_TYPE, valueId)`
where `valueId` is the raw id of a registered variable —
`(writer.addInteger(v) % 0x100000000L).toInt()` — NOT the literal value.
If the screen identity already implies the payload (e.g. a per-user detail
screen), prefer a payload-less `hostAction(name)` and resolve it host-side.

## Known alpha12 pitfalls (verify before assuming fixed in later alphas)

1. `remoteThemedColor(Int, Int)` routes ARGB *values* into the id-based
   overload → negative ids → player crashes in `IntMap.findKey` at
   `setDocument`. Use `remoteThemedColor(remoteColor(light), remoteColor(dark))`.
2. `HostAction(name, INT_TYPE, x)`: `x` is a variable id (see above); a
   literal compiles but delivers garbage to the host.
3. Without `FEATURE_CLICK_VERSION=1` header tag, a legacy gesture detector
   ALSO fires click actions → ~3 firings per tap.
4. `StateLayout` directly inside a `Column` paints its child at the column's
   origin — wrap it in a `Box`.
5. `CircleShape` clip is ignored for backgrounds — use
   `RoundedRectShape(r,r,r,r)` with `r = size/2`.
6. `minutes()`/time variables can disagree with wall-clock minutes on
   emulators; treat as approximate.

## Client side

- `RemoteDocument(bytes).document` → `RemoteDocumentPlayer(document, documentWidth/HeightPx, onNamedAction = { name, value, stateUpdater -> ... })`.
  Both are `@RestrictTo` → `@SuppressLint("RestrictedApi")`.
- Wrap the player with proper Loading / Error+Retry states; a failed fetch or
  parse should never strand the user.
- Emulator → host server: `http://10.0.2.2:<port>` + `usesCleartextTraffic`.

## Debug workflow

```bash
curl -s -o /tmp/doc.bin -w "%{http_code} %{size_download}B" "localhost:8080/ui/...?w=448&h=997&d=3.0"
adb install -r app-debug.apk && adb shell am start -n <pkg>/.MainActivity
adb exec-out screencap -p > /tmp/s.png        # LOOK at it
adb shell input tap X Y / input swipe ...     # drive every interactive element
adb logcat -d | grep -E "<repo tag>|System.err"
```

Screenshot after every server change, and test every interaction (clicks
multi-firing, wrong payloads, and state resets are all invisible in a static
screenshot of the first frame).
