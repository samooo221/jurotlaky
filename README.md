# Tlaky — go-kart cold tyre-pressure helper

A single-file phone web app that tells you **what cold pressures to set** so each tyre
reaches its **target hot pressure** after a run. Built for Juro's karting; usable by
multiple drivers and across tracks.

- App: `index.html` (vanilla JS, no dependencies, no build step)
- Data: stored in the browser (`localStorage`), backed up via JSON export/import
- Built: 2026-06-29

---

## The idea in one line

> **cold to set = target − the pressure rise you expect this run**

You can't set hot pressure directly — the tyre heats up during the run and pressure climbs.
So you set a lower *cold* pressure and let it rise to the target. The only hard part is
predicting the rise, and the rise is driven mostly by **asphalt temperature**.

## The math (why it's honest, not magic)

Every logged run tells you the cold pressure you *should* have set:

```
ideal_cold = cold_used − (hot_measured − target)
```

Example (Franciacorta): set `0.69` cold, measured `0.82` hot, target `0.80`
→ ideal = `0.69 − (0.82 − 0.80)` = **0.67**. This reproduces Juro's own hand-written
"Teoreticky" numbers exactly — verified in the in-page self-check.

The recommender:
1. Builds a **track base** = seed + every logged run on that track, each normalised to a
   reference asphalt temp, then a **recency-weighted** average (recent runs count more).
2. Adds four **learned offsets** — how the **driver**, **kart class**, **tyre**, and
   **weather** each deviate from that base, learned across *all* runs with that value.
3. Adjusts for **today's asphalt temp** (hotter track → more rise → lower cold).

`recommendation = track base + driver + kart + tyre + weather − temp adj`

This is the "learning": no neural net, just transparent recency-weighted averages that get
sharper every time you log a run. With a few dozen data points that is the correct tool — a
model with more knobs would just fit noise.

### Factorised per-category learning ("F1 style")

Each category (driver, kart class, specific tyre, weather) gets its own **additive, per-tyre
offset** vs the track base — kept *separate* rather than one giant table of every combination,
because a joint table would need 100× the data. Each offset is **shrunk** toward zero until a
few runs exist (`K_DRIVER`) so one run can't swing it, and **decayed** so recent runs dominate
(`DECAY`). An unseen value (new driver, tyre never run) contributes zero → you get the plain
track base until data arrives. The Recommend screen shows each chosen value's contribution;
the Data tab lists every learned signature.

**Caveat (documented in code):** because all four offsets are measured against the same base,
two that always vary together (e.g. a class only ever run on one tyre) split that deviation
between them — correct for combinations you've actually run, mildly off only for novel ones.
A joint ridge fit is the upgrade path if data ever gets dense enough to need it.

## Per-tyre layout

```
Front L | Front R
Rear  L | Rear  R
```

Pressures in **bar** (e.g. `0.65`), temperatures in **°C**.

## Targets per track

Default hot target is **0.80 bar**. Overridden per track on the Data tab
(Třinec = `0.79`). Wet sessions can need pressures above `1.00` — set the target higher.

---

## Calibration knobs

These live at the top of the `<script>` in `index.html`. They are deliberate
defaults — tune them as real data shows whether recommendations over/undershoot.

| Knob | Default | Meaning |
|------|---------|---------|
| `REF_TEMP` | `45` | Asphalt °C the seed base pressures assume |
| `COEFF` | `0.004` | Bar of cold-pressure change per °C asphalt |
| `K_DRIVER` | `2` | Shrinkage — any category's offset is trusted only after a few runs |
| `SEED_W` | `1` | Seed base counts as ~1 (oldest) run, so real data overtakes it |
| `DECAY` | `0.85` | Recency — each older run counts this much less; `1` = flat average (never forgets) |
| `CLAMP_LO/HI` | `0.2 / 1.6` | A run whose implied cold lands outside this band (bar) is a typo → skipped |

## Seed data (from Juro's PDF notes)

Bases are `[FL, FR, RL, RR]` cold pressures at `REF_TEMP`:

| Track | Target | Base |
|-------|--------|------|
| Franciacorta | 0.80 | 0.67 / 0.71 / 0.66 / 0.69 |
| Slovakia ring | 0.80 | 0.67 / 0.65 / 0.65 / 0.55 |
| Třinec | 0.79 | 0.62 / 0.66 / 0.56 / 0.64 |
| Bruck | 0.80 | 0.62 / 0.64 / 0.61 / 0.66 |
| Lonato | 0.80 | *(not transcribed yet)* |

---

## Run it

**Locally:** open `index.html` in any browser. Check the console for `tlaky self-check passed`.

**On a phone (same Wi-Fi):**
```
python3 -m http.server 8000 --bind 0.0.0.0 --directory .
```
Then open `http://<laptop-LAN-IP>:8000/` on the phone. If it won't connect, open the
firewall port: `sudo firewall-cmd --add-port=8000/tcp`.

---

## Inputs and taxonomy

Three setup variables feed the math, each a learned offset:

- **Kart class** — Tillotson T4 (Bambino / Mini / Junior / Senior / Senior+) and
  Rotax (Micro / Mini / Junior / Senior / DD2 / DD2 Master).
- **Tyre** — MAXXIS and Mojo, brand → compound dropdowns grouped slick / wet. The
  **cadet/mini classes** (T4 Bambino/Mini, Rotax Micro/Mini) show only their own
  smaller-diameter compounds; every other class shows only the standard ones.
- **Weather** — Dry / Damp / Wet.

## Deliberately left out (and when to add it)

- **Per-weather target** — wet running often wants a *different* hot target, but for now
  there's one target per track and the weather/tyre offsets absorb the difference. Split it
  once enough wet runs exist. (Marked `ponytail:` in code.)
- **Session length, tyre life, track condition, ambient** — logged but *not* in the math yet.
  Fold in once there are enough runs to actually fit them.
- **Unipro import** — manual entry for now.
- **Edit a logged run** — you can only add or delete; no in-place edit yet.

## Open questions for Juro

1. **Seed Lonato?** Its PDF is 11 pages of Unipro screenshots, not yet transcribed.
2. **Does the Unipro app export data** (CSV / share / backup)? If yes, an importer means
   he never types pressures by hand — the biggest remaining time-saver.

---

## Roadmap to a finished product

Ordered by what actually turns this from a demo into a tool Juro uses every race weekend.

### 1. Get data in fast (the real bottleneck)
- Confirm whether **Unipro exports**. If yes → build an importer (parse its file, auto-fill runs).
- If no → keep manual entry; consider a faster bulk-paste mode.
- **Seed Lonato** so all five tracks start non-empty.

### 2. Make it real trackside (offline + installable)
- Host on **GitHub Pages** (free, permanent URL — any phone, no laptop needed).
- Turn it into a **PWA**: add a `manifest.json` + icon so it installs to the home screen,
  and a service worker so it **works with no signal** at the track (karting paddocks have
  bad Wi-Fi). This is the single biggest reliability upgrade.

### 3. Sharpen the model (once data accumulates)
- Fit the temp slope **per track / per tyre** instead of one global `COEFF`.
- Split the **hot target per weather** (wet wants a different target) once wet runs exist.
- Show the **predicted hot pressure + expected error**, so Juro sees confidence, not just a number.

### 4. UX polish
- **Edit** a logged run, not just delete.
- Per-track **history view** (a small chart of rise vs asphalt temp).
- Input validation and nicer empty states.

### 5. Protect the data
- `localStorage` is wiped if the browser is cleared. Add an **export reminder**, or later a
  simple cloud sync (only if multiple devices/drivers actually need shared data).

**Suggested next move:** do **#2 (GitHub Pages + PWA)** first so Juro can use it at the next
race regardless of signal, then chase the Unipro export in **#1**. The model (#3) improves on
its own as runs get logged — no rush.
