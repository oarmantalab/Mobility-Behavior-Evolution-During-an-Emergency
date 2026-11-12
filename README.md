# Mobility Behavior Evolution During an Emergency

Reproducible implementation of the returner/explorer analysis developed in:

> **Mobility Behavior Evolution During an Emergency**  
> Omid Armantalab, Jason Hawkins, and Wissam Kontar  
> Department of Civil & Environmental Engineering, University of Nebraska–Lincoln  
> Department of Civil Engineering, University of Calgary

This repository examines how individuals shift between two mobility behavior types during a 15-day emergency period in a densely populated Japanese metropolitan area, using the open **YJMob100K** dataset. **Returners** concentrate their movement around a few familiar locations; **explorers** spread their movement over many destinations. The analysis compares a 15-day normal period with the 15-day emergency period, tracks how behavior evolves day by day, separates weekdays from weekends and holidays, and evaluates mobility inside versus outside each person's nearby neighborhood.

---

## Method

### 1. Stops

Each YJMob100K record gives a user's 500 m × 500 m grid cell in a 30-minute slot. Consecutive records of a user in the same cell are merged into a **stop**, and the time attributed to each record (the gap to the next record, capped at 5 hours) is summed into the stop's dwell time.

### 2. Radius of gyration

The dwell-time-weighted radius of gyration measures the spatial extent of a user's mobility:

```text
r_g = sqrt( (1/T) * sum_i t_i * |r_i - r_cm|^2 ),     T = sum_i t_i
```

where `t_i` is the time spent at location `i` and `r_cm` is the dwell-time-weighted center of mass. The **k-radius of gyration** `r_g^(k)` applies the same formula to the `k` locations where the user spent the most time.

### 3. Returners and explorers

```text
S_k = r_g^(k) / r_g

S_k > 0.5   ->  k-returner   (mobility defined by the top-k locations)
S_k <= 0.5  ->  k-explorer   (mobility extends well beyond them)
```

The paper uses `k = 4`, the crossover point of the normal period, to classify users in both periods.

### 4. Indicators and tests

For each group the analysis computes:

- maximum distance from home and non-home dwelling time;
- real entropy of the time-ordered stop sequence;
- spatial distributions of stops and stay time;
- average daily time and distance outside the nearby neighborhood of home (cells within ±2 grid units).

Group differences are tested with Mann-Whitney U (rank-biserial effect size) and two-sample Kolmogorov-Smirnov tests, with bootstrap 95% confidence intervals. Distributions of `r_g` are fitted with exponential, lognormal, and truncated power-law models by maximum likelihood.

### 5. Transitions

Users classified in both periods fall into four transition groups: **R-R**, **R-E**, **E-E**, and **E-R**. The normal-to-emergency transition matrix is compared against transitions between the normal period and three other normal windows (a placebo baseline), and recomputed at alternative `S_k` cutoffs (`r_g/1.8`, `r_g/2`, `r_g/2.2`).

---

## Repository Files

| File | Purpose |
| --- | --- |
| `preprocess_yjmob100k.py` | Cleans YJMob100K Dataset 2, adds the calendar, infers home locations, computes dwell time and trip distance, and builds the stop tables for both periods. |
| `mobility_metrics.py` | Study configuration and all shared functions (radius of gyration, classification, entropy, statistical tests, distribution fitting). |
| `period_analysis.py` | Full single-period analysis; run once with `--period normal` and once with `--period emergency`. |
| `observation_window_analysis.py` | Determines how many days of data are needed to separate emergency from normal behavior, with robustness and extended-window checks. |
| `transition_analysis.py` | Transition matrices, placebo baseline, threshold sensitivity, four-group daily analysis, and the POI comparison across groups. |
| `requirements.txt` | Lists the Python packages required to run the repository code. |

---

## Data

### YJMob100K

The study uses **Dataset 2** of YJMob100K, which follows 25,000 anonymized individuals over 75 days (days 0–74). Days 0–59 represent business-as-usual behavior and days 60–74 an emergency period designated by the dataset creators. Locations are given on a 200 × 200 grid of 500 m cells, and time is discretized into 30-minute slots.

The data are available on Zenodo under a CC BY 4.0 license:

<https://zenodo.org/records/10836269>

Required files:

| File | Content |
| --- | --- |
| `yjmob100k-dataset2.csv.gz` | Mobility records with columns `uid, d, t, x, y`. |
| `cell_POIcat.csv.gz` | Number of points of interest per grid cell and category. |

The data descriptor is:

> Yabe, T., Tsubouchi, K., Shimizu, T., Sekimoto, Y., Sezaki, K., Moro, E., and Pentland, A.  
> *YJMob100K: City-scale and longitudinal dataset of anonymized human mobility trajectories.*  
> Scientific Data, 11, 397 (2024).  
> DOI: 10.1038/s41597-024-03237-9

The YJMob100K data are **not included in this repository**. Place the two files in `data/raw/`:

```text
data/
└── raw/
    ├── yjmob100k-dataset2.csv.gz
    └── cell_POIcat.csv.gz
```

---

## Data Preprocessing

Prepare the records with:

```bash
python preprocess_yjmob100k.py
```

The preprocessing script:

- removes day 27, as advised by the dataset creators;
- adds day-of-week, weekend, and holiday flags;
- infers each user's home as the cell with the most records between 8:00 PM and 8:00 AM over the 60 normal days;
- computes the dwell time of each record (gap to the next record, capped at 5 hours) and the trip distance from the previous record;
- flags records inside the nearby neighborhood of home;
- merges consecutive records at the same cell into stops for the normal window (days 43–57) and the emergency window (days 60–74); and
- aggregates the POI counts per grid cell.

Outputs are written to `data/processed/`:

```text
records.parquet
stops_normal.parquet
stops_emergency.parquet
poi_cells.csv
```

Other file locations can be passed with `--raw`, `--poi`, and `--out-dir`. A `.csv` or `.parquet` copy of Dataset 2 works as well, and a POI table already aggregated to `x, y, poi_sum` is accepted.

---

## Installation

Python 3.10 or newer is recommended.

Create and activate a virtual environment if desired, then install all required packages with:

```bash
pip install -r requirements.txt
```

The requirements file covers all scripts in this repository.

---

## Single-Period Analysis

The same pipeline is applied to each period:

```bash
python period_analysis.py --period normal
python period_analysis.py --period emergency
```

For the selected period the script:

1. fits exponential, lognormal, and truncated power-law distributions to `r_g`, for the full period and for each day;
2. fits `r_g^(k)` for `k = 2, 3, 4, 5` and records `S_k` for every user;
3. computes the share of returners and explorers for `k = 2, 3, 4, 5, 6, 8, 10`;
4. classifies users with `k = 4`;
5. compares returners and explorers on maximum distance from home, non-home dwelling time, and real entropy, with Mann-Whitney U and Kolmogorov-Smirnov tests and bootstrap confidence intervals;
6. computes the full-period and daily distributions of both mobility indicators;
7. computes the spatial distribution of stops and stay time per person;
8. computes time and distance outside nearby neighborhoods on weekdays and on weekends/holidays; and
9. computes the share of returners among the residents of each home cell.

Results are saved as CSV tables under:

```text
results/normal/
results/emergency/
```

---

## Observation Window Analysis

```bash
python observation_window_analysis.py
```

The script classifies users within windows of increasing length and reports the percentage of returners for each `k`:

- **Matched windows:** the first 1 to 14 days of the normal and emergency periods. For each length, it reports whether both periods agree on the majority group at each `k`; the first disagreement marks the minimum observation length.
- **Robustness segments:** three other 14-day normal segments (days 1–14, 15–28, 29–42), each observed for 7 and 14 days.
- **Extended windows:** 2, 4, 6, and 8 weeks of normal data ending on day 56.

Results are saved under:

```text
results/observation_window/
```

including a `crossover_summary.csv` file with the crossover `k` of every window.

---

## Transition Analysis

```bash
python transition_analysis.py
```

The script:

- builds the normal-to-emergency transition matrix (rows sum to 100%);
- builds the same matrix from the normal period to three other normal windows (days 1–15, 15–29, 29–43) as a placebo baseline;
- recomputes the transitions at `S_k` cutoffs of `r_g/1.8`, `r_g/2`, and `r_g/2.2`;
- assigns every user to the R-R, R-E, E-E, or E-R group and computes their daily maximum distance from home and non-home dwelling time during the emergency; and
- compares the average POI count of visited cells outside the nearby neighborhood (weekdays and holidays, both periods) and of the home neighborhood across the four groups, using Kruskal-Wallis tests with epsilon-squared, Holm-corrected pairwise Mann-Whitney tests, and a balanced-subsampling check.

Results are saved under:

```text
results/transitions/
```

---

## Study Configuration

All settings are defined once at the top of `mobility_metrics.py`:

| Setting | Value |
| --- | ---: |
| Normal window | days 43–57 |
| Emergency window | days 60–74 |
| Placebo normal windows | days 1–15, 15–29, 29–43 |
| Excluded days | 27 |
| Grid cell size | 500 m |
| Maximum dwell time per record | 5 hours |
| `k` for classification | 4 |
| `S_k` threshold | 0.5 |
| `k` for neighborhood and home-location analyses | 6 |
| Nearby neighborhood | ±2 grid cells around home |
| Distribution fit `x_min` | 1 km |
| Bootstrap replicates (distance, dwell time) | 2,000 |
| Bootstrap replicates (entropy) | 200 |
| Balanced-subsampling iterations | 500 |

### Calendar

On normal days, day `d` falls on weekday `d mod 7` (0 = Sunday). Weekends and the public holidays identified from the daily ping counts are flagged as holidays. The emergency block is spliced separately in the dataset, so it is re-indexed with day 60 as a Monday and day 67 as a holiday. This gives the normal and emergency windows the same weekly structure: a weekend on days 6–7 and 13–14 and a holiday on day 8 of each period.

---

## Reproducibility

To keep stochastic components reproducible, **random seed 12345 is used throughout the repository**. The generator is re-seeded for each test, so results do not depend on the order in which analyses are run.

This includes:

- bootstrap confidence intervals for the Mann-Whitney U and Kolmogorov-Smirnov tests; and
- the balanced-subsampling check of the POI comparison.

All other steps are deterministic. When two locations have identical dwell time at the boundary of a user's top-`k` set, the tie is broken by grid coordinates `(x, y)`, so the classification is identical across runs and machines.

---

## Typical Workflow

A complete repository workflow is:

```text
1. Download YJMob100K Dataset 2 and the POI file
            |
            v
2. preprocess_yjmob100k.py
            |
            v
   records.parquet, stops_normal.parquet,
   stops_emergency.parquet, poi_cells.csv
            |
            +------------------+------------------+
            |                  |                  |
            v                  v                  v
3. period_analysis.py   4. observation_       5. transition_
   (normal, emergency)     window_analysis.py    analysis.py
```

In command-line form:

```bash
python preprocess_yjmob100k.py
python period_analysis.py --period normal
python period_analysis.py --period emergency
python observation_window_analysis.py
python transition_analysis.py
```

---

## Contact

For questions about the analysis or repository, please contact the corresponding author through the contact information provided in the manuscript.
