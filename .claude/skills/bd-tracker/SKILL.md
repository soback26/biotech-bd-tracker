---
name: bd-tracker
description: >
  Update the biotech BD (business development) deal tracker workbook at
  `tracker/bd_tracker.xlsx` (in the biotech-bd-tracker repo) with new raw
  deal data exported from PharmCube (医药魔方) or equivalent biopharma BD
  database. Use this skill whenever the user mentions: "update bd tracker",
  "update biotech tracker", "BD deal tracker", "更新 BD tracker",
  "更新 deal list", "PharmCube export", "新 deal 数据",
  "out-license tracker update", "deal sourcing update",
  "process raw/YYYY-MM-DD.xlsx", or otherwise asks to refresh the GC/JP/KR
  outbound licensing deal tracker with new raw data. Also trigger if the
  user is inside the biotech-bd-tracker repo and asks to "update" or
  "refresh" without further specification.
---

# Biotech BD Tracker — Update Workflow

This skill updates `tracker/bd_tracker.xlsx` from a new raw deal data file in `raw/`. It is a **thin wrapper** over `CLAUDE.md` — the repo's `CLAUDE.md` is the source of truth for field mappings, vocabularies, formatting, and the full 5-phase pipeline. The skill's only job is to make the slash command and keyword triggers work, and to keep the core discipline (`Never skip Phase 4`) front-and-center.

## Prerequisites

- Invoke this skill from the **root of the biotech-bd-tracker repo** so that all paths (`raw/YYYY-MM-DD.xlsx`, `tracker/bd_tracker.xlsx`, `CLAUDE.md`, `company_names.md`) resolve relative to the repo.
- Python 3.9+ with `openpyxl` — the only runtime dependency, used to read and re-save the `.xlsx` tracker while preserving Bloomberg-style formatting.

## Authoritative rules — read these first, every time

Both files may have been edited since the skill was last invoked. Re-read at the start of every update session; do not rely on memory for any mapping.

1. **`CLAUDE.md`** — source of truth. Contains:
   - Field mapping (Source → Tracker) and the 15-column schema
   - Stage / modality / MoA / rights / disease vocabularies
   - Top-20 MNC canonical list
   - Note generation rules
   - Bloomberg-style formatting spec
   - Sheet assignment logic (`ex_china_stage = max(US, EU, JP)`, CN excluded)
   - Update-or-Append classification (NEW / UPDATE / PROMOTE / NO-OP)
   - The full 5-phase pipeline under `## Update Workflow (Append-Only Incremental)`
   - Quality checks
   - Legacy Chinese source vocabulary (Appendix A)

2. **`company_names.md`** — canonical CN→EN registry for licensor names. Required for two things:
   - Composing the `Licensor` column as `English (中文)` for CN/HK/TW entities (JP/KR licensors are English-only).
   - Normalizing licensor names in Phase 3 so the tracker-match key `(Licensor, Licensee, asset)` works across English and Chinese raws.

## Inputs

The user typically supplies one of:

- A new raw `.xlsx` file path (e.g. `raw/2026-04-15.xlsx`)
- An instruction like "update tracker with the latest raw file" → pick the newest `raw/*.xlsx` by filename date
- A dry-run request: "dry run only, don't write"

Raw files vary along **two independent axes**, and the workflow must handle all four combinations. `CLAUDE.md` has the detection and normalization logic — re-read the `## Data Source → Two orthogonal dimensions` and `### Raw preprocessing (Phase 1 → 2 transition)` sections before starting.

- **Content mode**: `incremental` (default, weekly net-new) vs `full-refresh` (legacy backfill, requires scope filters + dedup)
- **Row structure**: `flat` (one tracker-ready row per deal-asset) vs `dual-row` (PharmCube native — one `Deal Info` / `交易信息` header followed by one or more `Pipeline Info` / `管线信息` detail rows; normalize to flat in memory)

## The 5-phase pipeline (summary — full rules in `CLAUDE.md`)

| Phase | Purpose | Writes disk? |
|---|---|---|
| **1 — Inspection + Preprocessing** | Read raw; auto-detect flat vs dual-row; normalize dual-row → flat in memory; detect incremental vs full-refresh; read existing tracker; build the tracker-match index keyed on `(canonical English Licensor, Licensee normalized, asset normalized) → (sheet, row, current stages)` | No |
| **2 — Mapping** | Build the 15 tracker columns per `CLAUDE.md`. **Pause and ask** on any ambiguity: unfamiliar CN/HK/TW licensor (never guess — ask, then add to `company_names.md`), unresolvable modality, or a new Top-20-ish licensee not on the canonical list | No |
| **3 — Classification + Sheet Assignment** | Compute `ex_china_stage = max(US, EU, JP)` (**CN excluded**); classify each row as `NEW` / `UPDATE` / `PROMOTE` / `NO-OP` per the Update-or-Append Rules table; assign `NEW` and `PROMOTE` rows to `Actionable` or `Pipeline` | No |
| **4 — Diff Preview** ⚠️ | **Mandatory checkpoint.** Present projected sheet sizes and the full NEW / UPDATE / PROMOTE / NO-OP / CHECK diff. **Wait for explicit user confirmation** (`yes` / `go` / `OK`). If user says `dry run only`, stop here and exit cleanly | No |
| **5 — Write + Commit** | Apply writes preserving Bloomberg formatting and existing Notes; re-sort each touched sheet by `Date` desc + `Licensor` asc (values only, styles intact); run quality checks; readback-verify NEW/UPDATE/PROMOTE rows; `git add tracker/bd_tracker.xlsx raw/YYYY-MM-DD.xlsx`; commit with `update: YYYY-MM-DD \| +N new, +U updates, +P promotions \| <one-line summary>`; **ask before pushing** to `origin/main` | Yes (after confirmation only) |

## Key rules to never forget

- **Never write before Phase 4 confirmation.** Dry-run-by-default is the design.
- **Never skip Phase 4.** Every update must surface the full diff first.
- **Never guess Chinese-to-English company names.** Stop and ask; then update `company_names.md`.
- **Never downgrade stages.** Existing `Ph3` stays `Ph3` even if the new raw shows `Ph2`. Compare each stage cell individually; only `new > existing` (per ordering) gets overwritten.
- **Never overwrite manually-edited Notes.** On UPDATE / PROMOTE, only stage cells (and, if previously `—`, financials) change. The Note column is untouchable.
- **Never strip Bloomberg formatting** when re-saving — copy cell styles from an adjacent existing data row when inserting new rows.
- **Never commit the raw file without updating the tracker.** They go together in one commit.
- **Never `git add .`** — always specify `tracker/bd_tracker.xlsx` and `raw/YYYY-MM-DD.xlsx` explicitly.
- **Never push without asking.**

## Common edge cases (full list in `CLAUDE.md`)

- **Multi-asset NewCo deals** (e.g. Hengrui's Kailera): raw usually carries one row per asset pointing to the same NewCo licensee with identical headline financials. Append each row as-is.
- **Same asset, different licensee** (geographic carve-outs, e.g. RemeGen telitacicept to different partners for different territories): append as separate `NEW` rows; the match key differs on licensee so they are not flagged as duplicates.
- **Stale `(Top20 MNC)` tags in source**: always strip first, then re-apply against the canonical Top-20 list in `CLAUDE.md`.
- **English raw missing Chinese company name**: look up `company_names.md` to compose `English (中文)`. If not in the registry, stop and ask.
- **Ambiguous match** (same licensor + asset, different licensee): tag as `NEW` with `[CHECK]` prefix on the Note. Do not treat as UPDATE / PROMOTE.

## Quick-reference: file paths (all relative to the repo root)

| Purpose | Path |
|---|---|
| Authoritative rules | `CLAUDE.md` |
| Licensor name registry | `company_names.md` |
| Master tracker | `tracker/bd_tracker.xlsx` |
| Raw inputs | `raw/YYYY-MM-DD.xlsx` |
| Project intro (public-safe) | `README.md` |
