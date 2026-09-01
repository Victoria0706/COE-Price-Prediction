from google.colab import files
upload = files.upload()

# Initializing 
import pandas as pd

filename = "COEBiddingResultsPrices.xlsx"

df = pd.read_excel(filename, engine="openpyxl")
print(df.head())

import numpy as np
import pandas as pd
import seaborn as sns
import matplotlib.pyplot as plt

sns.set_theme(style="whitegrid")


# Work on a copy to avoid modifying the original dataframe unexpectedly.
df = df.copy()


# 2. Check that important columns exist

required_columns = [
    "month",
    "bidding_no",
    "vehicle_class",
    "quota",
    "bids_success",
    "bids_received",
    "premium",
]

missing_columns = [
    column
    for column in required_columns
    if column not in df.columns
]

if missing_columns:
    raise ValueError(f"Missing columns: {missing_columns}")


# 3. Convert data types

df["month"] = pd.to_datetime(
    df["month"],
    format="%Y-%m",
    errors="coerce",
)

numeric_columns = [
    "quota",
    "bids_success",
    "bids_received",
    "premium",
]

for column in numeric_columns:
    # Remove commas and spaces before numeric conversion.
    df[column] = (
        df[column]
        .astype("string")
        .str.replace(",", "", regex=False)
        .str.strip()
    )

    df[column] = pd.to_numeric(
        df[column],
        errors="coerce",
    )


# 4. Remove missing observations
df = df.dropna(
    subset=[
        "month",
        "vehicle_class",
        "quota",
        "bids_success",
        "bids_received",
        "premium",
    ]
).copy()


# 5. Remove invalid numeric observations
valid_observations = (
    (df["quota"] > 0)
    & (df["bids_success"] > 0)
    & (df["bids_received"] > 0)
    & (df["premium"] > 0)
)

df = df.loc[valid_observations].copy()


# 6. Standardise text columns
text_columns = [
    "vehicle_class",
    "bidding_no",
]

for column in text_columns:
    df[column] = (
        df[column]
        .astype("string")
        .str.strip()
        .str.upper()
    )


# 7. Sort the data
df = df.sort_values(
    by=["month", "bidding_no", "vehicle_class"]
).reset_index(drop=True)


# 8. Inspect the cleaned data
df.info()

print("\nFirst five rows:")
print(df.head())

print("\nDataset shape:")
print(df.shape)

print("\nVehicle classes:")
print(df["vehicle_class"].unique())

print("\nMissing values:")
print(df[required_columns].isna().sum())

# Monthly COE Premium Trends by Vehicle Category, 2010 to 2026
category_colours = {
    "A": "#4C78A8",
    "B": "#F58518",
    "C": "#54A24B",
    "D": "#E45756",
    "E": "#B279A2",
}

plt.figure(figsize=(16, 8))

sns.lineplot(
    data=monthly_premium,
    x="month",
    y="average_premium",
    hue="vehicle_class",
    hue_order=categories,
    palette=category_colours,
    linewidth=1.8,
)

plt.title(
    "Monthly COE Premium Trends by Vehicle Category, 2010–2026",
    fontsize=16,
    fontweight="bold",
)

plt.xlabel("Year")
plt.ylabel("Average COE Premium (SGD)")

plt.gca().yaxis.set_major_formatter(
    mtick.StrMethodFormatter("${x:,.0f}")
)

plt.xlim(
    pd.Timestamp("2010-01-01"),
    pd.Timestamp("2026-12-31"),
)

plt.legend(
    title="COE Category",
    bbox_to_anchor=(1.02, 1),
    loc="upper left",
)

plt.tight_layout()
plt.show()

# Annual Excess Demand by COE Category Categories A to E from 2010 to 2026

import pandas as pd
import matplotlib.pyplot as plt

# Copy existing data
data = df.copy()

# Make column names consistent
data.columns = (
    data.columns.astype(str)
    .str.lower()
    .str.strip()
    .str.replace(" ", "_")
)

print("Available columns:", data.columns.tolist())

# Use either bids_success or bids_successful, depending on your dataset
if "bids_success" in data.columns:
    successful_column = "bids_success"
elif "bids_successful" in data.columns:
    successful_column = "bids_successful"
else:
    raise ValueError(
        "Cannot find successful-bids column. "
        "Expected 'bids_success' or 'bids_successful'."
    )

# Clean data fields
data["month"] = pd.to_datetime(
    data["month"],
    errors="coerce"
)

data["bids_received"] = pd.to_numeric(
    data["bids_received"],
    errors="coerce"
)

data[successful_column] = pd.to_numeric(
    data[successful_column],
    errors="coerce"
)

# Extract vehicle category letter: A, B, C, D, or E
data["category"] = (
    data["vehicle_class"]
    .astype(str)
    .str.upper()
    .str.extract(r"\b([A-E])\b", expand=False)
)

data["year"] = data["month"].dt.year

# Calculate excess demand for each bidding exercise
data["excess_demand"] = (
    data["bids_received"]
    - data[successful_column]
)

# Keep valid records between 2010 and 2026
plot_data = data.loc[
    data["year"].between(2010, 2026)
    & data["category"].isin(["A", "B", "C", "D", "E"])
    & data["bids_received"].notna()
    & data[successful_column].notna()
].copy()

# Sum Bidding 1 and Bidding 2 excess demand within each year
annual_excess_demand = (
    plot_data.groupby(
        ["year", "category"],
        as_index=False
    )["excess_demand"]
    .sum()
)

display(annual_excess_demand.head(10))

colours = {
    "A": "#1565C0",  # Blue
    "B": "#E65100",  # Orange
    "C": "#2E7D32",  # Green
    "D": "#C62828",  # Red
    "E": "#6A1B9A",  # Purple
}

markers = {
    "A": "o",
    "B": "s",
    "C": "^",
    "D": "D",
    "E": "P",
}

fig, ax = plt.subplots(figsize=(28, 14))

for category in ["A", "B", "C", "D", "E"]:

    category_data = annual_excess_demand.loc[
        annual_excess_demand["category"] == category
    ].sort_values("year")

    if category_data.empty:
        print(f"No data found for Category {category}.")
        continue

    ax.plot(
        category_data["year"],
        category_data["excess_demand"],
        label=f"Category {category}",
        color=colours[category],
        marker=markers[category],
        linewidth=4,
        markersize=12,
    )

ax.set_title(
    "Annual Excess Demand by COE Category\n"
    "Categories A–E, 2010–2026",
    fontsize=30,
    fontweight="bold",
    pad=25,
)

ax.set_xlabel(
    "Year",
    fontsize=22,
    fontweight="bold",
)

ax.set_ylabel(
    "Annual Excess Demand\n(Bids Received − Bids Successful)",
    fontsize=22,
    fontweight="bold",
)

ax.set_xticks(range(2010, 2027))

ax.tick_params(
    axis="x",
    rotation=45,
    labelsize=16,
)

ax.tick_params(
    axis="y",
    labelsize=16,
)

ax.grid(
    linestyle="--",
    linewidth=1.3,
    alpha=0.4,
)

ax.legend(
    title="Vehicle Category",
    title_fontsize=18,
    fontsize=17,
    loc="upper left",
    frameon=True,
)

plt.tight_layout()

plt.savefig(
    "annual_excess_demand_categories_A_to_E.png",
    dpi=300,
    bbox_inches="tight",
)

plt.show()

# Annual Excess Supply by COE Category Categories A to E from 2010 to 2026

import pandas as pd
import matplotlib.pyplot as plt


# 1. Copy and prepare data

data = df.copy()

data.columns = (
    data.columns.astype(str)
    .str.lower()
    .str.strip()
    .str.replace(" ", "_")
)

print("Available columns:", data.columns.tolist())

# Find the successful-bids column
if "bids_success" in data.columns:
    successful_column = "bids_success"
elif "bids_successful" in data.columns:
    successful_column = "bids_successful"
else:
    raise ValueError(
        "Cannot find successful-bids column. "
        "Expected 'bids_success' or 'bids_successful'."
    )


# 2. Clean needed fields
data["month"] = pd.to_datetime(
    data["month"],
    errors="coerce",
)

data["quota"] = pd.to_numeric(
    data["quota"],
    errors="coerce",
)

data[successful_column] = pd.to_numeric(
    data[successful_column],
    errors="coerce",
)

# Extract category letter from names such as:
# "Category A", "A", "Cat B", etc.
data["category"] = (
    data["vehicle_class"]
    .astype(str)
    .str.upper()
    .str.extract(r"\b([A-E])\b", expand=False)
)

data["year"] = data["month"].dt.year


# 3. Calculate excess supply
data["excess_supply"] = (
    data["quota"]
    - data[successful_column]
)

# Retain valid observations from 2010 to 2026
plot_data = data.loc[
    data["year"].between(2010, 2026)
    & data["category"].isin(["A", "B", "C", "D", "E"])
    & data["quota"].notna()
    & data[successful_column].notna()
].copy()

# Combine Bidding 1 and Bidding 2 into annual total excess supply
annual_excess_supply = (
    plot_data.groupby(
        ["year", "category"],
        as_index=False,
    )["excess_supply"]
    .sum()
)

display(annual_excess_supply.head(10))

colours = {
    "A": "#1565C0",
    "B": "#E65100",
    "C": "#2E7D32",
    "D": "#C62828",
    "E": "#6A1B9A",
}

markers = {
    "A": "o",
    "B": "s",
    "C": "^",
    "D": "D",
    "E": "P",
}

fig, ax = plt.subplots(figsize=(28, 14))

for category in ["A", "B", "C", "D", "E"]:

    category_data = annual_excess_supply.loc[
        annual_excess_supply["category"] == category
    ].sort_values("year")

    if category_data.empty:
        print(f"No data found for Category {category}.")
        continue

    ax.plot(
        category_data["year"],
        category_data["excess_supply"],
        label=f"Category {category}",
        color=colours[category],
        marker=markers[category],
        linewidth=4,
        markersize=12,
    )

ax.axhline(
    y=0,
    color="black",
    linestyle="--",
    linewidth=2,
)

ax.set_title(
    "Annual Excess Supply by COE Category\n"
    "Categories A–E, 2010–2026",
    fontsize=30,
    fontweight="bold",
    pad=25,
)

ax.set_xlabel(
    "Year",
    fontsize=22,
    fontweight="bold",
)

ax.set_ylabel(
    "Annual Excess Supply\n(Quota − Bids Successful)",
    fontsize=22,
    fontweight="bold",
)

ax.set_xticks(range(2010, 2027))

ax.tick_params(
    axis="x",
    rotation=45,
    labelsize=16,
)

ax.tick_params(
    axis="y",
    labelsize=16,
)

ax.grid(
    linestyle="--",
    linewidth=1.3,
    alpha=0.4,
)

ax.legend(
    title="Vehicle Category",
    title_fontsize=18,
    fontsize=17,
    loc="best",
    frameon=True,
)

plt.tight_layout()

plt.savefig(
    "annual_excess_supply_categories_A_to_E.png",
    dpi=300,
    bbox_inches="tight",
)

plt.show()

# Average Monthly COE Premium by Category from 2010 to 2026

monthly_seasonality = (
    seasonal_data
    .groupby(
        ["category", "month_number", "month_name"],
        observed=True,
        as_index=False
    )
    .agg(
        average_premium=("premium", "mean"),
        median_premium=("premium", "median"),
        premium_std=("premium", "std"),
        observations=("premium", "count")
    )
    .sort_values(["category", "month_number"])
)

display(monthly_seasonality.head(15))

colours = {
    "A": "#1565C0",
    "B": "#E65100",
    "C": "#2E7D32",
    "D": "#C62828",
    "E": "#6A1B9A"
}

markers = {
    "A": "o",
    "B": "s",
    "C": "^",
    "D": "D",
    "E": "P"
}

fig, ax = plt.subplots(figsize=(24, 12))

for category in category_order:
    category_data = monthly_seasonality.loc[
        monthly_seasonality["category"] == category
    ].sort_values("month_number")

    if category_data.empty:
        print(f"No records found for Category {category}.")
        continue

    ax.plot(
        category_data["month_name"].astype(str),
        category_data["average_premium"],
        label=f"Category {category}",
        color=colours[category],
        marker=markers[category],
        linewidth=3.5,
        markersize=10
    )

ax.set_title(
    "Average Monthly COE Premium by Category, 2010–2016",
    fontsize=26,
    fontweight="bold",
    pad=20
)

ax.set_xlabel(
    "Calendar Month",
    fontsize=18,
    fontweight="bold"
)

ax.set_ylabel(
    "Average COE Premium",
    fontsize=18,
    fontweight="bold"
)

ax.tick_params(axis="both", labelsize=14)

ax.grid(
    linestyle="--",
    alpha=0.4
)

ax.legend(
    title="COE Category",
    title_fontsize=15,
    fontsize=14,
    ncol=5
)

plt.tight_layout()

plt.savefig(
    "monthly_seasonal_premium_categories_A_to_E_2010_2016.png",
    dpi=300,
    bbox_inches="tight"
)

plt.show()

# Annual COE Bid Pressure by Category from 2010 to 2026 

# 1. Filter data
bid_data = data.loc[
    data["year"].between(START_YEAR, END_YEAR)
    & data["category"].isin(CATEGORIES)
    & data["quota"].notna()
    & data["bids_received"].notna()
    & (data["quota"] > 0)
].copy()

if bid_data.empty:
    raise ValueError(
        "No valid observations were found after filtering. "
        "Check the date, category, quota, and bids_received columns."
    )

# Bid pressure for each bidding exercise
bid_data["bid_pressure"] = (
    bid_data["bids_received"] / bid_data["quota"]
)

# Optional demand surplus relative to quota
bid_data["bids_above_quota"] = (
    bid_data["bids_received"] - bid_data["quota"]
)

print("\nFiltered period:")
print(
    bid_data["month"].min(),
    "to",
    bid_data["month"].max(),
)

print("\nObservations by category:")
print(
    bid_data["category"]
    .value_counts()
    .reindex(CATEGORIES, fill_value=0)
)

display(
    bid_data[
        [
            "month",
            "category",
            "quota",
            "bids_received",
            "bid_pressure",
        ]
    ].head(10)
)

# 2. Calculate annual bid pressure

annual_bid_pressure = (
    bid_data
    .groupby(
        ["year", "category"],
        as_index=False,
    )
    .agg(
        total_bids_received=("bids_received", "sum"),
        total_quota=("quota", "sum"),
        bidding_exercises=("bid_pressure", "count"),
    )
)

annual_bid_pressure["bid_pressure"] = (
    annual_bid_pressure["total_bids_received"]
    / annual_bid_pressure["total_quota"]
)

annual_bid_pressure["bids_above_quota"] = (
    annual_bid_pressure["total_bids_received"]
    - annual_bid_pressure["total_quota"]
)

annual_bid_pressure = annual_bid_pressure.sort_values(
    ["category", "year"]
)

display(annual_bid_pressure.head(15))

annual_bid_pressure.to_csv(
    table_folder / "annual_bid_pressure_categories_A_to_E_2010_2026.csv",
    index=False,
)

# 3. Combined annual bid-pressure chart
fig, ax = plt.subplots(figsize=(26, 13))

for category in CATEGORIES:
    category_data = annual_bid_pressure.loc[
        annual_bid_pressure["category"] == category
    ].sort_values("year")

    if category_data.empty:
        print(f"No valid data found for Category {category}.")
        continue

    ax.plot(
        category_data["year"],
        category_data["bid_pressure"],
        color=colours[category],
        marker=markers[category],
        linewidth=3.5,
        markersize=10,
        label=f"Category {category}",
    )

# Demand equals quota reference line
ax.axhline(
    y=1,
    color="black",
    linestyle="--",
    linewidth=2.5,
    label="Bids equal quota",
)

ax.set_title(
    "Annual COE Bid Pressure by Category, 2010–2026",
    fontsize=28,
    fontweight="bold",
    pad=22,
)

ax.set_xlabel(
    "Year",
    fontsize=19,
    fontweight="bold",
)

ax.set_ylabel(
    "Bid Pressure",
    fontsize=19,
    fontweight="bold",
)

ax.set_xticks(range(START_YEAR, END_YEAR + 1))

ax.tick_params(
    axis="x",
    labelsize=14,
    rotation=45,
)

ax.tick_params(
    axis="y",
    labelsize=14,
)

ax.grid(
    linestyle="--",
    linewidth=1.2,
    alpha=0.4,
)

ax.legend(
    title="COE Category",
    title_fontsize=16,
    fontsize=14,
    ncol=3,
    frameon=True,
)

plt.tight_layout()

plt.savefig(
    output_folder / "annual_bid_pressure_categories_A_to_E_2010_2026.png",
    dpi=300,
    bbox_inches="tight",
)

plt.show()
