# Timeline Redesign Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Rebuild the `index.html` timeline as a compact, fluid, per-person strip that fits any viewport and draws no fact twice.

**Architecture:** Keep the existing `--s`/`--n` absolute-placement system and only change its units from pixels to percentages, so the grid becomes fluid. Then collapse eleven rows to seven: four per-person rows coloured by where each traveller sleeps replace the sleep row, the apart row and the four presence bars.

**Tech Stack:** Hand-written HTML + CSS in a single file, CSS grid, CSS custom properties. No build step, no framework, no test runner.

**Design doc:** `docs/plans/2026-08-25-timeline-redesign-design.md`

---

## Testing note — read before Task 1

This repo has **no test framework**. It is one static HTML file served as a GitHub Page. TDD as written in the skill does not apply; substituting a fake unit test would be theatre. Verification for every task is instead:

1. **Structural** — `grep`/`python3` assertions on the file (counts of rows, cells, class names).
2. **Syntax** — `node --check` on the extracted inline `<script>`.
3. **Visual** — serve on `localhost:8731` and screenshot in Chrome at three widths: **1400px** (design width), **900px** (laptop, where it scrolls today), **390px** (phone).

The visual check is the real gate. A task is not done until the screenshot is inspected. If the Chrome extension is unavailable, say so and stop rather than claiming a render was verified.

Serve once at the start:

```bash
cd /Users/jonas/GitHub/org/jonasjohansson/japan
(python3 -m http.server 8731 >/dev/null 2>&1 &)
```

Baseline before touching anything: screenshot at 1400 / 900 / 390 and keep them for comparison.

---

## Task 1: Make the grid fluid

The load-bearing change. Everything else assumes it.

**Files:** Modify `index.html` — `:root` (line ~81), `.grid` (~380), `.daysbg` (~408), `.seg` (~433), `.pbar` (~518), `@media` (~540).

**Step 1: Replace the geometry tokens**

`:root` line 81, currently `--col:68px; --label:150px;`

```css
  /* timeline geometry — --col is now a fallback for anything still in px;
     placement is percentage-based so the grid shrinks to any width */
  --col:68px; --label:72px;
```

**Step 2: Make `.grid` fluid**

`.grid`, replace the `grid-template-columns` and drop `min-width`:

```css
.grid{display:grid;grid-template-columns:var(--label) repeat(17,minmax(0,1fr));
  max-width:calc(var(--label) + 17 * var(--col));   /* never grows past today's 1306px */
  border-top:1px solid var(--line)}
```

**Step 3: Convert `.seg`/`.leg`/`.who`/`.stop`/`.lock`/`.pickmark`/`.pdiv` placement**

The shared rule at ~420. `100% / 17` is the width of one day column within a 17-column track:

```css
.seg,.leg,.stop,.who,.lock,.pickmark,.pdiv{position:absolute;
  left:calc(var(--s) * (100% / 17) + var(--seggap));
  width:calc(var(--n) * (100% / 17) - 2 * var(--seggap))}
```

**Step 4: Convert the `.daysbg` gradients**

Every `calc(N * var(--col))` becomes `calc(N * 100% / 17)`:

```css
.daysbg{background-image:
  linear-gradient(to right,
    transparent 0 calc(4 * 100% / 17),
    var(--we)   calc(4 * 100% / 17) calc(6 * 100% / 17),
    transparent calc(6 * 100% / 17) calc(11 * 100% / 17),
    var(--we)   calc(11 * 100% / 17) calc(13 * 100% / 17),
    transparent calc(13 * 100% / 17)),
  repeating-linear-gradient(to right, var(--line2) 0 1px, transparent 1px calc(100% / 17))}
```

**Step 5: Convert the `.seg` night hairlines**

`--col` cannot express one night inside a variable-width block. Use the block's own width divided by its span:

```css
  background-image:linear-gradient(to right,
    transparent calc(100% - 1px), var(--nightline) calc(100% - 1px));
  background-size:calc(100% / var(--n)) 100%;
  background-position-x:0}
```

**Step 6: Convert `.pbar`** (still the old presence bar; Task 4 replaces it, but it must not break in between)

```css
  left:calc(var(--s) * (100% / 17));width:calc((var(--e) - var(--s)) * (100% / 17));
```

**Step 7: Narrow-screen rule**

Replace the `@media (max-width:640px)` block:

```css
@media (max-width:700px){
  :root{--label:22px}                    /* initials only */
  .rowlabel{font-size:9px;padding-right:4px;text-align:center;align-items:center}
  .rowlabel small,.rowlabel .full{display:none}
  .rowlabel .init{display:block}
}
.rowlabel .init{display:none}
```

**Step 8: Verify**

```bash
python3 - <<'PY'
import io
s=io.open('index.html',encoding='utf-8').read()
assert 'min-width:calc(var(--label) + 17 * var(--col))' not in s, 'grid min-width still present'
assert s.count('var(--s) * (100% / 17)') >= 1, 'placement not converted'
assert 'calc(4 * var(--col))' not in s, 'daysbg not converted'
print('structure ok')
PY
```

Then screenshot at **1400 / 900 / 390**. Expected: at 1400 it looks essentially identical to the baseline; at 900 and 390 the full Oct 6–22 range is visible with **no horizontal scrollbar**. Gridlines and the weekend tint must still land on column boundaries — check the weekend shading covers exactly Oct 10–11 and Oct 17–18.

**Step 9: Commit**

```bash
git add index.html
git commit -m "Timeline columns go fluid, so the whole trip fits any screen"
```

---

## Task 2: Base band — numbers and split cells

**Files:** Modify `index.html` — `.track`/`.seg` CSS (~432), sleep-row markup (~736).

**Step 1: Add the split-cell and number CSS**

After the `.seg.part` rule, replacing it (the hatch is no longer how a split is shown):

```css
/* a numbered pin badge, tying the band to the map's stops 1-10 */
.seg i{font-style:normal;font-weight:700;font-size:9px;opacity:.75;margin-right:4px}
/* where one person sleeps elsewhere the cell splits: group on top, branch below */
.seg.split{padding-bottom:15px}
.segbr{position:absolute;left:0;right:0;bottom:0;height:13px;
  display:flex;align-items:center;justify-content:center;gap:4px;
  font-size:9px;font-weight:700;color:var(--branch-ink);
  white-space:nowrap;overflow:hidden}
.segbr.eri{background:var(--eri)} .segbr.jonas{background:var(--jonas)}
```

**Step 2: Rewrite the sleep-row markup as the base band**

Replace the whole `.track` block. Row label becomes `Where we sleep` / `numbered as on the map`:

```html
<div class="seg travel n"  style="--s:0;--n:1"><b>In the air</b><span>3 routes</span></div>
<div class="seg city"      style="--s:1;--n:3"><b><i>1</i>Tokyo</b><span>Graphy Shibuya</span></div>
<div class="seg party"     style="--s:4;--n:2"><b><i>2</i>Labyrinth</b><span>Lodge Sekigahara</span></div>
<div class="seg trek"      style="--s:6;--n:3"><b><i>3</i>Yakushima</b><span>Jomon Sugi + moss forest</span></div>
<div class="seg culture n" style="--s:9;--n:1"><b><i>4</i>Kagoshima</b></div>
<div class="seg culture n" style="--s:10;--n:1"><b><i>6</i>Fukuoka</b></div>
<div class="seg culture split" style="--s:11;--n:2"><b><i>7</i>Okayama</b><span>base for Naoshima</span>
  <span class="segbr eri">Eri: Ise</span></div>
<div class="seg culture n split" style="--s:13;--n:1"><b><i>9</i>Kyoto</b>
  <span class="segbr jonas">Jonas: Tokyo</span></div>
<div class="seg city n"    style="--s:14;--n:1"><b><i>10</i>Tokyo</b><span>all four</span></div>
<div class="seg city n"    style="--s:15;--n:1"><b><i>10</i>Tokyo</b><span>Eva + Nate</span></div>
<div class="seg travel n"  style="--s:16;--n:1"><b>In Japan</b><span>still here</span></div>
```

**Step 3: Verify**

Pins **5 (Kumamoto)** and **8 (Naoshima)** must be absent — they have no bed and belong to the day row. Confirm:

```bash
grep -o '<i>[0-9]*</i>' index.html | sort -u
# expect: 1 2 3 4 6 7 9 10  — no 5, no 8
```

Screenshot at 1400 / 900. The 17th–18th cell must read `7 Okayama` with a violet `Eri: Ise` strip along its bottom; the 19th `9 Kyoto` with a rust `Jonas: Tokyo` strip.

**Step 4: Commit**

```bash
git commit -am "Base band picks up the map's numbering and splits when someone is away"
```

---

## Task 3: Person rows

The core of the redesign.

**Files:** Modify `index.html` — `.prow`/`.pbar`/`.pdiv` CSS (~517), the four person rows (~810).

**Step 1: Replace the presence-bar CSS**

Delete `.pbar`, `.pbar em`, `.pbar span`, the `.pbar.oR/.oLR` hatch rules, `.pdiv` and its variants. Replace with:

```css
/* ── where each of us sleeps, night by night ──────────────────────────────── */
.prow{display:grid;grid-template-columns:repeat(17,minmax(0,1fr));height:22px;
  border-bottom:1px solid var(--line2)}
.pn{margin:3px 1px;border-radius:2px;background:var(--c,transparent);
  display:flex;align-items:center;justify-content:center;
  font-size:8px;font-weight:700;color:var(--on-accent);overflow:hidden}
.pn.city{--c:var(--city)} .pn.party{--c:var(--party)}
.pn.trek{--c:var(--trek)} .pn.culture{--c:var(--culture)}
.pn.travel{--c:var(--travel)} .pn.eri{--c:var(--eri)} .pn.jonas{--c:var(--jonas)}
.pn.eri,.pn.jonas{color:var(--branch-ink)}
.pn.gone{background:none}
/* runs off the right edge of the window */
.pn.on{background-image:var(--hatch);opacity:.55}
/* arrival and departure ticks */
.pt{font-size:8px;color:var(--muted);font-variant-numeric:tabular-nums}
```

**Step 2: Rewrite the four person rows**

Drop the `<div class="sechead">Who is in Japan</div>` and `.sechead-sp` pair. Each row is a `.rowlabel` plus 17 `.pn` cells. Per the design doc's night table:

- **Jonas** (`7–21 Oct`): `travel`, `city`×3, `party`×2, `trek`×3, `culture`, `culture`, `culture`×2, **`jonas`**, `city`, `gone`, `gone`
- **Eri** (`7–21 Oct`): `travel`, `city`×3, `party`×2, `trek`×3, `culture`, `culture`, **`eri`**×2, `culture`, `city`, `gone`, `gone`
- **Eva** (`7–24 Oct`): `travel`, `city`×3, `party`×2, `trek`×3, `culture`, `culture`, `culture`×2, `culture`, `city`, `city`, `on`
- **Nathanael** (`5 Oct–2 Nov`): `city`, `city`×3, `party`×2, `trek`×3, `culture`, `culture`, `culture`×2, `culture`, `city`, `city`, `on`

Row labels carry both a full name and an initial for the narrow rule:

```html
<div class="rowlabel"><span class="full">Jonas</span><span class="init">J</span><small>7–21 Oct</small></div>
```

Nathanael's initial is `N`; Eva's is `E`; to keep Eri and Eva distinct use `Eri → E` and `Eva → V` as in the approved sketch.

**Step 3: Verify**

```bash
python3 - <<'PY'
import re,io
s=io.open('index.html',encoding='utf-8').read()
rows=re.findall(r'<div class="prow[^"]*">(.*?)</div>\s*(?=<div class="rowlabel"|</div>)',s,re.S)
for i,r in enumerate(rows):
    n=r.count('class="pn')
    assert n==17, 'person row %d has %d cells, want 17' % (i,n)
print('4 rows x 17 cells ok' if len(rows)==4 else 'WRONG ROW COUNT %d'%len(rows))
PY
```

Screenshot at 1400 / 900 / 390. The read must be: four matching rows for thirteen days, Eri's row alone violet on the 17th–18th, Jonas' alone rust on the 19th, all four converging blue on the 20th, and Jonas + Eri ending after the 20th while Eva and Nathanael carry on hatched off the edge.

**If four rows of near-identical colour read as noise**, apply the fallback from the design doc's risks: mute the shared-base cells (`opacity:.55`) and let only `.pn.eri` / `.pn.jonas` stay fully saturated.

**Step 4: Commit**

```bash
git commit -am "Four person rows, coloured by where each of us sleeps"
```

---

## Task 4: Day row — merge daytime stops into the weight bars

**Files:** Modify `index.html` — `.stops`/`.stop`/`.pickmark` CSS (~464), `.load`/`.lcell` (~487), both markup blocks (~758, ~780).

**Step 1: Delete the `.stops`, `.stop`, `.stop.pick` and `.pickmark` CSS and their markup row entirely.**

**Step 2: Extend `.lcell` to carry the stop name**

```css
.load{display:grid;grid-template-columns:repeat(17,minmax(0,1fr));height:74px;
  border-bottom:1px solid var(--line2)}
.lcell em{font-style:normal;font-size:9px;font-weight:700;line-height:1.15;
  color:var(--culture);white-space:nowrap;text-overflow:ellipsis;overflow:hidden;max-width:100%}
.lcell em.pick{color:var(--muted)}          /* the Oct 17 fork */
.lcell em i{font-style:normal;opacity:.7;margin-right:2px}
```

**Step 3: Rewrite the 17 `.lcell`s**, adding `<em>` where the day has a stop:

| # | Date | Stop `<em>` | Bar | Labels |
|---|---|---|---|---|
| 2 | 8 | `Gear shops` | h0 | 0 / static |
| 7 | 13 | `Jomon Sugi` | h4 foot | 9–10 h / 22 km |
| 8 | 14 | `Shiratani` | h2 foot | 4–5 h / moss forest |
| 9 | 15 | `Sakurajima` | h2 | 3 h / ferry out |
| 10 | 16 | `<i>5</i>Kumamoto` | h2 | 2 hops / midday only |
| 11 | 17 | `<i>6</i>Fukuoka` + `<em class="pick">or Hiroshima</em>` | h3 | 1 h 45 / evening train |
| 12 | 18 | `<i>8</i>Naoshima` | h2 | 2 ferries / Uno ×2 |

Others keep their current bar and labels, no `<em>`.

**Step 4: Verify**

Screenshot at 1400 / 900. The Oct 17 cell must still read as a fork — `6 Fukuoka` *or* `Hiroshima` — and the numbers 5 and 8 must appear here, having been withheld from the base band in Task 2.

**Step 5: Commit**

```bash
git commit -am "Daytime stops fold into the weight row; 5 and 8 appear where the beds do not"
```

---

## Task 5: Delete the rows that no longer earn their space

**Files:** Modify `index.html` — `.apart`/`.who` CSS (~455), `.fixed`/`.lock` CSS (~508), their markup, the legend, the caption, the stats.

**Step 1:** Delete the `.apart` row markup and its `.apart`/`.who`/`.who.n`/`.who.eri`/`.who.jonas` CSS.

**Step 2:** Delete the `Locked in` row markup and its `.fixed`/`.lock` CSS. The anchors survive in the day-by-day table's highlighted `tr.anchor` rows and in the Practical section's "Fixed anchors" note — **confirm both are still present** before deleting:

```bash
grep -c 'tr class="anchor"' index.html   # expect 3
grep -c 'Fixed anchors' index.html       # expect 1
```

**Step 3:** Rewrite the legend. It drops from ten entries to seven — the `hatch`, `hollow` and `hollow dash` swatches all go with the rows that used them:

```
Tokyo · Festival · Yakushima · Kyushu / Seto / Kansai · Transit · Eri, apart · Jonas, apart
```

**Step 4:** Rewrite the caption under the grid. The old one explains hatching, "outlined blocks", and "two bars run off the right edge" — all obsolete. New text should explain: one column per night; four rows, one per person; matching colour means the same bed; violet and rust mean off on their own; the numbers match the map.

**Step 5:** Fix the stats row. `8 bases` is now checkable against the base band — count the numbered blocks (1, 2, 3, 4, 6, 7, 9, 10) and state the true figure.

**Step 6: Verify**

```bash
python3 - <<'PY'
import io
s=io.open('index.html',encoding='utf-8').read()
for dead in ['class="apart','class="who','class="fixed','class="lock','class="stop','class="pickmark','class="pbar','class="pdiv','class="sechead']:
    assert dead not in s, 'still present: '+dead
print('all removed rows gone')
PY
node --check <(python3 -c "
import re,io;print(re.findall(r'<script>(.*?)</script>',io.open('index.html',encoding='utf-8').read(),re.S)[0])")
```

**Step 7: Commit**

```bash
git commit -am "Drop the apart row, the lock row and the presence bars; retune legend and caption"
```

---

## Task 6: Final render pass

**Step 1:** Screenshot at **1400 / 900 / 390**, in both light and dark (`prefers-color-scheme`), and read the whole page top to bottom — not just the timeline. Check:

- no horizontal scrollbar at any width
- no clipped or ellipsised label anywhere in the strip
- weekend tint still lands exactly on Oct 10–11 and Oct 17–18
- the map below still renders (Task 1 touched no JS, but `node --check` it anyway)
- dark mode: `--eri` and `--jonas` still legible against `--branch-ink`

**Step 2:** Fix whatever the screenshots show. Expect at least one round of nudging.

**Step 3:** Update the design doc with anything that changed during implementation, then commit and push.

```bash
git add -A && git commit -m "..." && git push origin main
```

---

## Rollback

Every task is one commit on `main`. `git revert <sha>` backs out any single step. The pre-redesign timeline is at `d67b3f3`.
