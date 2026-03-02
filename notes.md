# VIX Reconstruction Data Sources on WRDS (OptionMetrics + Cboe)

This document summarizes which WRDS datasets to use to reconstruct VIX from options, what the relevant tables/fields contain, what the extracted data should look like in Python, and the key “gotchas” when pulling data via the `wrds` Python library.

---

## 1) Primary data sources

### 1.1 OptionMetrics IvyDB US (options + rates)
Use this to reconstruct a *VIX-style* index from an option strip.

**WRDS libraries (schemas) you have:**
- `optionm_all`

**Key tables:**
- `secnmd`
- `opprcdYYYY` (year-partitioned option prices)
- `zerocd` (zero curve)

#### `secnmd` — Security Name / ID Map
Purpose:
- Maps `secid` ↔ tickers/underlyings (e.g., SPX).
- Has `effect_date` history; you should select the latest `effect_date <= trade_date`.

Typical usage:
- Find the SPX `secid` “as of” each trade date.

#### `opprcdYYYY` — Option Prices (year-partitioned)
Purpose:
- End-of-day option quotes and related fields for a given year.
- Partitioned by year in your account, e.g. `opprcd1996`, …, `opprcd2022`, `opprcd2025`.

**Important fields (from your table description):**
- `secid`: underlying identifier
- `date`: trade date
- `exdate`: option expiration date
- `cp_flag`: `C` or `P`
- `strike_price`: **strike × 1000**
- `best_bid`: highest closing bid (across exchanges)
- `best_offer`: lowest closing ask (across exchanges)
- `am_settlement`: 1 = AM-settled, 0 = PM-settled
- `expiry_indicator`: expiry type code (e.g., regular/weekly/daily/EOM)
- `ss_flag`: settlement flag (`0` = standard; others = non-standard)
- `contract_size`: contract multiplier (typically 100)
- (optional for debugging) `optionid`, `symbol`, `root`, `forward_price`

What this means:
- Quotes are **best across all exchanges** (not Cboe-only).
- Quotes are a fixed end-of-day snapshot (time convention varies by era; modern history uses a consistent snapshot time).

#### `zerocd` — Zero Curve
Purpose:
- Continuously-compounded zero rates by maturity (days) for each curve date.

Fields:
- `date`: curve date
- `days`: days to maturity
- `rate`: continuously compounded zero-coupon rate (often reported in **percent**; check scaling)

---

### 1.2 Cboe VIX Index (for comparison)
Use this to compare your reconstructed series to the “official” VIX.

**WRDS dataset:**
- Cboe index data is typically in a `cboe_all` schema on WRDS (availability depends on subscription).

What you want:
- Daily VIX close (and possibly open/high/low if present) to validate reconstruction.

Note:
- Your reconstructed VIX from OptionMetrics will likely differ slightly from published VIX because:
  - OptionMetrics quotes are best-across-exchanges, not Cboe C1-only
  - timestamp conventions differ
  - rates methodology differs (Cboe uses Treasury CMT spline; OptionMetrics provides its own zero curve)

---

## 2) What the extracted data should look like (Python)

### 2.1 Options chain format (input to your VIX engine)

For each selected expiration (near and next term), you want a DataFrame like:

| column   | dtype   | description |
|---------|---------|-------------|
| `strike`| float   | strike in index points (convert from `strike_price / 1000`) |
| `cp_flag` | str   | `'C'` or `'P'` |
| `bid`   | float   | best bid |
| `ask`   | float   | best ask |

Example:
```text
   strike cp_flag    bid    ask
0  3600.0      C   75.2   76.0
1  3600.0      P   74.8   75.6
...
````

**Minimum cleaning expectations:**

* Drop rows with missing quotes
* Drop invalid markets where `bid > ask`
* Ensure `strike > 0`, `bid >= 0`, `ask >= 0`
* Deduplicate `(cp_flag, strike)` deterministically (rare but important; e.g., keep highest bid then lowest ask)

### 2.2 Candidate expirations format (for selecting near/next terms)

For a given trade date, pull distinct expirations and keep:

| column             | description                     |
| ------------------ | ------------------------------- |
| `trade_dt`         | trade date (constant per query) |
| `exdate`           | expiration date                 |
| `am_settlement`    | AM vs PM settle                 |
| `expiry_indicator` | regular/weekly/daily/etc        |
| `ss_flag`          | settlement type (keep standard) |
| `contract_size`    | keep 100                        |
| `minutes`          | minutes-to-expiry (computed)    |

Then filter to the VIX-eligible universe and pick the two expiries bracketing 30 days.

### 2.3 Zero curve format

For the curve date used:

| column | dtype | description                       |
| ------ | ----- | --------------------------------- |
| `days` | float | days to maturity                  |
| `rate` | float | continuously-compounded zero rate |

Interpolate linearly in `days` to get `R` for each term:

* `maturity_days = minutes_to_expiry / 1440`
* `R = interp(days, rate)` (often divide by 100 if rates are stored in percent)

---

## 3) What to consider when fetching via Python `wrds` library

### 3.1 Year-partitioned `opprcdYYYY`

**Your biggest structural constraint:**

* There is no `opprcd` table; you must query `opprcd{year}` based on trade date.

Example:

* For trade date `2025-06-02` query `optionm_all.opprcd2025`

### 3.2 Avoiding SQL wildcard pitfalls in `raw_sql`

Some environments can throw SQLAlchemy/pandas errors when using `%` in raw SQL strings.
Safer approaches:

* Use `db.list_tables(lib)` and filter in Python
* If you must query patterns, parameterize the pattern or escape `%` as `%%`

### 3.3 Date handling

When passing dates in `params`, use Python `date` objects:

```python
params={"d": pd.to_datetime("2025-06-02").date()}
```

This avoids timezone parsing issues.

### 3.4 Standard-contract filters

When pulling option rows from `opprcdYYYY`, apply:

* `ss_flag = '0'` (standard)
* `contract_size = 100`
  These reduce contamination from adjusted/nonstandard deliverables.

### 3.5 Strike scaling

Always convert:

* `strike = strike_price / 1000.0`

### 3.6 Rate scaling

`zerocd.rate` is continuously compounded, but in many setups it is stored in **percent units** (e.g., `4.5` means 4.5%).
Sanity check:

* If typical values are ~2–6, divide by 100.
* If typical values are ~0.02–0.06, keep as-is.

### 3.7 Expiration selection gotchas

To reconstruct VIX, you must:

* select near/next expirations that **bracket 30 days**
* use the correct VIX-eligible expiration universe:

  * AM-settled “regular” SPX expiries
  * PM-settled end-of-week SPXW expiries
  * exclude SPXW that share an exdate with AM-settled SPX

**Holiday issue:**

* End-of-week expirations may shift from Friday to Thursday when Friday is a market holiday.
  A robust selector should:
* prefer PM Friday expiries
* if Friday PM expiry does not exist, allow Thursday PM expiry
* avoid using third-Friday SPXW if that date coincides with an AM-settled SPX expiry

### 3.8 Methodology regime break (Feb 10, 2025)

Cboe’s strike truncation rule changed:

* Pre-change: tail truncation based on **two consecutive zero bids**
* Post-change: based on **two consecutive zero bid OR zero ask**
  If you compute long history, implement a date-based switch.

### 3.9 Performance considerations

Queries over `opprcdYYYY` can be large.
Best practice:

* First query `distinct exdate, am_settlement, expiry_indicator...` for a date (cheap)
* Select only two expirations (near/next)
* Then query chains for those expirations only (small)

---

## 4) Minimal “what to query” checklist

### For each trade date `d`:

1. Get `secid` for SPX as-of `d` from `secnmd`
2. Query `opprcdYYYY` for distinct expirations on `d`
3. Compute minutes-to-expiry and filter to VIX-eligible expirations
4. Pick near/next expirations bracketing 30 days
5. Pull option chains for both expirations from `opprcdYYYY`:

   * fields: `cp_flag, strike_price, best_bid, best_offer`
   * filters: `ss_flag='0'`, `contract_size=100`
6. Pull zero curve from `zerocd` for date `d` (or most recent date <= d)
7. Interpolate `R_near`, `R_next` and feed into VIX calculation

---

## 5) Typical outputs you should store for diagnostics

Even if you only need daily VIX, storing intermediate values makes validation easy:

* selected expirations, minutes-to-expiry, and AM/PM flags
* `K_atm`, forward `F`, `K0`
* number of strikes included in each term’s `Q(K)` table
* term variances `sigma2_near`, `sigma2_next`
* final interpolated VIX

These diagnostics let you distinguish:

* ingestion problems (wrong expiries, wrong flags, wrong strike scaling)
* methodology mismatches (wrong truncation rule for date)
* input differences vs published VIX (quote source/timestamp/rate curve differences)
