# Reflection

## 1. Cleaning Code Court: verdicts

| Line | AI code | Verdict | Evidence |
|------|---------|---------|----------|
| L1 | `pd.to_numeric(price.str.replace(",", ""), errors="coerce")` | **Modify** | Removing the comma fixes `"1,250"`, but `errors="coerce"` hides any other bad text. I check for non-numeric text first, keep `price_raw`, and flag changes. |
| L2 | `dropna(subset=["price"])` | **Reject** | 97 -> 96. Deletes the real observation Pulse C / City 2 / 2026-01-01. I keep the row and leave the price blank. |
| L3 | `drop_duplicates()` | **Keep** | 96 -> 95 (dropped the repeated 2026-01-08 Staple A / City 1 row). Exact duplicate, and every key appears once afterwards. |
| L4 | `item.str.title()` | **Reject** | Oil B goes 24 -> 25 and Staple A falls to 23, and a duplicate key appears (Oil B / City 1 / 2026-02-05: 108.5 and 318.0). Correct fix is relabel to Staple A, flagged. |
| L5 | IQR filter on `price` | **Reject** (replace with flag) | It catches 1250 and 9999 only because they are huge. It is pooled across items, so a bad Staple A value like 250 would pass. It also deletes rows silently. I flag instead. |
| L6 | `drop(columns=["unit","data_label"])` | **Reject** | Constant columns are the only warning the data is illustrative. Keep. |
| L7 | `pd.to_datetime(date)` | **Keep** | All 12 dates parse, all Thursdays, no rows lost. |

Check: any valid observation removed? L2 removed one, and L5 removed two rows that were wrong values but whose existence is real. Any label or number changed without my knowing? Yes, L4 changed `oil b` to `Oil B` and created a duplicate key. After my cleaning, every item appears 24 times and every (date, item, city) appears once.

## 2. Cleaning log (rows affected)

| row_id (csv line) | Row | Action | Reason | Flag |
|---|---|---|---|---|
| 98 | 2026-01-08 Staple A City 1 99.5 | Excluded (duplicate of line 10) | Exact copy | n/a (dropped) |
| 42 | 2026-02-05 `oil b` City 1 108.5 | Relabelled to Staple A | Price in Staple A range; Staple A / City 1 missing that date; Oil B / City 1 already exists | `item_relabelled_oil_b_to_staple_a` |
| 7 | 2026-01-01 Pulse C City 2 (blank) | Retained, price blank | Missing, not imputed | `price_missing` |
| 19 | 2026-01-15 Staple A City 2 `"1,250"` | Parsed to 1250.0, then blanked in `price_clean`; `price_raw` kept | 10x neighbours (105 to 122); cause unknown | `implausible_1250_10x_neighbours` |
| 32 | 2026-01-22 Sweetener D City 1 9999 | Retained, `price_clean` blank | Sentinel code | `sentinel_9999` |
| all | `unit`, `data_label` | Unchanged | Provide meaning | n/a |

Net effect: 97 -> 96 rows. Changed by me: 1 label. Changed in format only: 1 value (comma). Blank in the analysis column: 3. Left alone: everything else.

## 3. Where my decisions differed from the generated code's

- **Missing price.** The code dropped the row (L2). I kept it and flagged it. Mine was better supported: the visit happened, and deleting it makes Pulse C look like it had 23 weeks by design.
- **`oil b`.** The code normalised case (L4), which looks right and produces a wrong table. I used the price and the missing Staple A / City 1 slot as evidence. The AI's version is the one that would have gone unnoticed.
- **Outliers.** The code deleted them with a pooled IQR (L5). I flagged them and blanked the value but kept the row, since the observation exists even if the number is not trusted. Mine is better supported, but note I could not verify 1250 or 9999 against any source, so "implausible" is my judgement.
- **Constant columns.** The code dropped them (L6). I kept them for the illustrative-data label.
- Where we agreed: the exact duplicate (L3) and the date conversion (L7).

## 4. What I would have missed alone
- The L4 trap: I would have run `str.title()` and not noticed the duplicate key unless I checked per-item counts afterwards. The "does every item appear the expected number of times" check is what exposes it.
- The silent behaviour of `errors="coerce"` and why I keep `price_raw`.
- That a pooled outlier rule hides problems within an item, so I should compare each value with its own item's history.

## 5. Remaining uncertainty
- The `oil b` -> Staple A relabel is an inference, not proof; it is flagged so it can be reversed.
- I cannot tell whether 1250 is a typo (125?) or an error, so I did not correct it.
- The 3 blank prices reduce the sample for Pulse C, Staple A, and Sweetener D by one week each.
