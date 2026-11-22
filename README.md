# Multitone

Multitone is a shared way for several Digitone II firmwares to live on one unit and to share its +Drive.

In plain words: it is (1) a small boot kernel that can start a firmware kept on the +Drive instead of the one in
flash, and fall back to a safe one when that fails; and (2) one shared rule for where each firmware keeps its extra
data, how a firmware is named (brand, name, version), and how a project made with another firmware is recognised and
shown to the user.

## Status

**Design, not built.** Nothing here has run on a unit or in an emulator. The design is in
[docs/design.md](docs/design.md) (a copy of the study written in the Wishes repository, 2026-11-22). Its open
questions Q1..Q8 are still open.

## What is in this repository

| path | what |
|---|---|
| `docs/design.md` | the full design study (multi-boot, the shared data rule, the kernel as a component) |
| `spec/identity.md` | the firmware identity, the open list, and a proposed binary record (a proposal) |
| `registry/firmwares.txt` | the firmware registry: one line per firmware (proposal; first entries) |
| `site/about.html` | the one About page (shared w0ts style, teal accent) |

## How a firmware adopts it (later)

Planned, not built yet:

1. Add this repository as a git submodule (`w0ts/multitone`); Wishes first.
2. Run the build glue that places the kernel as the first code chunk of the firmware and fills its descriptor
   (number, 4-letter id, name, version, ranges, machine-type block, open list).
3. Take a registry line (number, machine-type block, +Drive and RAM ranges) by pull request.
4. Build against the per-stock-version address table (1.12 first); on a stock version without a table the kernel
   switches itself off and the firmware boots normally.

## Licence

**No licence file yet, on purpose.** Which licence the kernel gets is open question Q1 in the design (AGPL
for open firmwares only, or a permissive licence so any firmware can carry it), and the User has not decided. Until
then all rights are reserved and nothing here may be reused. A `LICENSE` is added when Q1 is answered.
