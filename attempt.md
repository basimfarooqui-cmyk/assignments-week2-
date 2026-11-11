# Initial attempt (written BEFORE using any AI tool)


## Diagnosis (before changing anything)

| # | Problem | Code that revealed it | Planned decision | Reason |
|---|---------|----------------------|------------------|--------|
| 1 | Exact duplicate row: 2026-01-08, Staple A, City 1, 99.5 (index 96, copy of index 8) | `raw.duplicated().sum()` -> 1; `raw[raw.duplicated(keep=False)]` shows index 8 and 96 | **Exclude** the second copy | Same observation recorded twice; it was appended after the sorted data. |
| 2 | Item `oil b` (lowercase), index 40: 2026-02-05, City 1, 108.5 | `raw["item"].value_counts()` shows `oil b` = 1 | **Investigate, then correct** | `str.title()` would create a 25th Oil B and a duplicate key with index 42 (Oil B / City 1 / 2026-02-05 = 318.0). 108.5 fits Staple A (City 1 values 99.5 to 116), and Staple A / City 1 is the only one missing for that date. So it is a mislabelled Staple A. It is an inference, so I will flag it. |
| 3 | Missing price: Pulse C, City 2, 2026-01-01 (index 5) | `raw.isna().sum()` -> price 1; `raw[raw["price"].isna()]` | **Retain, flag, do not impute** | A real visit with an unknown value. Filling it with a mean/median/neighbour would invent data. Leave NaN and exclude from statistics. |
| 4 | `price` is text (object/str), not numbers; `"1,250"` at index 17 | `pd.to_numeric(raw["price"], errors="coerce")`: missing before 1, after 2; `raw[number.isna()]` shows `1,250` | **Correct the format** | The comma is a formatting problem, so `str.replace(",", "")` before converting. Keep original text in `price_raw`. |
| 5 | After parsing, 1250 for Staple A / City 2 / 2026-01-15 is implausible | `number.describe()` (max, std); `raw[number > 1000]` | **Flag, set `price` to NaN, keep row** | About 10x neighbours (Staple A City 2 ranges 105.5 to 122). Typo or error, I cannot tell. I will not guess a replacement. |
| 6 | Sentinel 9999 for Sweetener D / City 1 / 2026-01-22 (index 30) | `number.describe()` max = 9999; `raw[number > 1000]` | **Flag, set `price` to NaN, keep row** | Typical "unknown" code. Not a price. Not imputed. |
| 7 | `unit` and `data_label` are the same in all rows | `raw.nunique()` -> 1 each; `raw["data_label"].value_counts()` | **Retain, do not touch** | They say the data is illustrative and not PBS. |
| 8 | `date` is text, not a date | `raw.info()` | **Correct the type** | `pd.to_datetime`; needed to sort and check. |

## Checked and fine
- `raw["city"].value_counts()` has only `City 1` / `City 2` (49 / 48 before fixes, 48 / 48 after).
- `raw["date"].nunique()` -> 12 weekly dates; all parse with `to_datetime`.

## Expected result after cleaning
96 rows (12 dates x 4 items x 2 cities), each item 24 times, `clean.duplicated(subset=["date","item","city"]).sum()` = 0, and 3 blank prices (index 5, 17, 30), all flagged.
