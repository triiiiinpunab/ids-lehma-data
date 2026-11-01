# Predicting a dairy cow's 305-day yield at calving (from project providers)
Predictive information available at calving:
| Information | Columns | What it is |
|---|---|---|
| **Herd** | `herd_avg_milk_305`, `herd_avg_fat_pct`, `herd_avg_protein_pct`, `herd_n_milking_cows`, `herd_avg_parity`, `herd_n_cows_total` | How the rest of this farm's cows have been producing, measured in a window that ends before this calving |
| **Previous lactation** | `prev_milk_305`, `prev_fat_305`, `prev_protein_305`, `prev_peak_milk`, `prev_calving_interval`, `prev_dry_days`, `has_prev_lact` | How this cow performed previously |
| **EBV** | `spav`, `spav_reliability`, `has_ebv` | Cow's estimated breeding value for milk from the national genetic evaluation in force before the calving |
| **Cow** | `parity`, `breed`, `age_at_calving_mo`, `calving_season`, `calving_month` | Cow's and her lactation's general information |
## Files
| File | Rows | What it holds |
|---|---|---|
| `train.csv.gz` / `train.parquet` | 317,546 | Calvings 2021-2024. Features and the three targets |
| `test.csv.gz` / `test.parquet` | 63,573 | Calvings 2025. **Identical columns, targets included** |
| `columns.csv` | 29 | What every column means, its role and where it came from |
The Parquet and CSV files hold the same data. Prefer Parquet if you can: it keeps the
column types, including the nullable integers and the ordered categories, which a CSV
loses.
**Targets** (all in kg, cumulative over DIM (days in milk) 1–305):
- `target_milk_305_kg` — the headline target;
- `target_fat_305_kg`, `target_protein_305_kg` — the same for fat and protein.
Predict `target_milk_305_kg` at minimum. The other two are there if you want a
multi-output model.
## The split is by time, not by farm
**Train on 2021-2024, predict 2025.** This is deliberate, and it is the
condition the model would actually face in use: you have history, you want next year.
Two consequences that will show up in your results.
**1. There is a level shift between the two sides.** Median 305-day milk is
**10,905 kg** in training and **11,555 kg** in the test year — a
**+650 kg** difference, about +6%. Estonian yields have risen for over twenty
consecutive years, through breeding and management. A model fitted on the earlier years
will under-predict the later one unless you do something about it.
**2. The same herds appear on both sides, and so do many of the cows.** 323 of
the 326 test herds are also in training, and **66% of the test cows
have an earlier lactation in the training set**. Neither is a leak:
- every feature here looks strictly *backwards* from the calving date. The herd averages
  are computed over other cows' records in a window ending before this calving; `spav`
  is the evaluation in force before it; `prev_*` describes the cow's previous lactation;
- a cow having an earlier lactation in training is precisely the mechanism that makes
  `prev_milk_305` exist. If you removed it you would be deleting the strongest predictor
  in the dataset.
What *is* a mistake: **using `lact_id` or `id` as a model input.** `lact_id` is
`id * 100 + parity`, so it encodes the cow's identity and her parity number. Treat both
as identifiers, along with `herd_id`, `calving_date` and `calving_year`.
## What to expect
### A third of the cows have neither of the strongest predictors
**34% of the rows are first-lactation heifers.** They have no previous
lactation, and structurally no EBV — a cow is only evaluated once she has records of her
own, so `has_ebv` is **0 for every parity-1 cow in this dataset**. For that third you
have herd information and her own descriptors, and nothing else. So report parity 1 separately from parity 2+.
Missing values are **real information, not something to fill in**. `prev_milk_305` is
absent because there is no previous lactation; `spav` is absent because the cow has not
been evaluated. Both facts predict. Zero-filling them tells the model the cow produced
nothing last year, which is false. The flags `has_prev_lact` and `has_ebv` mark the two
cases.
## One honest caveat about the labels
A 305-day record is built by the ICAR **test interval method**: the daily curve runs
through the cow's test days, and after her **last** test day it is extended flat to day
305. For most cows that tail is short — the median is **26 days** of 305.
But **40% of the test-year labels have more than 30 such days and
13% have more than 105**, where a cow was dried off or sold early. For those,
a meaningful part of the "total" is a convention rather than a measurement.
## Column reference
`columns.csv` describes all 29 columns. The `origin` field follows CDISC's
vocabulary: *Collected* straight from the database, *Derived* computed by a rule,
*Assigned* from a lookup, *Predecessor* copied from another variable.
Units, since the fat and protein columns are not all alike:
- the three targets and `prev_milk_305` are **kg**;
- `prev_fat_305` and `prev_protein_305` are **kg** too, not percentages (so roughly 400 kg
  of fat against 10,500 kg of milk, i.e. about 3.9%);
- `herd_avg_fat_pct` and `herd_avg_protein_pct` *are* **percentages**;
- `age_at_calving_mo` in months, `prev_calving_interval` and `prev_dry_days` in days;
- `spav` is an index centred near 100 and `spav_reliability` is a percentage.
Breeds are Estonian Holstein (`EHF`) and Estonian Red (`EPK`) for about 99% of
the data, with smaller groups beside them.
## Data origin and contact
One row per lactation, each the day the cow calved. Calvings from 01 January 2021 to
20 November 2025, drawn from 459 Estonian herds; a lactation is included only if a
complete 305-day record could be built for it.
Questions about the data, the columns or anything that looks wrong: ask — a column that
behaves strangely is as likely to be our bug as your misreading.
