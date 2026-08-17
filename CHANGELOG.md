# Changelog

Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [0.6.16] - 2026-08-17 — toolchain pin to 6.5.27

### Changed — `cyrius = "6.5.21"` -> **6.5.27**

Stack-wide sweep so every repo in the desktop stack declares one toolchain. Pins had drifted across
three lines (6.5.5 / 6.5.20 / 6.5.21) while the installed wrapper was 6.5.27, so every build ran with
a drift warning and the declared graph did not describe what was actually compiled.

⚠ **Measured byte-identical**: 6.5.21 and 6.5.27 produce the same artifact for this repo, so the bump carries no codegen risk here. Recorded because a pin that is assumed to be cosmetic is how a real change gets waved through later.

⚠ The vendored `lib/` was re-synced to the 6.5.27 bundled set, which clears the
`./lib/ shadows version-pinned` warning. Tests re-run green after both changes.

## [0.6.15] - 2026-08-17 — the reunion: puka's window is a dhancha widget tree

### Changed — the present path goes through the toolkit

dhancha's README calls it *"the spiritual extraction of puka's windowing code"*, and puka was the last
app still hand-drawing its entire window: `fb_render` painted the grid into an RGB24 buffer and
`pix_blit_region` copied it straight into the compositor's present buffer. That is now a **dhancha
widget tree** (`src/ui.cyr`) whose `CANVAS` leaf performs the blit.

⛔ **THE GRID IS NOT DECOMPOSED INTO WIDGETS, AND THAT IS THE DESIGN.** An 80×24 terminal is 1920
cells, each with a foreground, background, attributes, a wide-glyph spacer flag and a possible cursor.
As widgets that costs more memory than the scrollback and throws away the **dirty-row scanner** that
makes a keystroke echo repaint sixteen scanlines instead of the screen. `fb_render` is untouched; what
changed is where its pixels land. dhancha 0.9.9's `CANVAS` exists precisely so a renderer like this
keeps working while the window around it becomes a tree.

⚠ **The point is what is now possible, not what changed on screen.** A find bar (a dhancha
`TEXTINPUT`), tabs, or a scrollback indicator is now a widget insertion instead of a rewrite of the
present path — which is exactly what puka could not do while it hand-drew everything.

### Fixed — ⛔ the present blit left the alpha byte at ZERO

`pix_blit_region` packs `(r << 16) | (g << 8) | b`, so byte 3 was **0** on every pixel puka presented.
That is correct for the path it was written for and wrong for a compositor that reads it: under agnos's
`gpu_shader_op #92` op 0x01 — premultiplied src-over, `out = src + dst * (1 - src_a)` — an alpha of 0
collapses the blend to `out = src + dst`. The terminal does not vanish; it renders as an **additive
over-bright ghost** composited onto whatever is behind it, which is far harder to notice than a black
rectangle.

⚠ **It could not have been caught on the host before this release.** Every RGB dump of that buffer is
correct — the defect is in a byte no host-side check looked at. `dh_canvas_blit_rgb24` sets it to 255,
and `tests/ui.tcyr` asserts on the **full 32-bit word**.

⚠ `pix_blit_region` itself is unchanged and still correct for its own path — the fix is that the
present path no longer uses it.

### Changed — no extra copy was added

`dh_surface_wrap` (dhancha 0.9.9) points a sadish surface header **at** the buffer
`win_present_begin` returned, so the widget tree draws directly into the memory the compositor reads.
Rendering into a toolkit-owned surface and copying it across would have added a second full-frame copy
per keystroke — adopting the toolkit would have made puka measurably slower, which is the wrong kind
of port.

⚠ The surface is wrapped **per frame**, never cached: `win_present_begin` hands back a pointer valid
only until the matching commit.

### Changed — `[deps.dhancha]` -> **0.9.9**

### Testing

`tests/ui.tcyr` (14 checks): the root is a `WINDOW`, the grid is a `CANVAS` with a draw callback, a
rendered frame carries the grid's pixels into a caller-owned buffer with **alpha 255**, the glyph cell
has shape rather than one flat colour, the wrapped surface points at the caller's memory with the
caller's stride, and a null present buffer is refused.

⚠ The glyph check asserts *"not uniform, and every pixel opaque"* rather than a specific foreground
value: which RGB a cell resolves to is puka's colour model, which `render.tcyr` already covers against
the RGB24 buffer. Asserting it again here would test that layer twice and this one — whether the
widget-tree path carries pixels through intact — not at all.

Mutation-tested: reverting to the alpha-dropping `pix_blit_region` fails 3 checks, and a `CANVAS` with
no draw callback fails 5. Full suite 49/49 plus the new 14.

## [0.6.14] - 2026-08-17 — desktop-stack catch-up: dhancha 0.9.5, setu 0.8.6, one language version

### Changed — `[deps.dhancha]` 0.9.4 -> **0.9.5**

0.6.13 put puka's transport seam on the toolkit; this keeps it current. The two toolkit fixes are
`dh_hit_test` clipping to the parent and `dh_surface_present` refusing instead of silently succeeding.
⚠ **Neither is reachable from puka today**, and saying so matters more than claiming the upgrade:
puka routes CONNECT / FD / CLOSE through `dh_client_*` but renders a raw XRGB terminal buffer and maps
HID to evdev itself, so it touches no widget tree and never calls `dh_surface_present`. The bump keeps
the declared graph honest and takes the fixes for free when the widget tree is adopted.

### Changed — `[deps.setu]` 0.8.5 -> **0.8.6**

`present_probe` honours `SETU_CLOSE`. ⭐ Directly relevant here: the probe is what gets staged into the
`/bin/puka` slot by default, and it inherited the exact leak **puka itself fixed on 2026-08-08** — an
orphaned client holding one of 16 system-wide `#86` slots for the rest of the boot. Measured on iron
with the probe in the slot: 16 → 15 → 14 → 13, one per desktop launch. A fix puka had already paid for
came back through its stand-in.

### Changed — cyrius pin 6.5.9 -> **6.5.21**, matching agnos, aethersafha and dhancha

One language version across the desktop stack.

### Fixed — `[deps.kashi]` gains a `path` override

Every other dep here carries `git` + `path`; kashi did not. ⛔ `path` WINS over `tag` — re-verify the
tag against kashi's `VERSION` at each cut.

⚠ Unblocked a hard `cyrius deps` failure (*"dep dhancha requires 'kashi_font_data'"*) whose cause was
in dhancha's dist sidecar, not here — kashi is VENDORED and belongs to neither the stdlib nor a fetched
dep list. Fixed in dhancha 0.9.5.

**Verified**: `--agnos` build OK, 1,637,800 B. ⚠ Grown from 1,633,616 B; the recorded `spawn_path #43`
iron size hazard is unchanged in kind and QEMU still spawns it (`puka: terminal up -- 80x24, shell on a
pty` → `first present ok`), but that hazard's own note says QEMU does not reproduce it.

## [0.6.13] - 2026-08-16 — the transport seam moves to dhancha

### Changed — `win_open` / `win_fd` / `win_close` go through `dh_client_*`, and `[deps.dhancha]` lands

puka was the last app reaching the compositor around the shared toolkit instead of through it. It now
routes CONNECT / FD / CLOSE through dhancha's client layer.

⛔ **IT WAS NOT puka's FAULT, AND THE CAUSE IS WORTH RECORDING.** dhancha's dist bundle did not SHIP its
client layer until 0.9.4 — `src/dh_client.cyr`, `src/setu_client.cyr` and `src/setu_input.cyr` were
absent from its `modules` list — so `dh_client_connect` was **undefined downstream** and hand-rolling
setu was the only option any consumer had. Proven by rewiring first and getting
`2 reachable undefined function(s)`.

⚠ **SCOPE, STATED PLAINLY.** `setu_client_present` and `setu_client_poll_input` STAY on setu directly.
dhancha's `dh_client_present` renders a WIDGET SURFACE and `dh_client_next_event` returns a `DhEvent`;
puka presents a raw XRGB terminal buffer and maps HID to evdev itself. Those are different models — a
port, not a rename. What is now shared is the seam that has moved twice (TCP -> AF_UNIX -> the agnos
`#97` channel band) and stranded a consumer each time.

⚠ **SIZE, MEASURED RATHER THAN ASSERTED: 1,541,744 -> 1,633,616 B (+91,872, +6%).** An earlier draft of
this work was DEFERRED on the argument that adding dhancha's draw stack (sadish + rupa + rekha) would
grow puka into the recorded `spawn_path #43` iron size hazard. The growth is 6%, and the hazard was
never measured before it was invoked. ⇒ Measure the thing you are about to refuse on.

**Verified in QEMU** (`scripts/harness/puka-terminal-test.py`, agnos 1.56.45): `puka: terminal up --
80x24, shell on a pty` -> `puka: first present ok`, `presented: 2` (puka AND crab, both rewired), spawned
through `spawn_path #43`, `exit 95`. ⛔ That does NOT close the iron size hazard — its own record says
"QEMU does not reproduce it" — but the binary is the same order of size as the one that burned green.

## [0.6.12] - 2026-08-12 — one rendezvous, named by setu

⭐ Passes **0** to setu instead of hardcoding `"/tmp/aethersafha-setu.sock"`, so the socket is named in
one place — `setu_un_path` (setu **0.8.5**), which resolves an explicit path, then `$SETU_SOCKET`, then
`SETU_UNIX_PATH`. Four repos each carried that literal; they agreed, but all four had to be edited in
step for that to stay true.
⚠ `[deps.setu]` gains `path = "../setu"` alongside its tag — it was the one dep here declaring a tag
with no path override, so a local setu change could not be built against at all. Verified: with the old
vendored 0.8.4 this client silently ignored `$SETU_SOCKET` and `--clients` answered 94.
⛔ Also corrects a comment asserting the path was *"advisory and always was — setu ignores it"*, false
since setu 0.8.4.

## [0.6.11] - 2026-08-08 — puka EXITS when the compositor closes its window

⛔⛔ **puka used to ignore being closed.** aethersafha's F4 removed the window from its own vector and
told nobody, so on the 2026-08-08 iron burn the terminal was left **orphaned alive** — still holding its
`#97` channel end and its `#86` GPU-visible shm slot, of which there are only **16 system-wide**. The
operator saw it as *"not closing properly"*.

⭐ **The platform layer already had the vocabulary**: `WIN_EV_CLOSE` exists because the wayland backend
raises it (`src/platform/window.cyr:18`, `:101`). The setu backend now raises it too, on `SETU_CLOSE`
(kind 7 — in the protocol from the start, never sent and never handled), and the frame loop drops `live`
so the existing `pty_close` + `win_close` teardown runs. ⚠ That teardown and the process exit are what
release the endpoint and the slot; the kernel reclaims them on process death.

⚠ **`win_poll_events` is now called ONCE per frame with its result captured.** It consumes a message per
call, so testing its return value twice would have dropped every other event.

⚠ **Recorded, not silently fixed:** the `WIN_EV_KEY` test is equality, and `WIN_EV_*` are powers of two
that the wayland backend ORs together — so on the host path a frame carrying both a key and a frame-done
reports `KEY | FRAME` and the key is missed. That is a latent **host-build** defect (the setu backend
returns single events), and fixing it changes behaviour on a path this change does not exercise. The new
`CLOSE` test uses a bit test for exactly this reason.

## [0.6.10] - 2026-08-07 — the LINE DISCIPLINE: a shell you can type into, iron-proven

⭐ `win_poll_events` / `win_next_key` are no longer stubs in the setu backend. The compositor forwards
`SETU_INPUT_KEY` carrying the **HID usage code** — not a codepoint and not an evdev keycode — so the
seam translates before handing anything to the engine.

⭐ **HID usage → evdev keycode is the PS/2 set-1 make code for the main block**, because Linux derived
its keycodes from set-1 (Esc 1, digits 2..11, A 30, Enter 28, Backspace 14, Tab 15, Space 57). The
agnos kernel carries the same mapping for its own console (`hid_usage_to_ps2`), so this is that table's
shape rather than a second invention. An **unmapped usage returns 0 and is dropped** — emitting a wrong
keycode would type a plausible character the user never pressed.

The decoded keycode then goes through `input_from_keycode`, the SAME keycode→child-bytes bridge the
Wayland backend uses, so arrows and named keys produce proper CSI sequences instead of a hand-rolled
byte. The encode discipline already existed and is not duplicated.

### ⭐⭐ IRON-PROVEN 2026-08-07 — a live agnsh answering in a composited window, on real silicon

The `AE-T2` work below was burned on archaemenid (AGNOS 1.56.41, `smp: cpus online: 4`) and **passed**:
`puka: terminal up -- 80x24, shell on a pty` → `first present ok` → TAB → **`puka: key received` ×5**
(`h-e-l-p-Enter`, not one keystroke lost) → **`puka: line sent to the shell`**, and the panel shows agnsh's
help output rendered with a live `[ASSIST] >` prompt. 278 frames, clean Esc.

⭐ **The QEMU key-loss defect did NOT reproduce.** QEMU lost 5 of 9 keys at a ~100 ms hold because the xHCI
HID ring is drained once per compositor frame; on iron the operator typed at human speed and **19 of 19**
keys arrived (crab 2, puka 17). A 6.40 ms GPU frame polls fast enough. ⚠ Still real for any slow frame.

⭐ `puka: byte refused by the line discipline` fired once — the Esc that quit the compositor was forwarded
here too, and correctly declined rather than entering a command line.

### Fixed — ONLCR: the child's bare LF must also return the carriage (the burn found this, a re-flash confirmed it)

⭐⭐ **RE-FLASHED AND CONFIRMED ON IRON, same day.** Operator: *"flashed and clean… puka displays shell in
terminal as expected with expected wrap."* The kernel was **byte-identical** across the two flashes (same
1,969,248 B artifact, same burn-tag, re-prepped from unchanged source), so **`/bin/puka` was the only
variable** — the staircase and its absence are attributable to the terminal and nothing else. A reproducible
build turned the fix into a controlled experiment for free.

⇒ **The line discipline is now hardware-validated in both directions: ICRNL + echo + erase + ONLCR.**
⭐ And the layout gate's QEMU calibration transferred to the panel — because the numbers were derived from
the **mutant**, not from a passing run.

⛔ **The same burn rendered agnsh's output as a STAIRCASE** — every line starting where the previous one
ended, then breaking mid-word at the right edge. Reported as *"doesn't appear to respect the window
wrapping"*. **It is neither**, and width was eliminated by measurement before anything changed: agnsh's
longest `help` line is **77 of 80 columns**, so nothing on that screen should have wrapped.

- agnsh emits a **bare LF**, like every agnos program, because the agnos kernel console makes LF mean
  newline **and** carriage return (`agnos/kernel/arch/x86_64/fb_console.cyr:1051-1053`).
- puka's engine is a **correct VT100**: LF is `term_index()`, down one row and nothing else.
- ⇒ Line 2 began at column 53 where line 1 ended, overflowed 80, and broke mid-word. Every fragment in
  that photograph is arithmetic.

Every Unix tty closes this with **ONLCR** on the output path. This repo shipped ICRNL, echo and erase — the
**input half only** — and the echo code even stated the rule (*"a bare LF would stair-step every line to
the right"*) without applying it to the child's output. **One half of a line discipline is not a line
discipline.** New `ld_out_needs_cr` / `ld_out_feed`, called by `pty_pump`, gated on `LD_OWNED_HERE` so Linux
devpts (which already does ONLCR) is untouched. Follows POSIX ONLCR exactly: NL → CR-NL unconditionally, no
look-back, because a second carriage return is idempotent.

⛔ **Deliberately NOT fixed in the shell.** Every agnos program emits bare LF for the same correct reason,
so patching agnsh would leave owl, kriya and `iam` broken in a window and add stray CRs to their console
output. The terminal owns it — *"the child never learns it has one."*

⛔⛔ **The lesson is about the instrument: a pixel count cannot see a layout defect.**
`puka-terminal-test.py` passed this build **before and after** the fix with byte-identical numbers
(4991 → 5176 → 6032 both times), because a staircase draws **exactly the same characters** and only puts
them in the wrong places. The gate was blind by construction; the operator's eye was the only oracle that
could see it. It now counts occupied **text rows** — calibrated on both arms of the same build in QEMU:
**correct = 6 · staircase = 8 · ceiling 7**.

**49/49** in `tests/line_discipline.tcyr` (was 37), including a negative control that reproduces the
staircase in the real engine (raw LF leaves the cursor at column 3) and an end-to-end check that two real
agnsh help lines occupy exactly two rows. ⚠ The first version of that test reimplemented `pty_pump`'s loop
inside the test file, so deleting the real call site would have left it green — the shared `ld_out_feed`
exists so the test and the pump run the same code.

### ⭐⭐ The shell now ANSWERS — a line discipline, and it was one byte

**`src/line_discipline.cyr`** (new) is decomposition item **(iii)**: a PTY is a bidirectional local
channel, an end handed to a child at spawn, and a line discipline. agnos supplies the first two on the
`#97` band and deliberately supplies none of the third — there is no termios, no ICRNL, no ECHO — so it
belongs in the terminal.

⛔ **The defect was a single byte, traced statically end to end with no burn and no probe.** Enter is HID
usage `0x28` → evdev keycode 28 → `evdev__keymap` returns **`0x0D`** (`input/evdev.cyr:74`, "Enter -> CR")
→ `utf8_encode` → the single byte **13** on the child's stdin. agnoshi's `read_line` terminates a line on
**`ch == 10`** and on nothing else (`agnoshi/src/agnsh.cyr:366`). CR is 13, so the line could never
complete: every keystroke, Enter included, accumulated in the shell's carry buffer while it correctly
waited forever. ⭐ **puka is right to send CR** — a terminal sends CR for Enter (VT100/xterm), and the
byte that reaches a program is LF because a discipline translated it. On Linux devpts does that (ICRNL),
which is exactly why `tests/input_pty.tcyr` passed on the host and proved nothing about agnos.

⛔ **The previous entry's hypothesis was aimed right and wrong in mechanism**, and is corrected rather
than deleted: it read *"something in its agnos `read_line` path is not completing a line from single-byte
records"*. Single-byte records accumulate **correctly** — `agnsh.cyr:363-375` refills and keeps
accumulating across as many reads as it takes. Nothing about record size was ever wrong. What was missing
was **CR→LF and echo**.

**Cooked, not a pass-through translate**, and the reason is erase: agnsh has no backspace handling of its
own (the kernel console line discipline used to do it), so a BS forwarded to the child would leave the
mistyped byte in its buffer while the screen showed it gone — **the screen would lie about what the shell
is about to run**. Buffering the line here means the child receives exactly one complete, edited line,
byte-for-byte the contract agnsh already has with the console. The child cannot tell the difference.

- **CR or LF terminates** — the completed line is copied out with a trailing LF and the pending buffer is
  cleared **inside `ld_feed`**, so no caller has to remember to reset one. A caller that forgot would
  silently prepend the previous command to the next, which reads as a shell bug rather than a terminal one.
- **DEL (`0x7F`, what Backspace encodes to) and BS erase**, and on an empty line emit **nothing** — an
  unconditional erase walks the cursor left over the shell's own prompt.
- **Echo is puka's job now.** agnsh does not echo, deliberately: on the console *"the kernel now owns echo
  + line discipline"*. A channel fd has no kernel echo, which was the second half of why the panel never
  changed — a byte that arrived perfectly was invisible.
- ⛔ **Every byte is either shown and sent, or refused and counted — never sent-but-unshown.** Control
  bytes and CSI sequences are **refused** (`ld_drops_get`, and a console line per refusal) rather than
  forwarded un-echoed: agnsh has no line editor, so a forwarded `ESC[A` would land in its line as `^[[A`
  and make the command unrunnable while the screen showed nothing.
- A line is handed over in **≤64-byte records**, because the band is a record transport. Not a carve-out:
  the kernel rules on exactly this case at `agnos/kernel/core/syscall.cyr:7257-7266` — *"a stream writer's
  bytes arrive as several records — which is exactly what a tty does"*.

**`LD_OWNED_HERE`** gates it — 1 on agnos, **0 on Linux**, where devpts already cooks and echoes and
running ours would translate CR twice and print every character twice. It is a variable rather than an
`#ifdef` at the call site, so both paths compile on both targets and the cooked path stays reachable from
a host test.

### Measured — QEMU, agnos 1.56.41, `AE_CLIENTS_MODE=desktop`

`agnos/scripts/harness/puka-terminal-test.py`, repeated and byte-identical across runs:

| | |
|---|---|
| keys delivered to puka | **9 of 9** forwarded (tab is consumed by the compositor) |
| keystrokes echoed | glyph px **4991 → 5176** |
| a completed line reached the shell | **yes** |
| **the shell answered** | glyph px **5176 → 6032** (floor +52, derived from this run's own scale) |

**37/37** in the new `tests/line_discipline.tcyr` (host), **13 suites green**, both targets build.
⛔ **QEMU only — never burned.**

### ⚠ Found on the way: keys are LOST when a frame is slower than a keypress

Not a terminal defect, recorded because it presents as one. **A USB HID keyboard reports state on poll; it
does not queue events.** agnos drains the xHCI HID ring only inside `kbscan #42`'s bounded `sti` window
(`agnos/kernel/core/syscall.cyr:8746-8757`), and the compositor calls that **once per frame** — so a key
whose press and release both complete inside one frame is never sampled at all.

Measured on the QEMU CPU composite path at QEMU's default ~100 ms hold: **0 of 9**, **4 of 9**, **4 of 9**
keys delivered. ⚠ The 4-of-9 runs still completed a line and got an answer, which is precisely what makes
this so easy to misread as a line-discipline bug. At a 500 ms hold: **9 of 9**, twice, deterministically.

⚠ **A human holds a key ~100 ms and would lose keys on this same path.** The fix is a faster frame
(`AE-0a`) or IRQ-buffered HID reports — system work, not terminal work. The harness now counts delivered
keys and names the layer, so the loss can never hide behind a terminal verdict again.

### ⚠ Shift is still not reachable

⚠ A press-only surface (the default, and what a terminal wants —
FULL_KEYS would double-type every character) carries no modifier state in the message, and the
compositor does not forward the HID modifier byte as a usage. The engine therefore sees unshifted
keycodes. Stated rather than faked: inventing a mods value would silently produce the wrong glyph.

⚠ Requires aethersafha's **unreleased** TAB focus-cycling fix to be typable at all with two clients.

## [0.6.9] - 2026-08-07 — puka is a TERMINAL: a live agnsh in a composited window

⭐⭐ **`src/main.cyr` was still the M1 headless demo.** It now opens a setu window, mints a PTY on the
agnos `#97` channel band, spawns `/bin/agnsh` onto it, and paints the shell's output as glyphs into the
window it presents. That completes agnos **ipc bite 9** — *"a live agnsh prompt in a composited window,
the gate no candidate could pass"*.

The frame loop is small because every hard part already sat behind a seam, which is what the seams were
for: `pty_pump` reads the child and feeds `term_feed` · `fb_render` paints the cell grid ·
`pix_blit_region` converts RGB → the backend's XRGB8888 buffer · `win_*` is the setu backend.

### Fixed — `--demo` matched the wrong byte and reached the demo only by failing

⛔ The flag check compared the FIRST byte of `argv(1)` against `'d'`, which for `--demo` is `'-'`. The
flag never matched: `--demo` reached the demo by falling through the terminal path and failing, printing
two display-failure lines on the way. An explicitly requested mode must not report failures for a thing
it was told not to attempt. Leading dashes are now skipped, so `-d` and `--demo` both match.

⚠ **The fallback path still explains itself** — that is the difference between the two, and it is the
point: no flag means "be a terminal if you can", and the reason it could not is worth printing.

⚠ **The demo is now the FALLBACK, not the default.** puka tries to be a terminal first and prints the
M1 canned stream only when there is no compositor to host it — so a missing display degrades to
something visible, and CI (which has none) still exercises parse → grid → render exactly as before.

### Changed — setu pin 0.7.4 → 0.8.4

⛔ **0.7.4 predates the channel-band cutover and has no agnos arm at all**, so `setu_client_connect`
returned 0 on agnos with none of setu's own refusal messages — a silent failure that looks like "no
compositor running" and is not. `window_setu.cyr` now says so explicitly when a connect is refused.

### Verified — `harness/puka-terminal-test.py` (in agnos)

**PASS, three consecutive runs, deterministic.** puka opens a window + PTY, presents its surface, the
compositor sees a client, and the panel carries **4991 pixels of exact RGB (192,192,192)** — puka's
`fb_def_fg`.

⭐ **The oracle is external and controlled.** Measured negative control: the same desktop WITHOUT puka
has **0** such pixels. Nothing else on screen uses that colour — the compositor's chrome is dark greys
and cyan — so a nonzero count means glyphs were rasterised and composited, not merely that a window
appeared. puka's own markers and the compositor's claim are self-reports by the programs under test;
the pixel count is the only witness that is neither.

⚠ **The gate had to stop racing the clock.** On a fixed capture delay one run in three screendumped
before puka's first present and reported 0 glyph px on a boot where everything worked — which reads as
a rendering failure rather than a capture taken too early. The capture now waits for puka's
"first present ok" marker. A timing-dependent oracle that sometimes says zero is worse than none: it
teaches you to distrust a real red.

⚠ **`/bin/puka` is overridden in that harness's seed only.** The shared rootfs stages setu's slim
`present_probe` under that name, and `aethersafha-clients-test.py`'s oracle counts *its* colours —
swapping the shared rootfs would silently invalidate a passing gate. Confirmed still PASS (3500 px).

⚠ **Input is wired but unexercised.** `win_next_key` is still a stub in the setu backend, so keystrokes
reach `pty_write` only once the compositor's key forwarding is consumed. Output, PTY and rendering are
proven; typing is not.

## [0.6.8] - 2026-08-07 — the AGNOS-native PTY backend (agnos ipc bite 9)

### Added — `src/pty.cyr` has a real agnos arm; the M5 "kernel syscall gap" is closed

The stubs said *"AGNOS-native PTY is M5 (kernel syscall gap)"*. agnos **1.56.40** closed it. A PTY
decomposes into (i) a bidirectional local channel, (ii) an end handed to a child at spawn, (iii) a line
discipline — only (iii) is terminal-specific — and agnos now supplies (i) and (ii) directly:

| puka | agnos |
|---|---|
| `pty_open` | `sys_chan_mint` — no devpts, no `TIOCGPTN`, no slave path to build |
| `pty_spawn` | `CH_ENDOW` in PTY mode + `sys_spawn_path` — no fork, no `setsid`, no `TIOCSCTTY` |
| `pty_pump` / `pty_write` / `pty_close` | `sys_chan_recv` / `sys_chan_send` / `sys_chan_close` |

⛔ **This is not a port of the Linux arm, because there is no `fork()`.** Linux forks, opens the slave,
makes it the controlling terminal, dup2's it onto 0/1/2 and execs. On agnos the **kernel** installs the
endowed endpoint at the child's 0/1/2 at spawn time, so the child is *born* holding its terminal and
never learns it has one — `/bin/agnsh` reads fd 0 with a plain `sys_read`, which is the entire point.

⛔ **The idle wait is a preemptible spin, not `sys_sleep_ms` and not `sched_yield`.** agnos
`planning/ipc.md` §9.4: sleep_ms is `preempt_disable; sti; hlt` and would starve the very child being
waited on; sched_yield is a documented silent no-op under a foreground run. The kernel never blocks on
a channel (one SYSCALL stack per CPU), so waiting is userland's job and must stay preemptible.

⚠ **Honest gaps, stated rather than faked:** no window size (there is no `TIOCSWINSZ` equivalent —
`rows`/`cols` are accepted and ignored), and `pty_spawn`'s `arg1`/`arg2` **return an error** rather than
being silently dropped, because `spawn_path #43` takes a path, not an argv vector. `pty_write` is one
record of ≤ 64 bytes; a longer write is an error, not something to split, since splitting would
reintroduce message boundaries in userland — the problem the band exists to delete.

### Added — CI builds the `--agnos` target

⛔ **puka had never been built for agnos at all**, and the pipeline could not have told anyone. All
three blockers found while writing this backend — the `mabda` dep, the stale cyrius pin, the
unresolvable enum — were compile-time facts on the agnos arm and invisible to a Linux-only CI. The
entry and the test program now build under `--agnos`. They cannot be *run* (no agnos host), and the
build is a sufficient gate precisely because each of those failures was a build failure.

### Changed — cyrius pin 6.5.5 → 6.5.9 (a floor, not a preference)

The channel-band wrappers (`sys_chan_*`, the `CH_E_*` result codes) landed in **6.5.8**. On 6.5.5 the
agnos arm failed with `undefined variable 'CH_E_PEERGONE'` — which reads like a puka bug and is a
toolchain floor.

### Removed — the `mabda` dependency, which was blocking the entire agnos target

⛔ `dist/mabda.cyr` calls `syscall(SYS_IOCTL, …)` for its DRM path, and **`SYS_IOCTL` does not exist in
agnos's syscall peer** (agnos has no ioctl). cyrius prepends every declared dep module whether or not
the entry's include graph reaches it, so the `--agnos` build died with `undefined variable 'SYS_IOCTL'`
before one line of puka was compiled.

⚠ **And nothing in the build graph used it.** `src/main.cyr` includes parser/grid/unicode/terminal only;
the sole consumer is `src/platform/gpu/gpu.cyr`, which no entry point includes yet. It was a declared
dependency on code that is not built, costing a whole target. The 3.2.11 pin was also two majors stale
(mabda is 4.0.8). ⭐ Restore it — guarded, on a current tag — when the GPU platform is actually wired in.

⚠ **Scope:** `src/main.cyr` is still the M1 headless demo and does not include `pty.cyr` or any
platform, so nothing exercises this backend yet. Wiring the entry to PTY + the setu renderer is puka's
M2/M3/M4 and is what remains of agnos ipc bite 9. The backend itself builds on **both** targets and the
test suite passes.

⚠ `src/grid.cyr` carries pre-existing `cyrius fmt` drift, untouched by this work.

## [0.6.7] - 2026-08-02

### Changed — cyrius pin 6.4.71 -> 6.5.5; kashi 1.0.4, setu 0.7.1

Part of the whole-desktop-stack toolchain catch-up cut on this date.

⚠ **`[deps.mabda]` deliberately held at 3.2.11** while 4.0.8 is on disk. That is a MAJOR version
jump, and puka's mabda path is hardware-verified at the old one — it is a decision, not an
oversight, and it wants its own bite rather than a sweep.

⚠ Unrelated but adjacent, recorded so it is not lost: `src/render/pixfmt.cyr` writes byte 3 = 0.
Harmless on the CPU present path; under agnos's `gpu_shader_op` **#92** op 0x01 (premultiplied
src-over) a zero alpha byte yields `out = src + dst` — an **additive over-bright ghost**, not a
vanished window as three other documents in this stack claim. Not fixed here.

## [0.6.6] - 2026-07-23

### Changed — setu 0.7.0 (`SETU_SURF_PREMULTIPLIED`) + dep refresh

No behaviour change: the flag is opt-in and this client does not set it.

## [0.6.5] - 2026-07-23

### Changed — setu 0.6.0: client buffers are GPU-visible on agnos

Picks up `setu` **0.6.0**, whose `setu_buf_create` now asks for `shm_create_gpu` **#86** before falling back
to `shm_create` **#71**.

⚠ **Why this matters beyond a version number.** `#71` allocates **system RAM**, which the agnos GPU cannot
reach at all — bus-master is off by design and the engines see only the framebuffer aperture. The kernel
rejects a `#71` slot at both GPU entry points (`gpu_blit_shm` #87: `src_mc == 0 ⇒ the GPU cannot read it`;
`gpu_shader_op` #92: `GPO_E_BADSLOT`). Every shared surface in the desktop was allocated that way, so the
whole iron-proven ring-3 GPU band had **no reachable consumer**. Buffers from this release are eligible for
a hardware blit.

No API change and no call-site change here — the buffer id behaves identically, and `#86` falls back to
`#71` automatically on a machine with no GPU carveout (every QEMU boot).

### Changed — cyrius pin → 6.4.71

## [0.6.4] — 2026-07-08 — puka is the compositor's first resident (setu client, over the TCP transport)

> ⛔ **RETRACTED 2026-08-03 — the TCP transport this release adopts is RETIRED, and the agnos claims
> made *in this release's era* are FALSE GREENS.** TCP-on-loopback was the WRONG PRIMITIVE for a local
> display protocol — nothing to route, nothing to checksum, no window to negotiate, no business owning
> a port. That, not a failure, is why it is retired.
>
> **Scope the history exactly.** *Before* `net_src_for` (agnos 1.56.34) the handshake could not complete
> on an ordinary boot: the client's SYN carried `net_ip` as its source, so the SYN-ACK came back on a
> 4-tuple its own conn could not match. The only agnos test that passed in that era,
> `aethersafha-setu-smoke.sh`, passed because the `AETHERSAFHA_SETU_SELFTEST` kernel hook assigned
> `net_ip = 0x7F000001` and made src and dst agree by accident; hook and script are both deleted, and
> the agnos claims in this 0.6.4 entry trace to them. **"cross-platform on Linux and agnos" below was
> not yet true on agnos when it was written.**
>
> ⚠ *After* `net_src_for` it DID work un-rigged. On 2026-08-02 the honest harness
> `agnos/scripts/harness/aethersafha-clients-test.py` — which byte-scans `build/agnos` and hard-exits if
> the kernel carries any selftest hook — reached **`connected: 2, presented: 2`**, and **one of those two
> clients was setu's `present_probe` staged as `/bin/puka`**; the other was the real dhancha `crab`.
> Scope it honestly: QEMU at `-smp 1`, never shown on iron, `-smp 4` fault-kills. Do not restate this
> release as "puka never connected on agnos" — it did, later, on an honest kernel.
>
> The replacement is the agnos socket (`anu`) — agnos `docs/development/planning/ipc.md` §9/§10.
> ⚠ The Linux-side end-to-end observation is NOT withdrawn either: Linux is a different target with a
> different kernel, not an agnos fallback. The `win_*` backend code stands; what it dials changes.

puka gains a **setu client window backend** and becomes `aethersafha`'s first
resident app: the terminal engine renders a cell grid → pixels and presents them
over setu, so puka's window arrives on the sovereign desktop at runtime (no
compositor-seeded placeholder). Now pinned to **setu 0.3.0** — the CROSS-PLATFORM
TCP transport (item 3b), proven end-to-end: puka connects over TCP loopback:7700
and presents a rendered 320×192 terminal frame that the compositor accepts +
composites.

### Added

- **`src/platform/setu/window_setu.cyr`** — the setu `win_*` backend: fills the
  same window contract the terminal engine expects (`win_open` → `win_present` →
  `win_next_event` → `win_close`) over setu's persistent client
  (`setu_client_connect` / `setu_client_present` / `setu_client_recv` /
  `setu_client_close`). The engine runs unchanged; only the platform seam differs.
- **`programs/puka_setu_probe.cyr`** — renders a terminal grid and presents it
  over setu (the fork-free client half of the `aethersafha` e2e proof).
- **`programs/puka_setu_term.cyr`** — a real `$SHELL` session presented over setu.

### Changed

- **`[deps.setu]` → 0.3.0** — the reference client transport is now TCP over
  loopback (`net.cyr`), cross-platform on Linux and agnos, replacing the
  Linux-only AF_UNIX path. No puka code change beyond the pin — the client API is
  unchanged.

## [0.6.3] — 2026-06-19

**Scrollback.** Lines that scroll off the top of the primary screen are retained and
viewable — **Shift+PageUp/PageDown** scrolls through history; typing snaps back to the
live bottom. Plus the docs/roadmap handoff sweep and test-harness entry hygiene from
the same cycle.

### Added
- **Scrollback ring + viewport** — a lazily heap-allocated ring (`SCROLLBACK_LINES`=1000, primary screen only — the alt screen has none) captures lines scrolling off the top (`grid_scroll_up` when the region starts at row 0). The renderer (`fb.cyr`) reads through viewport-aware accessors (`grid_vglyph`/`grid_vfg`/…) — identical to the live grid at the bottom, history when scrolled back; the cursor hides while scrolled back. The viewport stays anchored as new lines push in. `puka_term`: **Shift+PageUp/PageDown** scroll by a page; any keystroke to the child resets to the live bottom. 16 conformance assertions in `tests/grid.tcyr` (capture, viewport mapping, anchoring, alt-screen exclusion, reset).

### Changed
- **Docs + roadmap handoff sweep** — README and `getting-started.md` rewritten for the 0.6.2 Wayland-desktop reality (build/run `puka_term` on Hyprland, real architecture + deps, `/dev/fb0` warning); roadmap M6 reorganized into shipped-by-version vs remaining (GPU cell renderer marked **paused pending mabda**, bite-10 narrowed, alt-screen recorded); `overview.md` data-flow + module table refreshed; new **ADR-0003** records the framebuffer-console → Wayland-desktop pivot.
- **Test-harness entry hygiene** — every `tests/*.{tcyr,bcyr,fcyr}` + `src/test.cyr` now use the compliant `_entry();` + `SYS_EXIT` pattern (was `var X = main(); syscall(60, …)`), matching the programs and CLAUDE.md. No behaviour change; 461 assertions still green.

## [0.6.2] — 2026-06-19

**GPU foundation + the alternate screen.** Two tracks land: puka now drives `mabda`'s
native AMD GPU end-to-end (render → `wl_shm` → Hyprland, verified) as a shader-agnostic
**foundation** — the actual GPU *cell* renderer is paused pending mabda maturing (a
64 KiB-align `va_map` fix + a higher-level shading API). And the **alternate screen**
(DEC 1049) closes the biggest daily-driver gap — `vim`/`less`/`htop`/`tmux` work
correctly now. The daily driver still renders cells on the CPU `fb.cyr` path.

### Added
- **GPU plumbing foundation** (M6 bite 7) — puka now drives `mabda`'s native AMD GFX9 backend end-to-end: **GPU render → CPU readback → `wl_shm` → Hyprland window**, verified live (a GPU-rendered frame presented through the compositor, 120 sustained frames, pixel-exact). New `src/platform/gpu/gpu.cyr` — the puka-generic `pgpu_*` seam (init / target / render / readback-with-RGBA8→XRGB8888-swizzle / release), the GPU analogue of the `win_*` seam (extracts to `aethersafha`). `mabda` wired as a git dep at **3.2.11** (the `wgpu` FFI backend stays forbidden; native AMD only); puka's `[deps] stdlib` extended to mabda's superset (`args/hashmap/tagged/fnptr/mmap/dynlib/sakshi`). Probes: `programs/gpu_probe.cyr` (headless render→readback) and `programs/gpu_win_probe.cyr` (the full windowed pipe). **The daily-driver `puka_term` is unchanged — cells still render on the CPU `fb.cyr` path**; this is the shader-agnostic foundation the bite-8 grid renderer builds on.
  - *Architecture correction (recon 2026-06-19):* mabda has **no instanced-vertex path and none is roadmapped** (3.2.x closes at 3.2.13). But texture sampling (3.2.2–3.2.3) and the SPIR-V→GFX9 compiler (3.2.11) are HW-verified on Cezanne, so bite 8 renders the grid via a **single full-screen pass** (compute or fullscreen-FS reading the grid as a storage buffer + the kashi atlas as a texture), not instanced quads.
  - *mabda dep gap found:* the native render target's `va_map` returns `EINVAL` unless the BO byte-size is **64 KiB-aligned** (powers of two pass by luck; `1260×682×4` does not). Worked around in `pgpu_target` by padding the allocated target to a 256-px multiple per axis (always 64 KiB-aligned) and reading back the visible sub-rect. The clean fix is mabda-side (round the GTT/va_map size up to 64 KiB).
- **Glyph atlas** (M6 bite 8a) — `src/render/atlas.cyr` packs kashi's 256 CP437 glyphs (VGA 8×16) into a 128×256 RGBA8 coverage texture for GPU sampling (`atlas_build_kashi` + geometry/cell-origin helpers); verified bit-for-bit against `kashi_glyph_row` (`tests/atlas.tcyr`, 14 assertions). Pure CPU — the data the bite-8 sampling shader consumes; no GPU yet.
- **Alternate screen buffer** (DEC 1049 / 1047 / 47) — the biggest daily-driver gap: `vim`/`less`/`htop`/`tmux` now draw on a separate screen and the primary is restored intact on exit. `grid.cyr` keeps the inactive screen in a **lazily heap-allocated** backup and SWAPS on switch (no static cost until used); `terminal.cyr` wires mode 1049 (save cursor → swap → clear → home on enter; swap back → restore cursor on exit, idempotent), 1047 (clear-on-enter), and 47 (bare swap). The renderer needs no change — the swap marks all rows dirty. 17 conformance assertions in `tests/terminal.tcyr`.

## [0.6.1] — 2026-06-19

**Daily-driver polish — resize + real shell config.** The 0.6.0 MVP gains the two
things a kitty replacement needs day one: the window **reflows** on resize (drag /
maximize / tile), and the hosted shell now loads **your** config (`$SHELL` as a login
shell with the full environment, so `.zprofile`/`.zshrc`/starship all source). GPU
rendering gets its own cut next (M6 bites 7–8).

### Added
- **Window resize** (M6 bite 6) — puka now reflows when the compositor resizes the window (drag, maximize, tile). `xdg_toplevel.configure` → `win_poll_events` raises `WIN_EV_RESIZE` → `win_resize_apply` adopts the new size and refits the `wl_shm` present buffer → `term_resize` reflows the grid, `fb_resize` refits the pixel buffer, `pty_set_winsize` SIGWINCHes the child, full repaint. Buffers are **grow-only** (the bump allocator has no `free`, so per-drag realloc would leak; growth converges to the high-water mark — `shm`'s memfd/pool are explicitly torn down on grow, so it never leaks).
- **Raised grid ceilings** — `GRID_MAX_COLS` 132 → **480**, `GRID_MAX_ROWS` 64 → **144** (covers a 4K window at 8×16; even a 1080p window is 67 rows, past the old 64 cap). The per-row damage bitset is now a **3-word array** (was a single u64, which capped rows at 64). Static data grows ~1 MB (the larger cell backing store) — acceptable for a desktop binary.

### Fixed
- **The hosted shell now loads the user's config** (`.zprofile`/`.zshrc`, starship, etc.). Two compounding bugs starved it: `puka_term` execed `/bin/sh` (which never reads `.zshrc`), and `pty_spawn` gave the child an environment of **only** `TERM` — no `$HOME`, `$PATH`, or `$SHELL`, so even zsh couldn't find its rc or resolve `starship`. Now:
  - `pty_spawn` **inherits the full parent environment** (read from `/proc/self/environ`), overriding only `TERM` → `xterm-256color` (puka's advertised capability, not the launching terminal's). Benefits every PTY consumer, not just the desktop loop.
  - `puka_term` execs the user's **`$SHELL`** (fallback `/bin/sh`) as a **login shell** via the new `pty_login_argv0` override (argv[0] = `-zsh`), so the full `.zprofile → .zshrc` chain sources. The auto-typed `/bin/sh` demo banner is retired — the real shell prints its own prompt.
  - Note: powerline/nerd-font glyphs in a starship prompt still render blank — kashi's built-in font is CP437 8×16 only (a separate font-coverage gap, not a config-loading bug).

## [0.6.0] — 2026-06-18

**The Wayland desktop terminal — direction corrected.** puka is now a real window
in a Wayland compositor (Hyprland), hosting a live shell — the **first windowed
program in the Cyrius ecosystem**, speaking the Wayland wire protocol from scratch
(no libwayland / toolkit / FFI). This supersedes the 0.5.0 framebuffer/console
approach: a desktop terminal is a *compositor client*, not a `/dev/fb0` console. v1
is a daily-drivable **kitty replacement** on Linux/Wayland; AGNOS-native framebuffer
is post-v1; the multi-pane coding-agent command center (panes hosting `thoth`) is v3.

### Added
- **`src/platform/window.cyr`** — the cross-platform window-backend seam (`win_open`/`win_present_begin`/`win_present_commit`/`win_poll_events`/`win_next_key`/`win_close`). Platform-generic names (never `wl_*`) so the contract extracts to the future **`aethersafha`** windowing crate as a *move*, not a rewrite; the engine never references `src/platform/`. mabda's GPU ctx is passed *through* `win_open`.
- **`src/platform/wayland/`** — a sovereign Wayland client over the unix socket: `wire.cyr` (the wire codec — 32-bit message framing, string/u32 arg encoders), `client.cyr` (connect, `wl_registry` bind, the full xdg-shell window lifecycle with configure/ack, `wl_seat`/`wl_keyboard`, **SCM_RIGHTS fd-passing**), `shm.cyr` (memfd-backed `wl_shm` present buffer).
- **`programs/puka_term.cyr`** — the desktop daily-driver: a single-threaded `poll()` over the Wayland fd + the PTY master, hosting `/bin/sh`; child output → grid → **damage-aware** repaint (only changed rows), `wl_keyboard` → `input_encode` → child. The interactive CPU-rendered MVP.
- **`src/render/pixfmt.cyr`** — RGB→XRGB8888 packing + a damage-aware row blit (the device-neutral core that survives the fbdev backend's retirement).
- **`src/input/keymap.cyr`** — the shared evdev-keycode→bytes bridge (`wl_keyboard` delivers *raw* evdev keycodes — not evdev+8 — straight into `evdev__keymap` + the encode discipline).
- **kashi → 1.0.2** and **cyrius pin → 6.2.22** (the latest language).

### Notes
- This is the interactive MVP: a window + a live shell + correct keyboard input + snappy damage-aware rendering, verified on Hyprland. Still ahead toward v1: window **resize** (reflow on `xdg_toplevel.configure`), **mabda GPU** rendering (glyph atlas + instanced cell quads, then zero-copy dmabuf), raising the grid ceilings for large windows, and **retiring the 0.5.0 framebuffer edges** (`fbdev`/`evdev` device layers, `puka_session`).
- The Wayland wire parser is a **new untrusted-input boundary** (compositor messages) — a hardening audit is queued alongside the VT parser.

## [0.5.0] — 2026-06-18

**M5 — Linux live terminal.** The live edges over the M1–M4 core: puka now runs as
a real interactive terminal on a Linux framebuffer + evdev keyboard, with no host
terminal underneath — the first build you can actually *use*. The pure cores are
headless-tested; the device layers are Linux-guarded + skip-clean; the on-screen
session runs on a bare Linux VT. (AGNOS-native bring-up is post-v1.0.)

### Added
- **`src/render/fbdev.cyr`** — Linux `/dev/fb0` display backend. Blits the renderer's 24-bit RGB buffer (`fb_buf()`) onto the mmap'd framebuffer, converting to the device's pixel layout (bpp + RGB bitfields from `FBIOGET_VSCREENINFO`) generically over any truecolor format (32/24/16-bpp, stride-aware). The pure pack/blit core is headless-tested against an in-memory fake fb; the open/ioctl/mmap device layer is Linux-guarded. The ABI struct offsets were validated read-only against real hardware (2560×1440 XRGB8888 `amdgpudrmfb`). Device geometry (`bpp ∈ {16,24,32}`, rows-fit-stride, image-fits-mmap) is validated before the blit trusts it.
- **`src/input/evdev.cyr`** — Linux `/dev/input/eventN` raw keyboard source. Decodes 24-byte `input_event` records via a US-QWERTY keymap with independent left/right modifier tracking, feeds `input_encode`, and emits child PTY bytes; `EVIOCGRAB`s the device so keystrokes don't leak to the host VT. The pure decode + keymap core is headless-tested on synthetic events; the open/read device layer is Linux-guarded. Untrusted device input is bounds-checked (whole records only, output headroom, unknown keycodes dropped).
- **`programs/puka_session.cyr`** — the interactive capstone: spawns `/bin/sh` in a PTY, busy-polls the keyboard + master fd (single-threaded, no threads), encodes keys → child, pumps child output → grid, renders dirty rows → framebuffer. Runs on a bare Linux VT (not under X/Wayland).
- **`tests/fbdev.tcyr`** (25) — pixel-pack + stride/offset/clamp blit against a fake framebuffer. **`tests/evdev.tcyr`** (49) — synthetic-event decode: shift/ctrl/alt, L/R-pair release, arrows/F-keys/nav, digit/symbol shift, no-ops.

### Reviewed
- Multi-agent adversarial review (fbdev bounds / untrusted-device / session-loop / idiom), each finding independently verified. Four fixes applied + regression-tested: validate device bpp + geometry before the blit (untrusted screeninfo → OOB guard); independent L/R modifier tracking (a held pair no longer mis-clears on one release); the `evdev_poll` output headroom guard; and the session loop exits on any `waitpid` result (no hang with the keyboard left grabbed).

## [0.4.0] — 2026-06-18

**M4 — input encoding.** The keyboard half of the terminal: a pure, headless
key→escape-sequence encoder, byte-exact against xterm `ctlseqs` and round-tripped
through puka's own parser. 0.4.0 ships the platform-agnostic encoder + a real PTY
round-trip; the live raw-key SOURCE (Linux evdev / AGNOS xHCI-HID) and the
interactive loop ride with the M5 display backend.

### Added
- **`src/input.cyr`** — keyboard→escape-sequence encoder. `input_encode(sym, mods, out)` maps one keystroke to its exact bytes: printable Unicode (UTF-8), Ctrl→C0 fold, Alt/Meta→ESC-prefix; cursor keys (CSI normally, SS3 under DECCKM, `CSI 1;<mod>` when modified); Home/End; editing keys (Insert/Delete/PageUp/PageDown tilde forms); F1–F12 (SS3 P/Q/R/S and the `15/17/18/19/20/21/23/24~` tilde family); BackTab; and the xterm modifier formula `(mods&15)+1` via the single chokepoint `input__xtmod`. Keysyms use a disjoint range (codepoints `0..0x10FFFF`, named keys at `0x110000+`) so a printable scalar can never collide with a named key — the same self-describing-tag idiom as the colour encoding. Reads terminal modes through getters, never globals.
- **`input_paste(text, len, cap, out)`** — bracketed-paste (mode 2004) wrapper: wraps the body in `ESC[200~ … ESC[201~`, strips both CSI introducers (7-bit `ESC` and 8-bit C1 `0x9B`) from the untrusted body so a paste cannot break out of the bracket to inject commands, and bounds-checks every write against `cap`.
- **terminal.cyr**: `term_app_cursor_get()` (DECCKM) and `term_bracket_paste_get()` getters for the encoder; wired DEC private mode 2004 (bracketed paste) in `term__dec_mode` + reset in `term_init`. +6 `terminal.tcyr` assertions.
- **`tests/input.tcyr`** (67) — byte-exact assertions for every key/modifier category + `vt_feed` round-trips (proving emitter and parser agree) + paste wrap/sanitize/cap-guard tests. **`tests/input_pty.tcyr`** (skip-clean) — encoded keystrokes drive a real `/bin/cat` through a PTY and its echo lands in the grid. **`programs/input_demo.cyr`** types two lines into `cat` via the encoder and re-renders.

### Reviewed
- Multi-agent adversarial review (conformance / bounds / untrusted-paste / idiom), each finding independently verified. One low-severity hardening applied: also strip the 8-bit C1 CSI (`0x9B`) from a bracketed paste body (defense-in-depth against a legacy 8-bit-C1 child), with a regression test.

## [0.3.0] — 2026-06-18

**M3 — framebuffer renderer + glyphs.** puka's first *visible* surface. 0.3.0
ships the platform-agnostic renderer (grid → RGB pixel buffer + glyphs + cursor +
per-row damage), verified headlessly via PPM and visually via `fb_demo`. The live
on-screen display backend — pushing this buffer to a real screen — is folded into
the AGNOS-native bring-up (**M5**); on Linux it is a thin fbdev/KMS adjunct over
the same `fb_buf()`.

### Added
- **`src/render/fb.cyr`** — the renderer. A pure read of the grid (single-source-of-truth invariant: the renderer never mutates) that resolves the full xterm colour model to 24-bit RGB — default fg/bg, the 16 ANSI colours, the 16..231 6×6×6 cube (`level(d)=d?d*40+55:0`), the 232..255 grayscale ramp, and truecolor — applying `bold`=bright / `dim` / `reverse` / `hidden` in the correct order (cursor reuses the reverse swap, so SGR-reverse under the cursor cancels). It paints each cell's background rectangle, blits real CP437 glyphs via **kashi**, draws the cursor block (DECTCEM-aware), tracks per-row damage so a frame repaints only changed rows, and serializes the pixel buffer to a PPM (P6) image — the headless, pixel-assertable verification seam (the renderer's analogue of `term_render_row` for text). Integer-only; every pixel write is bounds-clamped (the renderer is a fresh untrusted-input boundary — cell glyph/colour/attr ultimately come from adversarial PTY output). `programs/fb_demo.cyr` renders a styled grid (SGR colours, bold, reverse, 256-colour, truecolor, cursor) to `puka_frame.ppm`.
- **kashi dependency**: wired the **kashi** crate (v1.0.1, API-frozen) as a **pinned git dep** (`git = ".../kashi.git", tag = "1.0.1"`) — selecting only its **freestanding** `src/font_data.cyr` core (zero stdlib: `store8`/`load8` + arithmetic), the same core the agnos kernel consumes. A pinned git dep (rather than a sibling path dep) resolves identically on a devbox and in CI, so no sibling checkout is needed. Gives the built-in CP437 bitmap fonts (VGA 8×16 / CGA 8×8 / VGA 9×16); the PSF/BDF/PCF loader library face is deliberately *not* pulled in (`modules = ["src/font_data.cyr"]`).
- **Per-row damage tracking** (`grid.cyr`): a `u64` dirty-row bitset marked at every grid write chokepoint — cell writes, row copy/blank, insert/delete, cursor moves (mark **both** old and new rows), `init`/`resize` (mark all) — consumed + cleared by the renderer each frame. Public surface: `grid_row_dirty` / `grid_clear_row_dirty` / `grid_mark_row_dirty` / `grid_mark_all_dirty`. +14 `grid.tcyr` assertions.
- **`term_cursor_visible()`** (`terminal.cyr`): DECTCEM visibility accessor for the renderer; the DECTCEM (`?25h`/`?25l`) handler now dirties the cursor row on a real toggle so the block repaints even on an otherwise-quiescent screen.
- **`tests/render.tcyr`** — 44 assertions: palette/cube/grayscale/clamp, token decode, the attribute resolve order, background + cursor pixel paint, the kashi glyph blit ('A' lights pixels, blank cell does not), the DECTCEM re-dirty fix, the `fb_init` geometry-overflow guard, and the PPM P6 header.

### Reviewed
- Multi-agent adversarial review (correctness / bounds & overflow / Cyrius-idiom / untrusted-input), every finding independently verified before acceptance. Fixed two confirmed defects, both regression-tested: a DECTCEM dropped-repaint (a bare cursor hide/show on a settled screen left a ghost / failed to reappear), and a `fb_init` integer-overflow where an out-of-range cell size could wrap the buffer size to a small positive value and undersize the allocation while the plot bounds stayed huge (now cell metrics are capped and the geometry is overflow-checked before `alloc`).

## [0.2.0] — 2026-06-18

### Added
- **M2 — PTY + process plumbing (Linux)**: `src/pty.cyr` — allocate a pseudo-terminal (`/dev/ptmx` → `TIOCSPTLCK` unlock → `TIOCGPTN` → `/dev/pts/N`), `fork` + child-side controlling-tty setup (`setsid` / `TIOCSCTTY` / `dup2` slave→0,1,2) + `execve` with explicit argv (never a shell string), bounded non-blocking pump (`pty_pump`) that reads the master and feeds bytes into `term_feed`, plus `pty_write` (input path, used in M4), `pty_set_winsize`, `pty_wait`, `pty_close`. Linux-guarded (`CYRIUS_TARGET_LINUX`) with non-Linux stubs; agnos PTY is M5. `tests/pty.tcyr` — spawns a real `/bin/echo` and asserts its output lands in the grid (skip-clean if the sandbox blocks `/dev/ptmx`/fork).
- **Resize**: `grid_resize` + `term_resize` — logical screen resize (SIGWINCH/`TIOCSWINSZ`): newly-exposed cells blanked, scroll region reset, cursor clamped, no reflow (matches xterm/VT). Backing store is fixed-max so this is a dimension change, not a realloc. +9 grid assertions.
- **Live demo**: `programs/pty_demo.cyr` — runs `/bin/ls /` inside a PTY sized to the grid and re-renders the captured output (also demonstrates winsize propagation: `ls` columnates to the grid width).

## [0.1.0] — 2026-06-18

First cut: the headless, platform-agnostic VT core. Fully unit-tested on the
Linux host; no PTY / rendering yet (those are M2 / M3).

### Added
- Initial project scaffold (`cyrius init puka`); cyrius pin `6.2.21`.
- **VT parser** (`src/parser.cyr`): Paul Williams DEC ANSI parser state machine, pure (bytes in → one typed event out via `vt_feed`, no I/O / no rendering / no hot-path allocation). GROUND/ESCAPE/CSI/OSC fully handled; DCS + SOS/PM/APC consumed-and-discarded (full DCS passthrough is M6). Events PRINT/EXECUTE/CSI/ESC/OSC with `vt_param*`/`vt_prefix`/`vt_intermediate`/`vt_string_*` accessors. UTF-8-mode (bytes ≥0x80 printable; no 8-bit C1). `tests/parser.tcyr` — 70 assertions.
- **Cell grid** (`src/grid.cyr`): the screen model and single source of truth — cells packed 2×i64 (glyph+attrs / fg+bg) behind accessors, cursor, scroll region, tab stops; scroll (region + sub-region), erase, insert/delete-cell, bce blanking. `tests/grid.tcyr` — 43 assertions.
- **Unicode** (`src/unicode.cyr`): incremental UTF-8 decoder (RFC 3629; overlong/surrogate/out-of-range → U+FFFD), UTF-8 encoder, `char_width` wcwidth (UAX#11 East-Asian-Wide + main emoji + common combining ranges; sourced). `tests/unicode.tcyr` — 28 assertions.
- **Terminal** (`src/terminal.cyr`): the driver — pumps bytes through the parser, applies VT semantics to the grid. Printing with deferred autowrap + wide-glyph placement; C0 (BS/HT/LF/VT/FF/CR); CSI cursor moves (CUU/CUD/CUF/CUB/CNL/CPL/CHA/VPA/CUP/HVP), ED/EL, IL/DL/ICH/DCH/ECH, SU/SD, DECSTBM, SGR (attrs + 16/256/truecolor), TBC, IRM, save/restore cursor; DEC private modes DECCKM/DECOM/DECAWM/DECTCEM; ESC IND/RI/NEL/DECSC/DECRC/RIS/HTS. Headless text renderer (grid → UTF-8, wide-spacer aware). `tests/terminal.tcyr` — 49 end-to-end assertions. DA/DSR responses + charset designators + alt-screen deferred (need the PTY writer / M6).
- Demo entry (`src/main.cyr`): drives a canned byte stream through the full pipe parse → grid → render.
- Design docs: `CLAUDE.md`, `docs/development/roadmap.md` (M1–M7 + v1.0 criteria + phase-2 command center), `docs/architecture/overview.md`, ADR-0001 (sovereign reimplementation, no libghostty), ADR-0002 (app-first, engine extracted later), `docs/development/state.md`, README. Root docs the scaffolder skipped: `CONTRIBUTING.md`, `SECURITY.md`, `CODE_OF_CONDUCT.md`.

**192 assertions green across 5 test files.**
