# Changelog

Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

## [0.6.24] - 2026-09-27 — toolchain 6.6.6, current dependencies, and a manifest that is only configuration

No change under `src/`. The toolchain and all three direct dependencies move to their latest
releases. `cyrius.cyml` is cut back to configuration, and the lock now pins every git dependency by
commit.

### Changed — toolchain `6.6.2` → `6.6.6`

The 6.6.3–6.6.6 changelog was read for consumer-visible changes. The stdlib functions and constants
puka uses were diffed between the two tags.

- No stdlib name puka uses was removed or renamed, and none changed arity.
- The `#97` channel wrappers and `CH_E_*` codes `src/pty.cyr` uses are unchanged. `SYS_IOCTL` is
  still absent from the agnos syscall peer, so the reason mabda is not declared still holds.
- None of the silent miscompiles fixed in the range involves a construct puka uses. 6.6.3 also
  fixed `CYRIUS_DCE=1` binaries that could crash before `main`; the release build uses that flag.
- `cyrius.lock` gains the `cyrius 6.6.6` trailer (a 6.6.4 lock feature). The stdlib is re-vendored
  from the 6.6.6 snapshot, and `lib/alloc_cx.cyr` (new in 6.6.6) comes with it.
- **`lib/hashmap_fast.cyr` removed.** It had sat in `lib/` since the scaffold. Nothing includes it,
  and a clean `cyrius deps` does not vendor it; the lock listed it only because the file was there.

### Changed — dependencies to their latest tags

| dep | 0.6.23 manifest | 0.6.23 actually vendored | 0.6.24 |
|---|---|---|---|
| kashi | 1.0.6 | 1.0.10 | **1.0.10** |
| setu | 0.8.8 | 0.8.9 | **0.8.11** |
| dhancha | 0.9.26 | 0.9.29 | **0.10.4** |
| sadish (via dhancha) | — | 0.5.4 | **0.11.2** |
| rupa (via dhancha) | — | 0.1.7 | 0.1.7 (dhancha's pin; 0.1.8 exists) |
| rekha (via dhancha) | — | 0.3.6 | **0.9.0** |

⚠ **The middle column is a defect this release fixes.** At 0.6.23 the manifest named one set of tags
and `lib/` held another. Live `path = "../sibling"` lines vendored whatever the sibling checkouts
held, while CI, which has no siblings, built the tags. The dev box and CI were building different
dependencies. The `path` lines are now commented out, so every dependency resolves from its tag.
`cyrius.lock` pins all six git dependencies by commit (it pinned one). `lib/` and the lock are
byte-identical to a clean `cyrius deps` run in a scratch copy with no siblings and an empty `lib/`.

- **No call site changes.** The 16 `dh_*` functions puka calls keep their signatures (0.9.29 →
  0.10.4), as do the 5 `setu_*` functions (0.8.9 → 0.8.11). The `SETU_*` constants puka reads are
  unchanged.
- **setu resolves at puka's 0.8.11**, not the 0.8.9 dhancha 0.10.4 declares. puka calls setu
  directly and declares it itself.
- **dhancha 0.10.4 draws text by UTF-8 character** instead of by byte. puka draws no dhancha text
  (the grid is a canvas widget), so nothing changes on screen.
- **setu 0.8.11 changes present failures.** `setu_client_present` no longer falls back to inline
  pixels: a failed present returns -45 or -46 with nothing sent, and puka only propagates the code. A
  resize now keeps the old buffer until the new one is attached.
- ⚠ **setu 0.8.11 also makes `setu_client_poll_input` return -6 on Linux** when the compositor is
  gone. `win_poll_events` treats every result but 1 as no event, so a compositor that dies without
  sending `SETU_CLOSE` still leaves puka running, exactly as before. Mapping -6 to `WIN_EV_CLOSE` is
  a separate change.

### Changed — `cyrius.cyml` is configuration only

118 lines down to 48. The manifest had accumulated dependency history, measured sizes and dated notes.
The durable parts moved:

- **Why each dependency is declared as it is** now lives in
  [`docs/architecture/001-dependencies.md`](docs/architecture/001-dependencies.md) (new). It covers
  why every declared dep is compiled into every target, why mabda is absent, the cyrius 6.5.8 floor,
  what goes through dhancha and what calls setu directly, what `dist/puka.cyr` consumers must
  declare, and why the `path` lines stay commented.
- **Why kashi is consumed as its freestanding core** is now
  [ADR 0004](docs/adr/0004-kashi-freestanding-core-over-library-face.md) (new). It keeps the measured
  +50% cost of the library face.
- **The history** was already in this changelog (0.6.4, 0.6.8, 0.6.13, 0.6.21).

Two comment lines remain, pointing at those documents, plus the commented `path` lines.

### Fixed — docs

- The 0.6.23 entry and the `[Unreleased]` heading had been appended below 0.1.0. Both are back at the
  top.
- README and `getting-started.md` said `cyrius deps` resolves "(kashi, mabda)". mabda has not been a
  dependency since 0.6.8.
- README's dependency section named the 0.6.21 pins and sent readers to `cyrius.cyml` for the mabda
  reasoning.
- `state.md` claimed 604 tests across 17 files. The suite is **589 assertions across 15 files**,
  identical before and after this release. The 604 was 589 plus the runner's closing
  `15 passed, 0 failed` line, which counts files.

### Verification

For the baseline, 0.6.23 was rebuilt exactly. Its `lib/` was reproduced hash for hash, and its plain
build is byte-identical to the committed `puka`.

- **Tests:** `cyrius test` 589 passed, 0 failed, the same 589 as 0.6.23.
- **Fuzz:** `cyrius fuzz` 1,297 green, also under `--poison`.
- **Bench:** `cyrius bench` within run-to-run noise of 0.6.23 over three runs each.
- **Builds:** Linux with and without DCE, `--agnos` entry and tests, and `--aarch64` all build with no
  undefined functions.
- **Static checks:** `cyrius lint` 0 warnings across `src/`, `tests/` and `programs/`. `fmt --check`
  is clean, `dist/` included.
- **Dependency checks:** `vet` reports 14 deps, 0 untrusted, 0 missing. `distlib --check` reports
  current, and `deps --verify` 41/0.
- **`programs/`:** everything builds except `gpu_probe` and `gpu_win_probe`, which fail the same way on
  0.6.23 (see *Known* below).
- ⚠ **`cyrius audit` exits 1** on its docs step: 433 undocumented public functions. It exits 1
  identically on 0.6.23 under 6.6.2. Its fmt, lint, tests and bench steps pass.

| build (bytes) | 0.6.23 | 0.6.24 | Δ |
|---|---|---|---|
| Linux, `CYRIUS_DCE=1` (CI / release) | 1,484,832 | 1,577,544 | +92,712 (+6%) |
| Linux, plain | 1,710,112 | 2,073,160 | +363,048 (+21%) |
| `--agnos` | 1,696,376 | 2,055,200 | +358,824 (+21%) |
| `--agnos` tests | 502,504 | 861,320 | +358,816 (+71%) |
| `--aarch64` | 2,025,336 | 2,490,736 | +465,400 (+23%) |

Nearly all of the growth is the draw stack dhancha pulls in. `lib/sadish.cyr` went from 75,450 to
471,512 B and `lib/rekha.cyr` from 38,661 to 551,402 B. DCE removes most of it. ⚠ The `--agnos` build
is not DCE'd, and that is the binary that has to spawn on iron. 0.6.13 recorded the `spawn_path #43`
size question as never measured on iron; it is now 21% larger.

⚠ **No agnos or live-compositor run.** Every result above comes from the Linux dev host; for
`--agnos`, the result is that it compiles.

### Known, not changed

- `src/platform/gpu/gpu.cyr` and the two GPU probes do not compile on cyrius 6.6. The seam reads a
  `Result` through `payload()`, which 6.6.0 deleted, and binds a two-value return to one variable.
  Nothing that is built includes it, and it fails identically at 0.6.23. Port it when the GPU path is
  wired.
- `mmap`, `dynlib` and `sakshi` are declared stdlib leaves that nothing in `src/`, `programs/` or
  `tests/` references; they were added for mabda's bundle. They are still listed in
  `dist/puka.deps`, and pruning them changes what bundle consumers are told to declare, so it is a
  separate change.

## [0.6.23] - 2026-09-11

### Changed

- **Toolchain `6.5.36` → `6.6.2`.** No source change; the value form needed none.
  Build, tests, and any bench/fuzz/distlib target the repo ships re-verified at the new pin.

## [0.6.22] - 2026-08-31 — the first audit, and two gates that were green over nothing

P(-1) hardening sweep. All of `src/` (5,788 lines) read line-by-line; findings filed in
[`docs/audit/2026-08-31-audit.md`](docs/audit/2026-08-31-audit.md) and **repaired in this release**
rather than carried. `cyrius lint` reports 0 warnings across every file and found none of it.

### Fixed — 🔴 two lazy allocations that poisoned their own guard, then wrote to the null page

`grid_alt_screen` and `grid__sb_push` both published their FIRST pointer before knowing the second
had succeeded — and `grid_alt_g == 0` / `grid_sb_g == 0` *is* the "not yet allocated" test. A
half-successful pair therefore poisoned the guard permanently: every later call skipped the alloc
block and ran against a null second pointer — `store64(0 + i * 8, …)` across all **69,120** cells for
the alt screen, a full row per scroll for the ring.

⛔ **REACHABLE FROM UNTRUSTED PTY OUTPUT, not merely from memory pressure.** `ESC[?1049h` is what vim,
less, htop and tmux send on startup, so any byte stream can ask for the alt screen at will; the
scrollback path is reached by **ordinary scrolling**. Both functions returned `-1` on the first
failure and every one of the five call sites discarded it, so the first half of the sequence was
silent and the second half was corruption.

⇒ Both now allocate into locals and publish **both pointers or neither**, which is what makes the
zero-test an honest question again.

### Fixed — 🔴 an unvalidated wire length in the Wayland client: OOB read + information disclosure

`wl_registry.global`'s interface-name length is a `u32` straight off the socket, and nothing tied it
to the message that carried it. A `global` event declaring `L = 0xFFFFFFFF` made
`load32(&wl_rbuf + wire_str_next(o + 12, L))` a read roughly **4 GB past** an 8,192-byte buffer; with
`wl_verbose` on, `wl__emit(sp, L - 1)` wrote that many bytes of puka's own address space to stdout.

⭐ **`wl__streq` WAS ALREADY SAFE, AND IS THE MODEL THE FIX FOLLOWS** — it compares `strlen(lit)`
against `len` first, so a bogus length never reached its loop. The other two readers of `L` had no
such check. The length is now validated against the message's own declared size before anything
indexes with it.

⛔ **AND EVERY OTHER FIELD READ WAS UNBOUNDED TOO.** Handlers loaded at fixed offsets (`o + 8`,
`o + 16`, `o + 20`) with nothing checking the compositor sent that far. `wl__parse` bounds a message
against the bytes RECEIVED, which is a different fact from "long enough to hold this field": a
truncated 8-byte message carrying a key opcode, arriving at the tail of a full buffer, read 16 bytes
past it. All field reads now go through a size-checked `wl__u32`; a short field reads 0, which every
consumer already treats as absent.

⚠ Scope, honestly: the compositor is a semi-trusted peer and this is not remotely exploitable — puka
has no network surface. It matters because a sovereign client that decodes the wire itself does not
get to assume the peer's framing is honest, and `wire.cyr` is the file this project's own docs call
*"the new untrusted-input boundary"*.

### Fixed — unbounded heap leaks on the per-keystroke and per-frame paths

`alloc` has no `free`, so an allocation inside the event loop is permanent.

- **64 B per keystroke** — `src/main.cyr` allocated its key scratch twice inside the loop body.
  ⛔ **The file states this exact rule 130 lines earlier**, for `puka_szpx`: *"Allocated ONCE — the
  loop reads it every frame and `alloc` has no free, so a per-frame allocation would leak 16 B per
  present for the life of the terminal."* The key path predated that idiom. Both returns were also
  unchecked — a failed `alloc` returned 0 and `win_next_key(0, 0)` would have stored through null.
- **48 B per number printed** — `puka__say_num` used two `alloc(24)`. Not a cold path: the resize
  branch calls it **six times per compositor configure**, and a drag configures continuously.
- **128 B per frame on agnos** — `pty_pump` allocated per call, and `main` calls it once per frame:
  ~**7.7 KB/s at 60 fps**, unbounded. The Linux arm directly above already used a stack buffer.
- **`pty_open` (agnos)** allocated its fd pair per call — unchecked, and `pty_open` legitimately
  repeats after a `pty_close`.

### Fixed — a byte that aborted a UTF-8 sequence was eaten

`E2 41` — a truncated 3-byte lead followed by `'A'` — yielded U+FFFD and **silently dropped the
'A'**. The aborting byte was never part of the sequence it interrupted; it starts the next character.
The decoder now flags it unconsumed (`utf8_rejected`) and `term_feed` re-dispatches it exactly once —
provably once, because the decoder is back in ground state and that branch cannot flag again. Matches
Unicode 15 §3.9 maximal-subpart practice and every other terminal. ⚠ A caller that ignores the flag
behaves exactly as before.

### Fixed — the child could inherit a truncated environment variable

`pty__build_child_env` recorded each entry's pointer BEFORE scanning for its NUL, so an environment
cut off by the 8,191-byte read cap handed the child a complete-looking `PATH=/usr/bin:/usr/lo…`. A
silently shortened PATH or HOME misbehaves in ways that look like anything except a truncated
environment. An unterminated tail is now dropped — a missing variable fails loudly, a corrupt one
does not. ⚠ The caps themselves (8,191 B / 125 vars) stay silent, deliberately: raising them is a BSS
decision, not a correctness one.

### Fixed — bounds guards that were asymmetric with the rest of their own module

- `grid__copy_row`, `grid_insert_cells`, `grid_delete_cells` stride the backing store directly and
  carried **no row guard**, while every other write path in `grid.cyr` checks. Safe only because all
  callers happen to pass the cursor row — an invariant that was real and **written down nowhere**.
- `grid__view_gword` / `grid__view_cword` had no range checks, while their non-viewport twins
  (`grid_glyph` / `grid_fg` / `grid_attr` / `grid_bg`) all do.
- `win_open` had no overflow backstop on `win_w * win_h * 4`, though `win_resize_apply` did — and
  `win_open` is the path that takes cols/rows from the caller rather than from the grid's clamps.

### Fixed — 🔴 two P(-1) gates were passing while measuring nothing

⛔ **`cyrius fuzz` fuzzed NOTHING.** `fuzz_main` was `if (len == 0) { return 0; } return 0;` — it
printed `fuzz: ok` and passed, for the module CLAUDE.md names as *the highest-priority audit target,
the untrusted-input boundary*. A harness that cannot fail is worse than no harness: it is a claim of
coverage that does not exist.

⇒ Replaced with a real harness against **`term_feed`** — the whole untrusted path (parser → UTF-8 →
grid), not just `vt_feed`, because the interesting defects live at the seams. 16 targeted adversarial
sequences (every cursor/edit/scroll verb driven at the 65,535 parameter clamp, degenerate and
inverted scroll regions, alt-screen thrash across all three modes, ESC storms, OSC overflow past
`VT_STR_CAP`, truncated UTF-8 spanning an escape, overlong/surrogate/out-of-range encodings) plus 400
rounds of fixed-seed pseudo-random and escape-biased noise — roughly **820,000 adversarial bytes**,
asserting grid invariants after each. **1,297 assertions, all green, and also green under
`cyrius fuzz --poison`.** Deterministic by construction, so a failure reproduces exactly.

⭐ **MUTATION-VERIFIED — "the harness is real now" is the one claim here that must not be taken on
trust.** Deleting the cursor-row clamp from `grid_cursor_set` turns the run red: **4 failures** across
both the targeted and the escape-noise sections. The stub it replaced passed that same mutation in
silence.

⛔ **`cyrius bench` timed an empty function.** It could not have detected a parser rewritten to be ten
times slower. Replaced with five hot-path baselines — see below.

### Changed — measured optimizations, all provable rather than heuristic

| bench | before | after |
|---|---|---|
| `vt_feed` printable | 10 ns | 10 ns |
| `vt_feed` CSI SGR sequence (7 B) | 123 ns | 122 ns |
| **`term_feed` printable (full pipeline)** | **300 ns** | **186 ns** |
| `grid_scroll_up` 1 line (+scrollback) | 10.29 µs | 10.24 µs |
| **`fb_render` 80×24 all-dirty frame** | **3.177 ms** | **1.694 ms** |

- **`char_width` ASCII early-out.** Every plain `'A'` walked ~15 range checks in `uc__zero_width` and
  ~15 more in `uc__wide` — thirty calls to `uc__in` — to return 1. ⭐ The early-out is PROVABLE, not a
  guess: the lowest zero-width entry is U+0300 and the lowest wide entry is U+1100, so nothing in
  `0x20..0x7E` can be either (and NUL, which *is* zero-width, is excluded by the lower bound).
- **Damage bitset in shifts, not division**, and `grid_cursor_set` no longer marks the same row twice
  when the cursor stays on its line. That path runs three times per printed character.
- **`fb__fill_rect` clamps once per rectangle** instead of calling the per-pixel guarded `fb__plot`
  **245,760** times per frame. ⛔ **The safety property is unchanged and that is the point** — this is
  not "trust the caller": the clamp is the SAME clamp, hoisted out of the loop, so every offset
  written is inside `fb_w * fb_h * 3` by construction. `fb__plot` remains the guarded path for
  scattered glyph writes.

⚠ `fb_render` at 1.7 ms is an **all-dirty upper bound, not a per-frame cost** — the damage bitset
means a keystroke repaints one row. The remaining cost is the per-cell glyph blit; not pursued, and
recorded in the audit rather than silently dropped.

### Added — the wheel scrolls the scrollback (the deferred half of 0.6.21)

setu 0.8.8 shipped `SETU_INPUT_PTR_SCROLL` (kind 12) and 0.6.21 recorded it as available and
unconsumed. Now wired: a new `WIN_EV_SCROLL` bit, `win_next_scroll` on both backends, and the event
loop driving `grid_scroll_view`.

⛔ **`grid_scroll_view` AND THE WHOLE VIEWPORT RING HAVE EXISTED SINCE THE SCROLLBACK BITE, AND
`src/main.cyr` NEVER MOVED THEM.** `programs/puka_term.cyr` wired PageUp/PageDown on the Wayland
path, so the feature looked done — while the terminal that actually runs on agnos had no way to reach
history at all. Typing now also snaps back to the live bottom via `grid_view_reset`, which was
written for exactly that ("on user input", `grid.cyr`) and likewise had no caller on this path.

⚠ Deltas are accumulated, not latched-one: the compositor already sums detents since its last
forward, so a frame carrying two messages must not drop the first. ⚠ Only useful with
agnos >= 1.56.49 / bhumi >= 1.4.3 — below those the kernel's HID drain discarded the wheel byte.
⚠ The Wayland backend's `win_next_scroll` honestly returns 0: `wl_pointer` is not bound yet.

### Fixed — `fmt` was failing, and had been (the deferred half of 0.6.21)

11 `src/` and `tests/` files plus the generated bundle needed reformatting; the *committed*
`dist/puka.cyr` was already fmt-dirty before regeneration. Whitespace only — `git diff -w` over the
reformat is empty (the paren-continuation reindent). `cyrius audit`'s fmt gate is now clean
tree-wide, `programs/` and `dist/` included.

### Fixed — a header that under-claimed its own module

`window_setu.cyr` said *"Input over setu is a later bite … `win_poll_events` / `win_next_key` are
present-parity stubs for now"* — false for several releases; the file handles `SETU_CLOSE`,
`SETU_CONFIGURE` and `SETU_INPUT_KEY` with a full HID-usage→evdev translation. A header that
under-claims invites someone to re-implement a seam that already works.

### Testing

**604 passed, 0 failed** (566 → 604; +38). New regression coverage for every behavioural fix: the
UTF-8 re-dispatch at both decoder and terminal level, the grid row guards, the viewport accessor
guards, adversarial CSI params against every grid verb, and `fb__fill_rect`'s clamp (negative
origins, both-axis overruns, fully-off-screen and negative-extent rectangles).

⚠ **One of those tests failed first, and the test was wrong, not the code** — a rectangle at
(-100,-100) sized 40×40 spans x ∈ [-100,-60) and is entirely off-screen, so asserting it painted the
corner was the error. Corrected to a rect that genuinely straddles, plus a separate case for the
fully-negative extent.

⛔ **NOT COVERED, AND IT IS THE FIX THAT MOST DESERVES IT**: A-03/A-04, the Wayland wire validation.
`client.cyr` needs a live compositor and the subsystem has no headless harness, so those two are
verified by reading only. That gap is the strongest argument for the `wire.cyr` byte-vector tests
already scoped in the roadmap.

### Verification

`cyrius test` 604/0 · `cyrius fuzz` 1,297 assertions green (and under `--poison`) · `cyrius lint` 0
warnings · `cyrius fmt` clean tree-wide · `cyrius vet` 14 deps, 0 untrusted, 0 missing ·
`cyrius distlib --check` current · `--agnos` and `--aarch64` both build.

⚠ **No agnos or live-compositor run.** Source review plus headless tests on the Linux dev host; the
`--agnos` result is that it compiles. Every runtime claim in this entry is a benchmark or a test on
that host, not an iron burn.

## [0.6.21] - 2026-08-31 — the stack catches up, and it skipped a compiler bug on the way

### Changed — toolchain pin **6.5.28 → 6.5.36**

⭐ **puka was never exposed to the one critical defect in that range, because it sat below it.**
cyrius 6.5.36 fixes *"enum constants ≥ 2^62 were silently corrupted"* — **shipped in `.31`–`.35`**.
puka's pin was `.28`, so the whole broken window is behind us and no build ever emitted a corrupted
constant. Recorded because "we jumped eight patch versions" otherwise invites exactly that question.

⚠ **One BREAKING stdlib removal in the range, verified benign here**: yukti dropped the unprefixed
`str_starts_with_cstr/2`. **Zero callers in `src/` or `tests/`** (grep-verified); `lib/str.cyr` keeps
the real `str_starts_with`.

⚠ The pin/`cycc` drift warning is gone — the manifest now matches the installed toolchain. It was
`warning: cyrius.cyml pins 6.5.28 but cycc is 6.5.36` on every build before this.

⛔ **`sys_ioctl` landing in 6.5.36 does NOT unblock mabda, and the note in `cyrius.cyml` stands.**
That addition wraps ioctl for the ELF/Mach-O peers; the **agnos** peer still has no `SYS_IOCTL`
(0 occurrences in the freshly vendored `lib/syscalls_x86_64_agnos.cyr`, versus `SYS_IOCTL = 16` in
`lib/syscalls_x86_64_linux.cyr`), and `dist/mabda.cyr` still names it **45 times**. Declaring mabda
would still fail the `--agnos` build before a line of puka compiles. Checked, not assumed — a new
`sys_ioctl` in the changelog is precisely the thing that reads like the blocker was cleared.

### Changed — dependency refresh to current tags

- **dhancha 0.9.12 → 0.9.26** (14 tags). All **13** `dh_*` entry points puka calls —
  `dh_client_connect/_fd/_close`, `dh_widget_new/_add_child/_set_bg/_set_flex/_set_layout`,
  `dh_canvas_new/_blit_rgb24`, `dh_surface_wrap`, `dh_layout_apply`, `dh_draw_widget` — are
  **arity-identical** in 0.9.26; signatures were diffed against the call sites, not assumed from the
  changelog. The range is additive (LIST/GRID/MENU/SHEET, stable widget keys); puka consumes none of it.
- **setu 0.8.7 → 0.8.8** — additive only: `SETU_INPUT_PTR_SCROLL` (kind 12), `SETU_KIND_MAX` 11 → 12.
- **kashi 1.0.6 — unchanged, already latest.** Still the freestanding `src/font_data.cyr` core; the
  measured +50% cost of the library face recorded in `cyrius.cyml` is unchanged and still declined.
- Transitive, via dhancha: **sadish 0.5.2 → 0.5.3**, **rupa → 0.1.6**, **rekha 0.3.5** (unchanged).
- stdlib snapshot re-vendored from 6.5.36: `fmt`, `io`, `sakshi`, and the four `syscalls_*` peers.

⭐ **The target set is not a guess — it is dhancha 0.9.26's own manifest.** That release pins
cyrius 6.5.36 / setu 0.8.8 / sadish 0.5.3 / rupa 0.1.6 / rekha 0.3.5 / kashi 1.0.6, which is exactly
the set adopted here. The stack is internally consistent rather than independently latest.

⚠ **`SETU_INPUT_PTR_SCROLL` is available and puka does NOT consume it** — wheel input still does
nothing. Stated because "setu 0.8.8" plus "terminal with scrollback" reads like scroll now works.
It is **safely ignored, not misread**: `win_next_key` dispatches on explicit equality
(`== SETU_CLOSE`, `== SETU_CONFIGURE`, `!= SETU_INPUT_KEY → WIN_EV_NONE`), so kind 12 falls through
to `WIN_EV_NONE`. Wiring it to `grid_scroll_view` is a feature, not a dep bump.

### Changed — `dist/puka.cyr` regenerated

⚠ **The engine bundle was stale before this release, and not because of it**: `dist/` was last built
at `6012281`, `src/` changed later at `da4d4c1`, so `cyrius distlib --check` was already failing on a
clean tree. Regenerated — and the diff is **exactly one line**, the `# Version:` stamp (`0.6.17` →
`0.6.21`). Every engine module is **byte-identical**; the three files that had moved (`main.cyr`,
`pty.cyr`, `window_setu.cyr`) are all in the app half, none in `[lib] modules`. `distlib --check` is
now current and `tests/engine_bundle.tcyr` (11) passes against the new bundle.

### Verification

`cyrius test` **566 passed, 0 failed** · `cyrius fuzz` PASS (the parser's untrusted boundary) ·
`cyrius vet` 14 deps, **0 untrusted, 0 missing** · `cyrius distlib --check` current ·
cross-target builds green: **`--agnos`** and **`--aarch64`**, the two most likely to break on a
toolchain move. Every vendored `lib/` artifact **hash-matches** its upstream `dist/` bundle
(dhancha, setu, rupa, sadish, rekha, kashi) — the lock is not merely self-consistent, it agrees
with the sources on disk.

⚠ **`cyrius audit` still reports `fmt: FAIL`, and this release does NOT fix it.** Pre-existing: 11
`src/`/`tests/` files plus the generated bundle need reformatting, and the *committed* `dist/puka.cyr`
was already fmt-dirty before regeneration. No `src/` or `tests/` file was touched here, so
reformatting them would bundle an unrelated whole-tree diff into a dependency bump.

⚠ **No agnos or Wayland runtime run.** This is a build/test/hash-level verification on the Linux dev
host. The `--agnos` result above is that it **compiles**, nothing more.

## [0.6.20] - 2026-08-22 — the kernel floor is 1.56.46, because below it nothing you run prints

### Changed — documented kernel floor 1.56.40 -> **1.56.46**

No code change. `#97` has existed since 1.56.40, so puka *appeared* to work on 1.56.40..1.56.45: the
window opens, keys echo, agnsh answers and every BUILTIN renders — while **every program the shell
launches is silent**.

Cause is in the kernel, not here: `chan_auth` accepted only `chan_end_owner == proc_current_get()`,
and a child that inherits the terminal fd by fd-table copy is not the owner, so its writes were
refused (`CH_E_BADFD`) and discarded. agnos 1.56.46 adds `chan_end_pty[]` and admits a DESCENDANT of
the owner to a PTY-mode endpoint.

Measured, QEMU (`agnos/scripts/harness/puka-child-stdout-test.py`, glyph px at RGB 192,192,192):
before, `ls` / `ls /` / `kriya ls` / `kriya ls /bin` all rendered an identical **266** — agnsh's
prompt; after, `ls` renders **1560**. Builtin control `help` unchanged at +11,186 throughout.

⚠ puka already endowed with `CH_ENDOW_STDIO` and needed no change. The floor is recorded because
"the terminal works but nothing you run in it prints" is otherwise indistinguishable from a puka bug.

## [0.6.19] - 2026-08-19 — a silent present, and two tests that waited for silence

### Fixed — only frame 0 reported its present result

`win_present_commit`'s return code was checked on the first frame and discarded thereafter, so a
surface that STOPPED reaching the compositor looked exactly like one arriving fine. That is the
worst case for a resize: setu recreates the `#86` slot at the new byte count, and if that create is
refused the client keeps rendering into a buffer nobody reads. Now reported (bounded at 8) with the
rc and the extent.

⭐ It paid for itself immediately. In QEMU, maximizing produced
`puka: present REFUSED, rc 43 at 2048x2016` — rc 43 is setu's inline fallback failing after
`setu_buf_create` refused 16.5 MB. QEMU has no GPU carveout, so buffers go through `#71` at a **2 MB**
cap; on iron `#86` allows 32 MB and 2560x1408 = 14.4 MB fits. Without this line the run was
indistinguishable from the compositor-side defect it was being used to diagnose.

### Changed — cyrius pin 6.5.28 (unchanged); no functional change to the resize path

0.6.18's reflow is intact. The compositor-side half of that defect is aethersafha 0.16.13.


### Fixed — `tests/pty.tcyr` failed in CI and passed locally

`FAIL: cursor advanced past the echoed line (got 0, expected 1)`, with the row-content assertion
PASSING in the same run — so the text arrived and the trailing newline did not.

`pty_pump(max_idle)` returns after N **consecutive empty reads**, which is not the same fact as "the
child is done". `/bin/echo` writes `hello world\n` and the pts expands it to `hello world\r\n`; when
a loaded runner splits that across two reads, the idle budget can expire between them. The test then
asserts on a half-consumed stream. It is timing, not content — which is why it passed 6/6 locally
and failed on CI.

⇒ Both pty tests now pump **until the state under test is reached**, with a bounded round count so a
child that never writes still fails, and fails for the right reason.

⭐ **Demonstrated, not assumed.** Starving the budget to `pty_pump(1)` to force the split: the old
single-pump shape failed **2 assertions in 6 of 6 runs**; the new loop passed **6 of 6** on the same
budget. Passing runs alone proved nothing here — the flaky version also passed 6/6 at the normal
budget.

### Fixed — `tests/input_pty.tcyr` had the identical latent shape

`pty_pump(200)` then assert on the pts echo of `hi`. Same race, not yet observed failing. Hardened
the same way rather than waiting for a red CI to find it.


## [0.6.18] - 2026-08-19 — the terminal follows the window

### Fixed — maximizing puka grew the frame and not the terminal

Iron 2026-08-19: "fullscreen of puka expands the window but not the terminal display itself."
Maximize changes the WINDOW; the surface belongs to the client, so only the client can grow it.
aethersafha has SENT `SETU_CONFIGURE` since 0.16.8 and clamps its blit until the client re-attaches
— puka never handled the message, so the grid stayed 80x24 inside a full-screen frame.

⛔ **The contract existed at three layers with no consumer.** `WIN_EV_RESIZE` and `win_resize_apply`
were already in the platform ABI and the wayland backend already raises the event; the setu backend
stubbed `win_resize_apply` to `return 0` ("no compositor-driven resize yet"); and the engine loop
tested for neither, so **no** backend ever reflowed. Now: `SETU_CONFIGURE` maps to `WIN_EV_RESIZE`,
`win_resize_apply` adopts the size and refits the present buffer, and the loop reflows
`term_resize` -> `fb_resize` and repaints.

Verified in QEMU (`agnos/scripts/harness/puka-resize-test.py`): pre-fix the event never arrives at
the grid; post-fix **80x24 -> 256x126 cells (2048x2016 px)** on a 2048x2048 framebuffer — the screen
minus the 32 px titlebar. The oracle is the new grid size, not the event: "resized" alone only
proves a message arrived, and the defect was that the grid did not follow.

### Fixed — the present passed the FRAMEBUFFER width as the SURFACE stride

`puka_ui_present(dst, fb_width(), fb_height(), fb_width())` wrote into the backend's buffer using
the grid's pixel width as the row pitch. Those are equal at open by construction and stayed equal
only because nothing ever resized. They diverge once the grid clamps at `GRID_MAX_COLS` (480 cols =
3840 px), and a frame written at one pitch and read at another SKEWS diagonally — which reads as a
renderer fault, not a stride fault. Now passes the surface's own width.

### Changed — a configure is floored to WHOLE CELLS before it is adopted

The compositor asks in pixels and does not know the surface is a character grid. Adopting 1445 px
of height would leave a 5 px band no cell can address, and a surface that is not `cols*8 x rows*16`
reintroduces the stride mismatch above. `win_resize_apply` quantizes, so surface == grid extent by
construction. A degenerate ask (<= 0, or smaller than one cell) is refused rather than adopted.

⚠ The present buffer is **grow-only** with capacity tracked separately from `win_w*win_h*4`
(`fb_cap` in `render/fb.cyr` is the same idiom): `alloc` has no free, so reallocating per configure
during a drag would leak a full surface per frame, and testing growth against the *logical* size
would realloc a buffer that already fits.

⚠ **The child shell is NOT told.** `pty_set_winsize` is accepted and ignored on agnos — there is no
TIOCSWINSZ on the `#97` channel band (see the "NO WINDOW SIZE" note in `src/pty.cyr`). puka's grid
reflows, which is what the report was about; a resize-aware pty protocol is separate work.

### Changed — dhancha tag 0.9.10 -> **0.9.12**, matching the vendored bundle

`cyrius build` re-vendors `lib/dhancha.cyr` from `path = "../dhancha"`, so the bundle was 0.9.12
while the manifest still declared 0.9.10. ⛔ **The path WINS over the tag** — CI clones by tag and
would have built a different library than every local build and every staged binary. Corrected to
what was actually compiled; 0.9.12 is tagged and released.

### Changed — cyrius pin 6.5.27 -> 6.5.28

⚠ The pin selects the **stdlib snapshot**, so this also re-vendored `lib/dynlib.cyr` — whitespace
only, 6.5.28's paren-continuation reindent (2 spaces per open-paren level), the same reformat the
agnos kernel took.


## [0.6.17] - 2026-08-17 — puka publishes its terminal engine

### Added — `[lib]` + `dist/puka.cyr`, so other programs can embed a terminal

puka was the only repo in the desktop stack that published **nothing** — no `[lib]`, no `dist/`. A
program wanting a terminal inside it had to vendor puka's source and inherit its pty, platform layer
and input stack along with the parts it actually wanted.

The bundle is the **engine only**: parser, grid, unicode, terminal, and the render trio
(pixfmt/atlas/fb). 2,017 lines. The app half — main, pty, platform, input, line_discipline — is
excluded. Verified: `dist/puka.cyr` defines no `pty_*`, `win_*`, `input_from*`, `evdev_*`,
`puka_ui_*` or `setu_*` function.

⚠ Consumers must also declare `[deps.kashi]` — `fb.cyr` blits its glyphs from the system font.

⭐ **The engine was already clean; nothing had shipped it.** `terminal.cyr` has long carried the
rule — *"the caller (the PTY owner) is responsible for pushing the new size to the child; terminal.cyr
does not depend on the PTY layer"* — and no check enforced it and no bundle delivered on it.

⚠ Composes with dhancha's `CANVAS` (0.9.9): an embedder blits the engine's RGB24 buffer into a
widget-tree slot, which is exactly what puka's own `src/ui.cyr` does since 0.6.15. A terminal does
not have to decompose into per-cell widgets to live in a widget tree.

### Testing — `tests/engine_bundle.tcyr` (11 checks)

Drives the engine with no pty, no window and no app code: sizes a grid, feeds text, renders, and
asserts glyph pixels reach the RGB24 buffer an embedder blits from; runs an escape sequence; and
resizes.

⛔ **The suite includes only the engine modules — the same list as `[lib] modules` — so the boundary
is asserted by the build, not by a comment.** Mutation-tested: making `term_resize` call
`pty_set_winsize` fails to compile with `undefined function`, which is the whole guarantee.

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
