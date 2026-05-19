# Date Format Revisited

macOS fork of [rcav8tr/CS1Mod-DateFormat](https://github.com/rcav8tr/CS1Mod-DateFormat) (MIT).
A Cities: Skylines 1 mod that reformats displayed dates. This fork adds Mac build
support and a display-only date offset feature (in progress); Race Day support
is planned post-1.0.

## Roadmap & process
- `PLAN.md` is the authoritative phased plan. Work phase-by-phase; each phase has
  a verified done-state — **do not skip ahead**. Phases 1–3 complete; Phase 4
  (offset data model) decided — see "Offset feature" below.
- One feature branch per phase (`feat/rename`, `feat/date-offset`, …); merge
  `--no-ff` to `master` when the phase verifies. `master` stays releasable.
- Commit/push only when the user asks.

## Environment (macOS)
- C# .NET Framework 3.5 / Mono, runs in Unity inside CS1.
- Game DLLs referenced via `$(CitiesSkylinesPath)` from gitignored
  `Directory.Build.props` (copy from `Directory.Build.props.template`).
- LocalMods deploy: `~/Library/Application Support/Colossal Order/Cities_Skylines/Addons/Mods/DateFormatRevisited/`
- Player log: `~/Library/Logs/Unity/Player.log`
- Build (auto-deploys mod DLL + CitiesHarmony.API.dll):
  ```bash
  msbuild DateFormat.csproj /t:Rebuild /p:Configuration=Debug
  ```
  Note `TreatWarningsAsErrors=true`. CS1 loads the DLL at startup — fully quit &
  relaunch the game to test changes.

## Architecture
- Harmony patching via CitiesHarmony.API; `HarmonyHelper` gates availability.
- Patches are **transpilers** (`HarmonyPatcher.cs`) that swap hardcoded
  date-format strings. The offset feature **extends the same transpiler pass**
  rather than adding prefix patches (prefixes can't reach the local-variable
  date in ChirpX/Football/Varsity-PastMatches — see Offset feature below).
- Must work with or without each optionally-patched mod present (graceful
  degradation; `CreateTranspilerPatchForMod` handles "mod not subscribed").

## Offset feature (Phase 4 decided → Phase 5 MVP)
- **Display-only**: shifts only *displayed* dates; never the simulation or saves.
- Config: single `OffsetYears` (int, default `0`; `0` = off) on
  `DateFormatConfiguration`. No mode enum — Fixed Anchor Date deferred unless
  requested.
- Semantics: `displayed = actual.AddYears(OffsetYears)` — **negative = past,
  positive = future** (intuitive UX sign). Clamp config to a sane band and clamp
  the *result* to `DateTime`'s 1–9999 range so `AddYears` never throws.
- MVP scope: offset every base-game date the original mod patches (main HUD +
  ChirpX + Festival + Football + Varsity Sports) — this **merges PLAN.md
  Phase 5 + 6**. Race Day/new panels (Phase 7) and mod-to-mod patches (Phase 8)
  are **post-1.0**.
- Mechanism (Phase 5, revised from the Phase 4 prefix decision): the transpiler
  is split into `ReplaceDateFormatString` (format-only — used by
  `CreateTranspilerPatchForMod`, offset deferred to Phase 8) and
  `ReplaceDateFormatStringWithOffset` (used by the 7 base-game patches via
  `CreateTranspilerPatch(..., applyOffset: true)`). The offset variant, after
  swapping the format string, retargets the paired
  `DateTime.ToString(string)` call to the static
  `HarmonyPatcher.OffsetAndFormat(ref DateTime, string)` — IL-compatible
  because a value-type instance call already pushes the receiver as a
  `DateTime&`. `OffsetAndFormat` reads the offset live (so runtime config
  changes apply on next refresh) and never throws.
- **Why not prefix** (the Phase 4 plan): `ChirpXPanel.UpdateBindings`,
  `FootballPanel.RefreshMatchInfo`, `VarsitySportsArenaPanel.RefreshPastMatches`
  read the date from a local copied out of the `EventManager` buffer
  mid-method — a prefix runs before the body and can't reach it. The transpiler
  retarget works uniformly for all 7 regardless of param/field/local origin.

## Identity (set in Phase 2 — keep consistent)
- Mod name: **Date Format Revisited** · Harmony ID:
  `com.github.shodgdon.DateFormatRevisited` · log prefix `[DateFormatRevisited]`
  · config file `DateFormatRevisitedConfig.xml` · assembly `DateFormatRevisited.dll`.
- **Deliberately NOT renamed** (do not "fix" these): the internal C# `namespace
  DateFormat`, `<RootNamespace>`, and the `DateFormat.csproj`/`DateFormat.sln`
  filenames. The log prefix is hardcoded in `LogUtil.cs`, decoupled from the
  namespace on purpose.

## Known deferred work
- Config is written to the process working dir (the Steam game install dir),
  not the Colossal Order user folder. Inherited from the original; fragile but
  **intentionally deferred to Phase 9** — do not change early. See `PLAN.md`.

## Reference
- `reference/` holds decompiled CS1 game source for patched classes. It is
  **gitignored** — Colossal Order IP, never commit or redistribute. Read it
  before writing patches against game internals.
- The original mod's `HarmonyPatcher.cs` enumerates all patch targets.

## Conventions
- All logging via `LogUtil`.
- Configuration follows the existing `DateFormatConfig.cs` / `Configuration<C>`
  pattern.
