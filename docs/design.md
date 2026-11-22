# MULTITONE: multi-boot and the shared data rule, a design study (not built)

**Multitone** is the name of the protocol (the User, 2026-11-22): the shared kernel that boots a firmware kept on
the +Drive, and the shared rule for each firmware's data.

2026-11-22. The User asked for a MULTI-BOOT that other Digitone II firmwares can plug into, as a shared kernel that
can be taken out of Wishes and used elsewhere, plus a shared rule for each firmware's extra data on the +Drive: a
firmware id, where each firmware stores its data, and how a project made with another firmware is recognised and
shown to the user.

This is a design study. Nothing here is built, no build was made, nothing ran on the farm or on the DN2. Every
address is stock Digitone II OS 1.12 unless a row says otherwise. Grades as in `docs/drive-storage-research.md`:
**[V]** measured (emulator, script or unit), **[D]** read from the code or docs, **[S]** proposed or guessed.

It builds on:

- SAFE MODE: `docs/safe-mode.md`, `csrc/safemode/` (the kernel chunk, the unpatch table, the RAM mark, the boot
  record, the soft reset);
- the dual-boot work (branch `claude/dualboot`): `docs/safe-mode-stock-dsp.md` "Stage 1 design", `csrc/safemode/stock7*.c`
  (stock's section 7 on the +Drive, sent over SysEx `7D 02`, header written last);
- `docs/memory-map.md` (the +Drive map; the rule "our +Drive data lives where upstream names nothing"; the RAM above BSS);
- `csrc/include/dmove.h`, `csrc/sampler/dmove.c` (the one-time moves);
- `docs/wishes-record.md`, `csrc/wishes/` (the link + record, TLV with `WT_MUST`);
- `docs/bootstrap.md`, `docs/firmware-sections.md`, `docs/usb-upgrade-limit.md`, `docs/ideas-backlog.md` 1b/1c (the
  boot chain and the container);
- `docs/drive-storage-research.md` (the stock slot store), `csrc/sampler/lib.c`, `csrc/sampler/erase.c`.

---

## 0. The short answer

1. **A "firmware" on the unit is three things:** its MAIN OS image (section 3, with the dnfw area appended), its DSP
   program (section 7), and its data ranges (RAM above BSS and +Drive sectors). Sections 2 and 4 (bootstrap, updater)
   and section 8 (Outbox) are the unit's, not a firmware's: multi-boot never writes or replaces them.
2. **The flashed firmware is the host.** It carries the kernel. Other firmwares (guests) are stored on the +Drive in
   a small directory (entries with id, version, CRCs, header written last). Nothing is ever written to the SPI NOR
   flash, the bootstrap or the updater.
3. **Booting a guest is a staged handover through a reset** [S]: the host, at its eMMC-ready point (before its own DSP
   upload, the +Drive driver up), reads the guest from the +Drive into a reserved RAM window, writes a RAM mark, and
   does the soft reset SAFE MODE already uses. After the reset the bootstrap loads the host again; the kernel's init
   (the first CODE chunk, before `main`, caches off) sees the mark, checks the staged image, copies it over
   `0x40000400` and jumps to the guest's entry. The guest starts on peripherals the reset made fresh. Its reads of
   "its" container (section 7 for the DSP, section 8) are redirected to the staged copy.
4. **Failure falls back to the host**, and the host's own failure falls back to SAFE MODE as today. A guest that
   never reports healthy is not booted again by default.
5. **The data rule:** a public registry gives each firmware a number and a 4-letter id, its machine-type block, its
   +Drive ranges and its RAM ranges. Data is owned by a **format**, not only by a firmware, so two firmwares that
   carry the same feature share its place (the rule Wishes already follows for upstream's features).
6. **Per-item records become one shared envelope**: the 14-byte link in the stored sound, kit, pattern, song and
   project settings points to one record whose top-level fields are one per firmware number. A firmware edits its own
   section and keeps every other section byte for byte (Wishes' `tlv.c` already keeps unknown tags). The kernel's own
   section says which firmware made the project and which last saved it.
7. **On open**, a project with sections or machine types of a firmware this one does not run gets a question
   (stock's own ConfirmWindow, as SAFE MODE's): open read-only (recommended) or open and accept that saving drops
   what this firmware cannot keep.
8. **The kernel is a C library + a dnfw build step**, version-independent C with a per-stock-version site table
   (1.11 and 1.12 to start). Its pre-main part stays small (about 2 to 3 KB more than SAFE MODE's 1,792 B of code [S]);
   the normal-boot part (directory, staging, menu, receiver, envelope) about 6 to 10 KB [S]. Licence: a question for
   the User (section 3.5).

What is proven today: nothing of the handover. SAFE MODE's parts it reuses are proven in the emulator (the mark, the
soft reset, the unpatch table, the update RPC in SAFE MODE) [V]. Section 1.6 lists what is known, what is not, and
the emulator proof for each.

---

## 1. Multi-boot

### 1.1 What a firmware is on the unit

| part | where it lives today | whose | multi-boot |
|---|---|---|---|
| bootstrap (section 2) | SPI NOR, from the flashed `.syx`; runs at `0x800003fc` [D] | the unit (Elektron's) | never touched |
| updater (section 4) | SPI NOR, runs from SRAM at `0x80000400` [D] | the unit | never touched |
| Outbox firmware (section 8) | SPI NOR; read at boot by `0x400f2934` (`find_section_by_id(8)`) [D] | the unit | the guest reads the host's copy, or its own staged copy (1.4) |
| meta (section 5) | SPI NOR, a build stamp | the firmware | stored with the guest |
| **MAIN OS (section 3) + the dnfw area** | SPI NOR (packed), depacked to `0x40000400` before MAIN OS runs; the area after stock's image (`0x4030C980` on 1.12) [D] | **the firmware** | stored in the guest's directory slot |
| **DSP program (section 7)** | SPI NOR; read by the uploader through `find_section_by_id(7)`, pushed to the SHARC at every boot [D] | **the firmware** | stored in the guest's slot; the guest's read is redirected |
| **RAM above BSS** | e.g. Wishes `0x4670c000..0x47800000` (`docs/memory-map.md`) | **the firmware** | per-firmware, declared; the kernel's window reserved for all |
| **+Drive data** | stock below `0x5D8000`; Wishes `0x5D8000..0x5FFFFF`; upstream `0x600000..` | **the firmware (by format)** | per format, registered (section 2) |

The container's section table is looked up by id, and `dest` is never read (`docs/ideas-backlog.md` 1b) [D]. The
container sits in SPI NOR at offset `0x80000`, read on demand through a flash read routine (DT2's at `0x401296fe`;
DN2 1.12's address: to find, see 1.6 U3) [D]. This is the seam multi-boot uses for a guest's own sections.

### 1.2 The +Drive firmware directory

**Where.** Two parts:

- **the kernel root**, a fixed place every kernel knows: proposed `0x5FFE00..0x5FFFFF` (512 sectors), taken from
  Wishes' free `0x5FFC01..0x5FFFFF` and given to the kernel [S]. It is below `0x600000`, where upstream names nothing,
  above the move record `0x5FFC00`.
- **the image store**, slots of a fixed size. Stage 1: Wishes' free `0x5EF200..0x5FEFFF` (past the dual-boot stock
  DSP copy `0x5EE800..0x5EF000`), **5 slots of 6 MiB** (12,288 sectors each, 61,440 in all, ending at `0x5FE200`) [S].
  6 MiB holds a guest's sections in stored form (stock 1.12's sections 3, 5, 7 and 8 together are about 1.9 MB) with
  room for a guest larger than the 2 MiB USB limit, which binds only a flashed firmware
  (`docs/usb-upgrade-limit.md`). Long term: see 2.3 (a store at the device's top needs upstream's library to agree on
  a ceiling).

**Root layout** (512 B sectors, big-endian words, each sector with its own CRC-32 over the bytes before it):

| sector | what |
|---|---|
| R+0, R+1 | the root header, two copies (A/B): `MBK0`, layout version, generation, `min_reader` (the lowest kernel version that may write here), the store's first sector, slot size, slot count. The valid copy with the higher generation wins; a write goes to the other copy |
| R+2, R+3 | the multi-boot record, A/B (1.5): default slot, one-shot slot, pending slot, tries, the last booted slot, why |
| R+16 .. R+47 | the directory: 32 entries, one sector each (up to 32 slots in a later, larger store) |
| R+64 .. R+191 | the range table: what each installed firmware declares on this unit (2.3) |
| the rest | spare |

**A directory entry** (one sector, `MBFE`):

| field | |
|---|---|
| magic `MBFE`, entry version 1 | |
| slot number, state | 0 empty, 1 valid (a state "writing" is never stored: the entry is written last) |
| firmware number (u16) and 4-letter id | from the registry (2.1) |
| firmware version | major, minor, build |
| name | 16 characters, as the UI shows it |
| stock base | stock OS version (e.g. 1.12) and the CRC-32 of that stock MAIN OS |
| build id | CRC-32 of its MAIN OS + area, as SAFE MODE's build id |
| sections | for each stored section: id, form (as in the `.syx`: aPLib/LZMA/raw), offset in the slot, stored length, CRC-32 of the stored bytes, unpacked length, CRC-32 of the unpacked bytes |
| needs | the kernel ABI it was built against; whether it carries a kernel (it can report healthy, 1.5) |
| the `.syx` sha256 | the file the page sent, so the page and the unit name the same file |
| CRC-32 of the entry | |

**Header written last** (the `stock7rx.c` rule, generalised): BEGIN zeroes the slot's entry first (an old guest in
that slot is dead from here); WRITEs fill the slot in order; END reads every sector back, checks every section's CRC
and **then** writes the entry. A cut anywhere leaves no entry: the slot is empty. A slot whose entry does not check
is empty.

**How a guest gets there.** A SysEx channel of the host's normal boot, like `7D 02`: proposed `F0 00 20 3C 7D 03`
with INFO / BEGIN slot id version lengths CRCs / WRITE off data / END / FORGET slot, one 512 B sector a message [S].
It runs only in the host's normal boot, never in SAFE MODE (SAFE MODE writes nothing to the +Drive). The builder page
sends the guest's sections from the guest's `.syx`, checked by its sha256 against the page's published list. At
about 600 B on the wire per 512 B sector, a 2 MB guest is about 4,000 messages; the time per message over USB with a
reply awaited each time is not measured.

**What may be stored.** Only a firmware whose `.syx` the page has from a published list (Wishes' builds, upstream's,
stock Elektron files the User already holds; registry members' releases). The unit checks the CRCs; it does not
check a signature, so the page is the gate on what is offered. Question Q4.

### 1.3 How the kernel picks and boots one

**Who decides, and when.** The host's normal boot, at the eMMC-ready point `0x400cf0f2` (where `sm_late` runs today,
after the eMMC's set-up, before the DSP upload, the UI, MIDI and any +Drive task) [D]. In this order:

1. the host's own SAFE MODE logic first, unchanged (its record, `pending`, `safe_tries`): a host that is failing goes
   to SAFE MODE, never to a guest;
2. the multi-boot record: a **one-shot** request (set by the menu), else the **default** (set in the host's
   settings);
3. the guarding rules of 1.5 (a guest that did not become healthy last time is not booted by default again);
4. if the choice is the host (or nothing valid): the normal boot goes on, no cost but one 512 B read;
5. if the choice is a guest: stage it (1.4), write the multi-boot record (pending = that slot, tries + 1, **the
   one-shot cleared in this same write**), write the handover mark through the uncached alias, soft reset (stock's
   `0x40005608` path without its wait, as `sm_reset_safe`). The one-shot is consumed before the reset on every path,
   never after the guest runs: a guest that crashes at start (and a kernel-less guest cannot report anything) must
   not be picked again by the next power-on.

**The record belongs to one host build** (as SAFE MODE's `DN2B` record): the multi-boot record carries the host's
build id, and a host that finds another build's record treats it as empty (default = the host, no one-shot, no
pending). So flashing a new host, the recovery route of last resort, always ends a guest choice, a bad default
included; the directory and the stored guests stay.

**The user's ways to choose** [S]:

- a **BOOT** page in the host's settings: the list of valid slots (name, version, stock base), DEFAULT and BOOT ONCE;
- **a key at power-on**: held in the first ~3 s of the host's UI, as SAFE MODE's STOP entry reads stock's button
  state (`0x445a2e30`), opens the boot list (stock's own list popup or ConfirmWindow, as SAFE MODE's question). The
  panel is not read by stock before the UI exists, so the key cannot be seen at the eMMC-ready point: choosing by key
  costs one host boot to the UI, then the staged reset. Which key: question Q6 (STOP is taken; FUNC never reaches the
  UI queue as a key item, measured for SAFE MODE);
- from a computer: the page sends "boot slot N once" over the same channel, then the host resets.

**A default guest boots without the host's UI.** The decision is made before the host's DSP upload, so the host's
SHARC program is never sent on that boot; the cost is the host's start-up to the eMMC-ready point plus the staging
read (1.6 U5).

**Never touched:** the SPI NOR flash, the bootstrap, the updater, the Early Start-up Menu. Flashing a new host
replaces only the host; the guests stay in their slots.

**The OS update runs in the host only** [S]. Transfer's update RPC (`OsUpgradeStart / Write / End`) is the one path
left that writes the SPI NOR flash, and in a guest it would run that guest's MAIN OS against the unit's updater and
bootstrap (U7: a 1.11 MAIN OS over 1.12's sections 2 and 4 is not measured). So on a guest boot the kernel makes
`OsUpgradeStart` answer "refused" (one more site in the guest's patch list, checked against stock's bytes as the
others); the user flashes from the host (or SAFE MODE, or the Early Start-up Menu). A guest without a kernel gets
the override from the host's kernel at step 7 of 1.4. A guest whose site is not stock's is refused (1.5), so no guest
runs with an update path the kernel has not closed.

### 1.4 The handover: loading a whole OS from a running OS

This is the hard part, and the reason for the design: the +Drive driver exists only after stock's init, so the
kernel's pre-main code cannot read the +Drive, and a jump into a new OS from a running one would leave every
peripheral (eSDHC and its eDMA, timers, interrupt controller, USB, UARTs, the SHARC's SPI, caches) as the old OS left
it.

**The staged handover through a reset** [S]:

| step | where | what |
|---|---|---|
| 1 | host, eMMC-ready point, normal boot | read the guest's stored sections from its slot (64 KB driver calls, as `stock7load.c`) |
| 2 | host | depack section 3 (MAIN OS + area) into the **staging window** with stock's own aPLib (`0x40136820`, in the host's RAM) or the host's LZMA decoder, as the slot's form says; keep sections 7, 8 and 5 in stored form beside it; check every CRC (stored and unpacked) against the entry |
| 3 | host | the handover mark at the kernel's RAM: `MBHO`, the slot, the build id, the staged image's place, length and CRC, the staged sections table (id, place, length), a CRC of the mark; the multi-boot record written; soft reset |
| 4 | bootstrap | loads the **host** as at every boot (it reads only the flash) |
| 5 | host's startup, then the loader (`csrc/runtime/loader.S`) | the kernel is the first CODE chunk; its init runs before `main`, before the BSS clear, caches off |
| 6 | kernel init | reads and wipes the mark (a mark lasts one boot, as SAFE MODE's). A valid MBHO mark of this host build whose staged image's CRC checks: go on; anything else: a normal host boot |
| 7 | kernel init | copies the staged image over `0x40000400` (the host's MAIN OS, including the loader's cave and the call it returns to: this path never returns), patches the guest's site(s) (below), sets the stack and the boot-flags word as the bootstrap passes them, and jumps through the vector at `0x40000400` as the bootstrap does (`moveal 0x40000400,%a0 / jsr %a0@`, read on 1.10E) [D] |
| 8 | the guest's own startup | its data init, BSS clear, its own loader and area (its own kernel, if it has one, sees the MBHO mark the host's kernel left for it) |

Why this order:

- **Peripherals:** the reset (RCR SOFTRST) restarts the ColdFire and its on-chip peripherals, and the bootstrap runs
  its usual set-up. The guest starts as after a Transfer flash's reset. Only the copy and the jump run between the
  bootstrap and the guest, both with caches off, interrupts never enabled, no driver used.
- **External chips:** the SHARC, the eMMC card, the codec, the panel's controller and the Outbox are not reset by
  SOFTRST. Stock already copes: every soft reset (after a Transfer flash, SAFE MODE's own reset) restarts a ColdFire
  whose SHARC holds a running program, and stock's set-up re-initialises the card. How the uploader brings the SHARC
  back to its boot state (a reset line, a command) is to be read (U6).
- **The +Drive is read only by the host's normal boot**, with stock's driver, in the place where `normal.c` and the
  DSP copy already read it at every boot [V in the emulator, unit: the normal boot].

**The staging window.** The staged image must sit where neither the bootstrap, the host's startup up to the loader,
nor the guest's own startup and loader write before the guest is done with it:

- section 3 + area: needed only until step 7's copy;
- section 7 (and 8): needed until the guest's uploader (and Outbox loader) read them, after the guest's BSS clear and
  area load.

So the window must be outside every firmware's BSS and RAM claims: **a RAM range reserved by the registry for the
kernel, which no firmware may use** [S]. Candidate: `0x47800000..0x47FFFFFF` (8 MiB, the top of the 128 MiB DDR).
On 1.12 Wishes' highest claim ends at `0x47800000` (the CPU profiler from `0x47700000`) [D]. Stock 1.12 writes above
its BSS end `0x466b84d0` too (its +Drive bounce buffer at `0x466f9300`, 64 KiB, measured) [V]; that the window is free
of every stock write is for `rammap` to show, on 1.11 and 1.12 (P2). 8 MiB holds a depacked MAIN OS + area (stock
3.19 MB; Wishes' area: to measure) and the stored sections 7 and 8 (under 0.8 MB). The kernel's mark and code stay at
`0x476F8000..0x47700000` (SAFE MODE's `SM_RAM`), which every member must also leave alone.

**Redirecting the guest's own sections.** The guest's uploader reads section 7 from the flash through
`find_section_by_id` and the flash read routine (stock), so on a guest boot it would read the **host's** DSP program.
A MAIN OS with another firmware's DSP program is the mismatched pair: no brick (the flash is untouched), but wrong or
loud audio is possible. Two ways out, both a single override over stock bytes the kernel checks first (as SAFE MODE's
O1..O4):

- **R1 (preferred)** [S]: the flash read primitive behind `find_section_by_id` / `section_data_address`
  (`0x4013459a` / `0x4013458a` on 1.11) answers from the staged container table for the guest's section ids, and from
  the flash otherwise. One site per stock version. The guest's own code (stock's uploader, Wishes' `dn_depack7`, the
  Outbox loader) then runs unchanged and reads its own sections.
- **R2**: the uploader's depack call (`0x400cf73a` on 1.12, SAFE MODE's O1 site) answers from the staged section 7.
  Narrower, but it leaves section 8 and any other section read on the host's bytes, and it is a different site for a
  guest that already moved its own depack (Wishes' `dn_depack7`).

For a guest without a kernel (stock Elektron, an upstream build), the host's kernel patches the guest's site in RAM
after the copy (step 7), by the guest's stock version, after checking the site holds stock's bytes; if it does not,
the guest is refused (the host boots normally and says why). For a guest with a kernel, the guest's own kernel does
it from the MBHO mark: each firmware patches only code it was built from.

**Caches.** The copy runs with the caches off (the kernel's init runs before `0x4000055a` turns them on [D]); the
reset invalidated whatever the old run cached. No cache maintenance is needed, as for the loader today.

**What is lost against a direct boot:** the guest's boot is slower by the host's start-up to the eMMC-ready point,
the staging read and depack, and a second bootstrap pass (1.6 U5). Its UI shows the boot screen twice.

### 1.5 Failure and fallback

The rule: **a guest failure falls back to the host; a host failure falls back to SAFE MODE (as today); SAFE MODE's
failure falls back to the Early Start-up Menu (as today).**

The multi-boot record (root R+2/R+3): `MBR1`, default slot, one-shot slot, pending slot, tries, last booted slot, why.

| case | what happens |
|---|---|
| staging fails (read, CRC, form) | no reset: the host boots normally and shows "BOOT <NAME> FAILED: <why>" at its UI; the default is kept, the one-shot cleared |
| the mark or the staged image is lost or wrong after the reset | the kernel boots the host (fail open, as SAFE MODE) |
| the guest site is not stock's | the host boots, the slot is marked refused for this host build |
| a guest **with** a kernel | it reports healthy after 30 s of UI (SAFE MODE's `SM_HEALTHY`, 3,630 ticks) by clearing `pending` in the multi-boot record. A next host boot that finds `pending` still set for that slot: tries + 1; at 2, the default goes back to the host for this boot and the host's UI asks "LAST START OF <NAME> FAILED. BOOT IT AGAIN? Y/N" |
| a guest **without** a kernel (stock, an upstream build) | it cannot report healthy, so a power-on cannot tell "it crashed" from "the user switched off". Such a guest is **BOOT ONCE only, never the default** [S]: every power-on returns to the host. Q3 |
| the host itself fails | its SAFE MODE as today (`safe_tries`); SAFE MODE never boots a guest, never writes the +Drive |

A guest booted by the host does not run its own SAFE MODE logic on that boot (its kernel sees the MBHO mark): the
host is the recovery path, because the host is what is in the flash and what Transfer replaces.

### 1.6 Known, unknown, and how each is proven in the emulator first

| | state | how to prove it |
|---|---|---|
| K1 the kernel's init runs before `main`, BSS clear and caches, as the first CODE chunk | [V] SAFE MODE, `safeboot` | as is |
| K2 a RAM mark survives our soft reset | [V] emulator (`safeboot` part t); on the unit: a restart after a flash kept the RAM above BSS, a 20-minute power-off did not (`docs/memory-map.md`, 2026-11-18) [V]; our soft reset's own case is SAFE MODE's open unit check | SAFE MODE's first unit test settles it for multi-boot too |
| K3 the host reads the +Drive at the eMMC-ready point | [V] emulator, every boot (`normal.c`, the DSP copy) | as is |
| K4 the container is read by id, `dest` unused, offset `0x80000` in SPI NOR | [D] 1.11 (`docs/ideas-backlog.md`) | read again on 1.12 (U3) |
| U1 the bootstrap's hand-off to MAIN OS on 1.11 and 1.12: the entry vector, the stack, the boot-flags word it passes (bit 0 EMPTY RESET, bit 5 maintenance), what else is in registers | read on 1.10E only [D] | read the 1.12 bootstrap; then **P1** below |
| U2 that a copy + jump from the kernel's init gives the guest the same machine as a direct boot | unknown | **P1, the equality test:** digikit boots guest G from reset directly (its flash HLE); then host H + G staged + MBHO + soft reset. At G's `main` (0x40000568 on 1.12) compare RAM (minus the declared window) and every MMIO write so far; then the UI composes and the SHARC receives `0x03` + G's own stream byte for byte. G = stock 1.12, then a Wishes build, then stock 1.11 |
| U3 the flash read primitive on 1.12 and 1.11 and its callers (only `find_section_by_id` 7 and 8 on 1.11?) | [D] 1.11 two callers; 1.12 not read | a static callers scan on both, then **P3:** with R1, the emulator logs every flash read of a guest boot: none outside the bootstrap's reads lands on the host's container |
| U4 the staging window is free of every stock write, both versions, and of every member's claims | partly: 1.12 bounce buffer known [V] | **P2:** `rammap` over a from-reset run of stock 1.11 and 1.12 and the UI fuzz, window `0x47800000..` |
| U5 time: the eMMC driver's read rate, the depack, the second bootstrap pass | not measured (`docs/lzma7.md` lists the eMMC read time as open too) | instruction counts in the emulator; times only on the unit |
| U6 the SHARC after a soft reset: how stock's uploader brings it back to boot | not read | read the uploader around `0x400cf4b0`; the emulator's SHARC model shows the sequence; on the unit SAFE MODE already resets with a running SHARC |
| U7 a 1.11 MAIN OS started after the 1.12 bootstrap and updater set the machine up (the flash holds 1.12's sections 2 and 4) | unknown | P1 with G = stock 1.11; and compare 1.11 and 1.12 section 8 (if identical, the Outbox question is moot) |
| U8 stock data across stock versions: a 1.11 guest reading projects 1.12 wrote, a 1.12 host reading what a 1.11 guest wrote. 1.10E -> 1.11 re-wrote every occupied project slot (`projectStorage_v14_t`, `docs/os-versions.md`) [V] | unknown for 1.11 <-> 1.12 | list 1.12's storage struct names against 1.11's (the method in `docs/os-versions.md`); if any differ, a guest on another stock base is refused by default (2.6) |
| U9 the guest's own boot screen and the eMMC card's state after the host touched it | unknown | P1 covers the card in the emulator's eSDHC model; the unit for the real card |
| U10 the kernel's site checks on a guest image that is not stock under its area (a modded MAIN OS) | design | **P4:** hostile and modded guests: a site not stock -> refused, host boots, no write |
| U11 no SPI NOR write a guest can reach other than the update RPC: MAIN OS 1.11 and 1.12 hold no flash erase or program routine outside the update path (the bootstrap's own `BOOTSTRAP UPGRADE` is in section 2, which a guest never carries) | not read: assumed | a static scan of both MAIN OS images for the flash controller's registers and every caller of the flash write/erase routines; then **P8:** the emulator logs every SPI NOR write of a guest boot, the UI fuzz and a refused `OsUpgradeStart`: none |

**Proof list before any unit test** [S]: P1 (equality, three guests), P2 (window), P3 (no stray flash read), P4
(hostile marks, images, sites: fail open), P5 (a guest with a kernel reports healthy; one that hangs: pending, tries,
back to the host), P6 (every +Drive write of a guest boot inside the guest's registered ranges or stock's store, the
trace as `erase`'s), P7 (the host's SAFE MODE path unchanged: `safeboot` passes byte for byte), P8 (no SPI NOR write
on a guest boot, `OsUpgradeStart` refused there; U11), P9 (a one-shot guest that crashes before its UI: the next
power-on boots the host; a host flashed over a record whose default is a guest: the host boots). All brick-risk steps
in `ship_gate.py` for the host build that carries the kernel.

**The risk the handover cannot remove:** the guest runs on the unit. A guest is as safe as its own gate. The host's
kernel guarantees only that the flash, bootstrap and updater are untouched and that the host comes back.

### 1.7 The target: a thin host, a two-stage boot (the User, 2026-11-22)

**The end state:** the flashed firmware is **stock 1.12 + the multi-boot kernel** and nothing else (the thin host).
Wishes, upstream builds and stock 1.11 are all guests on the +Drive. The boot has two stages:

| stage | what | where it comes from | changes |
|---|---|---|---|
| 1 | the bootstrap loads the thin host; the kernel decides what boots and keeps the fallbacks | SPI NOR flash | rarely: only when the kernel itself changes (a reflash) |
| 2 | the host stages the chosen firmware and hands over through the soft reset (1.4) | the +Drive | every release, by a copy (1.2), never a flash write |

**The rule: recovery code lives in flash and is frozen; everything that changes lives on the +Drive.** The part that
decides what boots and that guarantees the host comes back stays in the flash, because the bootstrap reads only the
flash: a decision loaded from the +Drive could be the thing that fails. Each guest still carries its own kernel for
its own side (the MBHO mark, the envelope, healthy reporting, its patches), and those versions change freely. A
chained "stage-1 kernel" loaded from the +Drive to run the menu is not proposed: it moves nothing that matters off
the flash.

Why not a minimal kernel written from scratch as the host: the host must read the +Drive (stock's eMMC set-up and
driver), take Transfer's update over USB (stock's USB stack and RPC), show the boot list (stock's UI) and recover by
itself. Writing those again is the riskiest code there is to write; stock 1.12 already carries all of it.

What the thin host gives:

- **the fallback is a working synth:** a missing or damaged guest, an empty +Drive, a guest that never becomes
  healthy: the user has a stock Digitone II with its sound. SAFE MODE shrinks to "boot the host, not the guest";
- **releases stop writing the flash:** a Wishes release is a +Drive copy (header written last, a cut copy is an empty
  slot); the flash, with its only brick risk, is written once for the host and again only for a new kernel;
- **guests leave the 2 MiB USB limit** (`docs/usb-upgrade-limit.md`): a guest is bounded by its slot (6 MiB) and the
  staging window (8 MiB), not by the update buffer (Wishes had 57,356 B left in v0.69.2);
- **the flashed code is small to gate:** stock bytes + the kernel (estimate 3 to 10 KB [S]).

What it costs, and the conditions for keeping it as the target:

- **every Wishes boot is two boots** (stock to its eMMC-ready point, staging, the soft reset, the bootstrap again):
  the time is not measured (U5). If it is too long for the User, Wishes-as-host (1.1 to 1.6) stays the design;
- **it stands on the handover:** P1 (equality) must pass with Wishes as the guest; if it cannot, same fallback;
- **U12, the working project at the decision point:** if stock reads the working project (`0x400f21e4`) before the
  eMMC-ready point, the thin host loads a Wishes working project with stock's code before the guest is chosen
  (Wishes' machine types clamped, Q7). To read in stage 1: the order of the working-project read and `0x400cf0f2`
  on 1.12, and whether the host can decide before it;
- one more step for the user: flash the thin host once; Wishes then updates from the page as a +Drive copy.

**Speed (the User, 2026-11-22): "as fast as possible ... 5 sec would be really the max acceptable".** The boot the
User wants: power on; **if no key is pressed** to pick another firmware, mount the +Drive, load the firmware picked
last, hot-boot into it. So:

- **the default is the last pick**: picking a firmware (BOOT page, key list, the page) also makes it the default
  (for a guest with a kernel; a kernel-less guest stays BOOT ONCE, 1.5). No menu, no UI, no question on the default
  path: the decision is one 512 B record read;
- **the budget: at most 5 s more than a direct boot of the same firmware flashed**, measured from power-on to its
  main screen; the aim is as little as possible;
- **the key window:** the key that opens the list must be seen before the staging starts, or the default path pays
  for a host UI it does not need. Stock reads the panel only once its UI runs (1.3), so this is a new unknown,
  **U13: can the kernel see a held key at the eMMC-ready point** (the panel controller's own state, a read the
  kernel makes itself, or a short wait there)? If not, the key costs a host boot to the UI only when it is held: the
  host checks the held key in its first UI pass (as SAFE MODE's STOP entry), and the default path never waits for it;
- **the hot boot** is the handover of 1.4 (staging, the soft reset, the copy and jump). A jump without the reset
  would save the second bootstrap pass but hands the guest peripherals the host left running; it is considered only
  if P1 shows the two give the guest the same machine, and only if the bootstrap pass is a large share of the time.

Where the time goes, and the levers (none measured, U5):

| step | lever |
|---|---|
| the bootstrap unpacks the host | none: the bootstrap is never touched |
| stock 1.12's start-up to the eMMC-ready point | **the host's own DSP upload moved after the decision** (the dual-boot work already moves it: O5/O6, `sm_late7`), so a guest boot never sends the host's SHARC program; anything else stock does before that point that a guest boot does not need |
| reading the guest from the +Drive | read only sections 3 and 7; section 8 only if it differs from the host's (compare 1.11 and 1.12's) |
| unpacking | store the guest unpacked if the eMMC read (U5) is faster than the depack |
| the soft reset and the second bootstrap pass | none, unless the jump without reset above |
| the guest's own start-up | as a direct boot |

**The measurement first, before any build:** instruction counts per step of stock 1.12 from reset in the emulator
(where the start-up's time goes before `0x400cf0f2`, the depack's cost), and a stopwatch on the DN2 from power-on to
the main screen for stock 1.12 and for a Wishes build (the bootstrap pass's real time, the baseline).

**The choice the time makes (the User, 2026-11-22: "depending on the speed we might want to flash our own firmware
still... and have 1.12 on disk"):**

| measured extra time of a guest boot | flashed (the host) | on the +Drive (guests) |
|---|---|---|
| within the budget | the thin host: stock 1.12 + the kernel | Wishes (the default), upstream builds, stock 1.11 |
| over the budget | **Wishes + the kernel** (1.1 to 1.6) | **stock 1.12** (stage 3, BOOT ONCE), others |

Both are the same kernel and the same handover; only what is flashed differs, so nothing built for one is lost for
the other. In the second, Wishes boots at today's speed, and the slow boot is paid only when stock or another
firmware is picked.

---

## 2. The shared data rule

### 2.1 The firmware id registry

**A plain text file in a public repository** [S], the kernel's own repository (2.7, 3.4), one line per firmware,
assigned first-come by a pull request, never reused:

    # number  id    name               home (public repo)      machine types  stock base  +Drive ranges (sectors)            RAM ranges
    0         STOK  Elektron stock     -                       0..17 (stock)  any         below 0x5D8000                     BSS
    1         MBK1  multi-boot kernel  (this repo)             -              -           0x5FFE00..0x5FFFFF, image store    0x476F8000..0x47700000, 0x47800000..0x47FFFFFF
    2         DPAT  dn2mk2 (upstream)  De-Pattern/dn2mk2-firmware  18..49     1.11        (its own list, below)              (its list)
    3         WISH  Wishes             (Wishes' repo)          50..62         1.12        0x5D8000..0x5FFDFF less the kernel's  0x4670c000..0x47800000

(The numbers, the machine-type blocks and the ranges in this sketch are proposals, not agreed; the stock machine-type
count is to be read.)

- **number** (u16, 1..0x3FFF): what the records carry (2.4). 0 means stock, never a firmware's section.
- **id** (4 ASCII letters): what the directory and logs show.
- **machine types**: the block of machine-type numbers the firmware may give its engines. The type number is stored
  in stock's own sound bytes, so two firmwares that give one number to two engines make a sound play the wrong engine
  in the other firmware. It already happened: upstream v0.59 took 18..20 for FM+, which older Wishes builds had used
  for PLAITS, TB303 and SOPHIE, so those sounds load as FM+ (the User's decision 2026-11-18: Wishes moved to 50..55,
  no conversion). The limits: stock's LOAD bound is a `moveq` (up to 127, `docs/machine-types.md`), the DSP's machine
  lookup has 63 words (types up to 62).
- **+Drive ranges and RAM ranges**: section 2.3.
- **stock base**: the stock OS the firmware's MAIN OS is built on.

Who keeps it: the kernel repository's maintainer merges a line after checking it overlaps no other line; a member's
gate refuses a build whose declared ranges or types differ from its registry line (a static step, like
`safeboot-table`). Q2 asks who maintains it and where it lives.

### 2.2 Ranges are owned by formats

Wishes already shares data with upstream on purpose: "an upstream feature we take keeps upstream's place, so its data
is the same on both firmwares" (`docs/memory-map.md`). Its tunings, PERFORM, bank lists, the pool image and the
library sit at upstream's places, in upstream's formats. So a range belongs to a **format** (a magic, a version, an
owner who defines it), and any firmware that implements that format exactly may read and write it.

The registry therefore carries a second table, **formats**: format id (the magic, e.g. `DN2Q`, `DN2L`, `DN2W`),
owner (firmware number), its ranges, its document. A firmware's line lists the formats it implements. The rules:

1. a firmware writes only stock's own store (through stock's own code) and the ranges of formats it implements;
2. a format's owner alone changes its layout; a version bump is the owner's, and a reader that does not know the new
   version leaves the range alone (no write);
3. a firmware that does not implement a format never reads it as anything else and never erases it, except through
   a whole "erase all mod data" that the kernel offers and names as such;
4. a range moves only with a one-time copy on the unit by its owner (`dmove.c`'s pattern: copy whole records, set a
   bit in a move record, never write the old place);
5. **one sector, one owner**: no sector holds bits of two owners. Today's settings sector `0x6FF000` (`DN2S`,
   upstream's place) carries Wishes' bits 1..3 beside DIRECT JUMP's bit 0 (`docs/memory-map.md`): under this rule
   Wishes' bits move to a settings sector of its own.

### 2.3 Where each firmware's ranges are

Today (from `docs/memory-map.md`, v0.69.2) [D]:

| range | owner (format) |
|---|---|
| below `0x458000` | stock's slot store (projects, presets, kits, the working project, MRAM) |
| `0x458000..0x5D7FFF` | stock's factory buffer |
| `0x5D8000..0x5FFFFF` | Wishes (`DN2W`, `DN2I`, `DN2M`, `DN7C`, `DN2B`, `DN2V`, and `DN7S` on `claude/dualboot`) |
| `0x600000..0x6FFFFF` | upstream (`DN2Q` pool image, `DN2J`/`DN2K` lists, `DN2T` tunings, PERFORM, bank lists, `DN2X`, `DN2S`, folders) |
| `0x700000..` the device's end | upstream's library (`DN2L`), first-fit from its data start up to the device's last sector (`lib.c`) |

The proposal [S]:

- **the kernel's root and stage-1 image store** come out of Wishes' range (1.2), so no other owner is asked to give
  anything for stage 1;
- **new firmwares** need room, and only the library's far end is large and unused on most units. The library already
  allocates first-fit from the bottom; a **ceiling** it respects (e.g. the device's size less 1 GiB, rounded down to
  a 1 MiB boundary; the device is 48,340,992 sectors after the factory reconfigure, 61,120,512 before
  (`docs/drive-storage-research.md`), so the ceiling is computed, never a constant) would free the top for the
  kernel's image store and for new members' ranges, assigned from the top down. This is upstream's format; it needs
  upstream's agreement (Q5). A unit whose library already holds samples above the ceiling keeps them; the kernel
  never takes a sector a `DN2L` extent names;
- every member keeps `_Static_assert`s against its neighbours in its C, as Wishes does today, and the gate step P6
  checks every write.

**Warning, measured on the DT2 [D]:** the Digitakt II keeps a real filesystem (`ekFS`) from sector `0x5D8000` to the
device's end (`docs/drive-storage-research.md`). If Elektron ever brings it to the DN2, every mod range would be
inside it. No registry can prevent that; a stock release that does it is a reason to stop every member before its
next build.

### 2.4 Per-item records: one shared envelope

**Today [D]:** each stored item (sound 359 B, kit, pattern, song, project settings) has exactly one free place for a
14-byte link (`docs/wishes-record.md`: the sound's lane ids 4..28, kit bytes 10,416..10,429, a pattern's last unused
lock record, song 2,900..2,913, project settings 544..557). Stock's saver writes the sound's lane as zeros; Wishes'
save hook writes its link over them. The link names a content-addressed record (two xxHash32), filed in one sector
cell. Unknown tags are kept byte for byte; tag bit 15 is "must understand" (`WT_MUST`: a record with such a field this
build does not know is read-only here); bit 14 is "tied to the stock part".

**The problem with many firmwares:** there is one free place per item, so a second firmware with its own link would
overwrite the first one's at its next save. So the link and the record become **shared**, owned by the kernel:

- **The link** stays 14 bytes, `'W'`, link version **2**, the hash, length, check. Version 1 links (Wishes' today)
  stay valid: they are read as "a record whose fields are all Wishes' (number 3)".
- **The record** (envelope) is TLV as today, sorted by tag; at the top level **one field per firmware number** (tag =
  the number, bit 15 MUST, bit 14 tied to the stock part), its value that firmware's own TLV. Number 1 (the kernel)
  holds the shared fields (2.5).
- **A firmware edits only its own section** and keeps every other section byte for byte; the hash then changes, the
  new record is filed, the link rewritten. This is `tlv.c`'s rule one level up.
- **The cells** (`DN2W`, 92,160 at `0x5D8000`) become the kernel's format, shared by all members (they are content
  addressed, so a record written by any member is read by any other). A record larger than a cell (488 B today)
  needs a chain or a larger cell: to design when a member needs it.
- **A firmware without the kernel library** (stock, an upstream build without it) still zeroes the lane at its save:
  the item's link is lost (its cell stays, orphaned, and is found again if a copy with the link comes back). The
  kernel cannot prevent that; the UI warns about it (2.6). For upstream to keep links, its save and load need the same
  hooks on 1.11 (sites to find).
- **Per-slot data** (upstream's per-project lists, tunings, PERFORM, bank lists, keyed by the project's slot number,
  not by a link) does not follow a project that is copied or moved, and stays with the slot when another firmware
  saves a different project there. The rule [S]: each per-slot record carries the project's identity (a random 64-bit
  project id kept in the kernel's section of the project's envelope), and a reader that finds a different id treats
  the per-slot record as stale.

### 2.5 How a project records which firmware made it

The kernel's section (number 1) of the **project settings' envelope** (`WI_PROJECT`, the link at stored 544..557):

| field | |
|---|---|
| `PID` | the project id (2.4), set at the first save by a member |
| `MADE` | firmware number, version, the first save |
| `LAST` | firmware number, version, the last save |
| `USES` | the firmware numbers whose sections are anywhere in the project (sounds, kits, patterns, songs) and whose machine types its sounds use, each with "MUST" if any of its fields is MUST |

`USES` is rebuilt at each project save by the saving member (it walks what the project's save writes, which the
member's hooks see already). A project saved by stock or by a non-member has no envelope: it reads as "stock or
unknown".

### 2.6 What the user sees when opening a project from another firmware

At the project LOAD hook (`0x400df58c` on 1.12, where Wishes already hooks), the kernel compares `USES` (and, for a
project without an envelope, the machine types its sounds carry against this firmware's block and the registry's
blocks) with what this firmware runs:

| the project | the user sees | what this firmware does |
|---|---|---|
| made and saved by this firmware, stock, or only by firmwares whose sections are not MUST and none of whose machine types it uses | nothing | normal |
| sections of a firmware this one does not run, none MUST | a short notice: "HAS DATA FROM <NAME>: KEPT, NOT USED" | keeps them byte for byte; normal saves |
| MUST sections, or sounds with another firmware's machine types | stock's ConfirmWindow: "MADE WITH <NAME> V<x>" / "OPEN READ-ONLY? Y/N" | YES (recommended): the project's SAVE is refused with a message ("READ-ONLY: SAVE AS A NEW PROJECT"), the working copy still works; NO: opened normally, with a second line "SAVING LOSES <NAME>'S SOUNDS" |
| a project last saved by a firmware on another stock base (U8) | the same question | read-only unless U8 shows the storage formats are the same |

Why read-only: stock's LOAD clamps an unknown machine type (the bound Wishes raises, `WR_LOAD_BOUND`), so a sound of
another firmware's engine is changed when it is loaded, and the change is written at the next save. Records are kept
(2.4), but the sound's own stock bytes are not. The name shown comes from the directory on this unit (installed
firmwares) or from the registry table compiled into the kernel; else "FIRMWARE <number>".

**The working project.** Stock reads the working project (slot 128 / MRAM) at every boot, so switching firmware opens
the last firmware's working state at once, before any question. Three ways, to choose (Q7):
(a) the boot of another firmware starts **empty** and does not write over the working state (SAFE MODE's mechanism:
boot flag bit 0 and O4); the user opens projects explicitly and gets the question above;
(b) **a working slot per firmware**: the kernel redirects stock's slot-128 sector per firmware number to a range of
that firmware's (a project image is about 12.9 MB, `docs/old-project-load.md`), one site per stock version;
(c) the host asks "SAVE ALL BEFORE SWITCHING?" (stock's own question) and the guest loads the working state with the
check above. (a) is the smallest and reuses proven parts.

### 2.7 Erase

ERASE MOD DATA (`csrc/sampler/erase.c`) erases the ranges of the formats the running firmware owns. The kernel adds
"ERASE ALL MOD DATA" (every registered range on this unit, from the root's range table, not the root itself unless
asked) and "REMOVE FIRMWARE <slot>" (the entry first, then the slot's sectors).

---

## 3. The shared kernel as a component

### 3.1 What it is

Two halves, as SAFE MODE today:

| half | where it runs | what |
|---|---|---|
| **pre-main** (the kernel chunk) | the first CODE chunk of the area, at `SM_LOAD` `0x476F8100`, from the loader before `main` | the RAM mark; SAFE MODE (the unpatch table, O1..O4); the handover (check, copy, site patches, jump); the guest-side handling of an MBHO mark |
| **normal** (in the firmware's normal code) | the eMMC-ready point, the UI loop, the SysEx router, the item LOAD/SAVE hooks | the boot record and SAFE MODE's decision (`normal.c`); the directory and the root; staging; the BOOT page and the power-on key; the guest receiver (`7D 03`); healthy; the envelope (TLV, hash, cells); the provenance check and its question |

### 3.2 What a firmware provides (the interface) [S]

A descriptor in the firmware's area, found by its magic, filled by the build:

    struct mbk_fw {                         /* 'MBFW', in the area */
        u32 magic, abi;                     /* 'MBFW', the kernel ABI it was built for */
        u16 number; char id[4]; char name[16];
        u32 version;                        /* major << 16 | minor << 8 | build */
        u32 stock_os, stock_crc;            /* its stock base: 0x0112 and the CRC-32 of stock 1.12's MAIN OS */
        u32 build_id;                       /* CRC-32 of its MAIN OS + area (SAFE MODE's) */
        const struct mbk_range *drive; u32 n_drive;   /* sector, count, format id: its own and the formats it implements */
        const struct mbk_range *ram;   u32 n_ram;     /* its RAM above BSS */
        u8 machine_lo, machine_hi;          /* its machine-type block */
        const u8 *unpatch; u32 unpatch_len, unpatch_crc;   /* SAFE MODE's table (dnfw.mods.safemode) */
        const struct mbk_known *formats; u32 n_formats;    /* the envelope sections and MUST tags it understands */
    };

and four calls the kernel makes into it (all optional): `on_healthy()`, `on_guest_mark(mark)` (a guest with its own
DSP path, e.g. Wishes' `dn_depack7`), `section_known(number, tag)`, `draw_name(number, buf)`.

The kernel provides: `mbk_boot_once(slot)`, `mbk_set_default(slot)`, `mbk_dir(i, entry)`, `mbk_env_get/put(item,
number, tag, ...)` (Wishes' `wish_get/put` one level down), `mbk_provenance(project)`, `mbk_erase_all()`.

**Stock sites** are not in the firmware: a **site table per stock version**, generated from the stock files and
checked byte for byte (`port_os112`-style tooling): the eMMC-ready point, the DSP upload sites (O1, O5, O6), the flash
read primitive (R1), isDirty (O2), the empty-project store (O4), the boot-flags word, stock's button state,
ConfirmWindow and its class, the block driver (read `0x4012C87A`, write, TRIM, ready, sector count), the soft reset,
the item LOAD/SAVE sites. A stock version not in the table: the kernel turns itself off (a normal boot, no BOOT page),
never guesses.

### 3.3 Size budget [S]

| | today | multi-boot adds (estimate, not built) |
|---|---|---|
| pre-main kernel | 1,792 B code + the unpatch table (9,142 B of runs on up-v067) | handover: mark check, CRC over the staged image (the CRC exists), copy, site patches from a small table, the jump: about 0.6 to 1.2 KB; the guest-side MBHO handling about 0.3 KB |
| normal half | `normal.c` + `question.c`; the stock7 receiver and loader (dual boot) | root and directory with A/B, staging (driver loop, depack call, CRC), the BOOT page and key, the receiver `7D 03`, the provenance check and its windows: about 6 to 10 KB; the envelope reuses `tlv.c`, `store.c` |
| Wishes' room | 57,356 B in the 2 MiB USB buffer (v0.69.2) | fits; upstream's room: to ask |
| RAM | `0x476F8000..0x47700000` (the mark + kernel, about 10 KB used of 32 KB) | the staging window `0x47800000..0x47FFFFFF` (8 MiB), reserved for all |
| CPU | 0 per audio frame | 0 per audio frame; one extra 512 B read at boot (the multi-boot record) |
| +Drive | one read and one write per boot, one at healthy | the same, in the root |

### 3.4 How another firmware builds it in

- **A firmware built with dnfw** (upstream and every fork of it: Wishes, the planned second firmware in
  `docs/HANDOFF-second-firmware.md`): the area, the CODE chunks and the loader (`csrc/runtime/loader.S`) are the
  shared ABI already. Adding the kernel is a dnfw mod (`multiboot`, as `dnfw.mods.safemode` today): it compiles the
  kernel's C with the firmware's toolchain (`cbuild`), places the kernel as the **first** CODE chunk, generates the
  unpatch table from the stock file and the firmware's own MAIN OS, writes the `MBFW` descriptor from a small
  `multiboot.toml` (number, id, name, ranges, machine block), and adds the gate steps (`safeboot-table`, `safeboot`,
  and P1..P7).
- **A firmware not built with dnfw** needs: a pre-main hook that calls the kernel's init before its BSS clear, the
  kernel's RAM left alone, and the same table generator (a standalone Python tool reading two `.syx` files).
- **Repository** [S]: a public repository of its own holding `kernel/` (C: `mbk.h`, the pre-main and normal halves, the
  envelope), `sites/` (one generated table per stock version, with its check script), `tools/` (the table and
  descriptor generators), `emu/` (the proof scripts as a library on digikit), `REGISTRY.txt`, `FORMATS.txt`, and the
  documents. Wishes takes it in as a subtree or a pinned copy; the gate checks the pinned copy's hash.

### 3.5 Licence

Wishes is AGPL-3.0-or-later (`LICENSE`). The kernel's own C was written for Wishes; the loader it relies on
(`csrc/runtime/loader.S`) and the dnfw framework come from upstream under upstream's licence. Choices (Q1):

- **AGPL-3.0-or-later** for the kernel: every firmware that ships it must offer its sources to its users. It suits
  open firmwares; it rules out a closed one (the second firmware's plan says its repository "never goes public",
  `docs/HANDOFF-second-firmware.md`);
- **a permissive licence (MIT or BSD-2) or LGPL-style** for the kernel alone: any firmware may carry it. Only code
  whose authors agree can be relicensed, so the kernel must not contain upstream-derived code (the loader stays
  upstream's, under its licence, and is not part of the kernel library).

The registry and format texts: CC0 or CC-BY, so anyone can implement them.

No Elektron bytes are in the kernel library. The unpatch table holds stock bytes, made at build time from the stock
file the builder already holds; it ships inside a firmware as today.

### 3.6 Staying compatible

- **Across firmwares:** the root, the directory, the records and the envelope carry versions; a reader that finds a
  newer layout reads what it knows and writes nothing (the root's `min_reader`, the envelope's MUST bit). Fields
  grow only at their end; tag numbers are never reused (`tlv.c`'s rules).
- **Across kernel versions in one unit:** the host's kernel and a guest's kernel may differ. The MBHO mark carries
  its own version; a guest kernel that does not know it ignores the mark (it boots as if flashed, then reads its
  container from the flash: the mismatched pair). So **the host refuses a guest whose `abi` is older than the mark it
  would write** (the directory entry says the guest's ABI).
- **Across stock versions (1.11 vs 1.12):** the C is version-free; every stock address comes from the site table; the
  RAM windows are chosen above both versions' BSS ends (1.11 `0x466b74d0`, 1.12 `0x466b84d0`) [D]; U7 and U8 decide
  whether a guest on another stock base may boot at all.

---

## 4. The staged plan, the risks, the questions

### 4.1 Stages

| stage | what | for whom | ends with |
|---|---|---|---|
| **0** (now) | this study; the dual-boot stock DSP (`claude/dualboot`) waits for its own go | the User | the User's answers to Q1..Q8 |
| **1** read and measure, no firmware change | U11 first (no reachable flash write in MAIN OS), U12 (the working project vs the decision point, 1.7), U13 (a held key at the eMMC-ready point), U5 timing (1.7: emulator counts, DN2 stopwatch), U1 (the 1.12 bootstrap's hand-off), U3 (the flash read primitive on 1.12 and 1.11), U4/P2 (`rammap` of the window on both), U6, U8 (storage struct names), section 8 compared; Wishes' area size unpacked | Wishes | a note with the numbers; the window and the root agreed |
| **2** the handover in the emulator only | a test kernel (not shipped): P1 with stock 1.12 as the guest, then a Wishes build, then stock 1.11; P3, P4 | Wishes | P1 equal on three guests, or the reason it is not |
| **3** Wishes as host, stock 1.12 as the only guest | the root and directory in Wishes' range, the receiver `7D 03`, the page sending stock's `.syx`, BOOT ONCE only (stock has no kernel); this replaces the narrower SAFE-MODE-with-stock-DSP if the User prefers | the User | a cleared test build; on the unit the User boots stock 1.12 once and comes back |
| **4** the envelope | link version 2, the kernel section (`PID`, `MADE`, `LAST`, `USES`), the open question; Wishes' v1 records read as its own section | Wishes | Wishes keeps a foreign section byte for byte (host tests, then `emu_wishes`) |
| **5** the kernel as a separate repository | the C, the site tables (1.11 from upstream's base), the dnfw mod, the registry and formats files, the proof scripts | other firmwares | a second firmware (upstream or the second firmware) builds it in and passes P1..P7 |
| **6** guests with a kernel | healthy reporting, DEFAULT for such guests, the per-firmware working state (if Q7 says so) | all members | a Wishes host booting an upstream build with the kernel, and back |

What other firmwares need, in order: a registry line (number, machine block, ranges); their data moved out of other
owners' ranges (rule 2.2.5); the kernel library and its site table for their stock base; the gate steps; for upstream,
the save/load link hooks on 1.11 if it is to keep other firmwares' records.

### 4.2 Risks

| risk | how bad | what holds it |
|---|---|---|
| the handover leaves state the guest does not expect (U1, U2) | guest crash; flash untouched, host returns | P1 equality before anything ships; fail open; BOOT ONCE first |
| an OS update started inside a guest (its MAIN OS over the unit's updater and bootstrap) | the only flash write left; a failed write is the ordinary recovery case, a bootstrap write would be the one true brick (`docs/SAFETY.md`) | the update RPC refused on guest boots (1.3); U11/P8 prove no other SPI NOR write is reachable |
| a guest choice that outlives a crash or a reflash (a boot loop) | the unit keeps booting a crashing guest; recovery only by flashing stock | one-shot cleared before the reset; the record tied to the host's build id (1.3); P9 |
| a guest's DSP program mismatched with its MAIN OS (R1 missing or wrong) | wrong or loud audio | R1 checked by P3; the mark names the staged section CRCs; a guest whose kernel ABI is too old is refused |
| a guest writes another owner's +Drive range | data loss for the user | the registry, the `_Static_assert`s, P6 in each member's gate; the root's range table; but a guest is only as safe as its own gate |
| a guest on another stock base migrates stock's projects (U8) | projects unreadable by the host | refused by default until U8 is measured |
| the staging window used by a future stock or member | the staged image damaged | the CRC check fails open; `rammap` on every stock release; the registry reserves the window |
| our soft reset loses the RAM on the unit | multi-boot never boots a guest (fails open to the host) | SAFE MODE's first unit test answers it |
| the eMMC read is slow | a guest boot takes several seconds more | measure (U5); store section 3 raw instead of packed if the depack is the slow part |
| Elektron brings `ekFS` to the DN2 at `0x5D8000` (as on the DT2) | every mod range inside a stock filesystem | stop all members at that release |
| a link lost when a non-member saves | a member's sound plays its stock part | the cells stay; the open notice says so; upstream adopting the hooks |
| two kernel versions in one unit disagree on the root | a write a newer kernel cannot read | A/B root with `min_reader`; read-only on a newer layout |

### 4.3 Questions for the User

- **Q1, licence of the kernel library:** AGPL-3.0-or-later (open firmwares only), or a permissive licence (any
  firmware, closed ones included, if the kernel holds no upstream-derived code)?
- **Q2, the registry:** in a new public repository that also holds the kernel, kept by the User; or a file in Wishes'
  public line; or proposed to upstream to keep?
- **Q3, guests without a kernel** (stock Elektron, upstream builds today): BOOT ONCE only, never the default (the
  study's proposal); or allow them as the default and accept that a guest that crashes at start needs the power-on key
  to get back to the host?
- **Q4, what may be stored as a guest:** only files from the page's published list (cleared Wishes builds, upstream
  releases, stock Elektron files, registry members' releases); or any `.syx` the user picks, with a warning?
- **Q5, room for other firmwares:** ask upstream for a ceiling on its library (the device's top for the image store
  and new members), or keep multi-boot inside Wishes' range (5 guest slots of 6 MiB) for now?
- **Q6, the power-on key** for the boot list (STOP is SAFE MODE's; FUNC is not seen by the UI queue): which key?
- **Q7, the working project on a switch:** (a) the other firmware starts empty and leaves the working state alone
  (SAFE MODE's way, smallest); (b) a working slot per firmware; (c) "SAVE ALL BEFORE SWITCHING?" then the shared
  working state?
- **Q8, the dual-boot stock DSP on `claude/dualboot`:** go on with it as is (SAFE MODE with sound), or fold it into
  stage 3 (stock 1.12 as the first guest, with its own DSP program and its own update RPC)?

## Multitone identity and the projects a firmware opens (the User, 2026-11-22)

Every firmware carries an identity: **brand . name . version** (e.g. `Elektron . DN2 . 1.12`,
`w0ts . Wishes . 0.69.6`). Every firmware also carries a list of the project formats it can open, one entry per
line: **brand . name . min version . max version**. A project records the identity of the firmware that saved it;
a firmware opens a project only when that identity falls inside one of its entries, and otherwise tells the user
which firmware made it.

Examples:

| firmware | opens |
|---|---|
| Elektron . DN2 . 1.12 | Elektron . DN2 . 1.0 . 1.12 |
| w0ts . Wishes . 0.69.6 | Elektron . DN2 . 1.0 . 1.12; w0ts . Wishes . 0.1 . 0.69.6 (not De-Pattern . dn2mk2) |

Stock firmware has no list of its own (it cannot read one): the kernel holds stock's list for it, and a stock
project (no identity record) counts as `Elektron . DN2 . <the OS version in its header>`.

## Multitone as its own repository (the User, 2026-11-22)

- **Repository:** `w0ts/multitone`, included by each firmware as a git submodule (Wishes first). It holds the kernel
  (C + the dnfw placement glue), the per-stock-version address tables, the proof scripts, the identity / open-list
  format and the registry.
- **Site:** one About page only (what Multitone is, how a firmware adopts it, the registry), served from the repo
  with the shared w0ts site style and the same accent as Wishes (teal).
- **Stock versions:** the kernel is not tied to 1.12 by design: stock addresses come from a byte-checked table per
  stock version, and the kernel switches itself off on a version without a table. The first table is 1.12 (what
  Wishes runs on); a 1.11 table is added when a 1.11-based firmware (e.g. upstream dn2mk2) adopts it.
- **Not created yet:** the repository and its site wait for the User's go (an outward step).

- **Mixing versions** (the User, 2026-11-22: start with 1.12 only): each firmware builds the kernel against its own stock version's table, so a 1.11-based firmware can carry its own kernel. A 1.12 host booting a 1.11 guest needs: the host-side handover (copy + jump, version-independent), the 1.11 table for the guest's section-7 redirect (R1), and U7 proven (a 1.11 OS on a unit whose boot chain came from 1.12). Projects across 1.11 / 1.12 are governed by the open lists.
