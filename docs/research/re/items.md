# Item accessors and helpers

These functions take an item unit. They require a non-null unit of type 4
with a non-null item-data pointer at unit `+0x14`; rejected calls do not
write item data. `FUN_0048c060` uses the quality to select item-name
formatting, and the call sites cited below identify fields where a semantic
name is known.

The 16-bit getters write only `AX`, zeroing it on the rejection paths above.
On success the upper `EAX` bits come from the item-data pointer. A wrong-type
call instead retains the unit pointer's upper bits; null-unit and null-data
calls return zero.

| Address | Traced claim |
|---|---|
| `FUN_00627e70` | Read the 32-bit quality at item data `+0x00`; return 2 (normal) for a rejected call. |
| `FUN_00627ea0` | Write the 32-bit quality at item data `+0x00`; do nothing for a rejected call. |
| `FUN_00627ec0` | Read MagicPrefix ID `i` (0..2) from item data `+0x38 + 2*i`. `FUN_0048c060` formats slot 0 through `FUN_00633ee0`; `FUN_0055f5d0` compares slots 0, 1 and 2. |
| `FUN_00627ef0` | Write the low 16 bits of a value to MagicPrefix slot `i` at item data `+0x38 + 2*i`; do nothing for a rejected call. |
| `FUN_00627f20` | Read the 1-based AutoMagic ID at item data `+0x36`. `FUN_0062b5b0` includes it in required-level calculation through `FUN_00633ee0`. |
| `FUN_00627f50` | Write the low 16 bits of a value to the AutoMagic ID at item data `+0x36`; do nothing for a rejected call. |
| `FUN_00627fb0` | Write the low 16 bits of a value to MagicSuffix slot `i` (0..2) at item data `+0x3e + 2*i`; do nothing for a rejected call. `FUN_0055f5d0` compares suffix slots 0, 1 and 2 through `FUN_00627f80`. |
| `FUN_00627fe0` | Read the rare-name prefix ID at item data `+0x32`. `FUN_0048c060` formats rare and crafted names through `FUN_00634260`. |
| `FUN_00628010` | Write the low 16 bits of a value to the rare-name prefix ID at item data `+0x32`; do nothing for a rejected call. |
| `FUN_00628040` | Read the rare-name suffix ID at item data `+0x34`. `FUN_0048c060` formats rare and crafted names through `FUN_00634260`. |
| `FUN_00628070` | Write the low 16 bits of a value to the rare-name suffix ID at item data `+0x34`; do nothing for a rejected call. |
| `FUN_00628110` | Read the 32-bit item flags at item data `+0x18`; return 0 for a rejected call. |
| `FUN_00628170` | For a mask and boolean-like third argument, nonzero sets `item_data[+0x14] \|= mask` and zero applies `item_data[+0x14] &= ~mask`; do nothing for a rejected call. |
| `FUN_006281e0` | Read the signed item level at item data `+0x2c`, overwriting values below 1 with 1; return 1 for a rejected call. `FUN_00628220` is the corresponding item-level setter. |
| `FUN_00628280` | Write the low byte of a value to the inventory page at item data `+0x45`; do nothing for a rejected call. `FUN_00628250` is the corresponding getter; `tools/emu/auto_belt.py` exercises this field as the inventory page. |
| `FUN_00628320` | Write the low byte of a value at item data `+0x47`; do nothing for a rejected call. `FUN_006282f0` is the corresponding getter. |
| `FUN_00628340` | Return `item_data + 0x4a`, the ear-name string pointer; return 0 for a rejected call. |
| `FUN_00628370` | Copy a NUL-terminated source string to item data `+0x4a`, including the terminator; do nothing for a rejected call. `FUN_00567070` copies ear names through this setter. |
| `FUN_006283a0` | Read the ear-class byte at item data `+0x48`, writing only `AL`; rejected calls set `AL=1`. Successful calls retain the item-data pointer's upper 24 bits, wrong-type calls retain the unit pointer's upper 24 bits, and null-unit or null-data calls return 1. |
| `FUN_006283d0` | Write the low byte of a value to the ear-class byte at item data `+0x48`; do nothing for a rejected call. `FUN_00567070` sets the ear class through this function. |
| `FUN_00628420` | Write the low byte of a value to the picture ID at item data `+0x49`; do nothing for a rejected call. `FUN_00557a70` sets it from the item graphics code. |

## Item-record accessors and helpers

`FUN_006335f0` resolves the unit class at unit `+0x04` to the item record used
by the record accessors below. The oracle substitutes a synthetic record.

| Address | Traced claim |
|---|---|
| `FUN_00628500` | Read the 32-bit value at item record `+0xc0`; return 0 for a null unit, a wrong unit type, or a failed class lookup. |
| `FUN_00628590` | Return the 32-bit item code at item record `+0x80` for a valid item unit with a successful class lookup. `FUN_00483c50` compares this field with the Horadric Cube code `box `. |
| `FUN_006286c0` | Copy item record bytes `+0x10f` and `+0x110` to the two output pointers; leave them unchanged for a null unit, a wrong unit type, or a failed class lookup. The function returns with `RET 0x14`, so the oracle supplies the two additional assert arguments used by its call sites. |
| `FUN_00628740` | Read the base item level at item record `+0xfd`, writing only `AL` and retaining the record pointer's upper 24 bits. `FUN_005c1560` uses it as a floor when selecting an affix level. |
| `FUN_006287a0` | Return the bitwise AND of its two 32-bit arguments. |
| `FUN_006287b0` | Update the 32-bit value at the first argument: a nonzero third argument sets the bits in the second argument with `\|=`, while zero clears them with `&= ~mask`. |
| `FUN_006287d0` | For a valid item unit and class lookup, return item record `normcode` at `+0x84` when nonzero, otherwise `code` at `+0x80`. Invalid inputs take assertion paths. `FUN_0062efb0` uses this fallback when pricing gambled items. |
| `FUN_00628830` | Clamp an affix required level using a custom fastcall convention: `ECX` is the affix record, `EDX` the optional player unit, and `EAX` the initial value. A null affix preserves `EAX`. Select byte `+0x65`, or byte `+0x68` when byte `+0x67` is not `0xff`, the player is nonnull, and that byte equals the player class at `+0x04`; raise a signed initial value to the selected byte, but preserve higher values. `FUN_0062b5b0` calls this helper. |
| `FUN_00628860` | Resolve the data-table state through global pointer `0x744304`; index the item-type table at state `+0xbf8`, with count at `+0xbfc` and record stride `0xe4`. Return the zero-extended byte at item-type record `+0x10`, or 0 for a negative index, an index greater than or equal to the count, or a computed null record. `FUN_0055a6d0` uses the result when rerolling classic drops. |

Re-check these claims against game.exe 1.14d:

```sh
cd tools/emu && uv run python item_accessors.py
```

## A belt put on or taken off (`FUN_005608c0`)

The server runs it in two cases:

- A belt swapped in from the cursor (C->S 0x1d, `54af50` -> `FUN_00560f00`).
  After the swap passes `FUN_0063de60`, an incoming item of type 0x13
  (`FUN_0062b400`) calls it with the new belt.
- A belt taken to the cursor (C->S 0x1c, `54aec0` -> `FUN_00560cd0`). It
  calls it with no belt, so Belts.txt row 2 (one row) is used.

It reads the belt's row (`FUN_00621ed0`, its Belts.txt index;
`FUN_00660cb0`) for `numboxes`. Then, for box 0 up to 15, each item in a
box at or past `numboxes`:

1. Comes out of the belt (`FUN_0063ad90`), and the client is told
   (`FUN_0053eed0`).
2. Gets inventory page 0 (`FUN_00628280`) and goes to the inventory's
   free spot (`FUN_0063b950`, a player's best fit).
3. With no room, drops at the nearest free spot to the player
   (`FUN_00555da0`, `FUN_00558aa0`).

The swap itself checks nothing about the rows. The one belt check in the
0x1d handler, `FUN_00567840`, refuses the swap only while a trade with
another player is open (interaction type 0, `FUN_00554100`) and the belt
holds an item (`FUN_0063e020` page 2).

d2d: `rules::spill_belt`, called by the World's `cmd::ToBody` (belt
slot) and `cmd::ToCursor` (the worn belt).
## The player's client (traced 2026-11-06)

| Address | What |
|---|---|
| `FUN_005531c0` | Fastcall, unit in ECX. Returns 0 for a null unit or one whose type (`*unit`) is not 0 (player); else the player data's client field: `FUN_006221a0(unit)` (asserts 0xfb4 null, 0xfb6 wrong type; returns `unit+0x14`) read at `+0x9c`. d2d agrees: objects.cpp:378 calls it the unit's client. Ghidra shows constants as its argument at call sites (0xf94, 0x136d, 0x1370); quests-act5.md:284 reads such constants as assert source lines (not re-checked from the asm here). |

### Taking an item out of an inventory (traced 2026-11-06)


### ItemMode.cpp: the stat-use wrapper (traced 2026-11-06)

Both functions below belong to source file `.\ITEMS\ItemMode.cpp`; registers are read from the decompile only, not the asm.

| Address | What |
|---|---|
| `FUN_0055c270` | Fastcall, two arguments (unit in ECX, a second value in EDX). A null ECX asserts line argument 0x24d, a null EDX 0x24e (both through `FUN_00408090` / `FUN_00408a60` / `FUN_00681e09(-1)`, the exit path). Otherwise, only when the unit's type (`*unit`) is 0 (player), it calls `FUN_0055c110(1)` and returns; other unit types do nothing. `FUN_0055c110` (not traced here) reads a stat (0x46) and the player's skill state via `FUN_006439b0`, and sends through `FUN_0053c520`. Callers: `FUN_0055c2c0`, `FUN_0055e9b0`, `FUN_0055eb30`, `FUN_005600a0`, `FUN_00560200`. d2d: not ported (no mention in `components/` or `apps/`). |
| `FUN_0055c2c0` | Item mode change. Decompile order: item in EDX, then a unit `U` and a flag on the stack (ECX unused). Returns for a null `U` or item, an item not of type 4, or a type-0 `U` with state 7 (`FUN_00639df0`). Then by item type (`FUN_00629bb0`): rows 0x14 and 0x4a each require `*U == 4` and the item's class record (`FUN_006335f0(item+4)`, then `FUN_006372c0(record+0xf0)`) and call `FUN_0065fec0` (kind 2 for row 0x14, 5 for row 0x4a); any other item of quality 5 (`FUN_00627e70`) goes to `FUN_00663cc0(U, item, 0, 0)`. Then `FUN_0055c270`; `on` = 1, except 0 when `FUN_006235a0(U)` is true, the item is not `FUN_0063bef0(U+0x60)` and is of row 0x2d. `FUN_00627910(U, item, on)` runs if the item flag word `+0x18` & 0x100 (`FUN_006280a0`, assert line 0x2b8 when `FUN_006235a0(U)`, else 0x2c1) is clear or the flag argument is nonzero. Last, a nonzero `U+0x2c` calls `FUN_00623f50(U)` (assert 0x2cf). Callers: `FUN_0055c460`, `FUN_0055d970`, `FUN_0055dbc0`, `FUN_0055e9b0`, `FUN_0055eb30`. Not checked against the asm. d2d: not ported. |

### Item to its table row: the unique / set index and the SetItems row (traced 2026-11-06)

Rows in call order (callee first). The SetItems table is `[0x744304] +0xc18` (rows), `+0xc1c` (count), stride `0x1b8`; the Sets table is `+0xc0c` / `+0xc10`, stride `0x128`.

| Address | What |
|---|---|
| `FUN_00629da0` | Fastcall-looking, one argument (a unit). Returns the item's named-row index: `*(unit->data + 0x28)` when the unit's type (`*unit`) is 4 (item) and its data pointer (`unit[5]`) is non-null. Otherwise it asserts and exits: a null unit 0x1070, a non-item 0x1071, a null data 0x1073 (`FUN_00408090` / `FUN_00408a60` / `FUN_00681e09(-1)`). The index is a UniqueItems row for quality 7, a SetItems row for quality 5 (the Cube input check reads it as "the named row", cube.md). 74 bytes; 5 direct callers in the work list (`FUN_00486770`, `FUN_0048c060`, `FUN_0048d1d0`, `FUN_004c16d0`, `FUN_004c17f0`). d2d: agrees, the field is `Item::set_id` / the unique index in the parsed item; the assert numbers are not ported. |
| Address | What |
|---|---|
| `FUN_0055ed30` | Fastcall (item in the third argument, an owner unit in EDX; ECX is not read). Clears bit 1 of item `+0xc4` (`[0x31]`), then `FUN_0063ad90(owner+0x60, item)`: null if the inventory is null, or the unit is not type 4 (item), or `FUN_0063ad50` says it is not in that inventory; else `FUN_0063aaf0()` and the item back. Null returns assert 0x12e9; a returned unit other than `item` asserts 0x12ea (both through `FUN_00408090` / `FUN_00408a60`, then exit). On success: class = item data `+0xc` for unit types 2, 4, 5, else 0 for a null data pointer, else `FUN_006488c0(data)`; `FUN_0063c550(inventory, class)` (empty body in the decompile: a stub, argument count unconfirmed); then `FUN_00557fd0` (`FUN_005538d0` calls a callback for each of 128 buckets at game `+0x1120`, then `FUN_00555600`). d2d: not ported. |
