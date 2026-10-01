# Rental Crisis in Portugal

**Pastel de Data, S.A.** · Ironhack Data Analytics Bootcamp · Module 1 project (Data Wrangling)
**Team:** Andrea Carlo Cerri (H1) · Juliana Therezo (H2, H3)

> Fictional client brief: the TV channel **SIC** asked us to test three popular claims about Portugal's rental crisis using **official data only**.

**Presentation:** [Rental Crisis in Portugal – slides](https://docs.google.com/presentation/d/141kplpqkM6aTQ9PDCH6gl_GLeDpNCA-B1SAW3Skba1U/edit?usp=sharing)

---

## Introduction

Rents in Portugal are one of the country's most discussed topics. Three claims come up again and again: wages can't keep up, short-term rentals (Airbnb-style *Alojamento Local*) are pushing locals out, and it's mainly a Lisbon and Porto problem. We tested each claim with official data from INE (Statistics Portugal) and the national short-term rental register (RNAL).

**Key definition:** "rent" always means the **long-term rent**: the median monthly rent per m² of **new lease contracts actually signed** (INE), not asking prices and not tourist prices.

---

## Research questions and answers

| # | Question | Verdict | Key result |
|---|---|---|---|
| **H1** | Are long-term rents rising faster than wages? | ✅ Supported | Rent **+109%** vs wages **+43%** (2017 → 12 months to June 2026) |
| **H2** | Do municipalities with more short-term rentals have higher long-term rents? | ⚠️ Partly supported | **+41%** higher rent in *large* municipalities with more short-term rentals; no difference in small ones; growth since 2020 not faster |
| **H3** | Is rent growth spreading beyond Lisbon and Porto? | ✅ Supported | Since 2020: **+73%** around the cities, **+71%** rest of Portugal, **+52%** in Lisbon & Porto |

---

## Data sources

About **146,000 source records** from two official sources:

| Data | Source | Indicator / file | How we get it | Used in |
|---|---|---|---|---|
| Long-term rent, 2021 methodology (old regional map), 2017–2023 | INE | `0009817` | API | H1 |
| Long-term rent, 2021 methodology (new regional map), 2020–2024 | INE | `0012598` | API | H1 |
| Long-term rent, 2026 methodology, 1Q 2020 – 2Q 2026 | INE | `0014696` | API | H1, H2, H3 |
| Average gross monthly earnings per employee (incl. holiday and Christmas pay) | INE | `0011132` | API | H1 |
| Resident population by municipality (2025) | INE | `0012918` | API | H1, H2, H3 |
| Short-term rental registrations (111,852, up to 29 Sep 2026) | Turismo de Portugal (RNAL) | `rnal.csv` | Dataset (CSV download) | H2 |

- INE API documentation: https://www.ine.pt/xportal/xmain?xpid=INE&xpgid=ine_api_db
- INE API endpoint: `https://www.ine.pt/ine/json_indicador/pindica.jsp?op=2&varcd=<code>`
- RNAL dataset: https://dadosabertos.turismodeportugal.pt/datasets/estabelecimentos-de-alojamento-local

---

## Methodology

### 1. Data collection
- **INE API:** one request per indicator. Raw answers are saved in `data/raw/`, so the API is only called once. The H2/H3 notebooks pause 1 second after each request and retry if the server answers *429 – too many requests*.
- **RNAL:** CSV downloaded from Turismo de Portugal's open-data portal and saved as `data/raw/rnal.csv`.
- A scraped population table from an earlier version was replaced by the INE API, so every number is official.

### 2. Data cleaning

| Problem | Technique |
|---|---|
| One INE table mixes country, regions, municipalities and parishes | Filter on the length of `geocod` (7 characters = municipality; mainland codes start with `1`) |
| Numbers stored as text | `pd.to_numeric()` |
| Dates and periods stored as text ("2014/12/03 09:47:03+00", "2nd Semi-annual 2017") | `pd.to_datetime()`, string methods (`.str.startswith()`, `.str[-4:]`) |
| Wages split by component and sector | Keep the totals (`dim_3 == "T"`, `dim_4 == "T"`) |
| Quarterly wages | Yearly average with `groupby()`; latest 12 months = average of the last 4 quarters |
| INE and RNAL store the municipality code differently; INE's regional map changed | Join on the **4-digit municipality code** (last 4 digits of INE `geocod` = first 4 digits of RNAL parish code), never on names |
| Municipalities with too few contracts have no published rent | Dropped and reported: **41** today, **64** since 2020, **92** since 2017 (mostly small towns) |
| Possible duplicate registrations | Checked with `drop_duplicates` on the registration number: **0 duplicates** |
| Unused columns | Dropped |
| **INE changed its rent methodology in 2026** (different levels: end of 2020 = 5.79 vs 5.61 €/m²) | **Chain-linking (H1):** old methodology up to 2024, extended with the growth of the new one. Robustness check without the join: rent +82% vs wages +32% (2017–2024) |

Each notebook has a **main cleaning function** that runs these steps in order (`build_h1_table()`, `clean_ine()`, `clean_rnal()`).

### 3. Analysis
- **H1:** one value per calendar year 2017–2025 plus the latest 12 months (to June 2026); both series indexed to 2017 = 100. Rent of an 80 m² flat as a share of the average gross salary. Repeated for small (≤ 50,000 residents) vs large municipalities (median across municipalities).
- **H2:** short-term rentals per 1,000 residents; municipalities split into "more" / "fewer" (above / below the median) **within the same size group**, so that size doesn't hide the effect. Compared rent levels (2Q 2026) and rent growth (1Q 2020 → 2Q 2026, using new registrations since April 2020).
- **H3:** municipalities grouped by INE region code: Lisbon & Porto (2), around them (33), rest of Portugal (179). Median rent growth per group, the cities' price premium, and the fastest-rising municipalities.

### 4. Visualisation
`matplotlib`, with one shared chart style across all notebooks: white background, one blue accent, values written on the charts. All charts are saved in `figures/`.

---

## Main findings and insights

**H1: Rents are outrunning pay.**
- Long-term rent **+109%** (4.39 → 9.18 €/m²) vs average gross wages **+43%** (1,215 → 1,738 €): rents grew 2.5× as fast.
- An 80 m² flat went from ≈ 351 € to ≈ 735 € a month, from **29% to 42%** of an average gross salary.
- Not only a big-city problem: small municipalities **+93%**, large **+104%**, both far above wages.

**H2: Short-term rentals matter for rent levels in big towns, not for the recent rise.**
- Short-term rental registrations peaked in **2018 (≈14,700)** and **2023 (≈12,800)**, concentrated in Porto, Lisbon and the Algarve.
- **Lisbon's boom stopped:** 3,714 new registrations in 2018, only 28 in 2024.
- Large municipalities with more short-term rentals: **9.75 vs 6.92 €/m² (+41%)**. Small municipalities: no real difference (5.09 vs 5.26 €/m²).
- Since 2020, long-term rents rose **≈71%** almost everywhere, **not faster** where more short-term rentals were added.

**H3: The crisis is moving outward.**
- Since 1Q 2020: Lisbon & Porto **+52%** (Lisbon +44%, Porto +59%), around them **+73%**, rest of Portugal **+71%**.
- The cities' premium over their surroundings shrank from **80% to 57%**.
- Fastest rises around the cities: **Moita +104%**, Trofa +91%, Barreiro +87%.
- Most of the 10 fastest-rising municipalities with ≥ 20,000 residents are outside the two metropolitan areas.

**Overall:** affordability is now a national problem, and limiting short-term rentals alone is unlikely to fix it.

---

## Limitations
- **Correlation is not causation:** the data show where and how fast rents grew, not why.
- Rent is the **median of new contracts**; wages are the **average of all jobs**. H1 describes people signing a new lease, not all tenants.
- Wages are only available for Portugal as a whole, so municipal comparisons use the national wage.
- Rent after 2024 (H1) depends on chain-linking two INE methodologies.
- RNAL only lists registrations that are **still active**; closed ones are missing, so earlier years are undercounted.
- Municipalities with too few contracts have no rent value and are excluded. Azores and Madeira are not part of the municipality comparisons.
- All values are nominal (not adjusted for inflation). Only contracts declared to the tax authority are included.

---

## Further questions and next steps
- **Why** do rents rise? Test supply (new housing), migration and remote work.
- Compare rents with **local incomes** (INE's local income statistics based on tax data) instead of national wages.
- Adjust rents and wages for **inflation**.
- Check whether changes in short-term rental rules line up with the drops in new registrations.
- Extend the municipality analysis to the **Azores and Madeira**.

---

## Repository structure

```
data-wrangling-project/
├── README.md
├── h1_long_term_rent_vs_wages.ipynb      # H1 – Andrea
├── h2_short_term_rentals_vs_rent.ipynb   # H2 – Juliana
├── h3_long_term_rent_spreading.ipynb     # H3 – Juliana
├── data/
│   ├── raw/      # API answers and RNAL CSV (saved on first run)
│   └── clean/    # h1_clean.csv, h1_municipalities_clean.csv, h2_clean.csv, h3_clean.csv
└── figures/      # all charts used in the presentation
```

## How to run
1. Install the libraries: `pip install pandas numpy matplotlib requests`
2. Download the RNAL CSV (link above) and save it as `data/raw/rnal.csv`.
3. Run the notebooks top to bottom. INE data is downloaded on the first run and read from `data/raw/` afterwards.

---

## Links
- **Presentation:** [Rental Crisis in Portugal – slides](https://docs.google.com/presentation/d/141kplpqkM6aTQ9PDCH6gl_GLeDpNCA-B1SAW3Skba1U/edit?usp=sharing)
- **Kanban board (Trello):** ADD LINK
- **INE API:** https://www.ine.pt/xportal/xmain?xpid=INE&xpgid=ine_api_db
- **RNAL dataset:** https://dadosabertos.turismodeportugal.pt/datasets/estabelecimentos-de-alojamento-local
