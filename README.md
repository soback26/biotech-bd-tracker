# Biotech BD Tracker

> **Cross-border out-licensing deal tracker for Greater China, Japan, and Korea biotech.**
> A sourcing engine for royalty financing and structured capital opportunities.

---

## Table of Contents

- [Purpose](#purpose)
- [Coverage Snapshot](#coverage-snapshot)
- [Repository Layout](#repository-layout)
- [Tracker Schema](#tracker-schema)
- [Sheet Split Logic](#sheet-split-logic)
- [Update Workflow](#update-workflow)
- [Data Source](#data-source)
- [Notes](#notes)

---

## Purpose

The pharmaceutical out-licensing market in Greater China, Japan, and Korea has become one of the fastest-growing pipelines of cross-border BD activity globally. Deals like **Innovent–Takeda IBI363** ($11.4bn), **DualityBio–BioNTech HER2 ADC** ($1.7bn), and **Akeso–Summit ivonescimab** have reset benchmarks for what GC/JP/KR assets can command in ex-Asia markets.

This tracker exists to systematically monitor that flow and answer one core question:

> **Which licensors might benefit from royalty monetization, synthetic royalties, or milestone receivable financing — and which deals are large enough, late enough, and structured cleanly enough to support it?**

Every deal in this database is read through the lens of a **structured capital provider**. The 15-column schema is deliberately built to surface the inputs an underwriter needs at a glance: who got which rights in which territories, how late-stage is the asset, what's the disclosed cash flow (upfront / total), and is the licensor a credible counterparty for a structured deal.

---

## Coverage Snapshot

As of the most recent update:

| Metric | Actionable | Pipeline |
|---|---|---|
| **Definition** | Highest global stage ≥ Ph3 | IND Filed through Ph2/3 |
| **Deals tracked** | 35 | 140 |
| **Date range** | 2020 – 2026 | 2020 – 2026 |
| **Primary use case** | Near-term underwriting universe | Watchlist for graduation to Actionable |

**Geographic mix** (Pipeline + Actionable combined, ~175 deals):

| Licensor HQ | Deals |
|---|---|
| China | 120 |
| Japan | 30 |
| Korea | 21 |
| Taiwan | 2 |
| Hong Kong | 1 |
| Japan; UK | 1 |

**Modality mix** (Pipeline sheet, n=140):

| Modality | Deals |
|---|---|
| Small Molecule | 50 |
| ADC | 37 |
| Monoclonal Antibody | 32 |
| siRNA | 5 |
| Peptide | 5 |
| Cell Therapy | 5 |
| Other | 6 |

---

## Repository Layout

```
biotech-bd-tracker/
├── README.md              # This file — project overview
├── CLAUDE.md              # Detailed update rules and field mappings
├── raw/
│   └── YYYY-MM-DD.xlsx    # Raw exports from PharmCube (one per refresh)
├── tracker/
│   └── bd_tracker.xlsx    # Master tracker — 2 sheets, 15 columns
├── scripts/               # (Reserved for future automation)
└── archive/               # (Reserved for archived versions)
```

---

## Tracker Schema

Both sheets (`Actionable` and `Pipeline`) share the same **15-column schema**:

| # | Column | What it captures |
|---|---|---|
| 1 | **Licensor** | Transferring party. CN/HK/TW shown as `English (中文)`; JP/KR English-only. |
| 2 | **HQ** | Country of licensor headquarters. |
| 3 | **Licensee** | Receiving party. Top-20 global pharma flagged with `(Top20 MNC)`. |
| 4 | **Asset / Target / MoA** | Drug name, target(s), and mechanism abbreviation, e.g. `ivonescimab (VEGF-A; PD1 \| anti-PD1×VEGF-A BsAb)`. |
| 5 | **Modality** | Standardized: `mAb`, `ADC`, `Small Mol.`, `Fusion Protein`, `Peptide`, `siRNA`, `Gene Therapy`, etc. |
| 6–9 | **US / CN / EU / JP** | Highest development stage in each region: `Preclnl`, `IND Filed`, `Ph1`, `Ph1/2`, `Ph2`, `Ph2/3`, `Ph3`, `NDA Filed`, `Approved`. |
| 10 | **Total ($M)** | Total announced deal value in USD. Format: `$X.Xbn` or `$XXXm`. `—` if undisclosed. |
| 11 | **Upfront ($M)** | Upfront payment in USD, same format. |
| 12 | **Lead Indications** | Top 2-3 indications, semicolon-separated and abbreviated. |
| 13 | **Rights** | Rights structure: `R&D rights: <terr> \| Mfg rights: <terr> \| Commercial rights: <terr>` (or `All rights: <terr>`). |
| 14 | **Date** | Announcement date, `YYYY-MM-DD`. |
| 15 | **Note** | 1-3 sentence analyst comment: FIC/BIC flag, deal structure, key milestones, and counterparty/financing relevance. |

> Full mapping rules (RAW Chinese → English tracker columns), modality and rights vocabularies, the Top-20 MNC list, and Note-generation conventions are documented in [`CLAUDE.md`](./CLAUDE.md).

### Sample row (`Actionable` sheet)

| Field | Value |
|---|---|
| Licensor | Innovent Biologics (信达生物) |
| HQ | China |
| Licensee | Takeda Pharmaceuticals |
| Asset / Target / MoA | IBI363 (IL-2; PD1 \| IL-Ab-fusion) |
| Modality | Fusion Protein |
| US / CN / EU / JP | Ph3 / Ph3 / Preclnl / Preclnl |
| Total ($M) | $11.4bn |
| Upfront ($M) | $1.2bn |
| Lead Indications | sq NSCLC; Melanoma; Pancreatic Ca |
| Rights | R&D rights: Global \| Mfg rights: US;EU;JP;ROW \| Commercial rights: US;EU;JP;ROW |
| Date | 2025-10-21 |
| Note | Potential FIC. Global R&D + ex-CN commercial rights to Takeda. IL-2×PD1 fusion protein, $11.4bn mega-deal with $1.2bn upfront, Takeda's largest BD deal ever. Innovent retains CN rights but faces heavy R&D burn across broad pipeline. Future royalty stream could be very large if approved. |

---

## Sheet Split Logic

Each deal-asset row is assigned to exactly one sheet based on the **highest global stage** across US, CN, EU, and JP:

| Highest Global Stage | Sheet |
|---|---|
| `Approved`, `NDA Filed`, `Ph3` | **`Actionable`** — assets within near-term underwriting range |
| `Ph2/3`, `Ph2`, `Ph1/2`, `Ph1`, `IND Filed` | **`Pipeline`** — watchlist; promote to `Actionable` upon stage advancement |
| `Preclnl` only | **Excluded** — too early to underwrite |
| `全球研发状态 = Inactive` | **Excluded** — programs no longer being developed |

**Stage ordering** (used for the highest-stage comparison):

```
Preclnl < IND Filed < Ph1 < Ph1/2 < Ph2 < Ph2/3 < Ph3 < NDA Filed < Approved
```

---

## Update Workflow

The tracker is updated **weekly with an append-only incremental pass**. Each week, a pre-filtered English deal list containing only net-new transactions is dropped into `raw/`, then appended to the appropriate sheet.

1. **Drop** the new file into `raw/` named as `YYYY-MM-DD.xlsx` (the export date). The file should already be **scoped to net-new deals only**, in English, and limited to in-scope out-licensing transactions from Greater China / Japan / Korea licensors.
2. **Map** the 15 tracker columns from each raw row per the rules in `CLAUDE.md` (stage vocabulary, modality vocabulary, MoA abbreviations, rights vocabulary, disease abbreviations, Top-20 MNC tagging, etc.).
3. **Compose** the `Licensor` column as `English (中文)` for CN/HK/TW companies using `company_names.md`; JP/KR licensors use English only.
4. **Auto-generate** the Note field with FIC/BIC flag, deal structure summary, stage highlight, and counterparty/financing relevance.
5. **Assign** each row to `Actionable` or `Pipeline` per the split logic above (drop Preclnl-only rows).
6. **Append** to the target sheet. A light sanity check flags any row whose `(Licensor, Licensee, asset name)` triple already exists in the tracker — surfaced as a warning for user review, not automatically deduped.
7. **Re-sort** each affected sheet by `Date` descending, then `Licensor` alphabetically.
8. **Save** to `tracker/bd_tracker.xlsx` preserving Bloomberg-style formatting, then commit with the message format:
   ```
   update: YYYY-MM-DD | +X new deals | <summary of notable deals>
   ```

---

## Data Source

Raw deal data is primarily sourced from **医药魔方 (PharmCube)**, a Chinese pharma intelligence database widely used by domestic biotech investors and BD professionals. PharmCube was selected over English-language alternatives (Cortellis, Evaluate Pharma, BioCentury) because GC/JP/KR domestic deals — especially smaller out-licensing transactions involving private CN biotech — are systematically better captured in Chinese-language sources.

**Incremental weekly input format**: the standard weekly drop into `raw/` is an **English-language, net-new-only** `.xlsx` file. The file is pre-filtered by the analyst to exclude domestic-only deals, Preclnl-only assets, and transactions already reflected in the master tracker. This keeps the append-only workflow simple and avoids the schema/dedup complications of full-refresh dumps. Legacy full-dump Chinese exports (dual-row structure with deal-level + pipeline-level rows) remain supported via the fallback rules in `CLAUDE.md`.

---

## Notes

- This is a **private working repository**. Tracker contents reflect curated analyst judgment and are not for external distribution.
- All deal data is sourced from public announcements; **no MNPI** is captured here.
- Counterparty notes and financing-relevance assessments are **internal opinions** for sourcing purposes only and do not constitute investment recommendations.
- Top-20 MNC list, modality vocabulary, and sourcing criteria are subject to periodic revision — see `CLAUDE.md` for the current canonical version.
