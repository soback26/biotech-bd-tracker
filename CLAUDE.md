# Biotech BD Tracker — Update Rules

## Project Overview

This repo maintains a **Greater China / Japan / Korea outbound licensing deal tracker** for the R-Bridge (Structured Capital) team at CBC Group. The tracker monitors cross-border BD transactions where GC/JP/KR biotech companies out-license assets to Western pharma/biotech, with a focus on identifying potential royalty financing and structured capital opportunities.

---

## Data Source

- **Raw data**: Exported from **医药魔方 (PharmCube)** as `.xlsx` files, placed in `raw/` directory
- **Naming convention**: `raw/YYYY-MM-DD.xlsx` (date of download)
- **Sheet name**: `RAW_Ph3` (may vary — always check sheet names on import)
- **Row structure**: Raw data uses a **dual-row pattern per deal**:
  - `数据层级 = "交易信息"` → deal-level header row (deal name, parties, financials, deal type)
  - `数据层级 = "管线信息"` → pipeline-level detail rows (target, MoA, stage, indications, modality)
  - One deal can have **multiple 管线信息 rows** (multi-asset deals). Each pipeline row becomes a separate row in the tracker.

---

## Tracker Output

- **File**: `tracker/GC_JP_KR_BD_Tracker.xlsx`
- **Two sheets**:
  - `Ph3+` — Assets with **any region at Ph3 or beyond** (Ph3, NDA Filed, Approved) at time of update
  - `IND+` — Assets at **IND Filed through Ph2/3** (IND Filed, Ph1, Ph1/2, Ph2, Ph2/3) in highest global stage
- **15 columns** (in order):

| # | Column | Description |
|---|--------|-------------|
| 1 | **Licensor** | Transferring party. Format: `English Name (中文名)` if Chinese company, e.g. `Akeso (康方生物)`. JP/KR companies use English only. |
| 2 | **HQ** | Licensor headquarters. Use: `China`, `Japan`, `Korea`, `Taiwan`, `HK`, or combinations like `Japan; UK` |
| 3 | **Licensee** | Receiving party. Append `(Top20 MNC)` for top-20 global pharma by revenue. |
| 4 | **Asset / Target / MoA** | Format: `asset_name (target | MoA_abbreviation)`, e.g. `ivonescimab (VEGF-A; PD1 \| anti-PD1×VEGF-A BsAb)` |
| 5 | **Modality** | Standardized modality (see mapping below) |
| 6 | **US** | US development stage (see stage mapping below) |
| 7 | **CN** | China development stage |
| 8 | **EU** | EU development stage |
| 9 | **JP** | Japan development stage |
| 10 | **Total ($M)** | Total deal value in USD millions. Format: `$X.Xbn` or `$XXXm`. Use `—` if undisclosed. |
| 11 | **Upfront ($M)** | Upfront payment in USD millions. Same format. Use `—` if undisclosed. |
| 12 | **Lead Indications** | Top 2-3 indications, semicolon-separated, abbreviated. Use standard oncology/disease abbreviations (e.g. `NSCLC`, `TNBC`, `RCC`, `Atopic Dermatitis`). |
| 13 | **Rights** | Territory and rights type. Format: `R&D rights: US;EU;JP;ROW \| Commercial rights: US;EU;JP;ROW` or `All rights: Global`. See rights mapping below. |
| 14 | **Date** | Deal announcement date. Format: `YYYY-MM-DD` |
| 15 | **Note** | 1-3 sentence analyst note in English covering: FIC/BIC status, deal structure summary, licensor profile, and **RBF relevance assessment** (e.g. capital needs, royalty monetization angle). |

---

## Field Mapping: RAW → Tracker

### Core Fields

| Tracker Column | RAW Column(s) | Transform |
|---|---|---|
| Licensor | `转让方` | Add `(中文名)` for CN/TW/HK companies if raw name is Chinese |
| HQ | `转让方所在国家/地区` | Map: `中国`→`China`, `日本`→`Japan`, `韩国`→`Korea`, `台湾`→`Taiwan`, `香港`→`HK` |
| Licensee | `受让方` | Strip `(Top20 MNC)` if present in raw, then re-add per the Top-20 MNC list |
| Asset / Target / MoA | `管线名称` + `靶点` + `作用机制` | Compose: `管线名称 (靶点 \| MoA简称)`. Use English names; translate Chinese drug names if possible. Separate multiple targets with `;`. |
| Modality | `药品类别三` + `药品类别二` | See Modality Mapping below |
| US | `美国最高研发阶段` | See Stage Mapping below |
| CN | `中国最高研发阶段` | See Stage Mapping below |
| EU | `欧洲最高研发阶段` | See Stage Mapping below |
| JP | `日本最高研发阶段` | See Stage Mapping below |
| Total ($M) | `总金额(百万)/美元` | Format to `$X.Xbn` if ≥1000, else `$XXXm`. `None`/blank → `—` |
| Upfront ($M) | `首付款(百万)/美元` | Same formatting. `None`/blank → `—` |
| Lead Indications | `疾病` | Take first 2-3 diseases, translate to English, abbreviate. Drop overly generic terms like `Cancer` or `Solid tumors` if more specific terms are available. |
| Rights | `权益信息` | See Rights Mapping below |
| Date | `交易时间` | Format as `YYYY-MM-DD` |
| Note | — | **Auto-generate** from context (see Note Generation below) |

### Stage Mapping (Chinese → English)

| RAW (Chinese) | Tracker (English) |
|---|---|
| `临床前` | `Preclnl` |
| `申报临床` | `IND Filed` |
| `I期临床` | `Ph1` |
| `I/II期临床` | `Ph1/2` |
| `II期临床` | `Ph2` |
| `II/III期临床` | `Ph2/3` |
| `III期临床` | `Ph3` |
| `申请上市` | `NDA Filed` |
| `批准上市` | `Approved` |
| `None` / blank | Leave blank or omit |

### Modality Mapping

| RAW `药品类别三` | Tracker Modality |
|---|---|
| `抗体` | `mAb` |
| `抗体; 偶联药物` | `ADC` |
| `小分子` | `Small Mol.` |
| `融合蛋白` | `Fusion Protein` |
| `多肽` | `Peptide` |
| `核酸` (siRNA/ASO context) | `siRNA` (or specify if ASO/mRNA) |
| `基因疗法` | `Gene Therapy` |
| `放射性药物; 小分子` | `Small Mol.` (note: radiopharmaceutical) |
| Not available | Infer from `药品类别二`: `化药`→`Small Mol.`, `生物`→`mAb` (default). Check `管线类型` and `作用机制` for refinement. |

### Rights Mapping (Chinese → English)

| RAW Chinese | Tracker English |
|---|---|
| `所有权益` | `All rights` |
| `研发权益` | `R&D rights` |
| `商业化权益` | `Commercial rights` |
| `生产权益` | `Manufacturing rights` |
| `(期权)` suffix | `(opt.)` suffix |
| Territory: `全球` | `Global` |
| Territory: `美国` | `US` |
| Territory: `欧洲` | `EU` |
| Territory: `日本` | `JP` |
| Territory: `中国(内地)` | `CN` |
| Territory: `中国(港澳台)` | `HK/TW` |
| Territory: `其他` | `ROW` |

**Format**: Concatenate with `|` separator. Example:
- RAW: `研发权益: 全球 | 商业化权益: 全球`
- Tracker: `R&D rights: Global | Commercial rights: Global`

### Top-20 MNC List (append `(Top20 MNC)` to Licensee)

Pfizer, Roche, Novartis, Merck & Co., Johnson & Johnson, AbbVie, AstraZeneca, Sanofi, Bristol-Myers Squibb, Eli Lilly, Amgen, GSK, Gilead, Bayer, Novo Nordisk, Takeda, Boehringer Ingelheim, Regeneron, Daiichi Sankyo, Astellas Pharma

---

## Note Generation Rules

Each row's `Note` field should be a concise 1-3 sentence English analyst comment covering:

1. **FIC/BIC flag**: Check `药品标签` for `First-in-Class` or `Potential First-in-Class`. If present, start with `Potential FIC.` or `FIC.`
2. **Deal structure summary**: What rights were transferred, to whom, in which territories. Example: `Ex-CN R&D + commercial to Summit.`
3. **Development status highlight**: Mention key stage milestones. Example: `US NDA filed, CN approved.`
4. **RBF relevance**: Brief assessment of royalty financing opportunity for the **licensor**:
   - Is the licensor a small/mid-cap that may need capital? → `may consider royalty monetization`
   - Is the licensor a large self-funded pharma (e.g. Daiichi Sankyo, Takeda, Astellas)? → `self-funded — no RBF need`
   - Is there a near-term revenue stream that could back a royalty deal? → `Near-term US revenue potential`
   - Is the deal too early-stage or too small? → `Limited royalty scale`

**Tone**: Terse, investment-memo style. No filler. Semicolons to separate thoughts. Use abbreviations (CN, US, EU, JP, MNC, RBF, FIC, BIC, Ph3, NDA).

---

## Sheet Assignment Logic

After mapping, assign each row to the correct sheet:

- **`Ph3+` sheet**: The asset's **highest global stage** (max of US, CN, EU, JP) is `Ph3`, `NDA Filed`, or `Approved`
- **`IND+` sheet**: The asset's **highest global stage** is `IND Filed`, `Ph1`, `Ph1/2`, `Ph2`, or `Ph2/3`
- **Exclude**: Assets where all stages are blank or `Preclnl` only (no clinical development)
- **Exclude**: Deals where `全球研发状态` = `Inactive`

Stage ordering for comparison: `Preclnl < IND Filed < Ph1 < Ph1/2 < Ph2 < Ph2/3 < Ph3 < NDA Filed < Approved`

---

## Deduplication Rules

Before appending new rows, check for existing entries by matching on:

**Primary key**: `Licensor` + `Licensee` + `Asset name` (first part of Asset / Target / MoA before the parenthesis)

- **Exact match found**: **Update** the existing row's stage columns (US/CN/EU/JP) if the new data shows advancement. Update financials only if previously `—` and now disclosed. Do NOT overwrite existing `Note` unless the row had no note.
- **No match**: **Append** as new row, tagged with `[NEW]` prefix in the Note field for easy identification during review.
- **Ambiguous match** (e.g. same licensor/licensee but different asset variant): Append as new row but add `[CHECK]` tag in Note for manual review.

---

## Update Workflow

When asked to process a new raw data file:

1. **Read** the new raw file from `raw/` directory
2. **Parse** dual-row structure: pair each `交易信息` row with its subsequent `管线信息` rows
3. **Filter**: Only process rows where `交易类型二` = `国内转国外` or `国外转国外` (skip domestic deals)
4. **Filter**: Only process rows where `转让方所在国家/地区` ∈ {`中国`, `日本`, `韩国`, `台湾`, `香港`}
5. **Map** fields per the mapping tables above
6. **Deduplicate** against existing tracker entries
7. **Assign** to `Ph3+` or `IND+` sheet
8. **Sort** each sheet by `Date` descending (newest first), then by `Licensor` alphabetically
9. **Save** updated tracker to `tracker/GC_JP_KR_BD_Tracker.xlsx`
10. **Commit** with message format: `update: YYYY-MM-DD | +X new deals | [summary of notable deals]`

---

## Important Context

- This tracker serves CBC Group's R-Bridge team for **royalty deal sourcing**
- Key lens: Which licensors might benefit from royalty monetization, synthetic royalties, or milestone receivable financing?
- Large self-funded JP pharma (Takeda, Daiichi Sankyo, Astellas, Chugai/Roche) are generally NOT RBF targets
- Small/mid-cap CN, TW, KR biotech with multiple out-licensing deals are higher-priority targets
- Pay special attention to deals with disclosed financials (upfront + milestones + royalties) as these define the cash flow structure
