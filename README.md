# Reproducibility Guide

**Paper**: *Waste heat recovery can alleviate environmental inequity embodied in China's large-scale computing power migration* (submitted to *Energy Economics*)

This folder (`实验过程/`) contains all **reproducible Python scripts** for the statistical analysis (Methods & Results) of the paper. The **upstream data pipeline** (computing power → energy → waste heat → carbon emissions) is implemented in `../数据处理代码/waste_heat_recovery.ipynb`; its **core intermediate outputs are consolidated into a single Excel workbook** (`../数据汇总/前端数据_可复现输入.xlsx`), which serves as the one-stop data source for the statistical scripts below.

---

## 1. Execution order (top-down; must follow this sequence)

```
[Upstream data pipeline]  ../数据处理代码/waste_heat_recovery.ipynb
        │  produces intermediate tables under ../数据汇总/ (computing power scale /
        │  energy / heat / carbon transfer, three-path abatement details, etc.),
        │  consolidated into ../数据汇总/前端数据_可复现输入.xlsx
        ▼
[Data pre-processing]  01_fix_ef_highspeed.py  → unify Beijing/Shanghai/Guangzhou/Shenzhen grid EF to the high-speed-scenario lower bound
                       02_fill_pop.py           → fill historical population of 13 cities
                       03_fix_gdp.py            → fix historical GDP of 13 cities
                       04_forecast_2035.py      → project population/GDP 2005–2035 → ../13城市人口GDP预测_2005-2035.xlsx
        ▼
[Core statistics]     05_gini_theil_calc.py    → Gini/Theil three-layer framework → ../碳不平等测算_Gini_Theil.xlsx ★CORE
                       07_DCEA.py               → DCEA concentration index + abatement cost → DCEA_results.xlsx + Fig.7*
                       08_compensation.py       → cross-regional compensation → compensation_results.xlsx + Fig.9/10
                       09_sensitivity.py        → Monte Carlo sensitivity → sensitivity_results.xlsx + Fig11 (=正文 Fig.8)
                       10_robust_inequality.py  → CV/RMD/Atkinson robust indices → robust_inequality_indices.xlsx (Table 1)
        ▼
[Figures]             06_visualization.py      → transitional Fig1–6 (single panels)
                       11_fig5a_2035_burden.py  → Fig.5a (final)
                       12_fig5_combined.py      → Fig.5 (2×2 combined)
                       13_fig7_combined.py      → Fig.7 (2×2 combined)
```

> Scripts 07/08/10 read only the output of 05 (`../碳不平等测算_Gini_Theil.xlsx`) and **must run after 05**; 09 additionally reads the incremental-transfer link table, the total-load table, and the EF table directly. The publication-quality versions of Fig.1–4, Fig.6 and Fig.9/10 are drawn in **Origin** (see `../图源文件/*.opju`).

**One-click entry point `run_all.py`** (same folder):
```
python run_all.py            # rerun 05→07→08→09→10 in order, then print verification anchors
python run_all.py 07 09      # run selected scripts only (07/09 depend on 05; ensure 05 has run)
python run_all.py --check    # verify existing results only (read xlsx and print anchors; no rerun)
```
Verified: all 5 scripts pass (≈28 s total); anchors match the paper exactly (2035 Gini = 0.9036, net = 0.9336, mitigation −3.32%, between-group share 60.5%).

### 1.1 Front-end data workbook structure (`../数据汇总/前端数据_可复现输入.xlsx`)

The consolidated data source for the statistical scripts; all sheet names are in English:

| Sheet | Content | Shape | Used by |
|---|---|---|---|
| `01_Total_Carbon_Transfer_S1S2` | 9 receiving cities × 10 yr; S1/S2 total transferred carbon (burden side) | 90 × 5 | 05 |
| `02_ThreePath_Abatement_S1` | S1 waste-heat three-path (heating/cooling/power) abatement, Plan 1/2/3 | 90 × 19 | 05 |
| `03_ThreePath_Abatement_S2` | same as 02, S2 scenario | 90 × 19 | 05 |
| `04_Incremental_Transfer_S1` | Source→Target link-level computing power/energy/heat/carbon (S1 unified caliber) | 210 × 9 | 05, 09 |
| `05_Incremental_Transfer_S2` | same as 04, S2 final (incl. training/inference Type split) | 270 × 9 | 05 |
| `06_Total_Load_S1` | 9 cities × 10 yr; stock + incremental total load/energy/heat/carbon | 90 × 13 | 09 |
| `07_NineCity_EF` | 9 cities × 10 yr grid emission factor (linear interpolation) | 130 × 4 | 05, 09 |
| `08a_Population_Forecast` | 13 cities 2005–2035 population projection (von Bertalanffy fit) | 31 × 14 | 05, 08 |
| `08b_GDP_MidScenario` | 13 cities 2005–2035 mid-scenario GDP projection (growth-rate convergence) | 31 × 14 | 05, 08 |
| `Data_Dictionary` | sheet-by-sheet glossary | — | readers |

---

## 2. Script inventory: function / input / output / paper section

| Script | Function | Input (relative to `../数据汇总/`, unless noted) | Output | Paper section |
|---|---|---|---|---|
| `01_fix_ef_highspeed.py` | Unify the grid EF of Beijing/Shanghai/Guangzhou/Shenzhen to the "high-speed development scenario" lower bound, consistent with the 9 receiving cities | `NineCity_EF_LinearInterp.csv` (overwritten in place) | same CSV | Methods 2.3 |
| `02_fill_pop.py` | Fill historical population of 13 cities (marked in yellow) | `298city_pop_density_1998-2024.xlsx` | `13城市人口GDP历史数据收集清单.xlsx` | Methods 2.1 |
| `03_fix_gdp.py` | Fix historical GDP of 13 cities (5th economic census series) | `2000-2024地级市GDP相关指标.xlsx` | same checklist | Methods 2.1 |
| `04_forecast_2035.py` | Population (von Bertalanffy fit) + GDP (nominal growth-rate convergence) projection | population/GDP history | `13城市人口GDP预测_2005-2035.xlsx` | Methods 2.1 |
| `05_gini_theil_calc.py` ★ | **Core**: 9-city burden–benefit–mitigation three-layer framework; population-weighted Gini / Theil (between coal-heavy vs clean grid) + Plan 1/3 sensitivity + 13-city net burden | `情景一+情景二_总转移碳排放.xlsx`; `情景一/二_年份+城市_三路径碳减排量_明细总表.xlsx`; `../13城市人口GDP预测_2005-2035.xlsx`; `NineCity_EF_LinearInterp.csv`; `情景一_增量转移_..._统一口径版.csv`; `情景二_增量转移_..._最终版.csv` | `../碳不平等测算_Gini_Theil.xlsx` | 3.3, Fig.5, Table 1 |
| `07_DCEA.py` | Population-weighted DCEA concentration index (pro-rich / pro-poor) + abatement-cost table (three cases) | `../碳不平等测算_Gini_Theil.xlsx` | `DCEA_results.xlsx`, Fig7 concentration curve / Fig8 net benefit (early) | 3.4, Fig.7 |
| `08_compensation.py` | Cross-regional compensation: 4 source cities contribute by GDP; positive-gap receiving cities receive; carbon price 50/70/100 | `../碳不平等测算_Gini_Theil.xlsx`, `../13城市人口GDP预测_2005-2035.xlsx` | `compensation_results.xlsx`, Fig9/Fig10 | 3.5 |
| `09_sensitivity.py` | Monte Carlo sensitivity: independent from-source perturbation of transfer volume ±20/±50% × EF bounds × combined (seed 42) | `情景一_增量转移_..._统一口径版.csv`, `情景一_存量+增量_总负荷_总表.csv`, `NineCity_EF_LinearInterp.csv`, `../碳不平等测算_Gini_Theil.xlsx` | `sensitivity_results.xlsx`, Fig11_sensitivity_distributions.png (=正文 Fig.8) | 3.6, Fig.8 |
| `10_robust_inequality.py` | CV / RMD (handle negatives natively) + Atkinson (ε=0.5/1.0, zero-floored), population-weighted | `../碳不平等测算_Gini_Theil.xlsx` | `robust_inequality_indices.xlsx` | 3.4-end, Table 1 |
| `06_visualization.py` | Transitional Fig1–6 (single panels) | `../碳不平等测算_Gini_Theil.xlsx` | Fig1–6 png | transitional (superseded by 12/13) |
| `11_fig5a_2035_burden.py` | Fig.5a (2035 TAS, city labels vertical 90° black, coal-heavy red / clean blue) | `../碳不平等测算_Gini_Theil.xlsx` | Fig5a_percap_burden_2035.png | Fig.5a |
| `12_fig5_combined.py` | Fig.5 combined (a burden / b Gini series / c Lorenz / d Theil; no subplot titles; c uses colored city markers + legend) | `../碳不平等测算_Gini_Theil.xlsx` | Fig5_inequality_diagnosis.png | Fig.5 |
| `13_fig7_combined.py` | Fig.7 combined (a DCEA concentration curves: cost red / benefit blue; b per-capita net welfare @70 yuan/t by city & income group) | `../碳不平等测算_Gini_Theil.xlsx` | Fig7_distributional_welfare.png | Fig.7 |

---

## 3. Core reproducible results (verification anchors; do not change the caliber)

After running 05 (deterministic output), TAS scenario, Plan 2 (medium recovery rate):

| Year | Gini (burden, no recovery) | Gini (net, zero-floored) | Mitigation (zero-floored, %) | Theil total | Between (coal vs clean) | Between share (%) |
|---|---|---|---|---|---|---|
| 2026 | 0.9067 | 0.9178 | −1.22 | 2.1382 | 1.0997 | 51.4 |
| 2030 | 0.9094 | 0.9374 | −3.08 | 2.1827 | 1.1911 | 54.6 |
| 2035 | 0.9036 | 0.9336 | −3.32 | 2.1112 | 1.2777 | 60.5 |

> The paper reports: **Gini(burden) = 0.904, Gini(net, zero-floored) = 0.934 (mitigation −3.32%)**, 2035 between-group share 60.5%. A rerun should reproduce these numbers exactly.

Other anchors (from 07/08/09):
- Per-capita net welfare @70 yuan/t: Zhongwei −190.7, Hohhot −110.1 (yuan/cap); income groups Low −29.63 / Mid +3.79 / High −13.94 (yuan/cap).
- Compensation pool: 693 million RMB @70 (495 @50, 989 @100); Hohhot 63.2% + Zhongwei 30.0% = 93.2%; contributions Shanghai 31.9% / Beijing 28.8% / Shenzhen 22.9% / Guangzhou 16.4%.
- Sensitivity verdicts C1–C5 all PASS: Gini > 0.7, mitigation < 0, loss center = Hohhot (total) / Zhongwei (per-capita), positive total gap, C_benefit < 0 (pro-poor).
- Abatement cost: gross ≈276 yuan/tCO₂ → net ≈−162 yuan/tCO₂ after heating revenue.

---

## 4. Dependencies (requirements.txt)

- python ≥ 3.9; pandas, numpy, scipy, matplotlib, openpyxl.
- No networked / paid dependencies; the scripts are **deterministic** (09 uses a fixed seed `np.random.default_rng(42)`, fully reproducible).
- Run: `python 05_gini_theil_calc.py` (core); scripts 07/08/09/10 depend on the output of 05 — always run 05 first.

---

## 5. Terminology and caliber (consistency constraints)

- Use the full term **waste heat recovery** throughout (no WH / WHR abbreviations).
- Scenarios: BS = S1 baseline; TAS = S2 computing-power-migration. Recovery plans 1 low / 2 medium / 3 high (main result = Plan 2).
- Grouping: coal-heavy grid (Hohhot, Zhangjiakou, Wuhu, Qingyang, Zhongwei; 2026 EF ≥ 0.4) vs clean grid (Chengdu, Chongqing, Guiyang, Shaoguan).
- Index caliber: **population-weighted** Gini / Theil / DCEA (resident-level inequality); dual caliber reported — raw (including negatives) vs zero-floored (`max(net, 0)`); zero-floored isolates the true structural effect (mechanism A).
- Accounting boundary: a negative net burden means that, in accounting terms, recovery abatement exceeds the migration-attributed carbon; it does not imply physical negative emissions (must be stated in the paper).
- The 13-city net-burden table uses the incremental-transfer CSV caliber (different from the 9-city total caliber; for showing source–receiving directionality only).

---

## 6. Data sources (external inputs; citations to be completed)

The following inputs require formal sources in the paper (Methods 2.1 / 3.2.2; enter into EndNote):
- Resident population: municipal statistical yearbooks / 7th census; GDP: municipal statistical yearbooks / Bureaus of Statistics.
- Grid emission factors: China Carbon Emission Accounting platform / China Academy of Environmental Planning (2023) projection.
- Carbon price (50–100/70): National Carbon Emissions Trading Market (CEA).
- Residential income Gini (0.4–0.5): National Bureau of Statistics / World Bank.
- Abatement-cost cases: ① Lanzhou Municipal Bureau of Industry & IT (Gansu Telecom Hub-2, 665.69 t); ② CCTV News (China Mobile Hohhot, 5500 t); ③ China Heat Pump Association 47-project compilation (1 MW, 3.8–5.5 million yuan, 48,000 GJ) — the source for ③ is to be provided by the authors; **not fabricated**.
