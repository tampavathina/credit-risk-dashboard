# Data Dictionary

Source: Lending Club accepted loans 2007–2018Q4 (Kaggle, `wordsforthewise/lending-club`), file `accepted_2007_to_2018Q4.csv`.
Column meanings: `LCDataDictionary.xlsx`, sheet `LoanStatNew`.

## Subset rule
Keep a row only if **all** of these are true:
- `term` = 36 months
- `issue_d` year is 2013, 2014 or 2015
- `loan_status` is final (see Default flag below). Drop `Current`, `Late`, `In Grace Period` and similar.

PD and LGD are estimated on the **2013–2014** vintages and backtested on **2015**.

## Leakage rule
PD may only use fields **known at origination** (✅). Post-origination fields (❌) are used **only** to compute the outcome or historical LGD, never as PD predictors.

## Raw columns kept

| Column | Meaning | Raw format | Origination? | Used for |
|---|---|---|---|---|
| `id` | Unique loan ID | integer | ✅ | Primary key |
| `issue_d` | Month the loan was issued | text, e.g. `Dec-2015` | ✅ | Subset filter, vintage split, filter in app |
| `term` | Loan length | text, e.g. ` 36 months` (check for leading space) | ✅ | Subset filter, PD bucket (grade × term) |
| `grade` | LC risk grade | `A`–`G` | ✅ | PD bucket, portfolio filter, charts |
| `sub_grade` | Finer grade | `A1`–`G5` | ✅ | Display, stretch PD model |
| `int_rate` | Interest rate (%) | number, `%` already stripped, e.g. `13.99` | ✅ | Weighted-average rate |
| `funded_amnt` | Amount actually lent | number | ✅ | **EAD**, exposure, LGD denominator |
| `purpose` | Stated loan purpose | text, e.g. `debt_consolidation` | ✅ | Portfolio filter, exposure chart |
| `addr_state` | Borrower's state | 2-letter code | ✅ | Portfolio filter, concentration (top-5, HHI) |
| `annual_inc` | Self-reported annual income | number | ✅ | Income-band filter, stretch PD model |
| `dti` | Debt-to-income ratio (%) | number | ✅ | DTI-band filter, stretch PD model |
| `fico_range_low` | FICO band low end at origination | number | ✅ | Derived `fico`, stretch PD model |
| `fico_range_high` | FICO band high end at origination | number | ✅ | Derived `fico` |
| `emp_length` | Years employed | text, e.g. `10+ years`, `< 1 year`, blank | ✅ | Stretch PD model |
| `home_ownership` | Housing status | `RENT` / `MORTGAGE` / `OWN` / … | ✅ | Stretch PD model |
| `verification_status` | Was income verified | text | ✅ | Stretch PD model |
| `delinq_2yrs` | 30+ day delinquencies in past 2 years | number | ✅ | Stretch PD model |
| `inq_last_6mths` | Credit inquiries in past 6 months | number | ✅ | Stretch PD model |
| `revol_util` | Revolving credit utilization (%) | number, `%` already stripped | ✅ | Stretch PD model |
| `loan_status` | Final outcome | text | ❌ outcome | **Default flag** |
| `total_rec_prncp` | Principal repaid before default/payoff | number | ❌ | **LGD (and EAD factor) only** |
| `recoveries` | Amount recovered after charge-off | number | ❌ | **LGD only** |
| `collection_recovery_fee` | Fee paid to collect recoveries | number | ❌ | **LGD only** |

## Derived columns (computed by `etl.py`)

| Column | Definition | Notes |
|---|---|---|
| `issue_date` | `issue_d` parsed to `YYYY-MM-01` | ISO date, so SQL can sort and filter it |
| `vintage` | Year of `issue_date` | 2013 / 2014 = estimation, 2015 = backtest |
| `fico` | (`fico_range_low` + `fico_range_high`) / 2 | Single score per loan |
| `default_flag` | 1 if defaulted, 0 if fully paid | See mapping below |
| `lgd` | 1 − (`recoveries` − `collection_recovery_fee`) / (`funded_amnt` − `total_rec_prncp`), defaulted loans only | Clip to [0, 1]. NULL for non-defaults. Skip rows where the denominator is 0. LGD is measured on the **balance outstanding at default**. |
| `outstanding_share` | (`funded_amnt` − `total_rec_prncp`) / `funded_amnt`, defaulted loans only | Share of principal still owed when the loan defaulted. NULL for non-defaults. Feeds the EAD factor. |

## Risk parameter tables (computed by `etl.py` from the 2013–2014 vintages only)

| Table | Columns | Definition |
|---|---|---|
| `pd_table` | `grade`, `term`, `pd`, `n` | `pd` = mean `default_flag`; `n` = loan count |
| `lgd_table` | `grade`, `lgd`, `ead_factor`, `n_defaults` | `lgd` = mean `lgd`; `ead_factor` = mean `outstanding_share`; both over defaulted loans |

## EAD and Expected Loss (decided 2026-11-23: option A)
- **EAD** = `funded_amnt` × `ead_factor` (for the loan's grade)
- **EL** = PD × LGD × EAD
- Why: LGD is a fraction of the balance *outstanding at default*, so EAD must be that same balance, not the original funded amount. Using `funded_amnt` directly would overstate EL, because borrowers repay part of the principal before they default.

### Default flag mapping
| `loan_status` | `default_flag` |
|---|---|
| `Charged Off` | 1 |
| `Default` | 1 |
| `Does not meet the credit policy. Status:Charged Off` | 1 |
| `Fully Paid` | 0 |
| `Does not meet the credit policy. Status:Fully Paid` | 0 |
| anything else | row dropped |

## Deliberately excluded (leakage)
These are known only after origination, so they are never PD inputs:
`last_fico_range_high`, `last_fico_range_low`, `last_pymnt_d`, `last_pymnt_amnt`, `total_pymnt`, `total_rec_int`, `out_prncp`, `next_pymnt_d`, `hardship_*`, `settlement_*`, `debt_settlement_flag`.
`recoveries`, `collection_recovery_fee` and `total_rec_prncp` are kept, but only for LGD.

## Also excluded
`member_id`, `url`, `desc` (empty or unused). `emp_title` is free text with too many unique values for this project. `funded_amnt_inv` is the investor share of `funded_amnt`, and EAD uses the full funded amount.
