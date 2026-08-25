# Timeline redesign — design

Date: 2026-08-25
Scope: the timeline grid in `index.html`, from `<div class="scroll">` after `<h1>` down to the caption above `.legend`.

## Problem

Four failures, in order of severity.

**It does not fit.** `--col:68px` and `--label:150px` make the grid exactly 1306px, hard-coded. At a 1018px viewport only Oct 6–16 is visible; the Kansai/Tokyo end sits behind a horizontal scroll most readers never make. An overview you cannot see at once is not an overview.

**Cells are narrower than their contents.** One-day blocks cannot hold their labels. `Jonas → Tokyo` rendered as `Jonas → To` and had to lose its arrow to fit. Kagoshima, Fukuoka, Sakurajima and Kumamoto have the same problem.

**Eleven rows, several of which barely earn the space.** "Locked in" holds 3 items across 17 days. "Sleeping apart" holds 2. The four person bars are near-solid slabs encoding only 4 arrival and 4 departure times.

**Three encodings of one fact.** Hatching on the sleep row, the Eri/Jonas chips, and the person bars all say who is separated from the group. "Locked in" repeats the "Book these now" list below it.

It is not one bad chart. It is seven charts sharing a space that fits three.

## Decisions taken

1. Compact strip; the day-by-day table keeps the detail it already carries.
2. Four per-person rows coloured by location, replacing the sleep row, the apart row and the four "who is in Japan" bars.
3. On narrow screens the strip degrades to pure colour rather than scrolling.

## Design

### Row inventory: 11 → 7

| Today | Becomes |
|---|---|
| Date header | Date header, tighter |
| Leg | Phase band, thin — the narrative spine |
| Where we sleep | Base band, numbered to match map pins 1–10 |
| Sleeping apart | deleted — folded into base band and person colour |
| Daytime stops | merged into the day row |
| Weight of the day | Day row — weight bar plus the day's headline |
| Locked in | deleted — anchors are in the table and the "Fixed anchors" note |
| "Who is in Japan" heading | deleted |
| 4 person bars | 4 person rows, coloured by where each sleeps |

### Person rows

Each traveller gets 17 cells, coloured by where *that person* sleeps that night. With the group, the base colour; away, their own branch colour (`--eri` violet, `--jonas` rust). Thirteen days read as four matching rows; the two splits read as a single row breaking pattern. Rows start and end where each person's trip does, so Eva and Nathanael visibly run off the right edge and the Oct 7 arrival stagger stays legible.

Night-by-night, index 0 = Oct 6:

| # | Date | Jonas | Eri | Eva | Nathanael |
|---|---|---|---|---|---|
| 0 | 6 | air | air | air | Tokyo |
| 1–3 | 7–9 | Tokyo | Tokyo | Tokyo | Tokyo |
| 4–5 | 10–11 | Labyrinth | Labyrinth | Labyrinth | Labyrinth |
| 6–8 | 12–14 | Yakushima | Yakushima | Yakushima | Yakushima |
| 9 | 15 | Kagoshima | Kagoshima | Kagoshima | Kagoshima |
| 10 | 16 | Fukuoka | Fukuoka | Fukuoka | Fukuoka |
| 11–12 | 17–18 | Okayama | **Ise** | Okayama | Okayama |
| 13 | 19 | **Tokyo** | Kyoto | Kyoto | Kyoto |
| 14 | 20 | Tokyo | Tokyo | Tokyo | Tokyo |
| 15 | 21 | — | — | Tokyo | Tokyo |
| 16 | 22 | — | — | Japan | Japan |

Jonas is wheels-up 07:50 on the 21st and Eri 20:45, so neither sleeps the night of the 21st.

### Base band with split cells

One row naming where the group sleeps, numbered 1–10 to match the map pins so the two graphics finally reference each other. Where someone is away the cell splits horizontally: top is the group's base, bottom is the branch in that person's colour. The 17th–18th reads `7 Okayama / Ise`; the 19th reads `9 Kyoto / Tokyo`.

This is also what fixes the clipping. Names appear once, on a row whose blocks are as wide as the stay, instead of being repeated across four narrow person cells.

Blocks: `0 in the air` · `1 Tokyo ×3` · `2 Labyrinth ×2` · `3 Yakushima ×3` · `4 Kagoshima` · `6 Fukuoka` · `7 Okayama ×2 / Ise` · `9 Kyoto / Tokyo` · `10 Tokyo` · `10 Tokyo (Eva + Nate)` · `still here`.

Pins 5 (Kumamoto) and 8 (Naoshima) are absent by design — those are the places with no bed, and they appear on the day row instead. The gaps in the numbering are informative.

### Day row

Merges "Daytime stops" into "Weight of the day": one cell per day carrying the weight bar plus that day's headline, with the map number where the stop has one — `5 Kumamoto`, `8 Naoshima`. The Oct 17 "pick one" marker (Fukuoka + Dazaifu **or** Hiroshima + Miyajima) survives here; it is a live decision the group has not made.

### Fluid columns

```
grid-template-columns: var(--label) repeat(17, minmax(0,1fr));   /* was 17 × 68px */
.seg { left: calc(var(--s) * 100% / 17) }                        /* was --s * --col */
```

The `--s` / `--n` placement system is unchanged; only the units move from pixels to percentages. `.daysbg` gradient stops convert the same way. Label column 150px → 72px. The grid caps at today's 1306px so it never grows, but can now shrink to any width. Below ~700px labels drop out and the strip goes pure colour, with the legend carrying the meaning.

## Out of scope

The day-by-day table, the map, the money block and the prose sections are untouched, except for the caption under the timeline and the legend, which both have to match the new rows.

## Risks

- The `.daysbg` weekend tint is built from `calc(n * var(--col))` stops and must convert cleanly to percentages or the weekend shading will drift off the gridlines.
- Four person rows for thirteen identical days is more ink than one sleep row. That redundancy is the message — "we are together" — but if it reads as noise at implementation time, the fallback is to mute the shared-base cells and let only the branch colours saturate.
- Removing "Locked in" assumes the table's highlighted anchor rows are prominent enough. Verify at render.

---

## What changed during implementation

Recorded after the fact, 2026-08-25.

- **The branch strip names the place, not the person.** `Eri: Ise` overflowed the one-day Kyoto cell. Since the legend already reads "Eri, at Ise" and "Jonas, in Tokyo", the strip says only `Ise` and `Tokyo` and lets colour carry who.
- **One-day base cells put the number on top and the name underneath.** `<i>4</i>Kagoshima` on one line still truncated at ~48px. Two lines fit, and the band grew 52px → 58px so a split one-day cell holds number, name and branch strip.
- **Day-row stop names wrap to two lines** instead of ellipsising. `Kumamoto` and `Naoshima` do not fit one line in a one-day column; the row grew to 86px.
- **Arrival and departure times moved into the person cells** — `13:50` in Jonas' Oct 7 cell, `07:50` in his Oct 21 cell.
- **The legend kept a hatch swatch**, repurposed from "not all four" to "still here after the 22nd", so it is 8 entries rather than the 7 predicted.

## Known loss

The old `.pbar` positioned each traveller's bar at their *fractional* landing time (`--s:1.574` for Jonas' 13:50), so the Oct 7 arrival stagger was visible as three bars starting at three points inside one column. A cell grid cannot express sub-day precision. The times are now text inside the arrival cell and are also in the caption and the day-by-day table, but the *visual* stagger is gone. This was a real, if small, cost of the change.

## Risks that did not materialise

- The `.daysbg` weekend tint converted cleanly to percentages; shading still lands exactly on Oct 10–11 and Oct 17–18.
- Four rows of near-identical colour did **not** read as noise, so the muting fallback was not needed.
- `--hatch` is built from `var(--bg)`, so every new class inherits dark mode with no extra work.
