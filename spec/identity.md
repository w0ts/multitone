# Firmware identity and the open list

Status: **design, not built.** The text parts (identity, open list, the rules) are the User's decisions of
2026-11-22 (`docs/design.md`, "Multitone identity and the projects a firmware opens"). The binary record at the end
is a **proposal**, not agreed.

## 1. Identity: brand . name . version

Every firmware carries one identity:

    brand . name . version

| field | what | examples |
|---|---|---|
| brand | who makes it | `Elektron`, `w0ts`, `De-Pattern` |
| name | the firmware's name under that brand | `DN2`, `Wishes`, `dn2mk2` |
| version | dotted numbers, compared number by number (1.9 < 1.12) | `1.12`, `0.69.6` |

Examples: `Elektron . DN2 . 1.12`, `w0ts . Wishes . 0.69.6`.

A project records the identity of the firmware that **saved** it.

## 2. The open list: brand . name . min version . max version

Every firmware also carries the list of project formats it can open, one entry per line:

    brand . name . min version . max version

A firmware opens a project only when the project's identity has the same brand and name as one entry and its version
lies between that entry's min and max (both included). Otherwise the firmware does not open it and tells the user
which firmware made it (brand, name, version).

### Examples

| firmware | its open list |
|---|---|
| `Elektron . DN2 . 1.12` | `Elektron . DN2 . 1.0 . 1.12` |
| `w0ts . Wishes . 0.69.6` | `Elektron . DN2 . 1.0 . 1.12`<br>`w0ts . Wishes . 0.1 . 0.69.6` |

So Wishes 0.69.6 opens stock projects up to 1.12 and Wishes projects up to its own version. It does **not** open
`De-Pattern . dn2mk2` projects: they are a different brand and name, even where the formats look alike.

### Stock firmware

Stock firmware has no list of its own (it cannot read one). The kernel holds stock's list for it
(`Elektron . DN2 . 1.0 . <the stock version>`).

A stock project carries no identity record. It counts as:

    Elektron . DN2 . <the OS version in the project's header>

## 3. Proposed binary record (proposal, not agreed)

Fixed size, little-endian, versioned. Text fields are ASCII, padded with zero bytes, not necessarily zero-terminated
when full. Versions are packed as four bytes `major.minor.patch.build` (unused parts 0), so a plain unsigned compare
of the u32 orders them.

### 3.1 Identity record (64 bytes)

| offset | size | field | notes |
|---|---|---|---|
| 0 | 4 | magic | `MTID` |
| 4 | 2 | layout version | 1 |
| 6 | 2 | record size | 64; a reader skips bytes past what it knows |
| 8 | 2 | firmware number | from `registry/firmwares.txt` (0 = stock) |
| 10 | 2 | flags | 0 for now |
| 12 | 4 | version | packed, e.g. 1.12 = `01 0C 00 00`, 0.69.6 = `00 45 06 00` |
| 16 | 16 | brand | `Elektron`, `w0ts` |
| 32 | 16 | name | `DN2`, `Wishes` |
| 48 | 12 | reserved | zero |
| 60 | 4 | CRC-32 | over bytes 0..59 |

### 3.2 Open-list entry (48 bytes)

| offset | size | field | notes |
|---|---|---|---|
| 0 | 16 | brand | as in the identity record |
| 16 | 16 | name | as in the identity record |
| 32 | 4 | min version | packed |
| 36 | 4 | max version | packed |
| 40 | 8 | reserved | zero |

### 3.3 Open list (header + entries)

| offset | size | field | notes |
|---|---|---|---|
| 0 | 4 | magic | `MTOL` |
| 4 | 2 | layout version | 1 |
| 6 | 2 | entry count | at most 16 |
| 8 | 48 × count | entries | 3.2 |
| end | 4 | CRC-32 | over everything before it |

### 3.4 Rules for the binary form

- A reader that finds a higher layout version reads the fields it knows and writes nothing.
- Fields only grow at the end; a field is never reused for another meaning.
- Matching is on brand and name byte for byte (case kept), then the packed version range.
- Where the identity record sits in a project (the per-item envelope of `docs/design.md` 2.4/2.5) is not decided.
