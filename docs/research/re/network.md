# Network — packets and the server loop (1.14d game.exe)

What d2d's client/server split follows (docs/design/multiplayer.md).
Single player runs the server inside the client process. The client's
main loop (`FUN_0044cf20`, `FUN_0044efa0`) calls the server's frame and
flush passes itself, and the two sides trade the same packets a
TCP / Battle.net game uses.

## The server loop

- **Game frames** (`FUN_0052fc20`): by `timeGetTime`, once
  `DAT_00883d60` ms have passed. That is `1000 / fps`, set by
  `FUN_0052df80` (reached through a pointer; the rate passed isn't
  traced, 25 per the animation and sound timings). With `param_1` set it
  catches up at most one frame. Every live game slot (`DAT_00882d38`, 0x400
  of them, so up to 1024 games) is locked (`FUN_0052e860`), gets a speed
  factor (game +0x1dbc, 0x400 = 1.0, from the real time since its last
  frame, clamped 10..0x800), and runs one frame.
- **One frame** (`FUN_0052d870`): frame counter + 1 (game +0xa8), then in
  order `FUN_0052d7b0`, `FUN_0052d720`, `FUN_0052d160`, `FUN_005414d0`,
  `FUN_0052d440` (132 lines, the unit updates), `FUN_0053b000`,
  `FUN_0053a820`. Then every 20 frames the quests (`FUN_00543e10`), every
  12 `FUN_0052d240`, every 11 each act's `FUN_0061aa20`, and last
  `FUN_0052d310`. The subsystems aren't traced one by one.
- **Flush** (`FUN_0052fd90`): at most every 40 ms (0x28) unless forced.
  Each game is locked, and each client's queued outgoing packets are sent
  (`FUN_0052e320` → `FUN_0052b330`; three failures in a row drop the
  client). A game with nobody in it for 300000 ms (5 min) is closed.
- Clients' packets are queued as they come and handled by the dispatcher
  below, not from the socket callback. That is the "commands in, then
  the tick" rule.

## Client → server

- **Sizes** (`DAT_00730dc0`, u32 per id 0x00..0x70; `FUN_0052bc20`): 0
  means not valid, −1 variable: 0x14 / 0x15 are chat, 0x66 is `u16 len +
  3`, 0x6c is the save upload. `FUN_0052b100` frames them. Ids under 0x67
  are game packets (class 1); 0x67..0x70 are join and system packets
  (class 0); anything over 0x70 other than 0xff is refused.
- **Dispatch** (`FUN_0054d750(game, player unit, packet, len)`): table
  `DAT_006e0d18`, 8 bytes an id: handler, then a flag (1 on most item /
  NPC ids; not traced). Ids 0 and over 0x66 are refused. A player who's
  busy (`FUN_0057eec0`) gets nothing through except 0x14, 0x15, 0x3c,
  0x43 and 0x66. 0x41 only goes through while the player's mode is 0x11
  (dead).
- **Layout**: `[id]`, then the fields, little-endian. A 5-byte packet is
  `u16 x, u16 y` (subtiles); a 9-byte one is `u32 unit type, u32 unit id`.
  Unit types: 0 player, 1 monster, 2 object, 3 missile, 4 item, 5 warp.
  Handlers refuse a type over 5 or a wrong length (return 2 / 3).

Names: the community's (d2-clientless, D2MOO) — unverified labels
(re/unverified.md); ✓ = the handler was read and agrees.

| Id | Size | Handler | What |
|---|---|---|---|
| 01 | 5 | 0x5497e0 ✓ | walk to x, y (`FUN_005809d0`, mode 2) |
| 02 | 9 | 0x549890 | walk to unit |
| 03 | 5 | 0x5498d0 | run to x, y |
| 04 | 9 | 0x549920 | run to unit |
| 05 | 5 | 0x549d00 ✓ | left skill at x, y |
| 06, 07, 09, 0a | 9 | 0x549d80.. | left skill on unit (variants) |
| 08 | 5 | 0x549e80 | left skill at x, y (shift) |
| 0c, 0f | 5 | 0x549fc0, 0x54a140 | right skill at x, y |
| 0d, 0e, 10, 11 | 9 | 0x54a040.. | right skill on unit |
| 13 | 9 | 0x54aa90 ✓ | interact with unit (type ≤ 5, id) |
| 14, 15 | var | 0x54a290, 0x54a5d0 | overhead / chat message |
| 16 | 13 | 0x54aad0 ✓ | pick up item (type, id, action) |
| 17 | 5 | 0x54ab40 | drop item: the cursor's item (its id at +1) at the nearest free spot to the player (`FUN_00563c00` → `FUN_00555da0`); refused while dead or trading. d2d: `cmd::Drop` |
| 18..1f | | 0x54abb0.. | item to buffer / body / swap / switch |
| 20 | 13 | 0x54b1e0 | use item |
| 21..26 | | | stack; belt: to, remove, switch, use |
| 27 | 9 | 0x54b280 | identify item |
| 28 | 9 | 0x54b650 | socket an item |
| 29, 2a | 9 | | scroll to book; item to cube |
| 2f, 30 | 9 | 0x54b930, 0x54b9f0 ✓ | start / end NPC chat |
| 31 | 9 | | quest message |
| 32, 33 | 17 | | buy / sell |
| 34 | 5 | | Cain identifies |
| 35 | 17 | | repair |
| 36 | 9 | | hire merc |
| 38 | 13 | | NPC action (gamble, trade menu) |
| 3a | 3 | 0x54bd10 | add stat point |
| 3b | 3 | 0x54bd90 ✓ | add skill point (a skill lookup, `FUN_006439b0`) |
| 3c | 9 | 0x54be70 | select skill (let through when busy) |
| 3d | 5 | 0x54bf10 | object action (`FUN_005845d0`) |
| 3f | 3 | | character phrase |
| 40 | 1 | | update quests |
| 41 | 1 | 0x54c0e0 | resurrect (only when dead) |
| 44..47 | | | staff in orifice; merc interact / move |
| 48 | 1 | | busy state off |
| 49 | 9 | 0x54c5d0 | waypoint |
| 4b | 9 | 0x54c6d0 | request unit update |
| 4c | 5 | 0x54c760 | transmute (cube) |
| 4f | 7 | 0x54c7c0 | click button (trade, stash, ...) |
| 50 | 9 | 0x54c800 | drop gold |
| 51 | 9 | 0x54c870 | bind hotkey |
| 53, 54 | 1 | | stamina on / off (run) |
| 58 | 3 | | quest completed |
| 59 | 17 | | move unit |
| 5d, 5e | 7, 6 | | squelch / hostile; party |
| 5f | 5 | 0x54cd50 | update player position |
| 60 | 1 | 0x54ce70 | swap weapons |
| 61, 62 | 3, 5 | | merc item; resurrect merc |
| 63 | 5 | 0x54d520 | item to belt |
| 66 | var | 0x54d740 | warden |
| 67, 68 | 46, 37 | (not in the table) | join game |
| 69 | 1 | | leave game |
| 6d | 13 | | ping |

## Server → client

The client's table `DAT_007114d0`: 12 bytes an id, `{handler, size,
second handler}` for 0x00..0xb5 (−1 variable; `0x45c900` is the no-op).
Ids 0x0c..0x10, 0x17, 0x24, 0x25, 0x4c, 0x4d have a unit handler
third; the table beyond 0xae holds other data.

| Id | Size | What (community names) |
|---|---|---|
| 00..06 | 1, 8, 1, 12, 1, 1, 1 | loading, game flags, load done, load act, load complete, unload, exit |
| 07, 08 | 6 | map reveal / hide |
| 09 | 11 | assign warp |
| 0a | 6 | remove unit |
| 0b | 6 | handshake |
| 0c | 9 | monster hit |
| 0d | 13 | player stop |
| 0e | 12 | object state |
| 0f, 10 | 16 | player moves (to x, y / to unit) |
| 11 | 8 | report kill |
| 15 | 11 | reassign player (position) |
| 18..20 | 15, 2..6, 10 | life / mana, gold, experience, attributes |
| 21..23 | 12, 12, 13 | item skill, skill level, set skill |
| 26 | var | chat |
| 27..2a | 40, 103, 97, 15 | NPC info, quest info, game quest info, NPC transaction |
| 2c | 8 | play sound |
| 3e, 3f | var, 8 | item stats; stackable used |
| 42 | 6 | clear cursor |
| 47, 48 | 11 | relators |
| 4c, 4d | 16, 17 | unit casts a skill (on unit / at x, y) |
| 4e, 4f | 7, 1 | merc for hire; merc list |
| 51 | 14 | assign object |
| 52 | 42 | player quest info |
| 53 | 10 | darkness |
| 57 | 14 | monster enchants |
| 58 | 7 | open UI |
| 59 | 26 | assign player |
| 5a | 40 | event messages |
| 5b, 5c | var, 5 | player joins / leaves |
| 60 | 7 | town portal state |
| 63 | 21 | waypoints |
| 67..6d | 16, 21, 12, 12, 16, 16, 10 | monster move, move to unit, state, update, action, attack, stop |
| 76, 77 | 6, 2 | player in proximity; button actions |
| 81, 82 | 20, 29 | assign merc; portal owner |
| 89..8f | | unique events, NPC wants to talk, relationship, party, corpse, pong |
| 94..97 | var, 13, 9, 1 | skill levels, life / mana / stamina, walk verify, weapon switch |
| 9c, 9d | var | item in the world / owned |
| 9e..a0 | 7, 8, 10 | merc attributes |
| a7..a9 | 7, var, 7 | delayed state, set state, end state |
| aa | var | add unit |
| ab, ac | 7, var | monster heal; assign monster |
| ae | var | warden |

## What d2d takes from this

- The **message shapes** for `Command` (client → server): walk / run to
  x, y or a unit, left / right skill at x, y or on a unit, interact
  (operate) with a unit, pick up / drop / move items, stat and skill
  points, select skill, NPC chat, buy / sell, waypoint, swap weapons. Units
  are named by (type, id), never by index.
- The **events** (server → client): unit added / removed, moves, casts,
  hits, state changes, life / mana, items in the world or owned, sounds.
- The **tick**: game frames at 1000 / 25 = 40 ms; commands are queued and
  applied inside the frame; outgoing updates are flushed after it.
- Not taken: the byte-exact layouts, Warden, the save upload. Those belong
  to a later D2GS codec.

d2d's `cmd::UseItem` is 0x20 (use an inventory item); right-clicking a
belt potion sends it too, where game.exe sends 0x26 with the item's id.

The field layouts of the gameplay packets (both directions), what the
client works out for itself, and how they map onto d2d's View:
net-packets.md.

## MCP 0x11 reply: `FUN_0044afc0`, a chunk reassembler (cited)

Traced from the decompiled C only; the register and stack arguments were
not checked in the disassembly, and no oracle has run it: **cited**, not
proven. d2d ports none of it (`git grep -i -E "44afc0|00799310|0079930c"` in
`components/` and `apps/` finds nothing).

`FUN_0044afc0` (339 bytes, `__fastcall`: ECX = packet, EDX = length) is the
handler at `0x70ed44`, entry 0x11 of the main MCP table `0x70ed00`
(`0x70ed00 + 4 * 0x11`). Its sender is `FUN_0044aa10`, which builds
`11, <ecx low byte>, <u16 edx>` and sends 4 bytes through `FUN_006bc8a0`.
docs/design/battlenet.md:148 names the pair REQUESTLADDERDATA; the handler
itself does only the chunk protocol below; what the callbacks do with the
records (ladder data or not) was not traced. It
returns 0 for a bad packet and 1 otherwise.

Packet: +1 byte `tag` (handed to the callback), +2 u16 `total` (size of the
whole message), +4 u16 `chunk` (bytes in this packet), +6 u16 `offset`,
data from +8. A length under 8 returns 0.

- `total`, `chunk` and `offset` all 0: calls `DAT_00799300(0, 0, 0)`, clears
  `DAT_00799310` (bytes still expected) and `DAT_0079930c` (bytes held),
  returns 1.
- `total` <= 0xb returns 0.
- Otherwise `DAT_00799310 = total` and `DAT_0079930c = offset`. If
  `chunk + offset > total` it returns 0. Else `_memcpy` (`FUN_006829c0`,
  the CRT one, which handles overlap; its callee `__VEC_memcpy`
  `FUN_0068c9c0` is the CRT's SSE helper) copies the chunk to
  `0x799318 + offset`, and `DAT_0079930c = chunk + offset`. The only bound
  is `total`; the buffer's size was not measured.
- While `total > chunk + offset` it returns 1 and waits for more.
- When complete: calls `DAT_00799304()` if set, then walks records in the
  buffer. A record is three i32 `{ a, n, m }` then `n * (m + 0xc)` bytes. The
  walk stops (leaving `DAT_00799310` at the bytes left minus 0xc, not 0) if
  `n` or `m` is negative, `n` > 0x100, `m` > 0x10, `a` > 2000, or the
  record is larger than what is left. For each valid record it calls
  `DAT_00799300(m, tag, record + 12)`, then moves on 12 + size bytes. A clean
  walk ends with `DAT_00799310 = 0`.

Who sets the two callbacks `DAT_00799300` / `DAT_00799304` was not traced
(game-calls.tsv has no row for them). d2d side: **not ported**.

## zlib-shaped helpers around `_memcpy` (cited)

Read from the decompiled C only, no oracle: **cited**. The names are a
reading of the field use (no assert string or source file name backs them),
so they are a structural match to zlib's deflate, not proven to be it.

- `FUN_0041d2a0` (27 bytes): `_memcpy(dst, src, n)` and nothing else, a
  plain wrapper.
- `FUN_00427450` (71 bytes, stream in EDI, not an argument): like zlib's
  `flush_pending`. It calls `FUN_0042d150` (which only calls `FUN_0042cb40`),
  then copies `min(stream +0x10, state +0x14)` bytes from the state's
  pending-out pointer (+0x10) to the stream's output pointer (+0xc), moves
  both pointers, adds to the stream's total (+0x14), takes the count off the
  stream's room (+0x10) and the state's pending (+0x14), and when pending
  hits 0 resets pending-out to the state's buffer (+8).
- `FUN_00427560` (405 bytes, `__fastcall`: ECX = dest stream, EDX = source
  stream): like zlib's `deflateCopy`. Returns -2 if either stream or the
  source's state (+0x1c) is null; copies the 0x38-byte stream (0xe dwords),
  allocates 0x16c4 bytes through the stream's allocator (+0x20), copies the
  state, allocates three 2-byte-unit arrays and a 4-byte-unit buffer, copies
  them, and re-points the tree pointers into the new state. On an
  allocation failure it calls `FUN_004274a0` and returns -4. It has no
  caller in game-calls.tsv.

`_memcpy` `FUN_006829c0` has 123 call rows in game-calls.tsv; the other work-list callers (`FUN_0040a1f0`, `FUN_00410040`,
`FUN_00427980`, `FUN_00427bb0`, `FUN_00428900`) were not traced.
