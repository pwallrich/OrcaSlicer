# Shared toolheads — High Level Design

## Purpose and scope

A toolchanger without single-extruder multi-material, such as the Snapmaker U1
with four toolheads, normally holds one filament per toolhead. Shared toolheads
let a project use more filaments than the printer has toolheads. Each extra
filament is loaded into one of the physical toolheads. Filaments on the same
toolhead are either used on different plates, or swapped by hand during the
print. This document covers how a filament is assigned to a toolhead, how the
G-code addresses toolheads, what the printer is told about them, and where the
design stops.

## Assignment

`filament_map` (1-based, one entry per filament) names the toolhead of every
filament. The first filaments, one per toolhead, are fixed to their own toolhead.
Only the filaments after them can be assigned, through "Load into toolhead" in
the filament's menu in the sidebar. `toolchanger_toolhead_of_filament` reads the
map with that rule, so a filament below the toolhead count always reports its
own toolhead. The choice is written to the project map and to every plate's
map, because a plate's map overrides the project's once it has been sliced.

Sharing exists only when a filament is actually mapped to another filament's
toolhead. `GCodeWriter::has_shared_toolheads` is true only then, on a non-BBL,
non-SEMM, non-IMEX printer without `manual_filament_change` and with at least
two toolheads. Mixed filaments are not counted, since they print through their
components. The sidebar offers the assignment under the same conditions. Every behaviour below is gated by it,
so a printer whose filaments each have their own toolhead produces the same
G-code as before.

## G-code

Tool changes address the toolhead, not the filament: `T<toolhead>`. A change
between two filaments of one toolhead is a manual swap. `GCodeWriter::toolchange`
tracks which filament each toolhead holds. When the incoming filament differs
from the loaded one it emits a message and the printer's pause G-code:

```
; FILAMENT_TOOLHEAD 4
M117 Load filament 5 (PETG #FF0000) into the active toolhead
RESPOND MSG="Load filament 5 (PETG #FF0000) into the active toolhead"   ; Klipper only
M600                                    ; machine_pause_gcode, M600 if empty
M117                                    ; clears the message once the print resumes
M109 S235 T0                            ; the new filament's temperature on its toolhead
```

`RESPOND` needs a `[respond]` section in the Klipper config; without it Klipper
logs an unknown command and carries on, and the message still shows on the
display.

The swap pauses whether the toolhead also changes or not. `change_filament_gcode`
is skipped for a swap on the same toolhead. For a real head change its
placeholders are remapped to toolheads: `next_extruder`, `previous_extruder`, and
the toolhead slots of the temperature and `filament_type` arrays. Templates
written for one filament per toolhead, such as the U1's, keep working. The start
G-code sees `is_extruder_used`, the temperature arrays and `initial_tool` per
toolhead too. `GCode::m_toolhead_filaments` records the filament each toolhead
prints first, and those values come from it. When printing by object it covers
the filaments of every object, not only the first.

Temperature commands name the toolhead as well. `GCodeWriter::set_temperature`
maps a filament to its toolhead, the first-layer and second-layer temperature
passes skip filaments their toolhead does not hold at that point, and the ooze
prevention preheat is skipped for a swap on the same toolhead.

The `FILAMENT_TOOLHEAD` tag carries the filament id for `GCodeProcessor`, which
selects that filament and ignores the `T` that follows. The preview keeps one
colour per filament even though the file only names toolheads. The processor
uses the tag only for ids within its filament count. A saved file's config block
describes toolheads, so opening it again falls back to the `T` commands and
attributes every extrusion to its toolhead.

`Print::validate` warns when a plate uses two filaments on one toolhead, because
the print will pause. A mixed filament whose components share a toolhead is an
error, since it would pause on every layer.

## What the printer is told

The file addresses `T0` to `T<toolheads-1>` only, so the lists a printer reads
from the file describe toolheads, not filaments. In the config block, every list
with one entry per filament (a `filament_` key or a filament preset option) is
collapsed to one entry per toolhead. Each entry
holds the values of the filament that toolhead prints first, or of its own
filament when it is unused. `filament_map` becomes the identity, and
`flush_volumes_matrix` is reduced the same way. `GCodeProcessor` sums the
`filament used` statistics per toolhead, and the header's `; filament:` line lists
the toolheads in use.

This matters on the U1. Its Moonraker passes these lists to the firmware when a
print starts (`SET_PRINT_TASK_PARAMETERS`), and its screen asks which spool each
listed tool uses. With one entry per filament, the screen would offer a choice
for filament 5 that the file never reads.

Opening such a G-code file in the viewer on its own gives one colour per
toolhead. The project, not the file, holds the per-filament data.

## Printer-side head mapping

The U1 firmware keeps a table from logical tools to physical heads
(`extruder_map_table`, set at print start and by auto-replenish). `T<n>`,
`M104`/`M109 T<n>` and `SM_PRINT_PREEXTRUDE_FILAMENT` all go through it. Moving a
whole toolhead to another physical head at print start therefore works: every
filament on it moves along, the swap pauses stay correct, and the temperature
commands reach the head the filament is in. The swap message
names no toolhead for that reason. The swap always happens on the toolhead the
file has just selected, which the printer has already mapped, so "the active
toolhead" is correct after any remapping. `M117` and `RESPOND` print their text
as written; the printer translates tool numbers only in commands that act on a
toolhead. The message gives the filament's number and `filament_colour`, and its
`filament_type` when the print uses more than one type.

A single filament cannot be moved to another head on the printer. The pauses are
fixed at slice time from the grouping in `filament_map`, and a filament on a
shared toolhead has no tool number of its own in the file.

### Deciding swaps on the printer

Honouring a per-filament choice made at print start would need the printer to
decide when to pause:

- The file uses one logical tool per filament (`T4` for filament 5) and carries
  no pauses.
- A macro on the printer wraps every tool change. It looks up the target head in
  `printer.print_task_config.extruder_map_table`, compares the logical tool with
  the one that head last printed, and pauses with a message naming the physical
  head when they differ.
- The same macro could unload the old filament first, with
  `FEED_AUTO MODULE=<left|right> CHANNEL=<0|1> UNLOAD=1` for that head. On the
  U1, T0 is left channel 1, T1 left channel 0, T2 right channel 0 and T3 right
  channel 1. The unload moves the head to the discard position and switches
  heads with `A0`, which bypasses the map. The macro has to save and restore the
  G-code state and select the mapped head again afterwards.

The slicer would then no longer know where the pauses are. Its preview, swap
warnings and start G-code would describe a grouping the printer may change. Every
printer also needs the macro in its `printer.cfg`.
