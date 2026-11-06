# Table loaders in `FUN_00619300` (game.exe 1.14d): CharTemplate, Experience, LowQualityItems

`FUN_00619300` (404 bytes, `DATATBLS/DataTbls.cpp` family) is the start-up
list of excel table loaders: it calls about 55 loaders in a row, passing its
first argument on to some of them. This note covers the three loaders and
helpers of that list traced so far. It is a trace only: d2d ports none of
them (`git grep -i -E "chartemplate|0096d600|00664b80|00613d30|00637500"`
in `components/` and `apps/` finds nothing).

The three loaders read here build a stack array of 5-dword column descriptors and pass it to
`FUN_006122f0(ctx, "<table>", descriptors, &count, rowSize)`. Reading the
descriptors in `FUN_00664b80`, `FUN_00613d30` and `FUN_00637500`, a
descriptor is `{ column name, type, size, offset, extra }`:

- the name is a string literal, or a shared pointer (`DAT_006ea354`, the
  first column of the table, type 1 = string, size = buffer length, offset 0);
- the array ends with a descriptor whose name is `DAT_006d9060`;
- `extra` is a lookup table for the lookup type (`class`, 0xd).

This layout is read from the decompiled stores only (the ECX/EDX and stack
arguments were not checked in the disassembly): **cited**, not proven.

## FUN_00664b80: the CharTemplate loader

- Caller: `FUN_00619300` (right after `FUN_006649b0`). `__fastcall`, its
  ECX is passed on to `FUN_006122f0`.
- Table name `"chartemplate"`, row size `0xf0`. The array pointer goes to
  `DAT_0096d600`, the row count to `DAT_0096d604`; then `FUN_00664b10`
  (below) runs on the array.
- Columns and row offsets, in the order the function writes them:

| Column | Offset | Type |
|---|---|---|
| (first column, `DAT_006ea354`) | 0x00 | string, 0x1d chars |
| `class` | 0x1e | 0xd, lookup in `PTR_DAT_00744304 + 4` |
| `level` | 0x1f | 4 |
| (`DAT_006ecc34`) | 0x20 | 4 |
| (`DAT_006e67f8`) | 0x21 | 4 |
| (`DAT_006e67f4`) | 0x22 | 4 |
| (`DAT_006e67f0`) | 0x23 | 4 |
| (`DAT_006e67ec`) | 0x24 | 4 |
| (`DAT_006e1614`) | 0x25 | 4 |
| `Hitpoints` | 0x26 | 4 |
| `ManaRegenBonus` | 0x27 | 4 |

Further columns, in the order the function names them: `Velocity`,
`AttackRate`, `OtherRate`, `RightSkill`, `Skill1`..`Skill9` each followed by
`SkillLevel<n>`, then `item1`..`item15` each followed by `item<n>loc` and
`item<n>count`. The last descriptor (`item15count`) has offset 0xed and the
one before it (`item15loc`) 0xec. Offsets between 0x28 and 0xeb were not
copied here; read them with the command below. The columns listed here use
type 4; its width was not checked. The
`item<n>*` strings make CharTemplate a starting-kit table (items, skills
per class and level).

`docs/research/re/todo.md` ("Tables") says CharTemplate is **not read by
game.exe at runtime**. This trace shows it **is loaded** by
`FUN_00619300` and indexed (below). Whether anything reads
`DAT_0096d600` / `DAT_0096d500` afterwards was not checked here: no xref
tool was available in this run. So the todo line stays, but "not read" is
at most "loaded and indexed; no reader found yet".

## FUN_00664b10: the level index over CharTemplate

- `FUN_00664b10(rows)`; the row count is in EDI (the decompiler shows an
  unaffected register, `unaff_EDI`; the one caller is `FUN_00664b80`, which has
  just stored the count in `DAT_0096d604`).
- `_memset(&DAT_0096d500, 0, 0x100)`, then walks the rows (stride 0xf0) reading
  the byte at offset `0x1f` (`level`). It keeps the highest level seen; the
  first time a level appears, `DAT_0096d500[level]` (a dword array) is set
  to that row's index.
- A row whose level is **lower** than an earlier one is fatal: it calls
  `FUN_00408090(0x5c)`, `FUN_00408a60(&DAT_006cc837, ...)` and
  `FUN_00681e09(-1)`, the same assert-and-exit sequence as elsewhere, so
  the rows of CharTemplate must be sorted by level.
- The index holds 64 dwords (0x100 bytes); a level above 63 would write
  outside it. The shipped table's levels were not read here.

## FUN_00613d30: the Experience loader

- Caller `FUN_00619300`. Table name `"experience"`, row size `0x20`.
  Result in `DAT_0096c8a8`; the count pointer is 0 (not kept).
- Columns: `Amazon` 0x00, `Sorceress` 0x04, `Necromancer` 0x08,
  `Paladin` 0x0c, `Barbarian` 0x10, `Druid` 0x14, `Assassin` 0x18
  (type 2 each), `ExpRatio` 0x1c. The row index is the level.
  The column order is also the class id order 0..6 times four bytes.

## FUN_00637500: the LowQualityItems loader

- Caller `FUN_00619300`. Table name `"lowqualityitems"`, row size `0x22`.
  Array in `DAT_0096cc5c`, count in `DAT_0096cc58`.
- One column: the first one (`DAT_006ea354`, string, 0x1f chars, offset 0).
- Then, in the loop read from the decompile (other paths not checked),
  `FUN_00524d30(row, &local)` looks the name up as a
  string key (patch, expansion, then base `.tbl`: the function adds 10000 or
  20000 for the first two) and stores the 16-bit result at row offset
  `0x20`.

## Small functions

- `FUN_00681ef0` is the CRT `_memset` (Visual Studio 2005/2008 library match
  by Ghidra; the body writes bytes up to a 4-byte boundary, then dwords of
  `byte * 0x01010101`, then the tail; for a zero fill of more than 255 bytes
  it uses `__VEC_memzero` when `DAT_00994c88` is set). It has many callers.
- `FUN_0041d2c0(ptr, size, byte)` is a wrapper: `_memset(ptr, byte, size)`.
- `FUN_0066a270` is an empty function (1 byte, `ret`), called once by
  `FUN_00619300` between `FUN_00654f20` and `FUN_00669130`.
- `005553cb` has size 0 in the work list and is not a function; it was not
  traced.
- `FUN_00619300` itself only calls the loaders in the order listed in its
  body; the order is the load order of the tables.

## Re-check

```
python3 tools/ghidra/fn.py 00664b80 | tail -12     # "chartemplate", 0xf0, DAT_0096d600/604
python3 tools/ghidra/fn.py 00664b10                # level index, byte at +0x1f, stride 0xf0
python3 tools/ghidra/fn.py 00613d30 | tail -8      # "experience", 0x20
python3 tools/ghidra/fn.py 00637500                # "lowqualityitems", 0x22
python3 tools/ghidra/fn.py 00681ef0                # _memset
```

d2d: not ported (no mention in `components/`, `apps/` or `tests/`).
