# Item names — save IDs to 1.14d tables

How a .d2s item's quality data (components/d2s/d2s_items.hpp) turns into
its displayed name. Verified by matching every item in 19 real 1.14d saves
against the 1.14d tables (installer-patched, see mpq-archives.md): each
mapping below is the only one of the candidates tried where every item's
row is allowed on that item (affix `itype` columns vs the item's
ItemTypes ancestry; unique/set `code`/`item`; runeword rune sequence vs
the socketed runes).

| quality        | ID bits            | row                                                        |
|----------------|--------------------|------------------------------------------------------------|
| 4 magic        | 11 prefix, 11 suffix | MagicPrefix / MagicSuffix **raw** data row — the blank "none" row is 0 and "Expansion" separators count (252/252, 223/223) |
| 5 set          | 12                 | SetItems row, separators dropped (36/36)                    |
| 7 unique       | 12                 | UniqueItems row, separators dropped (91/101; the other 10 are cube-upgraded bases, e.g. Shaftstop on an elite `uhn`) |
| 6 rare, 8 crafted | 8 + 8           | RarePrefix row `id - 156`, RareSuffix row `id - 1` (88/88)   |
| runeword       | 12                 | Runes.txt `RunewordN` rows sorted by N (80 and 96 don't exist): rank `id - 27` (11/11; 2718 is Delirium's special ID) |

Name strings are the row's key column (`index`, `Name`, `name`) looked
up in patchstring → expansionstring → string; base names are
armor/weapons/misc `namestr`. Two-line names (set, unique, rare,
runeword) show the base item on the second line.

## Property lines

The rest of the hover text comes from the item's stat list, described by
ItemStatCost's `descfunc`/`descval`/`descstrpos`/`descstrneg`/`descstr2`,
highest `descpriority` first (d2d `prop_lines`). Repeats of a
(stat, param) sum first, like the unit's stat list. `dgrp` groups
(all resistances, all attributes) print once when every member is
present with the same value. Min/max damage pairs are hard-coded to the
`strMod*DamageRange` strings ("Adds %d-%d fire damage"), poison scaled
by length/256 over length/25 seconds; 17+18 with equal values become
"+X% Enhanced Damage". Per-level stats (op 2..5) show `value * clvl >>
op param`.

Filled sockets add their bonuses: a jewel its own properties, a gem or
rune its gems.txt `weaponMod`/`helmMod`/`shieldMod` row (shield when the
parent's ItemTypes ancestry reaches `shld`), each mod code resolved
through Properties.txt (`func` 1/3 = value, 15/16 = min/max, 17 = param,
5/6/7 = min/max/% damage). Runewords get their rune bonuses this way —
the save only stores the runeword's own list.

## Hover text

game.exe's `FUN_0048dd90` (UI\inv.cpp) builds an item's tooltip as one
string drawn bottom-up: each helper appends its line(s) after the ones
below it, so the call order is bottom to top. Top line first (d2d
`item_lines`):

| Line | Function | String | Shown when / colour |
|---|---|---|---|
| name | `FUN_0048c060` | — | quality colour (as before) |
| Defense: N | `FUN_00485ee0` | 3461 | armor, total > 0; N blue when ≠ base |
| Chance to Block: N% | `FUN_00485be0` | 11018, `"%d%%"` 0x6d9bec | shields: block column + stat 20 + class BlockFactor, cap 75; blue over the column |
| Smite / Kick Damage: a to b | `FUN_00485d40` | 3468 / 21782, "to" 3464 | Paladin + shield (no other class's) / Assassin + boots; armor.txt mindam/maxdam |
| Throw / One-Hand / Two-Hand Damage: a to b | `FUN_00485410`, `FUN_00485240` | 3467 / 3465 / 3466 | weapons: stats 21-24/159-160 (`LAB_0062d300`: low quality ×3/4 floor 1/2, ethereal ×3/2), min ×(100+18)%, max ×(100+17+219·clvl/8)% + 218·clvl/8; throw line for ItemTypes Throwable; Barbarian + 1-or-2-handed gets both; numbers blue when raised |
| Quantity: N | `FUN_00486100` | 3462 | identified, not socketed, stacks |
| spelldesc | `FUN_00486370` | misc.txt spelldescstr | replaces quantity; mode 2 scales potions by class (`FUN_0062a5d0` life, `FUN_0062a620` mana: ×1.5 Ama/Pal/Asn, ×2 Barb life / Sor/Nec/Dru mana) |
| Keep in Inventory to Gain Bonus | — | 20438 | charms |
| socket filler | `FUN_004865d0`, `FUN_004e6850`, `FUN_004e67d0`, `FUN_004e6410` | 11080; 11075/11076/11073/11074 | type sock; gems/runes add Weapons/Armor/Helms/Shields groups (gems.txt), the label + " " (3995) before each group's top line |
| Durability: a of b | `FUN_00484e90`, `FUN_00629930` | 3457, "of" 3463 | durability column, not nodurability / indestructible / throwable; b blue with stat 75 |
| (Class Only) | — | 10917 + class | class items; red for another class (`FUN_0062eaf0`) |
| Required Dexterity / Strength: N | `FUN_00485170` / `FUN_004850a0` | 3459 / 3458 | weapons/armor: column + column·stat91/100 − 10 ethereal; red when unmet |
| Required Level: N | `FUN_00484ff0`, `FUN_0062b5b0` | 3469 | identified, > 1: affix/set/unique levelreq (crafted +10+3/affix, ≤ 98), base, sockets, + stat 92 |
| Class - Speed | `FUN_004861d0`, `FUN_0062a710` | class table 0x721eb0, " - " 3996, 4088 + band | weapons with a player: frames<<8 / ((100+IAS−WSM)·animspeed/100) of the class's A1 anim, banded by DAT_00721f10 (rows 10..27) and column 0x722078; speed blue with stat 93 |
| Unidentified | — | 3455 | red |
| properties | `FUN_004e6410` | | blue |
| Ethereal (Cannot be Repaired), Socketed (N) | `FUN_00484b10` | 22745, 3453 | blue |

Not yet: set item / set bonus lists (`FUN_004e6410` flags), the gold
line (`FUN_00486670`), Holy Shield's block and smite, time-of-day stats
268-273, throwing potions' missile damage, charged skill details.

## Hover helpers (UI\inv.cpp, traced)

- `FUN_00452170` (3 bytes, `__fastcall`): returns ECX unchanged, an identity
  function with no side effects. Its callers are `FUN_0048c060` (right
  after `FUN_00484c90`), `FUN_0048cff0`, `FUN_0048d1d0`, `FUN_0048db80`,
  `FUN_0048dd90` and `FUN_00497a40`; the decompile drops the ECX argument,
  so what they pass is not read. Porting it needs nothing. d2d: not
  ported, no mention. Cited: read from
  `python3 tools/ghidra/fn.py 00452170` and
  `python3 tools/ghidra/calls.py 00452170` (no callees).

- `FUN_0044dcd0` (6 bytes): returns the byte `DAT_007a060c`, the client's
  current difficulty (0 Normal, 1 Nightmare, 2 Hell). Evidence: Akara's menu
  `FUN_004b4830` compares it with 2 beside quest 41 (quests.md), the oracle
  `tools/emu/skill_tips.py` sets that address as "FUN_0044dcd0's difficulty",
  and `FUN_0048c060` (below) compares it with stat 0x164, the difficulty a
  questdiffcheck item was made in. It has about 25 callers
  (`game-calls.tsv`); only these three were read. d2d: has no
  function of that name; its difficulty (quests.md:176) reads the same,
  cited from the decompile, no oracle run. Cited.

- `FUN_0048c060` (UI\inv.cpp, the name line; `item_lines` ports its quality
  colours) ends with a quest-item recolour, read at its last 15 lines.
  After the name is built (`FUN_00484c90`, `FUN_00452170`), with `row` the
  item's Items.txt row (`FUN_006335f0`): if `row+0x12a` (quest) is set, then
  (a) if `row+0x12b` (questdiffcheck) is set and the item's stat 0x164
  (`FUN_00625480(item, 0x164, 0)`) is below `FUN_0044dcd0()`, it calls
  `FUN_004521c0` (the colour-code prefix, skill-tooltips.md) and returns;
  (b) otherwise, unless the code at `row+0x80` is `"leg "` (0x2067656c,
  Wirt's leg), it calls `FUN_004521c0` too. So quest items are recoloured,
  and a questdiffcheck item made in an easier difficulty than the current
  one is recoloured the same; Wirt's leg only in case (a). The colour code
  is in DL, which the decompile drops: not read, so which colour is open.
  d2d: not ported (`item_lines` has no quest-item branch; `rules.hpp:100`
  and `d2s_items.hpp:110` only carry stat 0x164 for the cube and save
  parsing). Cited.
- `FUN_0048cff0` (UI\inv.cpp, 475 bytes): `FUN_0048dd90` calls it when
  `FUN_0062b400(hovered item) == 0x12`, the tome type (`tome_use.py`
  header). It zeroes a 2 KB buffer. If the hovered item (`DAT_007bcbf4`)
  is not a tome, or `FUN_006335f0` finds no Items row, it fails through
  `FUN_00408090(0xcf1 / 0xcf5)`, `FUN_00408a60`, `FUN_00681e09(-1)`, the
  error exit `FUN_0048c060` also ends in. Otherwise it writes the name
  (`FUN_0048c060(0x80)`, `FUN_00452170`), then a second block: for hover
  mode `DAT_007bcbf0` 1..9 `FUN_0048cea0`, else `FUN_005267e0`. It
  measures the text (`FUN_00502520`, a value `n`) and draws it with
  `FUN_00502280` at `DAT_00721e40`, or at `DAT_00721e48 + n` when
  `DAT_00721e40 - n < 1`. `item_lines` has no tome-specific path (d2d:
  not ported). Cited; the second block (`FUN_0048cea0`) is not traced.

## Sockets

- Inserting (C→S 0x28, `FUN_0054b650` → `FUN_00562660`): the held item
  must be a socket filler (item type 53 `sock`: gems, runes, jewels) and
  identified; the target stored or worn, identified, socketed (0x800),
  able to hold sockets (Items +0x137) and with a free one.
- Runewords: after the filler goes in, `FUN_0062bed0` looks up the Runes.txt
  row (`FUN_006394a0`, 0x120 bytes: +0x80 complete, +0x81 server, +0x86
  itype1..6, +0x92 etype1..3, +0x98 Rune1..6, +0xb0 T1Code/Param/Min/Max
  x 7): not quality 4..9, not a quest base (Items +0x12a), every socket
  filled (`FUN_006299b0`), the fillers' classes Rune1.. in order, none of
  the etypes and one of the itypes (`FUN_00629bb0`). A `server` row needs a
  ladder game (game +0x74, `FUN_00562660`). `FUN_006600a0` (unless the item
  has stat list 0xab / 0x40) sets item flag 0x4000000 and puts each T1 mod
  through `FUN_0065fd70` kind 6 into that list. d2d: `rules::runeword_row`,
  `make_runeword` (the save id as item_text reads it; the seed is the
  item's fingerprint).
- The hover overlay (`FUN_00483e40`, a hovered identified socketed grid
  item, `Panel\GemSocket` = `0x7bef14`): per socket i a place in half
  cells from `0x6d91e8`, 48-byte records `[sockets-1][width-1][height-1]`
  of six {col, row}; the socket at x = (col - 1) * box_w / 2 + left - 1,
  bottom (row + 1) * box_h / 2 + top, the filler's inventory art over it.
- Worn items (`FUN_0048b540`): the same, from the item's cells centred
  in its body slot (the grid's cell size).
