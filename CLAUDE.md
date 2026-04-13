# Biotech BD Tracker — Update Rules

## Project Overview

A **Greater China / Japan / Korea outbound licensing deal tracker**. The tracker monitors cross-border BD transactions where GC/JP/KR biotech companies out-license assets to Western pharma/biotech, with a focus on identifying potential **royalty financing and structured capital opportunities**.

---

## File Layout

```
biotech-bd-tracker/
├── README.md              # Project intro (English, public-safe)
├── CLAUDE.md              # This file — update rules and conventions
├── company_names.md       # Canonical CN→EN licensor name registry
├── raw/
│   └── YYYY-MM-DD.xlsx    # Source exports (PharmCube or equivalent)
├── tracker/
│   └── bd_tracker.xlsx    # Master tracker — sheets: Actionable, Pipeline
├── scripts/               # (Reserved for future automation)
└── archive/               # (Reserved for archived versions)
```

---

## Data Source

- **Provider**: 医药魔方 (PharmCube), or any equivalent biopharma BD database that supplies the same field set
- **Naming convention**: `raw/YYYY-MM-DD.xlsx` (date of download / drop)
- **Primary mode — weekly incremental** (default going forward):
  - English-language `.xlsx`, pre-filtered by the analyst to contain **only net-new deals** not yet in the tracker
  - One row per deal-asset (flat structure); out-of-scope deals (domestic-only, Preclnl-only, Inactive, non-GC/JP/KR licensors) are already excluded by the analyst before saving
  - This is the **default input format** — the workflow is append-only, not a full refresh, and no dedup against the existing tracker is performed
- **Legacy mode — full-refresh Chinese dump** (supported for backfilling existing archives):
  - Full PharmCube search export with column names in Chinese
  - **Dual-row structure**: each deal has one `数据层级 = 交易信息` header row followed by one or more `数据层级 = 管线信息` detail rows. Multi-asset deals produce multiple pipeline rows.
  - Requires the full 10-step legacy workflow (parse dual rows → apply in-scope filters → dedup against existing tracker → merge) — see the "Legacy Full-Refresh Workflow" sub-section below if you ever need to backfill from this format.
- **Sheet name detection**: detect by content/structure, not by sheet name. Common names: `Search Results`, `检索结果`, or arbitrary user-chosen names.

---

## Tracker Output

**File**: `tracker/bd_tracker.xlsx`

### Sheet definitions

| Sheet | Definition | Current rows |
|---|---|---|
| `Actionable` | Highest global stage ∈ {`Ph3`, `NDA Filed`, `Approved`} — near-term cash flow potential, primary underwriting universe | ~35 deals |
| `Pipeline` | Highest global stage ∈ {`IND Filed`, `Ph1`, `Ph1/2`, `Ph2`, `Ph2/3`} — watchlist for graduation to Actionable | ~140 deals |

**Excluded**: All-`Preclnl`-only assets; deals where global R&D status is `Inactive`; pure domestic deals (out-licensing direction = "Domestic to Domestic" / `国内转国内`).

### 15-column schema (identical across both sheets)

| # | Column | Width | Description |
|---|---|---|---|
| 1 | **Licensor** | 24-30 | Transferring party. CN/HK/TW: `English (中文)` per `company_names.md`. JP/KR: English only. |
| 2 | **HQ** | 8 | Country: `China`, `Japan`, `Korea`, `Taiwan`, `HK`, or combinations like `Japan; UK`. |
| 3 | **Licensee** | 24 | Receiving party. Append `(Top20 MNC)` for top-20 global pharma. |
| 4 | **Asset / Target / MoA** | 42 | Format: `asset_name (target \| MoA_abbreviated)`, e.g. `ivonescimab (VEGF-A; PD1 \| anti-PD1×VEGF-A BsAb)`. If MoA just restates modality, keep target only: `sacituzumab tirumotecan (TROP2 \| ADC)`. |
| 5 | **Modality** | 12 | Standardized modality (see vocabulary below). |
| 6 | **US** | 10 | US development stage. |
| 7 | **CN** | 10 | China development stage. |
| 8 | **EU** | 10 | EU development stage. |
| 9 | **JP** | 10 | Japan development stage. |
| 10 | **Total ($M)** | 12 | Total deal value, USD millions. Format: `$X.Xbn` if ≥1000, else `$XXXm`. `—` if undisclosed. |
| 11 | **Upfront ($M)** | 12 | Upfront payment, USD millions. Same format. |
| 12 | **Lead Indications** | 28-34 | Top 2-3 indications, semicolon-separated and abbreviated (see disease abbreviations below). |
| 13 | **Rights** | 28 | Rights structure: `R&D rights: <terr> \| Mfg rights: <terr> \| Commercial rights: <terr>` or `All rights: <terr>`. |
| 14 | **Date** | 11 | `YYYY-MM-DD`. |
| 15 | **Note** | 70-170 | 1-3 sentence English analyst note: FIC/BIC flag, deal structure, key milestones, and counterparty/financing relevance. |

### Bloomberg-style formatting spec

Match the existing tracker exactly when writing — never strip these styles when re-saving:

```python
from openpyxl.styles import PatternFill, Font, Border, Side, Alignment

# Header row (row 1)
HDR_FILL = PatternFill('solid', fgColor='1B1B1B')                    # near-black
HDR_FONT = Font(name='Arial', bold=True, size=8.5, color='FF8C00')   # orange
HDR_HEIGHT = 21.9

# Data cells
CELL_FONT       = Font(name='Arial', size=8.5, color='333333')       # dark grey
NOTE_FONT       = Font(name='Arial', size=8,   color='888888')       # lighter grey, smaller
DATA_HEIGHT     = 26.1

# Stage columns (US/CN/EU/JP) — center aligned
STAGE_ALIGN     = Alignment(horizontal='center', vertical='center')
# Note column — wrap text, left aligned
NOTE_ALIGN      = Alignment(horizontal='left',   vertical='center', wrap_text=True)
# All other columns — left aligned
DEFAULT_ALIGN   = Alignment(horizontal='left',   vertical='center')

# Priority highlight (rows flagged as high-conviction sourcing targets)
HIGHLIGHT_FILL  = PatternFill('solid', fgColor='FFF2CC')             # light yellow

# Sheet-level
freeze_panes  = 'A2'
auto_filter   = 'A1:O1'
tab_color     = '1B1B1B'
```

### Highlighting rules

Apply the yellow `#FFF2CC` fill to the **entire row** when the Note flags the deal as a high-conviction sourcing target. As of the current tracker:

- **`Actionable` sheet**: typically all rows highlighted (every late-stage deal warrants screening attention)
- **`Pipeline` sheet**: highlight only standout candidates (private licensors with mega-deals, multi-asset NewCo structures, near-term IND→Ph2 inflection points)

A row is a high-conviction target when the Note's counterparty/financing relevance assessment is positive (see Note Generation Rules below).

---

## Field Mapping: Source → Tracker

### Source columns

The table below lists the **conceptual source fields** with both their likely English column header and the legacy Chinese header. Detect by content/position when names vary across exports.

| Tracker Column | Source Field (English) | Source Field (Chinese, legacy) | Transform |
|---|---|---|---|
| Licensor | Licensor / Transferor | 转让方 | Look up in `company_names.md`. CN/HK/TW: `English (中文)`. JP/KR: English only. |
| HQ | Licensor Country | 转让方所在国家/地区 | Map country names to short form (see HQ values below). |
| Licensee | Licensee / Transferee | 受让方 | Strip any pre-existing `(Top20 MNC)`, then re-evaluate against the canonical Top-20 list below. |
| Asset / Target / MoA | Drug Name + Target + MoA | 管线名称 + 靶点 + 作用机制 | Compose: `drug_name (target \| MoA_abbreviated)`. Translate any remaining Chinese drug names. Multiple targets → semicolon-separated. |
| Modality | Drug Class | 药品类别三 (fallback 药品类别二) | Apply Modality Vocabulary below. |
| US / CN / EU / JP | US/CN/EU/JP Highest Phase | 美国/中国/欧洲/日本最高研发阶段 | Apply Stage Vocabulary below. |
| Total ($M) | Total Deal Value ($M) | 总金额(百万)/美元 | `≥1000` → `$X.Xbn`; else `$XXXm`. Blank/None → `—`. |
| Upfront ($M) | Upfront Payment ($M) | 首付款(百万)/美元 | Same formatting. |
| Lead Indications | Indications | 疾病 | First 2-3, abbreviated (see Disease Abbreviations below). Drop generic terms (`Cancer`, `Solid tumors`) when more specific terms exist. |
| Rights | Rights | 权益信息 | Apply Rights Vocabulary below. |
| Date | Deal Date | 交易时间 | `YYYY-MM-DD`. |
| Note | — | — | Auto-generate per Note Generation Rules. |

**HQ values**: `China`, `Japan`, `Korea`, `Taiwan`, `HK`, or combinations like `Japan; UK`.

**Out-licensing direction filter** — only keep deals where direction is one of:
- `Domestic to Foreign` (legacy: `国内转国外`)
- `Foreign to Foreign` (legacy: `国外转国外`)

(Skip `Domestic to Domestic` / `国内转国内`.)

### Stage Vocabulary

Canonical English values used in tracker (US/CN/EU/JP columns):

`Preclnl < IND Filed < Ph1 < Ph1/2 < Ph2 < Ph2/3 < Ph3 < NDA Filed < Approved`

The `<` ordering is used for "highest global stage" comparison and for the never-downgrade rule (see Update Workflow).

### Modality Vocabulary

Standardize all values to one of:

| Tracker Modality | Source patterns |
|---|---|
| `mAb` | Monoclonal antibody, mAb, single antibody, 抗体 (alone), 单抗 |
| `BsAb` | Bispecific antibody, 双特异性抗体 |
| `TsAb` | Trispecific antibody, 三特异性抗体 |
| `ADC` | Antibody-drug conjugate, 抗体偶联药物, 抗体; 偶联药物 |
| `Fusion Protein` | Fusion protein, Fc-fusion, 融合蛋白 |
| `Small Mol.` | Small molecule, 小分子, 化药 (default if no further detail) |
| `Peptide` | Peptide, 多肽 |
| `siRNA` / `ASO` / `mRNA` | Use specific subtype if known; otherwise `siRNA` for `核酸` / nucleic acid |
| `Cell Therapy` / `CAR-T` | Cell therapy, CAR-T, 细胞疗法 |
| `Gene Therapy` | Gene therapy, AAV, 基因疗法 |
| `Radiopharm.` | Radiopharmaceutical, 放射性药物 (note in Note field) |
| `Degrader` | PROTAC, molecular degrader, 降解剂 |
| `Mol. Glue` | Molecular glue, 分子胶 |
| `Vaccine` | Vaccine, 疫苗 |

If the source field is empty, infer from broader category (e.g. `生物` / Biologic → default `mAb`; `化药` / Chemical → default `Small Mol.`) and refine using target/MoA.

### MoA Abbreviation Rules

Inside the `Asset / Target / MoA` column's MoA segment, apply these in-line replacements:

| Long form | Abbreviation |
|---|---|
| inhibitor / 抑制剂 | `inh.` |
| agonist / 激动剂 | `agonist` |
| antagonist / 拮抗剂 | `antagonist` |
| modulator / 调节剂 | `modulator` |
| positive allosteric modulator / 正向变构调节剂 | `PAM` |
| negative allosteric modulator / 负向变构调节剂 | `NAM` |
| orthosteric agonist / 正构激动剂 | `orthosteric agonist` |
| allosteric inhibitor / 变构抑制剂 | `allosteric inh.` |
| anti-X antibody-drug conjugate / anti-X抗体偶联药物 | `ADC` |
| anti-X bispecific antibody / anti-X双特异性抗体 | `BsAb` |
| anti-X monoclonal antibody / anti-X单抗 | `mAb` |
| siRNA therapy / siRNA疗法 | `siRNA` |
| CAR T-cell therapy / CAR T细胞疗法 | `CAR-T` |

If MoA is just a restatement of the modality (e.g. modality = `ADC` and MoA = `anti-HER2 ADC`), drop the MoA segment entirely and keep target only:
- `disitamab vedotin (HER2 | ADC)` → `disitamab vedotin (HER2)` + Modality column = `ADC`

### Rights Vocabulary

**Rights types**:
| Source | Tracker |
|---|---|
| All rights / 所有权益 | `All rights` |
| R&D rights / 研发权益 | `R&D rights` |
| Commercial rights / 商业化权益 | `Commercial rights` |
| Manufacturing rights / 生产权益 | `Mfg rights` |
| `(option)` / `(期权)` suffix | `(opt.)` suffix |

**Territories**:
| Source | Tracker |
|---|---|
| Global / 全球 | `Global` |
| US / 美国 | `US` |
| EU / 欧洲 | `EU` |
| JP / 日本 | `JP` |
| China (Mainland) / 中国(内地) | `CN` |
| China (HK/Macau/TW) / 中国(港澳台) | `HK/TW` |
| Other / 其他 | `ROW` |

**Format**: Concatenate rights types with ` | `. Examples:
- `R&D rights: 全球 | 商业化权益: 全球` → `R&D rights: Global | Commercial rights: Global`
- `所有权益: 美国;欧洲;日本;其他` → `All rights: US;EU;JP;ROW`

### Disease Abbreviations

Use standard medical abbreviations in the `Lead Indications` column. Keep top 2-3 indications, semicolon-separated. Drop overly generic terms (`Cancer`, `Solid tumors`) when more specific labels are available.

**Oncology** — NSCLC, sq NSCLC, SCLC, TNBC, HCC, CRC, RCC, GBM, AML, ALL, CLL, MM, MDS, DLBCL, FL, MCL, MZL, HER2+ Breast Ca, HER2-low Breast Ca, HR+ Breast Ca, Gastric Ca, Ovarian Ca, Pancreatic Ca, Endometrial Ca, Cervical Ca, Esophageal Ca, Bladder Ca, Prostate Ca, mCRPC, Melanoma, Glioma, GBM, NPC, HNSCC, Cholangio Ca

**Metabolic / Cardio** — T2DM, Obesity, MASH, NASH, CKD, HFpEF, ASCVD, HoFH, Hypertension

**Immunology / Inflammation** — SLE, RA, IBD, UC, CD, Psoriasis, Atopic Dermatitis, Asthma, COPD, MS, IgA Nephropathy, Sjögren's, Myasthenia Gravis, AAV, AIH

**Rare / Other** — DMD, SMA, Pompe, Fabry, Gaucher, AATD, ALS, Parkinson's, Alzheimer's

### Top-20 MNC List

Append `(Top20 MNC)` to Licensee for any of these (and **strip any pre-existing tag** from source data first, then re-apply against this canonical list):

> Pfizer, Roche, Novartis, Merck & Co., Johnson & Johnson, AbbVie, AstraZeneca, Sanofi, Bristol-Myers Squibb, Eli Lilly, Amgen, GSK, Gilead, Bayer, Novo Nordisk, Takeda, Boehringer Ingelheim, Regeneron, Daiichi Sankyo, Astellas Pharma

**Watchlist** (tagging may be added/removed as market caps shift): Vertex, Biogen.

---

## Company Name Registry

`company_names.md` is the **canonical CN→EN mapping** for licensor names. It currently contains ~80 Greater China biotech entries plus reference notes for JP/KR companies (which use English-only names).

### How it is used

1. **Composing the Licensor column** (primary use): For any CN/HK/TW licensor, format as `English Name (中文)` using the registry. Example: raw has `RemeGen` or `荣昌生物` → tracker shows `RemeGen (荣昌生物)`. If the weekly incremental raw is in English but lacks the Chinese name, look up the Chinese via the registry and append it in parentheses.
2. **Sanity-check duplicate detection** (light use): In the rare event that an analyst accidentally includes a deal that already exists in the tracker, normalize both sides to canonical English via the registry before comparing. This is a **warning surface**, not an automatic dedup — any overlap is flagged to the user for review.
3. **Legacy full-refresh dedup** (legacy use only): When backfilling from a full Chinese dump, the registry is the only reliable way to match Chinese-raw licensors against an English tracker.
4. **JP/KR licensors**: No Chinese annotation. Use the English name as-is from source.
5. **JV entities**: No Chinese annotation (e.g. `KYM Biosciences` for 康明百奥 (JV)).

### When to add new entries

If a raw file contains a licensor not yet in the registry:

- **Stop and ask the user** for the verified English name. Do not guess.
- Common pitfalls (real examples from the registry): `宜联生物 = MediLink Therapeutics` (NOT "Yilian"); `舶望制药 = Argo Biopharma` (NOT "BoWan"); `博奥信 = Biosion` (NOT "BioAtla"); `维立志博 = Leads Biolabs` (NOT "Viela Bio"); `橙帆医药 = VelaVigo Pharma` (NOT "VBC Biotech").
- Never confuse Chinese companies with similarly-named Western ones (e.g., `祐和医药 ≠ Yuhan Pharma Korea`; `科弈药业 ≠ Kinnate Biopharma US`).
- After confirmation, append the new row to `company_names.md` in alphabetical-by-Chinese order within the appropriate section.

---

## Note Generation Rules

Each row's `Note` is a **1-3 sentence English analyst comment**. The Note is the most important analytical column — it is what makes the tracker useful for screening, beyond a simple data dump.

### Required content (in order)

1. **FIC/BIC flag** — If source `药品标签` / `Tags` field contains `First-in-Class` or `Potential First-in-Class`, lead with `Potential FIC.` or `FIC.` (Always use `Potential FIC`, never bare `FIC` unless the source explicitly says so.)
2. **Deal structure summary** — Rights granted, to whom, in which territories. Example: `Ex-CN R&D + commercial to Summit.`
3. **Development status highlight** — Key stage milestones across regions. Example: `US NDA filed, CN approved.`
4. **Counterparty / financing relevance** — Royalty financing assessment for the **licensor**:
   - Small/mid-cap with capital needs → `may consider royalty monetization`
   - Large self-funded pharma (Daiichi Sankyo, Takeda, Astellas, Chugai/Roche) → `self-funded — no financing need`
   - Near-term revenue stream → `Near-term US revenue potential`
   - Too early-stage or too small → `Limited royalty scale`

### Tone

Direct, opinionated, terse, investment-memo style. No filler. Semicolons separate thoughts. Use abbreviations: CN, US, EU, JP, MNC, FIC, BIC, Ph3, NDA.

**Good examples** (write Notes like these):
- `Private, capital-intensive ADC platform — strong sourcing candidate.`
- `Hengrui is large-cap self-funded — no financing need.`
- `Worth exploring whether Innovent would monetize a portion of retained economics.`
- `Textbook target — private, needs growth capital, Novartis validation de-risks.`
- `Potential FIC. Ex-CN all rights to BioNTech. HER2 ADC in US/EU Ph3, CN NDA filed. $1.67bn deal. DualityBio is private with multiple ADC programs burning cash.`

**Do NOT write**:
- Generic structural ratings like `Angle: Med-High`, `Status: Active`
- Raw data restatements (the other 14 columns already cover that)
- Empty cells — use `—` if there is genuinely nothing useful to say

---

## Sheet Assignment Logic

Compute `highest_global_stage = max(US, CN, EU, JP)` using the stage ordering above.

| Highest Global Stage | Sheet |
|---|---|
| `Approved`, `NDA Filed`, `Ph3` | **`Actionable`** |
| `Ph2/3`, `Ph2`, `Ph1/2`, `Ph1`, `IND Filed` | **`Pipeline`** |
| `Preclnl` only or all blank | **Excluded** |
| Global R&D status = `Inactive` | **Excluded** |

Within each sheet, sort by `Date` descending (newest first), then by `Licensor` alphabetically.

---

## Sanity-Check Rules (not dedup)

The weekly incremental raw is **already net-new by analyst curation** — full dedup logic is not run on the primary workflow. However, a light sanity check runs in Phase 3 to catch accidental re-inclusion of already-tracked deals:

**Sanity-check key**: `(canonical English Licensor) + (Licensee, normalized) + (asset name, normalized)`

- **Canonical English Licensor**: translate via `company_names.md` so `荣昌生物`, `RemeGen`, and `RemeGen (荣昌生物)` all resolve to the same key
- **Asset name normalized**: lowercase, trim, strip parenthetical (e.g. `IBI363 (IL-2; PD1 | ...)` → `ibi363`)

**Behavior**:

| Check result | Action |
|---|---|
| No match in existing tracker | Append as normal (default case) |
| Match found | **Flag to user** in Phase 4 diff preview with the message: `"Warning: <asset> from <licensor> → <licensee> looks already tracked in <sheet> row <N>. Include anyway / skip / update existing?"` |

The default assumption is that if the analyst included it, they meant to include it — but the warning prevents silent duplication.

**Never-downgrade-stage rule** (still applies to manual update requests): If an analyst asks me to update an existing row with new stage data, never regress — e.g. existing tracker `US = Ph3`, new data says `US = Ph2` → keep `Ph3`. Stage downgrades almost always reflect data-quality issues, not real regression.

**Preserve-manual-notes rule**: If an existing row's Note has been hand-edited (doesn't match auto-generated template), do not overwrite on any update. Only stage/financial cells get touched.

---

## Update Workflow (Append-Only Incremental)

When asked to process a weekly raw file, run as a **5-phase pipeline with a mandatory review checkpoint**:

### Phase 1 — Inspection (read-only)
- Read new raw file from `raw/YYYY-MM-DD.xlsx` and count rows
- Read existing `tracker/bd_tracker.xlsx` for baseline sizes and for the sanity-check index
- Report: `N raw rows to append; existing tracker = X Actionable + Y Pipeline`

### Phase 2 — Mapping (in-memory, may pause for input)
- Map each raw row to the 15-column schema (stages, modality, rights, indications) per the Field Mapping rules below
- Compose `Licensor` as `English (中文)` for CN/HK/TW via `company_names.md`; JP/KR use English only
- Strip any pre-existing `(Top20 MNC)` from raw Licensee, then re-apply against the canonical Top-20 list
- Auto-generate Note per Note Generation Rules
- **Pause and ask** if any field requires a judgment call:
  - Unfamiliar CN/HK/TW licensor not yet in `company_names.md` (never guess — ask for verified English name, then add to registry)
  - Ambiguous modality that can't be unambiguously determined from the source fields
  - New licensee that looks Top-20-ish but isn't in the canonical list

### Phase 3 — Sheet Assignment + Sanity Check (in-memory)
- Compute `highest_global_stage = max(US, CN, EU, JP)` per stage ordering
- Assign each row to `Actionable`, `Pipeline`, or exclude (Preclnl-only / all-blank)
- Run the sanity-check index against the existing tracker — flag any row whose `(Licensor, Licensee, asset)` triple already exists
- Determine the correct insertion point so the final sheet remains sorted by `Date` descending, then `Licensor` ascending

### Phase 4 — Diff Preview (mandatory checkpoint, no files written) ⚠️
Show the user, before writing anything:
- Projected sheet sizes: `Actionable: 35 → 35+X; Pipeline: 140 → 140+Y`
- Full list of rows to append, grouped by target sheet, showing: `Licensor | Licensee | asset | stages US/CN/EU/JP | Total/Upfront | Note (truncated)`
- Any sanity-check warnings with the specific question to resolve
- Any Phase 2 pauses that got resolved, with a note showing what was decided

**Wait for user confirmation** (`yes` / `go` / `OK`) before proceeding to Phase 5. If user says `dry run only`, stop here and do not write any files.

### Phase 5 — Write + Commit (only after explicit confirmation)
- Append new rows to the correct sheet using openpyxl, **preserving Bloomberg formatting** (header fill, fonts, freeze panes, auto filter, row heights, highlight rules). Copy the style from an adjacent existing data row when inserting new cells so they inherit the right fonts/fills/borders.
- Re-sort the affected sheet(s) by `Date` descending, then `Licensor` ascending. Sorting should touch only values, not formatting.
- Run Quality Checks (see next section) on the in-memory output before saving.
- Readback: re-open the saved file and verify 3 of the newly appended cells match expected values.
- `git add tracker/bd_tracker.xlsx raw/YYYY-MM-DD.xlsx` (exactly these two paths — never `git add .`)
- Commit with the canonical message:
  ```
  update: YYYY-MM-DD | +X new deals | <one-line summary of notable deals>
  ```
- Ask before pushing to `origin/main`.

---

## Quality Checks (run before saving)

```python
# 1. No Chinese characters in any cell EXCEPT the Licensor column's parenthetical
import re
CN = re.compile(r'[\u4e00-\u9fff]')
for r in range(2, ws.max_row+1):
    for c in range(1, ws.max_column+1):
        val = ws.cell(r, c).value
        if val is None: continue
        col_name = headers[c-1]
        if col_name == 'Licensor':
            # Allow Chinese only inside parentheses
            outside_parens = re.sub(r'\([^)]*\)', '', str(val))
            assert not CN.search(outside_parens), f"CN outside parens in Licensor row {r}"
        else:
            assert not CN.search(str(val)), f"CN in {col_name} row {r}: {val!r}"

# 2. Stage fidelity — every stage cell must be in the canonical vocabulary or blank
VALID_STAGES = {'', 'Preclnl', 'IND Filed', 'Ph1', 'Ph1/2', 'Ph2', 'Ph2/3', 'Ph3', 'NDA Filed', 'Approved'}
for stage_col in ['US', 'CN', 'EU', 'JP']:
    for r in range(2, ws.max_row+1):
        v = (ws.cell(r, headers.index(stage_col)+1).value or '')
        assert v in VALID_STAGES, f"Invalid stage {v!r} at {stage_col} row {r}"

# 3. No empty Note cells
for r in range(2, ws.max_row+1):
    note = ws.cell(r, headers.index('Note')+1).value
    assert note and note.strip(), f"Empty Note at row {r}"

# 4. No banned phrases in Note
BANNED = ['Angle: Med', 'Angle: High', 'Angle: Low', 'Status: Active', 'angle:']
for r in range(2, ws.max_row+1):
    note = ws.cell(r, headers.index('Note')+1).value or ''
    for b in BANNED:
        assert b not in note, f"Banned phrase {b!r} in Note row {r}"

# 5. Sheet assignment sanity
# Every row in Actionable must have at least one stage in {Ph3, NDA Filed, Approved}
# Every row in Pipeline must have its highest stage in {IND Filed..Ph2/3}
```

If any check fails, do not save — fix the in-memory data and re-run.

---

## Important Context

- This tracker is a **sourcing engine** for royalty financing and structured capital opportunities.
- Key lens: **which licensors might benefit from royalty monetization**, synthetic royalties, or milestone receivable financing?
- **Large self-funded JP pharma** (Takeda, Daiichi Sankyo, Astellas, Chugai/Roche) are generally **NOT sourcing targets** — they don't need balance-sheet capital.
- **Small/mid-cap CN, TW, KR biotech** with multiple out-licensing deals are higher-priority targets — they typically have multiple cash-burning programs to fund.
- Pay special attention to deals with **disclosed financials** (upfront + milestones + royalties), as these define the cash flow structure available for monetization.

---

## Appendix A: Legacy Chinese Source Vocabulary

Used only when processing legacy Chinese-language raw exports (e.g., `raw/2026-04-10.xlsx`).

### Stage map (Chinese → English)

| Chinese | English |
|---|---|
| `临床前` | `Preclnl` |
| `申报临床` / `申请临床` | `IND Filed` |
| `I期临床` | `Ph1` |
| `I/II期临床` | `Ph1/2` |
| `II期临床` | `Ph2` |
| `II/III期临床` | `Ph2/3` |
| `III期临床` | `Ph3` |
| `申请上市` | `NDA Filed` |
| `批准上市` | `Approved` |

### Modality map (Chinese → English)

| Chinese | English |
|---|---|
| `抗体` | `mAb` |
| `双特异性抗体` | `BsAb` |
| `三特异性抗体` | `TsAb` |
| `抗体; 偶联药物` / `抗体偶联药物` | `ADC` |
| `融合蛋白` | `Fusion Protein` |
| `小分子` | `Small Mol.` |
| `多肽` | `Peptide` |
| `核酸` | `siRNA` (or refine to `ASO` / `mRNA`) |
| `细胞疗法` | `Cell Therapy` (refine to `CAR-T` if applicable) |
| `基因疗法` | `Gene Therapy` |
| `放射性药物; 小分子` | `Small Mol.` (note radiopharmaceutical in Note) |
| `化药` (default) | `Small Mol.` |
| `生物` (default) | `mAb` |

### Source column names (Chinese → conceptual)

| Chinese | Concept |
|---|---|
| `数据层级` | Data hierarchy (`交易信息` deal-level, `管线信息` pipeline-level) |
| `交易类型一` | Deal type 1 (e.g., licensing) |
| `交易类型二` | Out-licensing direction (`国内转国外` / `国外转国外`) |
| `转让方` / `转让方所在国家/地区` | Licensor / Licensor HQ |
| `受让方` | Licensee |
| `管线名称` / `靶点` / `作用机制` | Asset name / Target / MoA |
| `美国/中国/欧洲/日本最高研发阶段` | US/CN/EU/JP highest stage |
| `总金额(百万)/美元` / `首付款(百万)/美元` | Total / Upfront ($M) |
| `权益信息` | Rights |
| `疾病` | Indications |
| `药品类别二` / `药品类别三` | Drug class (level 2 / level 3) |
| `药品标签` | Tags (FIC, etc.) |
| `全球研发状态` | Global R&D status (skip if `Inactive`) |
| `交易时间` | Deal date |
