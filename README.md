# VIX Replication from OptionMetrics (WRDS)

Python implementation of the Cboe VIX methodology using OptionMetrics SPX option quotes and the OptionMetrics zero-coupon curve via WRDS. Builds the daily near/next term schedule, computes term variances, and interpolates to a constant 30-day maturity.

Includes the post–February 10, 2025 OTM strike selection rule (exclude bid==0 or ask==0; stop after two consecutive such strikes).

## Requirements

- Python 3.12+
- numpy, pandas, wrds
- WRDS access to:
  - OptionMetrics (`optionm_all.opprcdYYYY`, `optionm_all.zerocd`)
  - Optional: Cboe (`cboe.cboe`) for comparison

Install:

```bash
python -m venv .venv
source .venv/bin/activate
pip install numpy pandas wrds
````

## Usage

Typical flow:

1. Fetch available expirations (per date)
2. Build near/next schedule that brackets 30 days (within 23–37 day eligibility)
3. Pull quotes for those expirations
4. Pull `zerocd` and interpolate rates by days-to-expiration
5. Compute the VIX series

Minimal example:

```python
import wrds

start_date = "2025-02-10"
end_date   = "2025-08-29"

db = wrds.Connection()

cal = get_expiry_calendar(db, start_date, end_date, secid=SPX_SECID)
schedule = build_term_schedule(cal, target_minutes=N30)

quotes = quotes_for_schedule(db, schedule, secid=SPX_SECID)
zerocd = fetch_zerocd_range(db, start_date, end_date)

vix_df = compute_vix_series(schedule, quotes, zerocd, republish_on_fail=True)
print(vix_df.head())
```

## Validation against Cboe close (test span)

Test window: 2025-02-10 to 2025-08-29

* Overlap days: 130 (computed) vs 144 (official series points available)
* MAE: 0.175
* RMSE: 0.291
* Correlation: 0.9992

Plot:

Output columns:

* `date`, `vix`
* `republished`, `error`
* `near_exdate`, `next_exdate`, `N1`, `N2`

## Notes

* Timestamps are US/Eastern:

  * calculation: 16:00 ET
  * expiration: 09:30 ET (AM-settled) or 16:00 ET (PM-settled)
* Fail handling: optional “republish last valid” behavior when a day cannot be computed.