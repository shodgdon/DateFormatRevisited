# Date Format Revisited

A revival and successor of the original **Date Format** mod for *Cities: Skylines*
(CS1), continuing it for current game versions and Mac, with new features planned.

## What it does

Same date-formatting behavior as the original mod. You can specify:

- Date part order: DMY, MDY, or YMD
- Separator: slash (`/`), period (`.`), dash (`-`), or space
- Month/day with or without a leading zero for values under 10

Years are always shown with 4 digits (game dates are often year 2100+). The
format can be changed mid-game and applies immediately. Formatted dates include
the main game date and the ChirpX, Football, and Varsity Sports panels.

## Planned (not yet implemented)

These are the reasons this fork exists; they are **not in this build yet**:

- **Display-only date offset** — shift displayed dates by a configurable amount
  (e.g. "play in 2024") without changing the simulation.
- **Race Day expansion support** — formatting/offset for the new Race Day panels.

## Credits

This is a fork of the original **Date Format** mod by **rcav8tr**:

- Original mod / source: <https://github.com/rcav8tr/CS1Mod-DateFormat>
- This fork: <https://github.com/shodgdon/DateFormatRevisited>

All original date-formatting functionality is rcav8tr's work. This fork adds Mac
build support and (upcoming) the offset and Race Day features.

## License

MIT — see [LICENSE.txt](LICENSE.txt). The original copyright
(`Copyright (c) 2024 rcav8tr`) is preserved as required by the MIT license; the
revival is maintained by shodgdon.

## Building (macOS)

References the local *Cities: Skylines* install via a gitignored
`Directory.Build.props` (copy from `Directory.Build.props.template` and set your
path). Build and auto-deploy to the local Mods folder with:

```bash
msbuild DateFormat.csproj /t:Rebuild /p:Configuration=Debug
```
