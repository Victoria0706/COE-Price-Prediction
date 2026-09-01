# coe_model_simplified_fixed.py
import re
from pathlib import Path

import numpy as np
import pandas as pd
import statsmodels.api as sm


# Parsing utilities

MONTH_ABBR = {'Jan': 1, 'Feb': 2, 'Mar': 3, 'Apr': 4,
              'May': 5, 'Jun': 6, 'Jul': 7, 'Aug': 8,
              'Sep': 9, 'Oct': 10, 'Nov': 11, 'Dec': 12}

def parse_period(label: str) -> pd.Timestamp:
    m = re.match(r'^(\d{4})([A-Za-z]{3})$', str(label))
    if not m:
        raise ValueError(f"Unexpected period label: {label}")
    year = int(m.group(1))
    mon = MONTH_ABBR[m.group(2).title()]
    return pd.Timestamp(year=year, month=mon, day=1)

def load_long(xlsx_path: str | Path) -> pd.DataFrame:
    dfw = pd.read_excel(xlsx_path, header=0)
    period_cols = [c for c in dfw.columns if re.match(r'^\d{4}[A-Za-z]{3}$', str(c))]
    if not period_cols:
        raise ValueError("No 'YYYYMon' columns found in the sheet.")
    series_col = dfw.columns[0]
    dfw = dfw[[series_col] + period_cols].copy()
    dfl = dfw.melt(id_vars=series_col, var_name='period', value_name='value')
    dfl['date'] = dfl['period'].apply(parse_period)
    dfl = dfl.rename(columns={series_col: 'series'}).drop(columns=['period'])
    dfl['value'] = pd.to_numeric(dfl['value'], errors='coerce')
    return dfl

def find_series(dfl: pd.DataFrame, tokens: list[str]) -> pd.Series:
    s = dfl['series'].str.lower().fillna('')
    mask = np.ones(len(s), dtype=bool)
    for t in tokens:
        mask &= s.str.contains(t.lower(), regex=False)
    candidates = dfl.loc[mask, 'series'].unique()
    if len(candidates) == 0:
        raise KeyError(f"No series matched tokens: {tokens}")
    name = sorted(candidates, key=len, reverse=True)[0]
    ts = (dfl[dfl['series'] == name]
          .set_index('date')['value']
          .sort_index()
          .asfreq('MS'))
    return ts.rename(name)


# Model builder (log-OLS)
def build_design(price: pd.Series, quota: pd.Series, add_month_fe=True, lag1=True) -> pd.DataFrame:
    df = pd.concat({'price': price, 'quota': quota}, axis=1)
    df = df.replace([np.inf, -np.inf], np.nan).dropna()
    df = df[(df['price'] > 0) & (df['quota'] > 0)]
    df['ln_price'] = np.log(df['price'])
    df['ln_quota'] = np.log(df['quota'])
    if lag1:
        df['ln_price_lag1'] = df['ln_price'].shift(1)
    if add_month_fe:
        df['month'] = df.index.month.astype(int)
        month_fe = pd.get_dummies(df['month'], prefix='m', drop_first=True)
        df = pd.concat([df, month_fe], axis=1)
    df = df.dropna()
    return df

def fit_log_ols(df: pd.DataFrame, include_month_fe=True):
    X_cols = ['ln_quota', 'ln_price_lag1']
    if include_month_fe:
        X_cols += [c for c in df.columns if c.startswith('m_')]
    # Ensure numeric float dtypes
    X = df[X_cols].apply(pd.to_numeric, errors='coerce').astype(float)
    y = pd.to_numeric(df['ln_price'], errors='coerce').astype(float)
    X = sm.add_constant(X, has_constant='add')
    # Pass DataFrames/Series (not numpy arrays) so statsmodels retains column names
    model = sm.OLS(y, X, missing='drop')
    res = model.fit(cov_type='HC1')
    return res


# End-to-end runner

def main(xlsx_path: str | Path):
    dfl = load_long(xlsx_path)

    # Tokens (adjust if labels differ slightly in your file)
    # Category A: price (1st bidding) and quota (fallback: 2nd bidding)
    catA_price_tokens = ["Cars Up To 1600cc And 97kW", "Quota Premium", "1st Bidding"]
    catA_quota_options = [
        ["Cars Up To 1600cc And 97kW", "Quota", "1st Bidding"],
        ["Cars Up To 1600cc And 97kW", "Quota", "2nd Bidding"],
    ]
    # Category B: price (1st bidding) and quota (try 1st, fallback 2nd)
    catB_price_tokens = ["Cars Above 1600cc Or 97kW", "Quota Premium", "1st Bidding"]
    catB_quota_options = [
        ["Cars Above 1600cc Or 97kW", "Quota", "1st Bidding"],
        ["Cars Above 1600cc Or 97kW", "Quota", "2nd Bidding"],
    ]

    # Resolve series
    catA_price = find_series(dfl, catA_price_tokens)
    catA_quota = None
    for tok in catA_quota_options:
        try:
            catA_quota = find_series(dfl, tok)
            break
        except KeyError:
            continue
    if catA_quota is None:
        raise KeyError("Category A quota series not found. Adjust tokens.")

    catB_price = find_series(dfl, catB_price_tokens)
    catB_quota = None
    for tok in catB_quota_options:
        try:
            catB_quota = find_series(dfl, tok)
            break
        except KeyError:
            continue
    if catB_quota is None:
        raise KeyError("Category B quota series not found. Adjust tokens.")

    # Build designs
    A_df = build_design(catA_price, catA_quota, add_month_fe=True, lag1=True)
    B_df = build_design(catB_price, catB_quota, add_month_fe=True, lag1=True)

    # Fit and report
    print("=== Category A (log-OLS) ===")
    A_res = fit_log_ols(A_df, include_month_fe=True)
    print(A_res.summary())
    print(f"\nEstimated quota price elasticity (beta_A on ln_quota): {A_res.params['ln_quota']:.3f}")

    print("\n=== Category B (log-OLS) ===")
    B_res = fit_log_ols(B_df, include_month_fe=True)
    print(B_res.summary())
    print(f"\nEstimated quota price elasticity (beta_B on ln_quota): {B_res.params['ln_quota']:.3f}")

if __name__ == "__main__":
    xlsx_path = "MotorVehicleQuotaQuotaPremiumAndPrevailingQuotaPremiumMonthly.xlsx"
    main(xlsx_path)

    # coe_predictability_plots.py
import re
from pathlib import Path

import numpy as np
import pandas as pd
import statsmodels.api as sm
import matplotlib.pyplot as plt
import seaborn as sns

sns.set_style("whitegrid")

MONTH_ABBR = {'Jan': 1, 'Feb': 2, 'Mar': 3, 'Apr': 4,
              'May': 5, 'Jun': 6, 'Jul': 7, 'Aug': 8,
              'Sep': 9, 'Oct': 10, 'Nov': 11, 'Dec': 12}

def parse_period(label: str) -> pd.Timestamp:
    m = re.match(r'^(\d{4})([A-Za-z]{3})$', str(label))
    if not m:
        raise ValueError(f"Unexpected period label: {label}")
    year = int(m.group(1))
    mon = MONTH_ABBR[m.group(2).title()]
    return pd.Timestamp(year=year, month=mon, day=1)

def load_long(xlsx_path: str | Path) -> pd.DataFrame:
    dfw = pd.read_excel(xlsx_path, header=0)
    period_cols = [c for c in dfw.columns if re.match(r'^\d{4}[A-Za-z]{3}$', str(c))]
    if not period_cols:
        raise ValueError("No 'YYYYMon' columns found.")
    series_col = dfw.columns[0]
    dfw = dfw[[series_col] + period_cols].copy()
    dfl = dfw.melt(id_vars=series_col, var_name='period', value_name='value')
    dfl['date'] = dfl['period'].apply(parse_period)
    dfl = dfl.rename(columns={series_col: 'series'}).drop(columns=['period'])
    dfl['value'] = pd.to_numeric(dfl['value'], errors='coerce')
    return dfl

def find_series(dfl: pd.DataFrame, tokens: list[str]) -> pd.Series:
    s = dfl['series'].str.lower().fillna('')
    mask = np.ones(len(s), dtype=bool)
    for t in tokens:
        mask &= s.str.contains(t.lower(), regex=False)
    candidates = dfl.loc[mask, 'series'].unique()
    if len(candidates) == 0:
        raise KeyError(f"No series matched tokens: {tokens}")
    name = sorted(candidates, key=len, reverse=True)[0]
    ts = (dfl[dfl['series'] == name]
          .set_index('date')['value']
          .sort_index()
          .asfreq('MS'))
    return ts.rename(name)

def build_design(price: pd.Series,
                 quota: pd.Series | None,
                 add_month_fe=True,
                 lag1=True) -> pd.DataFrame:
    parts = {'price': price}
    if quota is not None:
        parts['quota'] = quota
    df = pd.concat(parts, axis=1)
    df = df.replace([np.inf, -np.inf], np.nan).dropna()
    df = df[df['price'] > 0]
    if 'quota' in df.columns:
        df = df[df['quota'] > 0]
    df['ln_price'] = np.log(df['price'])
    if 'quota' in df.columns:
        df['ln_quota'] = np.log(df['quota'])
    if lag1:
        df['ln_price_lag1'] = df['ln_price'].shift(1)
    if add_month_fe:
        df['month'] = df.index.month.astype(int)
        month_fe = pd.get_dummies(df['month'], prefix='m', drop_first=True)
        df = pd.concat([df, month_fe], axis=1)
    df = df.dropna()
    return df

def fit_log_ols(df: pd.DataFrame, include_month_fe=True):
    X_cols = ['ln_price_lag1']
    if 'ln_quota' in df.columns:
        X_cols = ['ln_quota'] + X_cols
    if include_month_fe:
        X_cols += [c for c in df.columns if c.startswith('m_')]
    X = df[X_cols].apply(pd.to_numeric, errors='coerce').astype(float)
    y = pd.to_numeric(df['ln_price'], errors='coerce').astype(float)
    X = sm.add_constant(X, has_constant='add')
    model = sm.OLS(y, X, missing='drop')
    res = model.fit(cov_type='HC1')
    return res, X_cols

def plot_predictability(df: pd.DataFrame,
                        res,
                        title_prefix: str,
                        outfile_prefix: str):
    # Align fitted on df index
    y_log = df['ln_price'].copy()
    yhat_log = pd.Series(res.fittedvalues, index=y_log.index)
    # Transform back to price scale (visual; ignores log-bias)
    y = np.exp(y_log)
    yhat = np.exp(yhat_log)

    # Metrics
    r2 = res.rsquared
    mape = np.mean(np.abs((y - yhat) / y)) * 100.0

    # 1) Time-series overlay
    plt.figure(figsize=(11, 5))
    plt.plot(y.index, y.values, label="Actual price", color="#1f77b4")
    plt.plot(yhat.index, yhat.values, label="Fitted price", color="#ff7f0e", alpha=0.9)
    plt.title(f"{title_prefix} — Actual vs Fitted (price scale)\nR^2={r2:.3f} (log scale), MAPE={mape:.1f}%")
    plt.ylabel("Price")
    plt.xlabel("Year")
    plt.legend()
    plt.tight_layout()
    plt.savefig(f"{outfile_prefix}_timeseries.png", dpi=150)
    plt.show()

    # 2) Scatter actual vs fitted with 45-degree line
    plt.figure(figsize=(6, 6))
    sns.scatterplot(x=y.values, y=yhat.values, s=24, alpha=0.7)
    lims = [0, max(np.nanmax(y.values), np.nanmax(yhat.values)) * 1.05]
    plt.plot(lims, lims, 'k--', linewidth=1, label="45° line")
    plt.xlim(lims); plt.ylim(lims)
    plt.xlabel("Actual price")
    plt.ylabel("Fitted price")
    plt.title(f"{title_prefix} — Actual vs Fitted Scatter\nR^2={r2:.3f} (log scale)")
    plt.legend()
    plt.tight_layout()
    plt.savefig(f"{outfile_prefix}_scatter.png", dpi=150)
    plt.show()

def main(xlsx_path: str | Path):
    dfl = load_long(xlsx_path)

    # Category A (price: 1st bidding). Quota may be unavailable; we’ll fit AR(1)+month FE if so.
    catA_price_tokens = ["Cars Up To 1600cc And 97kW", "Quota Premium", "1st Bidding"]
    catA_price = find_series(dfl, catA_price_tokens)
    catA_quota = None
    try:
        # Try common quota labels if present in your file
        catA_quota = find_series(dfl, ["Cars Up To 1600cc And 97kW", "Quota", "1st Bidding"])
    except KeyError:
        try:
            catA_quota = find_series(dfl, ["Cars Up To 1600cc And 97kW", "Quota", "2nd Bidding"])
        except KeyError:
            catA_quota = None  # proceed without quota

    A_df = build_design(catA_price, catA_quota, add_month_fe=True, lag1=True)
    A_res, _ = fit_log_ols(A_df, include_month_fe=True)
    plot_predictability(A_df, A_res,
                        title_prefix="Category A (≤1600cc/97kW)",
                        outfile_prefix="catA_predictability")

    # Category B (price: 1st bidding; quota: try 2nd bidding first, else 1st)
    catB_price_tokens = ["Cars Above 1600cc Or 97kW", "Quota Premium", "1st Bidding"]
    catB_price = find_series(dfl, catB_price_tokens)
    catB_quota = None
    for tok in [
        ["Cars Above 1600cc Or 97kW", "Quota", "2nd Bidding"],
        ["Cars Above 1600cc Or 97kW", "Quota", "1st Bidding"],
    ]:
        try:
            catB_quota = find_series(dfl, tok)
            break
        except KeyError:
            continue
    if catB_quota is None:
        raise KeyError("Category B quota series not found.")

    B_df = build_design(catB_price, catB_quota, add_month_fe=True, lag1=True)
    B_res, _ = fit_log_ols(B_df, include_month_fe=True)
    plot_predictability(B_df, B_res,
                        title_prefix="Category B (>1600cc/97kW)",
                        outfile_prefix="catB_predictability")

if __name__ == "__main__":
    xlsx_path = "MotorVehicleQuotaQuotaPremiumAndPrevailingQuotaPremiumMonthly.xlsx"
    main(xlsx_path)
