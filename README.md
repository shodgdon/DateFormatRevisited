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
the main game date and the ChirpX, Festival, Football, and Varsity Sports
panels.

### Display-only date offset

Shift every displayed date by a configurable number of years (e.g. set `-30`
so a game running in 2055 reads 2025; negative = past, positive = future,
`0` = off). This is **display-only** — it never changes the simulation or your
save files, and it composes with the format options above. It applies to the
same base-game dates listed above, and the options panel previews it live.

## Planned (not yet implemented)

- **Race Day expansion support** — formatting and offset for the Race Day
  panels.
- **Mod-to-mod offset** — the offset currently covers base-game dates only.
  Extending it to supported mods (Extended Info Panel, Real Time, More City
  Statistics, Enhanced Outside Connections) is planned; date *formatting* for
  those mods already works.

## Credits

This is a fork of the original **Date Format** mod by **rcav8tr**:

- Original mod / source: <https://github.com/rcav8tr/CS1Mod-DateFormat>
- This fork: <https://github.com/shodgdon/DateFormatRevisited>

All original date-formatting functionality is rcav8tr's work. This fork adds Mac
build support and the display-only date offset; Race Day support is planned.

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
