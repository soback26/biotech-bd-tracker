# Biotech BD Tracker — Update Rules

## Project Overview

A **Greater China / Japan / Korea outbound licensing deal tracker** maintained for the R-Bridge (Structured Capital) team at CBC Group. The tracker monitors cross-border BD transactions where GC/JP/KR biotech companies out-license assets to Western pharma/biotech, with a focus on identifying potential **royalty financing (RBF) and structured capital opportunities**.

---

## File Layout

```
biotech-bd-tracker/
├── CLAUDE.md              # This file — update rules
├── README.md              # Project intro (English)
├── raw/
│   └── YYYY-MM-DD.xlsx    # PharmCube exports (date of download)
├── tracker/
│   └── bd_tracker.xlsx    # Master tracker (2 sheets: Actionable, Pipeline)
├── scripts/               # (Reserved for future automation)
└── archive/               # (Reserved for old versions)
```

---

## Data Source

- **Provider**: 医药魔方 (PharmCube) — Chinese pharma intelligence database
- **Export format**: `.xlsx`, single sheet named `检索结果` (search results)
- **Naming convention**: `raw/YYYY-MM-DD.xlsx` (date of download)
- **Row structure** — dual-row pattern per deal:
  - `数据层级 = "交易信息"` → deal-level header (parties, financials, deal type)
  - `数据层级 = "管线信息"` → pipeline-level detail (target, MoA, stage, indications, modality)
  - One deal can have **multiple 管线信息 rows** (multi-asset deals). Each pipeline row becomes a separate row in the tracker.

---

## Tracker Output

**File**: `tracker/bd_tracker.xlsx`

**Two sheets**:
| Sheet | Definition | Current rows |
|---|---|---|
| `Actionable` | Highest global stage ∈ {`Ph3`, `NDA Filed`, `Approved`} — near-term cash flow potential, primary RBF universe | ~35 deals |
| `Pipeline` | Highest global stage ∈ {`IND Filed`, `Ph1`, `Ph1/2`, `Ph2`, `Ph2/3`} — watchlist for future graduation to Actionable | ~140 deals |

**Excluded**: All-`Preclnl`-only assets; deals where `全球研发状态 = Inactive`; domestic-only deals (`交易类型二 = 国内转国内`).

**15 columns (in order, identical across both sheets)**:

| # | Column | Description |
|---|---|---|
| 1 | **Licensor** | Transferring party. Format: `English Name (中文名)` for CN/HK/TW companies, e.g. `Akeso (康方生物)`. JP/KR companies use English only. |
| 2 | **HQ** | Licensor headquarters. Use: `China`, `Japan`, `Korea`, `Taiwan`, `HK`, or combinations like `Japan; UK`. |
| 3 | **Licensee** | Receiving party. Append `(Top20 MNC)` for top-20 global pharma by revenue (see list below). |
| 4 | **Asset / Target / MoA** | Format: `asset_name (target \| MoA_abbreviation)`, e.g. `ivonescimab (VEGF-A; PD1 \| anti-PD1×VEGF-A BsAb)` |
| 5 | **Modality** | Standardized modality (see mapping below) |
| 6 | **US** | US development stage (see stage mapping below) |
| 7 | **CN** | China development stage |
| 8 | **EU** | EU development stage |
| 9 | **JP** | Japan development stage |
| 10 | **Total ($M)** | Total deal value, USD millions. Format: `$X.Xbn` if ≥1000, else `$XXXm`. Use `—` if undisclosed. |
| 11 | **Upfront ($M)** | Upfront payment, USD millions. Same format. `—` if undisclosed. |
| 12 | **Lead Indications** | Top 2-3 indications, semicolon-separated, abbreviated (e.g. `NSCLC`, `TNBC`, `RCC`, `Atopic Dermatitis`). |
| 13 | **Rights** | Territory and rights type. Format: `R&D rights: <terr> \| Mfg rights: <terr> \| Commercial rights: <terr>` or `All rights: <terr>`. See rights mapping. |
| 14 | **Date** | Deal announcement date, `YYYY-MM-DD`. |
| 15 | **Note** | 1-3 sentence English analyst note: FIC/BIC flag, deal structure summary, licensor profile, **RBF relevance assessment**. |

---

## Field Mapping: RAW → Tracker

| Tracker Column | RAW Column(s) | Transform |
|---|---|---|
| Licensor | `转让方` | Add `(中文名)` for CN/TW/HK if raw is Chinese |
| HQ | `转让方所在国家/地区` | `中国`→`China`, `日本`→`Japan`, `韩国`→`Korea`, `台湾`→`Taiwan`, `香港`→`HK` |
| Licensee | `受让方` | Strip any existing `(Top20 MNC)`, then re-add per Top-20 list |
| Asset / Target / MoA | `管线名称` + `靶点` + `作用机制` | Compose: `管线名称 (靶点 \| MoA简称)`. Translate Chinese drug names where possible. Multiple targets → semicolon-separated. |
| Modality | `药品类别三` (fallback `药品类别二`) | See Modality Mapping below |
| US / CN / EU / JP | `美国/中国/欧洲/日本最高研发阶段` | See Stage Mapping below |
| Total ($M) | `总金额(百万)/美元` | `≥1000` → `$X.Xbn`; else `$XXXm`. Blank/None → `—` |
| Upfront ($M) | `首付款(百万)/美元` | Same formatting |
| Lead Indications | `疾病` | First 2-3, translate, abbreviate. Drop generic terms (`Cancer`, `Solid tumors`) when more specific terms exist. |
| Rights | `权益信息` | See Rights Mapping below |
| Date | `交易时间` | Format `YYYY-MM-DD` |
| Note | — | Auto-generate (see Note Generation Rules) |

### Stage Mapping

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
| `None` / blank | Leave blank |

**Stage ordering** (for highest-global-stage comparison):
`Preclnl < IND Filed < Ph1 < Ph1/2 < Ph2 < Ph2/3 < Ph3 < NDA Filed < Approved`

### Modality Mapping

| RAW `药品类别三` | Tracker Modality |
|---|---|
| `抗体` | `mAb` |
| `抗体; 偶联药物` | `ADC` |
| `小分子` | `Small Mol.` |
| `融合蛋白` | `Fusion Protein` |
| `多肽` | `Peptide` |
| `核酸` (siRNA/ASO) | `siRNA` (or `ASO` / `mRNA` if specific) |
| `基因疗法` | `Gene Therapy` |
| `放射性药物; 小分子` | `Small Mol.` (note radiopharmaceutical in Note field) |
| Not available | Infer from `药品类别二`: `化药`→`Small Mol.`, `生物`→`mAb` (default). Refine using `管线类型` and `作用机制`. |

### Rights Mapping

**Rights types**:
| RAW | Tracker |
|---|---|
| `所有权益` | `All rights` |
| `研发权益` | `R&D rights` |
| `商业化权益` | `Commercial rights` |
| `生产权益` | `Mfg rights` |
| `(期权)` suffix | `(opt.)` suffix |

**Territories**:
| RAW | Tracker |
|---|---|
| `全球` | `Global` |
| `美国` | `US` |
| `欧洲` | `EU` |
| `日本` | `JP` |
| `中国(内地)` | `CN` |
| `中国(港澳台)` | `HK/TW` |
| `其他` | `ROW` |

**Format**: Concatenate rights types with ` | ` separator. Examples:
- RAW `研发权益: 全球 | 商业化权益: 全球` → `R&D rights: Global | Commercial rights: Global`
- RAW `所有权益: 美国;欧洲;日本;其他` → `All rights: US;EU;JP;ROW`

### Top-20 MNC List

Append `(Top20 MNC)` to Licensee for any of these:

> Pfizer, Roche, Novartis, Merck & Co., Johnson & Johnson, AbbVie, AstraZeneca, Sanofi, Bristol-Myers Squibb, Eli Lilly, Amgen, GSK, Gilead, Bayer, Novo Nordisk, Takeda, Boehringer Ingelheim, Regeneron, Daiichi Sankyo, Astellas Pharma

---

## Note Generation Rules

Each row's `Note` is a 1-3 sentence English analyst comment covering, in order:

1. **FIC/BIC flag** — Check `药品标签` for `First-in-Class` / `Potential First-in-Class`. If present, lead with `FIC.` or `Potential FIC.`
2. **Deal structure summary** — What rights, to whom, in which territories. Example: `Ex-CN R&D + commercial to Summit.`
3. **Development status highlight** — Key stage milestones. Example: `US NDA filed, CN approved.`
4. **RBF relevance** — Royalty financing opportunity assessment for the **licensor**:
   - Small/mid-cap with capital needs → `may consider royalty monetization`
   - Large self-funded pharma (Daiichi Sankyo, Takeda, Astellas, Chugai/Roche) → `self-funded — no RBF need`
   - Near-term revenue stream → `Near-term US revenue potential`
   - Too early-stage or too small → `Limited royalty scale`

**Tone**: Terse, investment-memo style. No filler. Semicolons separate thoughts. Standard abbreviations: CN, US, EU, JP, MNC, RBF, FIC, BIC, Ph3, NDA.

---

## Deduplication Rules

Before appending new rows, match against existing entries on:

**Primary key**: `Licensor` + `Licensee` + asset name (the part before the first `(` in `Asset / Target / MoA`)

| Match type | Action |
|---|---|
| **Exact match** | **Update** stage columns (US/CN/EU/JP) if new data shows advancement. Update financials only if previously `—` and now disclosed. **Do NOT overwrite** existing `Note` unless empty. |
| **No match** | **Append** as new row, prefix Note with `[NEW]` for review. |
| **Ambiguous match** (same parties, different asset variant) | Append new row, prefix Note with `[CHECK]` for manual review. |

---

## Update Workflow

When asked to process a new raw file:

1. **Read** new raw file from `raw/YYYY-MM-DD.xlsx`, sheet `检索结果`
2. **Parse** dual-row structure: pair each `交易信息` row with its subsequent `管线信息` rows
3. **Filter**:
   - `交易类型二` ∈ {`国内转国外`, `国外转国外`} (skip pure domestic deals)
   - `转让方所在国家/地区` ∈ {`中国`, `日本`, `韩国`, `台湾`, `香港`}
   - `全球研发状态` ≠ `Inactive`
4. **Map** all 15 fields per the mapping tables above
5. **Auto-generate** Note per Note Generation Rules
6. **Determine sheet**: highest-global-stage logic → `Actionable` or `Pipeline`. Drop if Preclnl-only.
7. **Deduplicate** against existing tracker entries
8. **Sort** each sheet by `Date` descending, then `Licensor` alphabetically
9. **Save** to `tracker/bd_tracker.xlsx`
10. **Commit**: `update: YYYY-MM-DD | +X new deals | <summary of notable deals>`

---

## Important Context

- This tracker serves **CBC Group's R-Bridge team** for royalty deal sourcing.
- Key lens: **which licensors might benefit from royalty monetization**, synthetic royalties, or milestone receivable financing?
- **Large self-funded JP pharma** (Takeda, Daiichi Sankyo, Astellas, Chugai/Roche) are generally **NOT RBF targets** — they don't need balance-sheet capital.
- **Small/mid-cap CN, TW, KR biotech** with multiple out-licensing deals are higher-priority targets — they typically have multiple cash-burning programs to fund.
- Pay special attention to deals with **disclosed financials** (upfront + milestones + royalties), as these define the cash flow structure available for monetization.
