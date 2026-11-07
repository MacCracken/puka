# 001 — Dependencies: what each one is for, and what the graph constrains

Facts about puka's dependency graph that `cyrius.cyml` cannot state by itself. The pins and the
versions they resolve to are volatile: they live in `cyrius.cyml`, `cyrius.lock` and
[`state.md`](../development/state.md), not here. The history of how each dependency arrived is in
[`CHANGELOG.md`](../../CHANGELOG.md).

## Every declared dependency is compiled into every build

`cyrius build` prepends the modules of every declared dependency, and of the dependencies *they*
declare, to every entry point on every target, whether or not that entry's include graph reaches
them. Declaring a dependency is therefore a claim about all targets, not about the file that uses
it: a bundle that does not compile for one target breaks that target for every program. DCE
(`CYRIUS_DCE=1`) removes unreachable code after compilation. It cannot rescue a bundle that does not
compile, and CI's `--agnos` and test builds run without it, so they carry every declared bundle.

## mabda is not declared

Nothing that is built needs it: the one consumer, `src/platform/gpu/gpu.cyr` (the `pgpu_*` seam), is
not included by any entry point. The two GPU probes (`programs/gpu_probe.cyr`,
`programs/gpu_win_probe.cyr`) include `lib/mabda.cyr` themselves, along with `lib/sakshi.cyr` and
`lib/mmap.cyr`, which mabda calls and puka does not declare. See the next section for where those
includes resolve.

Declaring it would cost every build. Measured at cyrius 6.6.20 with mabda 4.1.7 declared, the
`--agnos` binary goes from 1,914,944 B to 3,247,144 B.

The agnos build no longer blocks it. Through mabda 4.1.4 the bundle issued `syscall(SYS_IOCTL, …)`
directly and the agnos syscall peer has no `SYS_IOCTL`, so declaring mabda failed `--agnos` with
`undefined variable 'SYS_IOCTL'`. Since 4.1.5 those calls go through `_mabda_ioctl`, whose agnos arm
returns -ENOSYS, and that build links. ⚠ The GPU path fails closed on agnos; it does not work there.

Declare it again when an entry point includes the GPU seam, on a current tag, consuming only mabda's
`@public` `gpu_*` API. Two things to settle at that point:

- **`color_rgb` collides.** `src/terminal.cyr` and mabda both define it, with different bodies. From
  cyrius 6.6.20 every call binds to the last definition, so include order decides which one the
  terminal gets, and the build only warns (`duplicate fn`). No program includes both today.
- **Stdlib leaves.** mabda's `dist/mabda.deps` decides which ones come back into `[deps] stdlib`; at
  4.1.7 it lists `mmap` and `sakshi` beyond puka's own set.

## An undeclared `lib/` include resolves from the toolchain

When `include "lib/X.cyr"` finds no `lib/X.cyr` in the project, cycc opens
`~/.cyrius/versions/<its own version>/lib/X.cyr` instead (the fallback in cyrius
`src/frontend/lex.cyr`). A program can therefore reach a stdlib module, or a bundle the toolchain
ships, that the manifest never declares. It gets it at the toolchain's version, not at a tag puka
chose, and without a `cyrius.lock` hash. The GPU probes rely on this for mabda (4.1.7 at cyrius
6.6.20), sakshi and mmap. It is acceptable for developer probes that CI does not build; anything an
entry point includes should be declared instead.

## Toolchain floor: cyrius 6.5.8

The agnos PTY backend in `src/pty.cyr` uses the `#97` channel-band wrappers (`sys_chan_mint`,
`sys_chan_send`, `sys_chan_recv`, `sys_chan_close`) and the `CH_E_*` result codes, which landed in
cyrius 6.5.8. Below it the `--agnos` build fails with `undefined variable 'CH_E_PEERGONE'`, which
reads like a puka bug and is a toolchain floor. The pin sits far above it; the floor matters only
if the pin is ever lowered.

## What goes through dhancha, and what does not

- **Connect, fd and close** go through dhancha's client layer: `win_open` / `win_fd` / `win_close`
  in `src/platform/setu/window_setu.cyr` call `dh_client_connect` / `_fd` / `_close`, which delegate
  to setu.
- **Present and input** call setu directly: `setu_client_present` and `setu_client_poll_input`.
  dhancha's `dh_client_present` renders a widget surface and `dh_client_next_event` returns a
  `DhEvent`; puka presents a raw XRGB terminal buffer and maps HID usages to evdev itself. Moving
  these onto dhancha is a port, not a rename.
- **The window's contents** are a dhancha widget tree (`src/ui.cyr`): a `WINDOW` root with the grid
  as a canvas widget, drawn by `dh_draw_widget`.

Because puka calls setu directly it declares setu itself, and puka's `[deps.setu]` tag is the one
that resolves: it overrides the setu tag dhancha declares. kashi works the same way.

puka calls none of sadish, rupa and rekha. It declares them only to override dhancha's tags: dhancha
0.10.4 is still pinned to cyrius 6.6.6, and each of the three has a tag on puka's pin. Their moves
carry no API change, so this is about staying on the pin, not about anything puka's code needs.
sadish 0.11.3 differs from the 0.11.2 that rekha 1.0.0 was cut against only in its requires block and
version probe. Drop the three declarations once dhancha has a tag on puka's pin, and they arrive
through dhancha again.

## `dist/puka.cyr` needs kashi

The `[lib]` bundle is the engine only (parser, grid, unicode, terminal, pixfmt, atlas, fb), with
no PTY, window or app code. `fb.cyr` and `atlas.cyr` read kashi's glyph tables and the bundle does
not carry them, so a program that consumes `dist/puka.cyr` must also declare `[deps.kashi]`; the
core module is enough. `dist/puka.deps` lists the stdlib leaves the bundle needs, and the bundle's
requires block includes them. The `[lib]` module list follows `src/main.cyr`'s include order because
`cyrius distlib` concatenates in list order.

### Regenerate it with `cyrius distlib --no-default-features`

A plain `cyrius distlib` writes a false result: it deletes `dist/puka.deps` and leaves out the
requires block, so a consumer that declares only kashi fails with `undefined function 'alloc'`.

Since cyrius 6.6.18 a sidecar is what distlib's verify can prove. It compiles the leaves found so far,
every named dep's modules and the bundle, and records a leaf only when a symbol comes up undefined.
The bundles of setu 0.8.14, sadish 0.11.3 and rupa 0.1.9 open with requires blocks of their own, which
define everything the engine calls, so nothing comes up undefined and nothing is recorded.

Those deps belong to the app half, so they are `optional` under the `app` feature, which is on by
default. Every build resolves them exactly as before; the feature split left `lib/`, `cyrius.lock`
and both binaries byte-identical. `--no-default-features` leaves the verify with kashi alone, which is
the engine's real dependency set, and it proves `syscalls`, `alloc`, `string` and `io`.

A plain `--check` reads the bundle as stale, and that is expected. cyrius 6.7.18 hides the leaves the
same way (checked 2026-11-06), so this stays until cyrius changes the verify.

### CI builds a consumer, because `distlib --check` depends on the filesystem

Below cyrius 6.7.7, `--check` cannot be a gate, with or without the flag. distlib gives a symbol to
the first leaf that defines it in directory listing order, and `O_WRONLY`, which `fb_dump_ppm` uses, is
defined by both `syscalls` and `io`. So the sidecar follows the filesystem: `syscalls alloc string io`
on the dev box, `io alloc string` on a GitHub runner, and `--check` compares bytes. Both bundles
compile, because `lib/io.cyr` includes the others, so the committed bundle can carry either list.

CI instead builds `programs/engine_reach.cyr` with `--no-deps` for Linux and agnos and runs it on
Linux. It is a consumer: kashi and the bundle are all that is in scope, and it reaches every stdlib
call the engine makes. A bundle with no requires block, or one missing `io`, fails on both targets.
Without `--no-deps` the manifest's leaves are prepended and the same bundle builds, so that flag is
what makes this a gate.

cyrius 6.7.7 sorts the listing, so `cyrius distlib --check --no-default-features` can be added back at
a pin of 6.7.7 or later. It would also catch a bundle that has drifted from `src/`, which this gate
does not.

## kashi is consumed as its freestanding core

See [ADR 0004](../adr/0004-kashi-freestanding-core-over-library-face.md).

## `path` lines stay commented in a commit

When a `[deps.X]` entry carries both `path` and `tag`, `cyrius deps` uses the path whenever it
exists, and a path-resolved dependency gets no `commit` line in `cyrius.lock`. On a machine with
sibling checkouts a live `path` therefore vendors whatever the sibling holds, while the manifest
still names a tag and CI, which has no siblings, resolves that tag. The two builds diverge silently.

So every `path` line is commented out in a commit. Uncomment one to build against an unpushed
sibling and comment it back before committing. With all of them commented, `cyrius.lock` pins every
git dependency by commit and matches a clean `cyrius deps` byte for byte.

## Editing `cyrius.cyml`

cyrius parses a string array by collecting every `"…"` between `[` and the first `]`
(`_parse_toml_str_array`, `cbt/deps.cyr`), comments included. A comment inside `stdlib = [...]`
or `modules = [...]` that contains a quoted word or a `]` silently changes the list, so comments go
above an array, never inside it.
