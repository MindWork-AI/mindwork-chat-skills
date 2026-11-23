
# Data analysis and charts

Read this when the user hands over data and wants an answer, a table or a chart. Sessions, input files and publishing are covered in `SKILL.md`.

The running example: two years of daily sales revenue that should become a quarterly trend, a chart and a
few sentences of interpretation.

## 1. Bring the data in

Pass an uploaded file as `url`, or small data as `content_text`:
`input_files=[{"name": "data/sales.csv", "url": "..."}]`. Large files must come by `url`, not pasted.
Public data can be downloaded in code. If you have no usable link for an attachment, ask for one.

## 2. Look before you compute

The first call only loads the data and prints shape, types, a few rows and missing values. Real files
have decimal commas, header rows in the middle, dates as text and totals mixed in with the data.

```python
import pandas as pd
df = pd.read_csv("data/sales.csv", parse_dates=["date"])
print(df.shape); print(df.dtypes); print(df.head()); print(df.isna().sum())
```

Fix the parsing (`decimal=","`, `sep=";"`, `skiprows`, `thousands`) before any aggregate.

## 3. Keep results in files, not variables

Every call is a fresh process, so DataFrames are gone after it. Save what the next step needs; Parquet keeps types:

```python
q = df.groupby(df.date.dt.to_period("Q")).revenue.sum()
q.to_frame().to_parquet("quarterly.parquet")
print(q.tail())
```

Print summaries, not whole tables: output is cut to about 16,000 characters (start and end). For SQL over big
files without loading them: `chdb.query("SELECT ... FROM file('data/sales.csv') GROUP BY ...", "PrettyCompact")`.

## 4. Charts

matplotlib writes PNGs you can look at yourself:

```python
import matplotlib.pyplot as plt, pandas as pd
q = pd.read_parquet("quarterly.parquet")
fig, ax = plt.subplots(figsize=(8, 4.5), dpi=150)
ax.plot(q.index.astype(str), q.revenue, marker="o", linewidth=2)
ax.set_title("Revenue by quarter"); ax.spines[["top", "right"]].set_visible(False)
fig.tight_layout(); fig.savefig("revenue.png")
```

Then `sandbox_read_file("revenue.png")` and check as a reader would: readable labels, honest axis (bars start
at zero), a title that says what it shows, a legend only for several series. For an interactive chart use
plotly: `fig.write_html("revenue.html", include_plotlyjs="cdn")`. Native editable charts for decks come from
pptxgenjs (see [documents.md](documents.md)).

## 5. Memory and time

2 GB RAM and up to 300 s per call. On `status: killed`, read in chunks (`chunksize=200_000`), select columns
(`usecols=`), use chdb or downcast types. On `timeout`, split the work and save intermediates between calls.

## 6. Hand over

Answer the question in words first, with the numbers that matter and any caveat (missing months, odd rows
you dropped). Then publish what supports it, such as `paths=["revenue.png", "quarterly.xlsx"]`. Do not publish
Parquet files or scratch CSVs. State assumptions you made when cleaning the data.
