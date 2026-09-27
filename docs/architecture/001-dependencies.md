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

`dist/mabda.cyr` issues `syscall(SYS_IOCTL, …)` for its DRM path, and the agnos syscall peer has no
`SYS_IOCTL` (`lib/syscalls_x86_64_agnos.cyr` has none; the Linux peer has `SYS_IOCTL = 16`).
Declaring mabda fails the `--agnos` build with `undefined variable 'SYS_IOCTL'` before a line of
puka compiles. The stdlib's `sys_ioctl` wrapper (cyrius 6.5.36) covers the ELF and Mach-O peers
only and does not change this.

Nothing that is built needs it: the one consumer, `src/platform/gpu/gpu.cyr` (the `pgpu_*` seam), is
not included by any entry point. The two GPU probes include `lib/mabda.cyr` themselves rather than
declaring the dependency.

Declare it again when an entry point includes the GPU seam: on a current tag, consuming only
mabda's `@public` `gpu_*` API, and only once `--agnos` stays green with it declared (an agnos
`SYS_IOCTL`, or a way to keep the bundle out of the agnos build). The seam also has to be ported to
cyrius 6.6's value-form `Result` first; `gpu.cyr` still reads one through `payload()`, which cyrius
6.6.0 deleted, so it does not compile today.

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
that resolves: it overrides the setu tag dhancha declares. sadish, rupa and rekha are not declared
by puka; they arrive through dhancha at the tags dhancha declares, and puka calls none of them.

## `dist/puka.cyr` needs kashi

The `[lib]` bundle is the engine only (parser, grid, unicode, terminal, pixfmt, atlas, fb), with
no PTY, window or app code. `fb.cyr` and `atlas.cyr` read kashi's glyph tables and the bundle does
not carry them, so a program that consumes `dist/puka.cyr` must also declare `[deps.kashi]`; the
core module is enough. `dist/puka.deps` lists the stdlib leaves the bundle needs. The `[lib]`
module list follows `src/main.cyr`'s include order because `cyrius distlib` concatenates in list
order.

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
