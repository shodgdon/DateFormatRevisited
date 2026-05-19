# Date Format Revisited: Development Plan

A phased plan for forking [rcav8tr/CS1Mod-DateFormat](https://github.com/rcav8tr/CS1Mod-DateFormat) (MIT licensed) into **Date Format Revisited**: adding a display-only date offset feature, Race Day expansion support, and releasing as a successor mod.

Each phase has a clear "done" state.  Do not move on until that state is reached.

---

## Phase 0: Pre-work (15 minutes)

Before touching any code, confirm you have:

- Rider installed and licensed (JetBrains Toolbox is the easiest way to manage versions)
- CS1 installed and working
- GitHub Desktop or the `gh` CLI
- A decompiler that runs on Mac

The decompiler is the one Mac-specific catch.  **dotPeek is Windows-only**, so:

- Best option: Rider has a built-in decompiler.  Right-click a DLL in the Solution Explorer → "Open in Assembly Explorer".
- Alternative: AvaloniaILSpy runs natively on Mac (`brew install --cask ilspy`, or grab the release from GitHub).

Confirm `Assembly-CSharp.dll` exists at:

```
/Users/sean/Library/Application Support/Steam/steamapps/common/Cities_Skylines/Cities.app/Contents/Resources/Data/Managed/
```

The `/Applications/Cities Skylines.app` shortcut is a thin launcher shell—not the actual game files.  Always reference the Steam path above.

### Rider plugins

When Rider prompts about featured plugins, skip almost all of them:

- **Skip AI Assistant** and **GitHub Copilot**—you are using Claude Code, no point running two AI tools.
- **Skip the Unity plugin**—it is for opening Unity projects directly, not building DLLs that interface with Unity at runtime.
- **Skip Docker, Database Tools, Code With Me, TeamCity**—all irrelevant here.
- **IdeaVim**—only if you are a Vim user.

The base Rider install has everything needed for .NET Framework C# development.  Add plugins later if a specific need arises.

---

## Phase 1: Get the Existing Mod Building Locally (1–2 hours, expect friction)

This is the gate.  If you cannot build and load the unmodified fork, none of the rest matters.  Do not try to add features until this works.

### Optional first step: install the original mod

Worth doing temporarily to see baseline behavior:

1. Subscribe to the original Date Format on Steam Workshop.
2. Run the game with just it enabled.  Note how the options panel looks, how formats render in each panel, what the date displays look like.  Take screenshots if useful.
3. Unsubscribe before testing your fork.  Do not run both simultaneously—even after renaming, both will try to patch the same game methods and Harmony will fail or silently drop one.
4. Restart Steam to make sure the unsubscribe took effect.  Verify the mod is gone from `~/Library/Application Support/Steam/steamapps/workshop/content/255710/` (255710 is CS1's Steam app ID).
5. Optionally delete any leftover `DateFormat*.xml` config files from `~/Library/Application Support/Colossal Order/Cities_Skylines/`.  Not strictly required since your renamed config will use a different filename.

### Steps

1. Repo is cloned at `/Users/sean/Developer/GitHub/DateFormatRevisited`.
2. Open `DateFormat.sln` in Rider.  It will restore NuGet packages (mainly `Lib.Harmony` and `CitiesHarmony.API`) automatically.
3. Fix the DLL references.  Open `DateFormat.csproj` and look for `<Reference Include="...">` blocks pointing at Windows paths.  References to `Assembly-CSharp`, `ColossalManaged`, `ICities`, `UnityEngine`, and others each need a `<HintPath>` pointing to the Steam install:

   ```xml
   <HintPath>/Users/sean/Library/Application Support/Steam/steamapps/common/Cities_Skylines/Cities.app/Contents/Resources/Data/Managed/Assembly-CSharp.dll</HintPath>
   ```

   Better: use an MSBuild property so the project stays portable.  Something like `<HintPath>$(CitiesSkylinesPath)/Assembly-CSharp.dll</HintPath>` with `CitiesSkylinesPath` set in a local `.props` file (gitignored) or via environment variable.

4. Build the project.  Warnings about older .NET Framework versions are expected—ignore them.  You want a `DateFormat.dll` (rename target later) in `bin/Debug` or `bin/Release`.
5. Set up the LocalMods deploy folder:

   ```
   ~/Library/Application Support/Colossal Order/Cities_Skylines/Addons/Mods/DateFormatRevisited/
   ```

6. Copy the built DLL to that folder.  Even better, add a post-build step to the `.csproj` so it copies automatically on every build:

   ```xml
   <Target Name="DeployMod" AfterTargets="Build">
     <Copy SourceFiles="$(OutputPath)$(AssemblyName).dll" 
           DestinationFolder="$(HOME)/Library/Application Support/Colossal Order/Cities_Skylines/Addons/Mods/DateFormatRevisited/" />
   </Target>
   ```

7. Launch CS1, go to Content Manager → Mods, find the mod in the local mods list, and enable it.  Restart the game when prompted.  Start a city or load a save, verify the date displays in your configured format.
8. Tail the log in another terminal:

   ```bash
   tail -f ~/Library/Logs/Unity/Player.log
   ```

   Look for the mod's startup messages.  The existing `LogUtil` writes here.

### Done state

You can change a string in the source code (like a log message), rebuild, restart the game, and see the change in the log.  That iteration loop is your foundation—confirm it works end to end before moving on.

### Common failure modes

- Missing `CitiesHarmony.API` reference → install via NuGet.
- Mod does not appear in Content Manager → check the LocalMods folder path, the DLL filename, and that the DLL sits in its own subfolder.
- Mod loads but throws on init → check the log; usually a DLL reference mismatch (you referenced a different version than the game has).

---

## Phase 2: Rename to Date Format Revisited (30–60 minutes)

Do this as a discrete phase so the rename and the new feature work do not get tangled.

### Steps

1. **`IUserMod.Name` property** (probably in `DateFormat.cs`)—update to "Date Format Revisited".  This is what shows in Content Manager.
2. **`IUserMod.Description` property**—update to reflect that this is a revival of the original mod with added features.
3. **Config file name**—the existing mod writes to something like `DateFormatConfig.xml` in the user folder.  Change to `DateFormatRevisitedConfig.xml` so your fork does not read or write the same file as the original.  Look for the path in `DateFormatConfig.cs` (or wherever serialization lives).
4. **Harmony identifier**—look for `Harmony.CreateAndPatchAll` or similar.  The ID passed should be your new mod name (something like `com.github.sean.dateformatrevisited`), not the original's.  Prevents conflicts if both are installed.
5. **Assembly name** (optional but cleaner)—rename the output assembly from `DateFormat.dll` to `DateFormatRevisited.dll` in the `.csproj`.  Update the post-build deploy step to match.
6. **README**—rewrite to credit the original author, link to the original repo, explain what is new, preserve the MIT license.

### Done state

Build, deploy, and load the renamed mod.  Verify Content Manager shows "Date Format Revisited".  Verify the config file written to disk has the new name.  All existing functionality should still work.

---

## Phase 3: Set Up Your Dev Environment for AI-assisted Work (30 minutes)

Now invest in tooling that pays back over the project.

### Steps

1. Create a `reference/` folder in your repo and add it to `.gitignore`.
2. Decompile the key game classes using Rider's Assembly Explorer:
   - `UIDateTimeWrapper` (the main HUD date—your first feature target)
   - `FestivalPanel`, `FootballPanel`, `ChirpXPanel`, `VarsitySportsArenaPanel` (the other existing patch targets)
   - `Bindings` (since the existing mod reaches into it)
   - `SimulationManager` (for context on where the actual sim DateTime lives)

   Export each as a `.cs` file to `reference/`.
3. Write `CLAUDE.md` at the repo root.  Suggested starter content:

   ```markdown
   # Date Format Revisited

   Fork of rcav8tr/CS1Mod-DateFormat (MIT). Adds display-only date offset 
   feature and Race Day expansion compatibility.

   ## Environment
   - macOS, Cities Skylines 1 patch 1.21.1 f4+
   - CS1 install: /Users/sean/Library/Application Support/Steam/steamapps/common/Cities_Skylines/Cities.app/Contents/Resources/Data/Managed/
   - LocalMods deploy: ~/Library/Application Support/Colossal Order/Cities_Skylines/Addons/Mods/DateFormatRevisited/
   - Player log: ~/Library/Logs/Unity/Player.log
   - Built DLL: bin/Debug/DateFormatRevisited.dll (auto-deployed via post-build step)

   ## Architecture
   - C# .NET 3.5 / Mono, runs in Unity inside CS1
   - Harmony patching via CitiesHarmony.API (HarmonyHelper handles availability)
   - Existing patches use transpilers to swap hardcoded format strings
   - New offset feature will use prefix/postfix patches instead

   ## Reference
   - reference/ contains decompiled game source for patched classes (gitignored)
   - Original mod's HarmonyPatcher.cs lists all patch targets

   ## Conventions
   - Use existing LogUtil for all logging
   - Configuration follows existing pattern in DateFormatConfig.cs
   - Mod must work with or without each patched mod present (graceful degradation)
   ```

4. Run `claude` in the project directory to verify it picks up `CLAUDE.md`.

---

## Phase 4: Plan the Data Model (15 minutes, mostly thinking)

Before writing code, decide what the user-facing configuration looks like.  Suggested starting structure:

- **Offset mode:** None (off), Fixed Years, Fixed Date Anchor
  - Fixed Years = subtract N years from the displayed date (simplest)
  - Fixed Date Anchor = "treat the game's start date as real-world date X" and offset everything accordingly (more useful for "play in 2024" feel)
- **Offset value:** an integer year count, or a date picker for the anchor
- **Apply to:** checkboxes for each panel category (main HUD always; festivals, sports, race events optional)

The "apply to" toggles matter because users might want offset on the HUD but real dates on event panels.

### Decided (2026-05-18)

- **Mode:** Fixed Years only.  Single `OffsetYears` (int, default `0`; `0` = off).  No mode enum; Fixed Date Anchor deferred unless requested.
- **Sign convention:** `displayed = actual.AddYears(OffsetYears)` — negative = past, positive = future (intuitive UX).  Clamp config to a sane band; clamp the *resulting* date to `DateTime`'s 1–9999 range so `AddYears` never throws.
- **Display-only:** never alters the simulation or save files.
- **MVP scope:** offset every base-game date the original mod already patches (main HUD + ChirpX + Festival + Football + Varsity Sports).  This **merges Phase 5 and Phase 6** into one offset effort on a single `feat/date-offset` branch.
- **Post-1.0 (1.X):** Race Day / newer panels (Phase 7) and mod-to-mod offset (Phase 8) — these need extra mods/expansions to test and add conditional surface area.

### Mechanism revised (2026-05-18, during Phase 5)

The Phase 4 plan called for **prefix** patches per panel.  Reading the
decompiled game source proved this unworkable for the full MVP:
`ChirpXPanel.UpdateBindings`, `FootballPanel.RefreshMatchInfo`, and
`VarsitySportsArenaPanel.RefreshPastMatches` read the displayed `DateTime` from
a **local** copied out of the `EventManager` buffer mid-method — a prefix runs
before the body and cannot reach it.

**Revised mechanism:** extend the existing transpiler instead.  Split it into
`ReplaceDateFormatString` (format-only — used by `CreateTranspilerPatchForMod`,
keeping mod-to-mod offset deferred to Phase 8) and
`ReplaceDateFormatStringWithOffset` (the 7 base-game patches, via
`CreateTranspilerPatch(..., applyOffset: true)`).  The offset variant swaps the
format string and then retargets the paired `DateTime.ToString(string)` call to
the static `HarmonyPatcher.OffsetAndFormat(ref DateTime, string)` — binary
IL-compatible because the value-type instance call already pushes the receiver
as a `DateTime&`.  This works uniformly for all 7 targets regardless of whether
the date originates as a parameter, field, or local.  UI is a parse+clamp
number text field.  Wherever Phases 5–6 below say "prefix/postfix patch", read
"the offset transpiler variant".

---

## Phase 5: First Feature—Main HUD Date Offset (2–4 hours)

This is the real proof of concept.  If this works, everything else is repetition.

> **Scope note (Phase 4 decision):** the MVP covers the main HUD *and* the other base-game panels (the old Phase 6) as one effort on `feat/date-offset`.  The steps below stay the HUD-first proof of concept; Phase 6 then mechanically repeats the pattern for the remaining panels before the MVP merges.

### Steps

1. Extend the configuration class with `OffsetYears` (int, default 0).
2. Add UI for it in the options panel.  Look at how existing fields render—probably a `UITextField` or `UISlider`.
3. Write the prefix patch for `UIDateTimeWrapper.Check`.  The skeleton looks roughly like:

   ```csharp
   [HarmonyPatch(typeof(UIDateTimeWrapper), "Check")]
   static class UIDateTimeWrapper_Check_Offset {
       static bool Prefix(UIDateTimeWrapper __instance, ref DateTime ___m_Value /* ... */) {
           // read sim time, apply offset, set the label, return false to skip original
       }
   }
   ```

   The actual signature depends on what `UIDateTimeWrapper.Check` does internally—this is why you decompiled it to `reference/`.  Have Claude Code read that file before writing the patch.

4. Be aware of patch ordering.  The existing transpiler patch on `Check` swaps the format string.  Your prefix patch needs to coexist with it.  Either:
   - Your prefix patches first, applies offset to the DateTime, then lets the original run (which then formats with the transpiled format string).  Return `true`.
   - Or your prefix replaces the whole method, including formatting.  Return `false`.

   The first approach is cleaner because it composes with the format feature.
5. Build, deploy, test.  Open a save, set offset to 30 years, verify the HUD shows 30 years earlier than the actual game date.  Festival panel should still show the real date (you have not patched those yet)—that is expected and proves your patch is surgical.

### Done state

Main HUD date is offset by your configured amount.  Other panels unaffected.  Format feature still works.

### Realistic expectation

There is a good chance your first prefix patch will have a subtle issue: wrong field access, patch ordering with the transpiler, or an edge case with paused simulation.  Expect to iterate two or three times.  Watch the log, observe the symptom, refine the patch.

---

## Phase 6: Extend Offset to Other Existing Panels (1–2 hours)

Same pattern as Phase 5, repeated for:

- `ChirpXPanel`
- `FestivalPanel.RefreshCurrentConcert`
- `FestivalPanel.RefreshFutureConcert`
- `FootballPanel.RefreshMatchInfo`
- `VarsitySportsArenaPanel.RefreshPastMatches`
- `VarsitySportsArenaPanel.RefreshNextMatchDates`

For each one: decompile, read what it does with the DateTime, write a prefix or postfix patch that applies the offset.  This is mostly mechanical—good Claude Code work.

Skip the mod-to-mod patches (Extended InfoPanel, Real Time, More City Statistics, Enhanced Outside Connections View) for now.  Come back to those after Race Day.

---

## Phase 7: Race Day Investigation and Support (2–4 hours)

### Steps

1. Decompile the post-1.21.1 `Assembly-CSharp.dll`.  It should already be the latest since you have CS1 up to date.
2. Search for new classes: "Race", "Event", "Motor", "Parade" prefixes.  Look for `*Panel` classes with `Refresh*` or `Update*` methods.
3. Critical: also search for new date format string literals.  The existing transpiler only swaps `"dd/MM/yyyy"`, `"yyyy-MM-dd"`, and `"d"`.  Search the decompiled output for `ToString("` to find every date format call—if Race Day uses something new like `"MMM yyyy"`, you need to add it to the operand check in `ReplaceDateFormatString`.
4. Add patches for the new panels for both:
   - Format substitution (extend the existing transpiler patch list)
   - Offset (new prefix or postfix patches)
5. Test with Race Day content: create a race event, verify dates display correctly with both format and offset applied.

---

## Phase 8: Restore Mod-to-mod Patches (1–2 hours)

Go back to Extended InfoPanel, Real Time, More City Statistics, and Enhanced Outside Connections View.  These need:

- Format substitution (already done by upstream code—you inherited it)
- Offset support (your new feature, needs adding)

Test with each of those mods installed.  Some users will have them, some will not—your mod must degrade gracefully when they are absent.  The existing `CreateTranspilerPatchForMod` already handles the "mod not subscribed" case.

---

## Phase 9: Polish for Release (2–3 hours)

- **Compatibility declaration:** declare incompatibility with the original DateFormat mod so users do not run both.  The mod loader supports this.
- **Logging cleanup:** dial down verbose dev logging, keep error logging.
- **Config file location (deferred from Phase 2):** `Configuration.cs` `GetConfigFile()` returns a bare filename, so the config is written to the process working directory (the Steam game install dir), where Steam "verify integrity" / game updates can wipe it.  Relocate to the standard user folder (`~/Library/Application Support/Colossal Order/Cities_Skylines/`) via the game's `ColossalFramework.IO.DataLocation` API.  Inherited from the original mod; needs its own retest.
- **Test on a fresh save:** start a new city, play through real-time events, verify nothing breaks.
- **Test enabling and disabling at runtime:** the mod should clean up its patches when disabled.
- **Final README review:** make sure new features, credits, and migration notes from the original are clear.

---

## Phase 10: Publishing (2–3 hours, mostly waiting)

### Steps

1. **GitHub release.**  Tag a version (`v1.0.0`), write release notes, publish.  Attach the built DLL as a release asset.
2. **Steam Workshop.**  The original repo has a `Steam Deployment` folder—use it as a template.  You will need:
   - A preview image (512x512 PNG, eye-catching)
   - Workshop description (markdown-ish formatting)
   - The built DLL and any supporting files
   - A Workshop ID (assigned on first upload)
3. **Announce.**  Post in the relevant subreddits, the CS modding Discord, and the original mod's Steam Workshop comments (where people are already asking for this).

---

## Cross-cutting Concerns

### Git workflow

For a project this size:

- `master` should always be in a stable, releasable state.
- Feature branches per phase: `feat/rename`, `feat/date-offset`, `feat/race-day`, etc.
- Merge to `master` when each phase is verified working.
- Tag releases on `master`.

This makes it easy to bisect if something breaks, and easy to accept PRs later if others contribute.

### When to use Claude Code most heavily

- **Phase 1:** Minimal.  The `.csproj` path fixes are simple manual edits.
- **Phase 2:** Medium.  Mechanical find-and-replace work, good Claude territory.
- **Phase 5:** Heavy.  This is the main creative work; Claude with `reference/` context can draft the patch.
- **Phase 6:** Heavy.  Mechanical repetition is where Claude shines.
- **Phase 7:** Medium.  The investigation is yours, but writing the patches once you have identified targets is Claude work.
- **Phase 9 and 10:** Light.  Mostly your judgment calls.

### Decompiled source and copyright

The decompiled game code in `reference/` is Colossal Order's IP.  Keep it gitignored.  Do not commit it, do not redistribute it, do not paste large chunks into public-facing materials.  Use it as local reference only.

---

## Quick Reference: Key Paths on Mac

| Purpose | Path |
| --- | --- |
| Game DLLs (build references) | `/Users/sean/Library/Application Support/Steam/steamapps/common/Cities_Skylines/Cities.app/Contents/Resources/Data/Managed/` |
| LocalMods deploy folder | `~/Library/Application Support/Colossal Order/Cities_Skylines/Addons/Mods/DateFormatRevisited/` |
| Player log | `~/Library/Logs/Unity/Player.log` |
| Game settings and saves | `~/Library/Application Support/Colossal Order/Cities_Skylines/` |
| Workshop subscriptions | `~/Library/Application Support/Steam/steamapps/workshop/content/255710/` |
| Local repo | `/Users/sean/Developer/GitHub/DateFormatRevisited` |
