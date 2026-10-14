# AGENTS.md — Pokémon Sea-Shore

Instructions for AI agents (and me) working in this repository.

> **This file is yours.** The last section, *Project instructions*, is where you
> record how you want this hack to work. Add to it freely — the more specific it
> is, the less I have to guess. Put hard rules in *Hard rules*, because those I
> will follow without needing to be reminded.

---

## 1. What this is

A **Pokémon Emerald ROM hack** built on
[rH-hideout/pokeemerald-expansion](https://github.com/rh-hideout/pokeemerald-expansion)
(v1.17.1). This is a decompilation project: the base game is reimplemented in C so
that the ROM can be modified far beyond what the original could do.

- The compiled ROM is `pokeemerald.gba` (32 MB).
- `origin` → this project's GitHub repo. `upstream` → rh-hideout. **Never push to
  `upstream`.**
- Build outputs (`*.gba`, `*.elf`, `*.map`, `build/`) are gitignored by design.
  Don't force-add them.

## 2. Quick commands

```bash
make -j$(nproc)          # build the ROM (incremental, usually fast)
make check               # run the test suite
make clean               # full clean, including tools and assets
make clean-assets        # drop generated graphics in build/assets, keep tools
make debug               # build with -Og -g for a debugger
make firered / leafgreen # other base games (usually not what we want)
```

Build is incremental. **After changing a source PNG you usually do not need to
clean** — but see §4.

## 3. Hard rules

1. **Never push to `upstream`.** Only `origin` is yours.
2. **Never run `make -n`.** It evaluates the `generated` prerequisite and dumps a
   single command containing every one of the ~940 map JSON paths. You will blow
   up your own context. Use `make <real target>` and read the tail of the log.
3. **Don't run `make clean` casually.** It also rebuilds every tool
   (agbcc/gbagfx/compresSmol/…), which takes a long time. Prefer
   `make clean-assets`.
4. **Never hand-edit `data/tilesets/**/metatile_attributes.bin` or a
   `metatiles.bin`.** They are *sparse and shorter than they look*, and
   `include/constants/metatile_labels.h` is **stale** — see §6.4. Use porymap.
5. **I cannot see the game.** No emulator screenshots, no pixel checks. Anything
   visual must be described to the user and confirmed by them in an emulator.
   Never report a visual change as "verified".
6. **Keep third-party asset credit** (§7). It is part of the work, not optional.

## 4. How graphics actually get compiled

This is the single most surprising part of this codebase, and getting it wrong
costs hours.

- **There is no Make rule that builds graphics.** `graphics_file_rules.mk` only
  lists *concatenation* rules (e.g. `move_types.4bpp` = `cat` of 25 icons).
  Everything else is generated as a side effect of compiling C, by
  `preproc -g build/assets`, which shells out to `gbagfx` / `compresSmol`.
- Therefore the target is `build/assets/<path>.png.4bpp`, **not**
  `graphics/<path>.4bpp`. Grepping the `.mk` files for a `.4bpp` rule for some
  asset will find nothing, and that is expected.
- C source references graphics through macros, not paths-plus-rules:

  | macro | meaning |
  |---|---|
  | `INCBIN_U16("file.bin")` | raw bytes |
  | `INCGFX_U8/U16/U32("file.png", ".4bpp")` | png → 4bpp, as 8/16/32-bit array |
  | `INCGFX_COMP("file.png", ".4bpp", "-mwidth 4 -mheight 4")` | + compresses to `.smol` |

- **Replacing a source PNG:** `preproc -g` regenerates when the source is newer
  than the artefact. When in doubt, delete the artefact under `build/assets/` or
  run `make clean-assets`. (Real example: swapping `latin_normal.png` needed the
  stale `build/assets/graphics/fonts/latin_normal.png.latfont` removed.)

- **Verifying something actually landed in the ROM** is worth doing, because a
  clean build does not prove your asset was used:

  ```bash
  python3 -c "
  rom=open('pokeemerald.gba','rb').read()
  b=open('<the built artefact under build/assets>','rb').read()
  print('FOUND' if rom.find(b)>=0 else 'NOT FOUND')"
  ```

  Remember OW sprites are smol-compressed, so search for the `.smol`, not the
  raw `.4bpp`.

## 5. Repository layout

```
src/            C sources
include/        headers; include/config/*.h holds most feature toggles
graphics/       PNG sources. graphics/pokemon/<species>/ is per-Pokémon
data/           maps (map.json + .inc), layouts (layouts.json + map.bin),
                tilesets (binary block data — see §6.4), trainers
src/data/       data tables, incl. src/data/graphics/pokemon.h which maps
                every species to its overworld art + palettes
tools/          build tools (gbagfx, mapjson, preproc, compresSmol, ...)
```

## 6. Graphics subsystems — what I have learned

### 6.1 Fonts

- `src/fonts.c` maps font variants to `graphics/fonts/<name>.png`, converted
  with `gbagfx` to `.latfont` (`png2latfont`).
- The sheet is **256×512, 4-colour, indexed**. A font PNG that differs in size or
  palette size will not behave.
- `graphics/fonts/latin_normal.png` is the main UI/dialogue font. It was
  replaced with Lhea's PurrfectDoodle (FRLG letterforms, RSE spacing). Verified
  drop-in: identical dimensions, identical 4-colour palette, identical output size.
- Font width/spacing is baked into the sheet and into `charmap.txt`. Changing
  widths means re-checking every UI, not just dialogue.

### 6.2 Type icons — there are TWO separate systems

Do not confuse these:

| system | size | file | used by |
|---|---|---|---|
| move types | **32×16** × 25 | `graphics/types/move_types.4bpp` | summary screen, DexNav, Pokédex+HGSS |
| battle icons | **8×16** | `graphics/types/battle_icons1/2.png` | the tiny icons in battle |

Only the first has been modernised. It lives in a **separate** directory
(`graphics/types_modern/`) so the vanilla sheet survives untouched and reverting
is a one-line change in `src/graphics.c`. Follow that pattern for any future
asset swap: **add a new directory, repoint the C symbol, leave vanilla alone.**
It makes every change trivially revertible and lets you A/B two versions.

The 8×16 in-battle icons are untouched. Nothing in the Aquas pack fits them; the
right answer is to author new art at that size, not to downscale 32×16 art.

### 6.3 Overworld Pokémon sprites

- Path pattern: `graphics/pokemon/<species>/overworld.png`, plus
  `overworld_normal.pal` and `overworld_shiny.pal`.
- Wired up in `src/data/graphics/pokemon.h`:
  `INCGFX_COMP(".../overworld.png", ".4bpp", "-mwidth 4 -mheight 4")` → 32×32
  frames. Larger Pokémon use bigger `-mwidth/-mheight` (Kyogre uses 8×8 → 64×64).
- **Frame count is derived from the data size, not hardcoded** —
  `overworld_ascending_frames()` sets `relativeFrames = TRUE`
  (`include/sprite.h:36`). So a size change silently changes the animation
  instead of erroring. **Only replace art whose pixel dimensions match exactly.**
- Palette rules, all verified the hard way:
  - **Index 0 is transparent** and must stay index 0. In practice ~80% of pixels
    are index 0 — the sprite is a tight drawing on an empty background. The RGB
    stored at index 0 is irrelevant (never displayed); do not "fix" it.
  - Normal and shiny palettes must be **index-aligned** with each other, or
    shiny Pokémon render with the wrong colours. The game's own normal/shiny
    pairs are aligned; a third-party pack's may not be.
  - Third-party OW packs are often 8-bit PNGs. That is fine, but they must be
    rewritten to 4bpp, and **only if the art actually uses ≤16 colours** —
    otherwise it is a lossy quantise and the sprite should be skipped, not
    approximated.
- Current state: 744 of 1025 species replaced. 282 have no art in the source
  pack, 59 are a different size, 239 were skipped because the pack's own palette
  contradicted its own art. Skipped species keep working vanilla art.

### 6.4 Tilesets and maps — read before touching

- Map geometry lives in `data/layouts/<name>/map.bin`: a flat array of `u16`,
  each `metatileId | collision<<10 | elevation<<12` (masks in
  `include/global.fieldmap.h`). It contains **only metatile ids**.
- Metatile *behaviour* is not per-map. It is read from the tileset's
  `metatileAttributes` (`src/fieldmap.c`), so **behaviour is global per
  metatile** — changing it changes every map using that metatile.
- ⚠️ **`metatile_attributes.bin` files are sparse and misleading.** Example:
  `primary/building/metatile_attributes.bin` is 16 bytes (8 entries) while 179
  Building-primary layouts reference metatile ids up to 738. Sizes are always
  `metatiles.bin / 8`. `include/constants/metatile_labels.h` is stale and names
  ids that no longer exist for the shrunken tilesets.
  **Consequence:** writing behaviours into these binaries either does nothing
  (the id is past the end) or, if you extend the file, clobbers whatever follows
  it in the ROM. Use **porymap** (`porymap.project.cfg` is already configured).

### 6.5 Sideways stairs — engine present, not activated

The sideways-stairs feature is **fully implemented and compiled in**:
`MB_SIDEWAYS_STAIRS_*` in `include/constants/metatile_behaviors.h`, the
direction remap in `src/field_player_avatar.c` (`GetRightSideStairsDirection` /
`GetLeftSideStairsDirection`), and the collision rules in
`src/event_object_movement.c`.

**No map uses them, so nothing is visible in game.** Activating requires setting
those behaviours on stair metatiles — which §6.4 says to do in porymap, not by
editing binaries.

Left/Right are mirror images. A sensible mapping once real stair metatiles are
identified: `StairsTop → MB_SIDEWAYS_STAIRS_RIGHT_SIDE_TOP`,
`StairsBottom → MB_SIDEWAYS_STAIRS_RIGHT_SIDE_BOTTOM`.

Known cases to **not** touch: `METATILE_SootopolisGym_Stairs` (puzzle),
Trick House stair metatiles (slide puzzle), Secret Base slide/stand stairs
(decorations).

## 7. Third-party assets

Assets in this repo come from
[Team Aquas Hideout's asset repo](https://github.com/TeamAquasHideout/Team-Aquas-Asset-Repo)
and carry their own credits — do not strip them:

- **Overworld sprites** — compiled by *Rahtak* for *Merrp's* follower system,
  from many artists. Rahtak's README credits MissingLukey, cSc-A7X, Artrios,
  LarryTurbo, princess-pheonix, SageDeoxys, princess-phoenix and others. Read
  `Overworld Pokemon Sprites/Rahtak/README.md` in the asset repo before
  redistributing.
- **Type icons** — "Lhea's Modern-Styled Type Icons", based on icons by
  *Squid Eevee* (spriters-resource 170390).
- **Font** — *PurrfectDoodle*.

Keep a note of provenance (source URL + author) for anything new, ideally in the
asset's own directory or in CREDITS.md.

## 8. Conventions

- Match the surrounding code. The project is C89-flavoured with expansion's own
  brace style; do not reformat files you don't need to touch.
- Feature toggles live in `include/config/*.h` and are commented. Add new ones
  there rather than scattering `#define`s.
- `make check` runs the test suite. Run it after anything touching battle logic,
  and prefer adding a test over asserting something works.

---

## 9. Project instructions

*(mine — add to this)*

### Still to decide

- Region / map / story direction for "Sea-Shore"
- How modern should the presentation be vs. vanilla Emerald? (Gen 4/5 styling is
  already in via the sideways-stairs engine support; the type icons are a
  deliberate departure.)
- Which Pokémon are actually obtainable — do we stay near the vanilla 386, or
  expand the pool?
- Should followers be enabled? `OW_FOLLOWERS_ENABLED` is currently `FALSE` in
  `include/config/overworld.h`. Enabling it is a one-line change and the new
  sprites are already installed.

### Working agreements

- (none recorded yet)