# Python Data Science & Data Analytics — Complete Interview Prep
### Whiteboard-Style Notes: WHAT → WHY → HOW → CODE → INTERPRETATION → OUTPUT → BUSINESS MEANING → FOLLOW-UP

> **How to use this document:** Core topics (Parts 1–14) get the full deep treatment on representative questions. Parts 24–27 (the 150+/75+/50-question banks) are deliberately answered in a tight rapid-reference format — question + strong short answer + a one-line "why" — because doing the full 8-step treatment for every one of 150+ questions would make this unusable as a document. Where a question in those banks deserves deeper treatment, it's cross-referenced back to the Part where it's already covered in full.

---

## PART 1 — PYTHON FUNDAMENTALS

### Q1. What's the difference between mutable and immutable objects?
**SHORT ANSWER (20-30s):** Mutable objects (list, dict, set) can be changed in place after creation; immutable objects (int, float, str, tuple) cannot — any "change" creates a new object.

**DETAILED:** This matters for function arguments (passing a mutable default argument is a classic bug), for using objects as dict keys (only immutable/hashable objects can be keys), and for understanding `==` vs `is`.

```python
a = [1, 2, 3]
b = a
b.append(4)
print(a)
```
- `a = [1, 2, 3]` → creates a list object, `a` points to it.
- `b = a` → `b` points to the *same* object, not a copy.
- `b.append(4)` → mutates the shared object in place.
- `print(a)` → shows the mutation, because `a` and `b` reference the same list.

**OUTPUT:** `[1, 2, 3, 4]`
**INTERPRETATION:** This surprises beginners — assignment doesn't copy. **Business meaning:** in a data pipeline, if you pass a DataFrame or config dict into a function and mutate it "for convenience," you may silently corrupt the caller's original object — a real source of production bugs.
**FOLLOW-UP:** *"How would you avoid this?"* → Use `.copy()` (shallow) or `copy.deepcopy()` (deep), or simply avoid mutating inputs inside functions (write pure functions that return new objects).

### Q2. Shallow copy vs deep copy?
**SHORT ANSWER:** A shallow copy duplicates the outer container but keeps references to the same nested objects; a deep copy recursively duplicates everything, so nested objects are fully independent.
```python
import copy
original = [[1, 2], [3, 4]]
shallow = copy.copy(original)
deep = copy.deepcopy(original)

shallow[0].append(99)
print(original)   # [[1, 2, 99], [3, 4]]  <- nested list was shared, so it changed too
print(deep)        # [[1, 2], [3, 4]]      <- fully independent
```
**Business meaning:** when duplicating a nested config or a list of feature-engineering settings before experimenting, a shallow copy can silently leak changes back into the "original" — always confirm which copy type you need.
**FOLLOW-UP:** *"Does `list.copy()` do a shallow or deep copy?"* → Shallow.

### Q3. `*args` vs `**kwargs`?
**SHORT ANSWER:** `*args` collects extra positional arguments into a tuple; `**kwargs` collects extra keyword arguments into a dict — both let a function accept a flexible, unknown number of inputs.
```python
def summarize(*args, **kwargs):
    print("positional:", args)
    print("keyword:", kwargs)

summarize(1, 2, 3, name="revenue", unit="USD")
```
**OUTPUT:**
```
positional: (1, 2, 3)
keyword: {'name': 'revenue', 'unit': 'USD'}
```
**Business meaning:** useful for writing generic data-processing wrapper functions (e.g., a `run_query(*args, **kwargs)` that forwards flexible parameters to different underlying query functions).
**FOLLOW-UP:** *"Can you use both in the same function? In what order?"* → Yes: `def f(a, b, *args, c=1, **kwargs)` — positional-only, then `*args`, then keyword-only, then `**kwargs`.

### Q4. Local vs global variable scope?
**SHORT ANSWER:** A local variable exists only inside the function where it's defined; a global variable is defined at module level and accessible everywhere (read-only inside functions unless declared `global`).
```python
total = 0                      # global

def add_revenue(amount):
    global total                # without this line, `total += amount` raises UnboundLocalError
    total += amount
    return total

add_revenue(100)
print(total)   # 100
```
**FOLLOW-UP:** *"Why is overusing `global` considered bad practice?"* → Makes code hard to reason about/test since any function can silently mutate shared state; prefer passing values in and returning results out.

### Q5. Lambda functions — when to use, when not to?
**SHORT ANSWER:** A `lambda` is a small anonymous, single-expression function — great for short, throwaway logic passed to `sort`, `apply`, `map`, `filter`; bad for anything needing multiple statements or good readability/debuggability.
```python
df["Age_Group"] = df["Age"].apply(lambda x: "Young" if x < 30 else "Adult")
```
**Business meaning:** quick categorical bucketing for segmentation — but for logic with 3+ branches, a named function is more maintainable and testable.
**FOLLOW-UP:** *"Why not always use lambda instead of def?"* → No name (harder tracebacks), can't have a docstring, limited to one expression — reduces readability/testability at scale.

### Q6. Recursion — explain with an example
```python
def factorial(n):
    if n <= 1:                 # base case — stops the recursion
        return 1
    return n * factorial(n - 1)  # recursive case — calls itself with a smaller problem

factorial(5)   # 120
```
**FOLLOW-UP:** *"What happens without a base case?"* → `RecursionError: maximum recursion depth exceeded` — infinite recursion, stack overflow.

**Other Part 1 topics (rapid form — same question shape applies to all):** variables/data types (`int, float, str, bool`; Python is dynamically typed — type is inferred at runtime, checked with `type()`); type conversion (`int("5")`, `str(5)`, `float("5.5")` — fails with `ValueError` on non-numeric strings); operators (arithmetic, comparison, logical, `is` vs `==` — identity vs equality); `if/elif/else`, `for`, `while`, `break/continue/pass` (standard control flow — `pass` is a no-op placeholder, common while stubbing out function bodies); function `return` (without it, a function returns `None` implicitly — a very common source of "why is my variable `None`" bugs).

---

## PART 2 — PYTHON DATA STRUCTURES

### List vs Tuple vs Set vs Dictionary

| | List | Tuple | Set | Dictionary |
|---|---|---|---|---|
| Mutable? | Yes | No | Yes | Yes |
| Ordered? | Yes | Yes | No (insertion order preserved since 3.7 for dict, NOT guaranteed for set) | Yes (3.7+) |
| Duplicates? | Allowed | Allowed | Not allowed | Keys unique, values can repeat |
| Use case | Ordered, changeable collection | Fixed record, hashable (usable as dict key) | Fast membership tests, dedup | Key-value lookup |
| Example | `[1,2,3]` | `(1,2)` | `{1,2,3}` | `{"a":1}` |

**Interview trap:** *"Why use a tuple instead of a list?"* → Tuples are immutable (safer, e.g., function returns that shouldn't be modified by the caller), hashable (usable as dict keys), and slightly faster/more memory-efficient than lists.

### Key operations — data-analysis flavored examples
```python
sales = [120, 340, 220, 90, 500]
sales.append(150)          # add to end -> [120,340,220,90,500,150]
sales.sort(reverse=True)   # in-place sort desc -> [500,340,220,150,120,90]
top_3 = sorted(sales, reverse=True)[:3]   # sorted() returns a NEW list, doesn't mutate

customers = {"C1": 500, "C2": 300, "C3": 700}
for cust_id, revenue in customers.items():
    print(cust_id, revenue)

top_customer = max(customers, key=customers.get)   # key with highest value
```
**Business meaning of `max(customers, key=customers.get)`:** identifies the highest-revenue customer without writing a manual loop — a common one-liner in ad hoc analysis.

**`.get()` vs `[]` for dict access:** `.get("C4", 0)` returns a default (0) instead of raising `KeyError` if the key is missing — critical when merging/aggregating data with inconsistent keys.

---

## PART 3 — LIST COMPREHENSIONS & PYTHONIC CODE

```python
numbers = [10, 55, 30, 80, 20]
high_values = [x for x in numbers if x > 50]
```
**How Python executes it (conceptually unrolled):**
```python
high_values = []
for x in numbers:
    if x > 50:
        high_values.append(x)
```
**OUTPUT:** `[55, 80]`

**Dict/set comprehension:**
```python
squared = {x: x**2 for x in range(5)}          # {0:0, 1:1, 2:4, 3:9, 4:16}
unique_categories = {row["category"] for row in records}   # dedup via set comprehension
```
**Nested comprehension:**
```python
matrix = [[1,2],[3,4]]
flattened = [val for row in matrix for val in row]   # [1,2,3,4]
```
**Readability/performance:** comprehensions are typically *faster* than an equivalent `for` loop with `.append()` (less interpreter overhead) and more Pythonic — but **when NOT to use them**: if the logic needs multiple statements, side effects, or more than ~2 levels of nesting, a comprehension hurts readability more than it helps — use a regular loop or `.apply()`/vectorized pandas instead.
**FOLLOW-UP:** *"Is a list comprehension always faster than `.apply()` in pandas?"* → Not necessarily faster than a fully vectorized pandas/numpy operation, but generally faster than `.apply()` with a Python-level function, since `.apply()` still loops in Python under the hood.

---

## PART 4 — FUNCTIONS

```python
def revenue_summary(df, group_col="Category", value_col="Purchase_Amount", agg="mean"):
    """Default arguments let callers override only what they need."""
    return df.groupby(group_col)[value_col].agg(agg)

revenue_summary(df)                          # uses all defaults
revenue_summary(df, group_col="Gender")       # overrides one default (keyword argument)
```
**`map`, `filter`, `enumerate`, `zip` — data-analysis use:**
```python
prices = [100, 200, 300]
discounted = list(map(lambda x: x * 0.9, prices))     # [90.0, 180.0, 270.0]
above_150 = list(filter(lambda x: x > 150, prices))     # [200, 300]

for i, price in enumerate(prices, start=1):              # (index, value) pairs, 1-indexed here
    print(f"Item {i}: {price}")

names = ["A", "B", "C"]
scores = [90, 85, 70]
paired = list(zip(names, scores))     # [('A',90), ('B',85), ('C',70)]
```
**`reduce` (from `functools`):**
```python
from functools import reduce
total = reduce(lambda acc, x: acc + x, prices)   # cumulative reduction -> 600
```
**FOLLOW-UP:** *"Would you use `map`/`filter` or a pandas vectorized operation in a real analysis?"* → Vectorized pandas/numpy, almost always — `map`/`filter`/`reduce` on plain Python lists are for general-purpose Python, not large tabular data, where pandas operations are implemented in C and far faster.

---

## PART 5 — NUMPY

### Why NumPy instead of Python lists?
**SHORT ANSWER:** NumPy arrays store homogeneous data in contiguous memory and use vectorized, compiled (C-level) operations — dramatically faster and more memory-efficient than looping over Python lists.

```python
import numpy as np
arr = np.array([1, 2, 3, 4, 5])
print(arr.shape, arr.size, arr.ndim, arr.dtype)
# (5,) 5 1 int64
```
- `.shape` → dimensions (rows, cols, ...). `.size` → total element count. `.ndim` → number of dimensions. `.dtype` → the single data type all elements share (this is *why* it's fast — no per-element type checking like a Python list).

### Vectorization
```python
prices = np.array([100, 200, 300])
discounted = prices * 0.9        # applied to EVERY element at once, no explicit loop
```
**Why faster:** the multiplication is executed in a single compiled C loop instead of Python bytecode looping element-by-element — avoids Python's per-iteration interpreter overhead.

### Broadcasting
```python
matrix = np.array([[1,2,3],[4,5,6]])
row_means = matrix.mean(axis=1)          # [2. 5.]  (mean across columns, per row)
centered = matrix - row_means[:, None]     # broadcasting: (2,3) - (2,1) -> (2,3)
```
**What broadcasting means:** NumPy automatically expands a smaller-shaped array across a larger one *without copying data*, following specific compatibility rules (dimensions match or one of them is 1) — this is what lets `matrix - row_means[:, None]` subtract a per-row mean from every element in that row in one operation.

### Aggregations
```python
data = np.array([10, 20, 30, 40, 50])
data.mean(), np.median(data), data.std(), data.var(), np.percentile(data, 90)
```
**FOLLOW-UP:** *"NumPy `std()` vs pandas `std()` — same result?"* → No! NumPy's default `ddof=0` (population std); pandas' default `ddof=1` (sample std) — a classic interview trap (see Part 28).

---

## PART 6 — PANDAS (Core Reference)

### Reading & first look
```python
df = pd.read_csv("sales.csv")
df.head()          # first 5 rows — sanity check the load
df.info()           # dtypes + non-null counts per column — spot missing data/wrong types fast
df.describe()        # numeric summary stats: count, mean, std, min, quartiles, max
df.shape             # (rows, cols)
df.dtypes            # column data types
```
**Business meaning of `.info()`/`.describe()` as a first step:** this is the fastest way to catch data quality problems (unexpected nulls, a numeric column loaded as object/string due to stray characters) *before* any analysis — skipping this step is a common junior-analyst mistake.

### Selection: `loc` vs `iloc`
```python
df.loc[0:2, "Category"]        # label-based: rows 0-2 INCLUSIVE, column by name
df.iloc[0:2, 1]                  # position-based: rows 0-1 (exclusive end), column by integer position
df[df["Purchase_Amount"] > 500]           # boolean filtering
df[(df["Gender"] == "Male") & (df["Purchase_Amount"] > 500)]   # multiple conditions — use & / | with parentheses, not and/or
df[df["Category"].isin(["Electronics", "Clothing"])]
df[df["Age"].between(18, 30)]
```
**Interview trap:** using Python's `and`/`or` instead of `&`/`|` on pandas boolean Series raises `ValueError: truth value of a Series is ambiguous` — because `and`/`or` expect a single boolean, not an element-wise array.

### Missing data
```python
df.isnull().sum()                 # count of missing values per column
df["Rating"].fillna(df["Rating"].median(), inplace=False)   # impute with median (robust to outliers)
df.fillna(method="ffill")           # forward fill — carry last valid value forward (good for time series)
df.dropna(subset=["Customer_ID"])    # drop rows only where a critical column is missing
```
**"Why median instead of mean?"** — median is robust to outliers/skew (e.g., a few very high purchase amounts won't distort it), whereas mean imputation with a skewed distribution biases the imputed value toward the outliers.
**"Why didn't you delete the missing rows?"** — if missingness is substantial (e.g., >5-10%) or not random (MNAR), dropping rows loses information and can bias the remaining sample; imputation preserves sample size while flagging the assumption made.

### Duplicates
```python
df.duplicated().sum()                       # count of exact duplicate rows
df.drop_duplicates(subset="Customer_ID", keep="first")   # keep first occurrence per customer
```

### Transformation
```python
df.rename(columns={"Purchase_Amount": "Revenue"})
df["Revenue"] = df["Revenue"].astype(float)
df["Category"] = df["Category"].replace({"Elec": "Electronics"})
df["Age_Group"] = df["Age"].apply(lambda x: "Young" if x < 30 else "Adult")
df = df.assign(Revenue_per_Item = df["Revenue"] / df["Quantity"])   # adds a column without mutating original in a chain
```

### Grouping — the core of most interview questions
```python
df.groupby("Category")["Revenue"].mean()                     # average revenue per category
df.groupby("Category").agg(avg_rev=("Revenue","mean"), count=("Revenue","size"))  # named aggregation, multiple stats
df.groupby("Customer_ID")["Revenue"].transform("sum")           # returns a Series SAME LENGTH as df — for row-level comparison to a group stat
df.groupby("Category").filter(lambda g: g["Revenue"].sum() > 10000)   # keeps only groups meeting a condition
```
**`apply` vs `transform` — a top interview question:** `apply` on a groupby can return a result of *any* shape (aggregated or not); `transform` **must return an output the same length as the input group** — used to broadcast a group statistic back onto every row (e.g., "% of category total" per row).

### Merging
```python
pd.merge(orders, customers, on="Customer_ID", how="left")     # SQL-style join
pd.concat([df_jan, df_feb], axis=0)                              # stack rows (or axis=1 for columns)
```
**`merge` vs `concat`:** `merge` combines DataFrames based on key column(s) like a SQL join; `concat` simply stacks DataFrames along an axis with no key-based matching — pick `merge` when combining related tables, `concat` when appending similarly-shaped chunks of the same data.

### Datetime
```python
df["Order_Date"] = pd.to_datetime(df["Order_Date"])
df["Month"] = df["Order_Date"].dt.month
df["DayOfWeek"] = df["Order_Date"].dt.dayofweek
monthly_revenue = df.set_index("Order_Date")["Revenue"].resample("M").sum()   # resample = time-based groupby
```

### String operations
```python
df["Gender"] = df["Gender"].str.strip().str.upper()   # standardize " Male ", "male" -> "MALE"
df[df["Product"].str.contains("phone", case=False, na=False)]
```

---

## PART 7 — 75 PANDAS INTERVIEW QUESTIONS (with code)

*Format: Question → Code → one-line business interpretation. (Full 8-step treatment already demonstrated for representative cases above; these apply the same thinking.)*

1. **Find missing values** → `df.isnull().sum()` — quantifies data quality per column.
2. **Remove duplicates** → `df.drop_duplicates()` — ensures one row = one true observation.
3. **Average salary by department** → `df.groupby("Dept")["Salary"].mean()` — compares departmental pay levels.
4. **Second-highest salary** → `df["Salary"].nlargest(2).iloc[-1]` or `df["Salary"].sort_values(ascending=False).drop_duplicates().iloc[1]` — the second form correctly handles ties at the top.
5. **Top 5 customers by revenue** → `df.groupby("Customer_ID")["Revenue"].sum().nlargest(5)`.
6. **Customers above-average purchase** → `df[df["Purchase_Amount"] > df["Purchase_Amount"].mean()]`.
7. **Male vs female revenue** → `df.groupby("Gender")["Revenue"].sum()`.
8. **Category-wise average purchase** → `df.groupby("Category")["Purchase_Amount"].mean()`.
9. **Highest bad-rate category** → `df.groupby("Category")["Default_Flag"].mean().idxmax()` — mean of a 0/1 flag = rate.
10. **Month-over-month revenue** → `df.groupby(df["Order_Date"].dt.to_period("M"))["Revenue"].sum().pct_change()`.
11. **Customers who purchased more than once** → `df["Customer_ID"].value_counts()` then filter `> 1`.
12. **Duplicate customer IDs** → `df[df["Customer_ID"].duplicated(keep=False)]`.
13. **Replace missing with median** → `df["col"].fillna(df["col"].median())`.
14. **Conditional imputation** (impute by group) → `df["Rating"] = df.groupby("Category")["Rating"].transform(lambda x: x.fillna(x.median()))` — more accurate than a global median since it respects category-level baselines.
15. **Merge two datasets** → `pd.merge(df1, df2, on="key", how="inner")`.
16. **merge vs concat** → see Part 6.
17. **loc vs iloc** → see Part 6.
18. **apply vs transform** → see Part 6.
19. **groupby vs pivot_table** → `pivot_table` is `groupby` + reshape into a wide table in one call — good when you want categories as columns, e.g., `df.pivot_table(index="Category", columns="Gender", values="Revenue", aggfunc="sum")`.
20. **Create bins** → `pd.cut(df["Age"], bins=[0,18,30,50,100], labels=["Teen","Young","Adult","Senior"])`.
21. **Percentage contribution** → `df["Revenue"] / df["Revenue"].sum() * 100`.
22. **Cumulative revenue** → `df.sort_values("Order_Date")["Revenue"].cumsum()`.
23. **Rank customers** → `df["Revenue_Rank"] = df["Revenue"].rank(ascending=False, method="dense")`.
24. **Rolling average** → `df.set_index("Order_Date")["Revenue"].rolling(window=7).mean()` — smooths daily noise into a 7-day trend.
25. **Detect outliers (IQR method)** →
```python
Q1, Q3 = df["Revenue"].quantile([0.25, 0.75])
IQR = Q3 - Q1
outliers = df[(df["Revenue"] < Q1 - 1.5*IQR) | (df["Revenue"] > Q3 + 1.5*IQR)]
```
26. **Find nulls in specific columns only** → `df[["A","B"]].isnull().sum()`.
27. **Fill nulls differently per column** → `df.fillna({"Rating": df["Rating"].median(), "Category": "Unknown"})`.
28. **Drop columns with too many nulls** → `df.dropna(thresh=len(df)*0.5, axis=1)` — drops columns missing >50% of data.
29. **Count unique values** → `df["Category"].nunique()`.
30. **List unique values** → `df["Category"].unique()`.
31. **Value counts as %** → `df["Category"].value_counts(normalize=True) * 100`.
32. **Sort by multiple columns** → `df.sort_values(["Category","Revenue"], ascending=[True, False])`.
33. **Reset index after filtering** → `df[df["Revenue"]>500].reset_index(drop=True)` — avoids leftover, non-contiguous index confusion downstream.
34. **Multi-column groupby** → `df.groupby(["Category","Gender"])["Revenue"].mean()`.
35. **Groupby with multiple aggregations** → `df.groupby("Category")["Revenue"].agg(["mean","sum","count"])`.
36. **Custom aggregation function** → `df.groupby("Category")["Revenue"].agg(lambda x: x.max()-x.min())` — range per group.
37. **Pivot table with margins (totals)** → `df.pivot_table(values="Revenue", index="Category", aggfunc="sum", margins=True)`.
38. **Cross-tabulation** → `pd.crosstab(df["Gender"], df["Category"])` — frequency table.
39. **Filter top N per group** → `df.groupby("Category").apply(lambda g: g.nlargest(3, "Revenue"))` — top 3 orders per category.
40. **Cumulative count per group** → `df.groupby("Customer_ID").cumcount() + 1` — order index of each customer's purchases.
41. **Difference from previous row** → `df["Revenue"].diff()`.
42. **% change between rows** → `df["Revenue"].pct_change()`.
43. **Shift a column (lag feature)** → `df["Prev_Month_Revenue"] = df["Revenue"].shift(1)`.
44. **Check if index is sorted/unique** → `df.index.is_monotonic_increasing`, `df.index.is_unique`.
45. **Rename multiple columns via dict** → `df.rename(columns={"old1":"new1","old2":"new2"})`.
46. **Change all column names to lowercase** → `df.columns = df.columns.str.lower()`.
47. **Select columns by dtype** → `df.select_dtypes(include="number")`.
48. **Apply function to multiple columns** → `df[["A","B"]] = df[["A","B"]].apply(lambda col: col.fillna(col.mean()))`.
49. **Combine two string columns** → `df["Full_Name"] = df["First"] + " " + df["Last"]`.
50. **Extract substring/pattern** → `df["Domain"] = df["Email"].str.extract(r"@(.+)$")`.
51. **Split a column into multiple** → `df[["First","Last"]] = df["Full_Name"].str.split(" ", expand=True)`.
52. **Filter rows containing a substring** → `df[df["Product"].str.contains("Pro", na=False)]`.
53. **Find rows with any null** → `df[df.isnull().any(axis=1)]`.
54. **Drop a column** → `df.drop(columns=["col"])`.
55. **Conditional column with np.select for multiple conditions** →
```python
conditions = [df["Revenue"]>1000, df["Revenue"]>500]
choices = ["High","Medium"]
df["Tier"] = np.select(conditions, choices, default="Low")
```
56. **Convert categorical to numeric (label encoding)** → `df["Category_Code"] = df["Category"].astype("category").cat.codes`.
57. **One-hot encode** → `pd.get_dummies(df, columns=["Category"])`.
58. **Group and get first/last row per group** → `df.groupby("Customer_ID").first()` / `.last()`.
59. **Find customers with only ONE purchase** → `df["Customer_ID"].value_counts()[lambda x: x==1].index`.
60. **Weighted average** → `(df["Revenue"]*df["Weight"]).sum() / df["Weight"].sum()`.
61. **Calculate correlation matrix** → `df.corr(numeric_only=True)`.
62. **Find highly correlated feature pairs** → filter the correlation matrix for `abs(corr) > 0.8` excluding the diagonal.
63. **Memory usage per column** → `df.memory_usage(deep=True)`.
64. **Downcast numeric dtypes to save memory** → `pd.to_numeric(df["col"], downcast="integer")`.
65. **Read only specific columns from CSV** → `pd.read_csv("file.csv", usecols=["A","B"])` — faster, less memory.
66. **Read large file in chunks** → `for chunk in pd.read_csv("file.csv", chunksize=100000): process(chunk)`.
67. **Query syntax alternative to boolean filter** → `df.query("Revenue > 500 and Gender == 'Male'")` — often more readable for complex conditions.
68. **Explode a list-valued column** → `df.explode("tags_list")` — one row per list element.
69. **Melt (wide to long)** → `pd.melt(df, id_vars=["Customer_ID"], value_vars=["Jan","Feb"])`.
70. **Pivot (long to wide)** → `df.pivot(index="Customer_ID", columns="Month", values="Revenue")`.
71. **Apply a function row-wise using multiple columns** → `df.apply(lambda row: row["Revenue"]*row["Discount"], axis=1)` — slower than a vectorized `df["Revenue"]*df["Discount"]`; prefer vectorization when possible.
72. **Check for infinite values** → `np.isinf(df["col"]).sum()` — common after divide-by-zero in ratio features.
73. **Set a column as index** → `df.set_index("Customer_ID")`.
74. **Combine first non-null values from two columns** → `df["Best_Estimate"] = df["Model_A"].combine_first(df["Model_B"])`.
75. **Sample a random subset** → `df.sample(frac=0.1, random_state=42)` — reproducible random sampling for quick exploratory checks.

**Strong general follow-up for any of the above:** *"How would this scale to 50 million rows?"* → See Part 18/19 (vectorization, chunking, Dask/PySpark, dtype optimization).

---

## PART 8 — DATA CLEANING INTERVIEW QUESTIONS

**Standardizing inconsistent categories:**
```python
df["Gender"] = df["Gender"].str.strip().str.upper().replace({
    "M": "MALE", "MALE": "MALE", "F": "FEMALE", "FEMALE": "FEMALE"
})
```
- `.str.strip()` removes leading/trailing whitespace (`" MALE "` → `"MALE"`).
- `.str.upper()` normalizes case (`"male"` → `"MALE"`).
- `.replace({...})` maps abbreviations to a single canonical label.

**Invalid dates:** `pd.to_datetime(df["date_col"], errors="coerce")` — invalid strings become `NaT` instead of crashing the whole load, so they can be inspected/handled explicitly rather than silently breaking downstream `.dt` operations.

**Negative/impossible values:** e.g., negative `Age` or `Purchase_Amount` — flag and investigate rather than blindly clip; could indicate a refund (legitimate negative) vs. a data entry error, and the fix differs.

**Common interviewer questions with strong answers:**
- *"Why did you use median instead of mean?"* → Median is robust to skew/outliers; using mean on a right-skewed revenue column inflates the imputed value.
- *"Why didn't you delete the missing rows?"* → Explain the missingness rate and mechanism (MCAR/MAR/MNAR informally); dropping rows loses sample size and can bias results if missingness correlates with the outcome.
- *"How do you decide whether an outlier is genuine?"* → Cross-check against domain plausibility (a $50,000 single retail purchase might be real for a wholesale account, implausible for a typical consumer app) and check whether it's a data entry error (e.g., a misplaced decimal) vs. a real extreme value — genuine extremes are often kept (perhaps flagged/segmented), while entry errors are corrected or removed.
- *"How do you prevent data leakage during preprocessing?"* → Fit any transformation that learns from data (imputation values, scalers, encoders) on the training split only, then apply (`.transform()`, never `.fit()`) to validation/test.

---

## PART 9 — EXPLORATORY DATA ANALYSIS (EDA)

```
Business Problem → Data Understanding → Data Quality → Univariate Analysis →
Bivariate Analysis → Multivariate Analysis → Outlier Analysis → Correlation →
Insights → Business Recommendations
```

```python
# Univariate
df["Revenue"].describe()
sns.histplot(df["Revenue"], kde=True)      # distribution shape: skew, spread, modality

# Bivariate
sns.boxplot(x="Category", y="Revenue", data=df)    # compare a numeric distribution across categories
df.groupby("Gender")["Revenue"].mean()

# Multivariate
sns.pairplot(df[["Revenue","Age","Rating"]])
sns.heatmap(df.corr(numeric_only=True), annot=True, cmap="coolwarm")
```
**EDA interview questions:**
- *"Walk me through your EDA process on a new dataset."* → structure the answer exactly as the flow above: understand the business question first (so you know what to look for), check data quality (nulls/dtypes/duplicates), profile individual variables, then relationships, then multivariate patterns, then translate findings into a recommendation.
- *"What's the difference between univariate, bivariate, and multivariate analysis?"* → one variable's own distribution; the relationship between two variables; patterns involving three or more variables simultaneously.
- *"How do you decide what chart to use?"* → matched to the question and data type — see Part 12's "when to use / when not to" table.

---

## PART 10 — STATISTICS WITH PYTHON

```python
import numpy as np
from scipy import stats

data = df["Revenue"]
mean, median, mode = data.mean(), data.median(), data.mode()[0]
variance, std = data.var(), data.std()          # pandas default ddof=1 (sample)
q1, q3 = data.quantile([0.25, 0.75])
iqr = q3 - q1
skewness, kurt = data.skew(), data.kurt()
z_scores = (data - mean) / std
```
**What each output means / how to explain it:**
- **Mean vs median:** mean is pulled toward outliers/skew; median is the "typical" middle value — if they differ a lot, the distribution is skewed. **Business meaning:** average purchase amount inflated by a few whale customers — median is the more representative "typical customer" figure.
- **Variance/std:** spread of the data around the mean, in squared units (variance) or original units (std) — higher std means more inconsistent purchase behavior.
- **IQR:** the middle 50% spread, robust to outliers — commonly used to define outlier bounds (`Q1 - 1.5*IQR`, `Q3 + 1.5*IQR`).
- **Skewness:** positive = long right tail (common for revenue/income data); negative = long left tail; near 0 = roughly symmetric.
- **Kurtosis:** how heavy the tails are relative to a normal distribution — high kurtosis means more extreme outliers than a normal distribution would predict.
- **Z-score:** how many standard deviations a value is from the mean — a common, simple outlier flag (`|z| > 3`).

**Central Limit Theorem (CLT):** the sampling distribution of the sample mean approaches a normal distribution as sample size grows, *regardless of the population's original distribution* — this is *why* t-tests and confidence intervals built on normal assumptions still work reasonably well on real-world (non-normal) business data, given a large enough sample.

**Confidence interval example:**
```python
import scipy.stats as st
sample = df["Revenue"].sample(200, random_state=42)
ci = st.t.interval(confidence=0.95, df=len(sample)-1, loc=sample.mean(), scale=st.sem(sample))
```
**Business meaning:** "we're 95% confident the true average revenue lies within this range" — used to communicate uncertainty around an estimate rather than presenting a single point number as if it were exact.

---

## PART 11 — HYPOTHESIS TESTING INTERVIEW

| Test | When to use | Null Hypothesis (H₀) |
|---|---|---|
| One-sample t-test | Compare a sample mean to a known/target value | Sample mean = target value |
| Independent t-test | Compare means of two independent groups | Group means are equal |
| Paired t-test | Compare means of the same group before/after | Mean difference = 0 |
| ANOVA | Compare means across 3+ groups | All group means are equal |
| Chi-square | Association between two categorical variables | Variables are independent |
| Mann-Whitney U | Independent t-test alternative, non-normal data | Distributions are equal |
| Wilcoxon | Paired t-test alternative, non-normal data | Median difference = 0 |
| Kruskal-Wallis | ANOVA alternative, non-normal data | All group distributions equal |
| Pearson correlation | Linear relationship between two continuous variables | True correlation = 0 |
| Spearman correlation | Monotonic (not necessarily linear) relationship, or ranked/ordinal data | True rank correlation = 0 |

```python
from scipy import stats

# Independent t-test: does average purchase differ between subscribers and non-subscribers?
sub = df[df["Subscription_Status"]=="Yes"]["Purchase_Amount"]
non_sub = df[df["Subscription_Status"]=="No"]["Purchase_Amount"]
t_stat, p_value = stats.ttest_ind(sub, non_sub, equal_var=False)   # Welch's t-test if variances differ

# Chi-square: is Gender associated with Category preference?
contingency = pd.crosstab(df["Gender"], df["Category"])
chi2, p, dof, expected = stats.chi2_contingency(contingency)

# ANOVA: does average revenue differ across 3+ regions?
groups = [df[df["Region"]==r]["Revenue"] for r in df["Region"].unique()]
f_stat, p_value = stats.f_oneway(*groups)
```
**Output interpretation (applies to all):** if `p_value < 0.05` (the conventional significance threshold), reject H₀ — there's statistically significant evidence of a difference/association; otherwise, fail to reject H₀ (not the same as proving no effect — could be underpowered).

**Business example:** *"Marketing believes the new checkout flow increases average order value."* → paired or independent t-test comparing before/after or A/B groups; a significant p-value with a practically meaningful effect size supports rolling out the change.

**Interview Q: "What's a p-value, in plain terms?"** — the probability of observing data this extreme (or more) *if the null hypothesis were actually true*. It is NOT the probability that H₀ is true — this exact distinction is a favorite interview trap.

**Type I vs Type II error:** Type I = false positive (rejecting a true H₀, e.g., concluding a marketing change worked when it didn't); Type II = false negative (failing to reject a false H₀, e.g., missing a real effect) — lowering one typically raises the other, all else equal (sample size held fixed).

---

## PART 12 — DATA VISUALIZATION

| Chart | When to use | When NOT to use | Insight provided |
|---|---|---|---|
| Bar | Compare a metric across categories | Too many categories (>15, gets cluttered) | Ranking/magnitude comparison |
| Histogram | Distribution shape of one numeric variable | Categorical data | Spread, skew, modality |
| Boxplot | Compare distributions across groups, spot outliers | Very small samples (<5 per group) | Median, IQR, outliers at a glance |
| Scatterplot | Relationship between two numeric variables | Categorical vs categorical | Correlation, clusters, outliers |
| Line chart | Trend over time | Non-sequential/non-time categories | Trend, seasonality |
| Pie chart | Simple part-to-whole, few (≤5) categories | Many categories or precise comparison needed | Rough proportion |
| Countplot | Frequency of categorical values | Numeric data | Category volume |
| Heatmap | Correlation matrix, or 2D categorical crosstab | Sparse/unrelated data | Pattern density at a glance |
| Violin | Distribution shape + comparison across groups | Small samples (shape estimate unreliable) | Distribution shape + summary stats combined |
| KDE | Smoothed distribution estimate | Small/discrete data | Underlying distribution shape without binning artifacts |

```python
sns.boxplot(x="Category", y="Purchase_Amount", data=df)      # spot category-level outliers/spread
sns.heatmap(df.corr(numeric_only=True), annot=True, cmap="coolwarm", fmt=".2f")
sns.countplot(x="Subscription_Status", data=df)
```
**Interview trap:** "Why not always use a pie chart?" — humans are bad at comparing angles/areas precisely; a bar chart communicates the same part-to-whole comparison more accurately, especially with more than a few categories.

---

## PART 13 — SQL + PYTHON INTERVIEW QUESTIONS

**"Find the top 3 customers by revenue."**
```sql
SELECT customer_id, SUM(revenue) AS total_revenue
FROM orders
GROUP BY customer_id
ORDER BY total_revenue DESC
LIMIT 3;
```
```python
df.groupby("customer_id")["revenue"].sum().nlargest(3)
```
**SQL vs Pandas:** SQL is declarative and often faster on data that already lives in a database (pushes computation to the DB engine, avoids pulling everything into memory); Pandas is more flexible for complex, multi-step, or custom Python-logic transformations once data is already in memory — in practice, do heavy filtering/aggregation in SQL first, then finer analysis in Pandas.

**Window functions — SQL vs Pandas:**
```sql
SELECT customer_id, revenue,
       RANK() OVER (ORDER BY revenue DESC) AS rank,
       ROW_NUMBER() OVER (ORDER BY revenue DESC) AS row_num,
       DENSE_RANK() OVER (ORDER BY revenue DESC) AS dense_rank
FROM orders;
```
```python
df["rank"] = df["revenue"].rank(method="min", ascending=False)
df["row_num"] = df["revenue"].rank(method="first", ascending=False)
df["dense_rank"] = df["revenue"].rank(method="dense", ascending=False)
```
**RANK vs ROW_NUMBER vs DENSE_RANK — the classic trap:** for tied values, `RANK` leaves gaps (1,1,3), `DENSE_RANK` doesn't (1,1,2), `ROW_NUMBER` breaks ties arbitrarily and never repeats (1,2,3) — mixing these up is a very common interview mistake.

**CASE WHEN vs `np.select`/`pd.cut`:**
```sql
SELECT *, CASE WHEN revenue > 1000 THEN 'High' WHEN revenue > 500 THEN 'Medium' ELSE 'Low' END AS tier
FROM orders;
```
```python
df["tier"] = np.select([df.revenue>1000, df.revenue>500], ["High","Medium"], default="Low")
```
**CTE vs intermediate DataFrame:** a SQL CTE (`WITH x AS (...)`) is directly analogous to assigning an intermediate result to a named DataFrame variable in Pandas — both exist for readability/reuse of a multi-step computation.

---

## PART 14 — BUSINESS DATA ANALYTICS QUESTIONS

*Format: Business Problem → Code → Business Insight/Recommendation.*

- **Highest-revenue customer segment** → `df.groupby("Segment")["Revenue"].sum().idxmax()` → focus retention marketing spend on this segment; it drives disproportionate revenue.
- **Highest average order value category** → `df.groupby("Category")["Revenue"].mean().idxmax()` → premium-positioning opportunity; consider bundling lower-AOV categories with this one.
- **Highest-sales month (seasonality)** → `df.groupby(df.Order_Date.dt.month)["Revenue"].sum().idxmax()` → align inventory/staffing/marketing spend ahead of that month next cycle.
- **Declining sales product** → compare recent vs prior period revenue per product; flag negative trend → investigate: pricing, competition, stockouts, or genuine demand decline.
- **Customer retention rate** → `retained = customers active in both period T and T-1; rate = retained / customers active in T-1` → low retention signals a churn problem worth deeper cohort analysis.
- **Repeat purchase rate** → `(df["Customer_ID"].value_counts() > 1).mean()` → indicates loyalty/product-market fit strength.
- **Conversion rate** → `converted_users / total_visitors` → funnel health metric; segment by channel to find the weakest step.
- **Bad rate / default rate (credit)** → `df["Default_Flag"].mean()` → core risk KPI; compare across segments to find where underwriting criteria may need tightening.
- **Revenue contribution by segment** → `df.groupby("Segment")["Revenue"].sum() / df["Revenue"].sum() * 100` → identifies concentration risk if one segment dominates.

**"Revenue increased but profit decreased. How would you investigate?"** — structured answer: check if costs (COGS, discounts, marketing spend, returns) grew faster than revenue; check the *mix* — did the growth come from lower-margin products/segments/discounted channels; check for one-off costs; then quantify each driver's contribution before recommending action.

---

## PART 15 — REAL DATASET INTERVIEW ROUND

**Dataset schema:** `Customer_ID, Age, Gender, Category, Purchase_Amount, Discount_Applied, Subscription_Status, Purchase_Frequency, Shipping_Type, Review_Rating, Previous_Purchases, Season`

```python
# EASY
avg_purchase = df["Purchase_Amount"].mean()
revenue_by_gender = df.groupby("Gender")["Purchase_Amount"].sum()
top_category = df.groupby("Category")["Purchase_Amount"].sum().idxmax()

# MEDIUM
above_avg = df[df["Purchase_Amount"] > df["Purchase_Amount"].mean()]
discount_but_high_spend = df[(df["Discount_Applied"]=="Yes") & (df["Purchase_Amount"] > df["Purchase_Amount"].mean())]
discount_by_gender = df[df["Discount_Applied"]=="Yes"].groupby("Gender").size()
rating_by_category = df.groupby("Category")["Review_Rating"].mean().sort_values(ascending=False)

# HARD
repeat_pct = (df["Previous_Purchases"] > 0).mean() * 100
top_10 = df.groupby("Customer_ID")["Purchase_Amount"].sum().nlargest(10)
revenue_contribution = df.groupby("Category")["Purchase_Amount"].sum() / df["Purchase_Amount"].sum() * 100
df["Age_Group"] = pd.cut(df["Age"], bins=[0,25,40,60,100], labels=["18-25","26-40","41-60","60+"])
highest_spend_age_group = df.groupby("Age_Group")["Purchase_Amount"].sum().idxmax()

# ADVANCED
seasonal_trend = df.groupby("Season")["Purchase_Amount"].sum()
loyal_customers = df[df["Previous_Purchases"] > 5]
sub_vs_nonsub = df.groupby("Subscription_Status")["Purchase_Amount"].agg(["mean","sum","count"])
```
**Business interpretation, e.g., `sub_vs_nonsub`:** if subscribers show materially higher mean spend, it validates the subscription program's value and supports continued investment in subscriber acquisition; if not, it's a signal the program isn't driving the intended behavior change and needs redesign.

---

## PART 16 — CODE INTERPRETATION ROUND

```python
df.groupby("Category")["Purchase_Amount"].mean()
```
**What it returns:** a Series indexed by `Category`, with the mean `Purchase_Amount` for each category. **Why:** `groupby` splits the DataFrame into groups by category, then `mean()` aggregates each group's `Purchase_Amount` independently.

```python
df[df["Purchase_Amount"] > df["Purchase_Amount"].mean()]
```
**What it returns:** a filtered DataFrame — only rows where the purchase amount exceeds the overall (not per-category) average. **Trap to watch for:** this compares against the *global* mean, not a group-specific mean — if the question intended "above their category's average," you'd need `transform` instead (see below).

```python
df.groupby("Gender")["Purchase_Amount"].sum()
```
**What it returns:** total purchase amount per gender — a Series, index = Gender values.

```python
df["Age"].apply(lambda x: "Young" if x < 30 else "Adult")
```
**What it returns:** a new Series, same length as `df`, with each age bucketed into "Young"/"Adult". **Line-by-line:** `.apply()` calls the lambda once per element in the `Age` Series; the lambda evaluates the condition per value and returns the matching label.

**Intentionally broken code — spot the error:**
```python
df["Purchase_Amount"] > df["Purchase_Amount].mean()   # <- missing closing quote on column name
```
**Error:** `SyntaxError` — unterminated string literal (`"Purchase_Amount]` is missing its closing `"`). **Fix:** `df["Purchase_Amount"].mean()`.

```python
avg = df.groupby("Category")["Purchase_Amount"].mean()
df["Above_Category_Avg"] = df["Purchase_Amount"] > avg   # <- shape mismatch bug
```
**Error:** this doesn't raise an exception but silently produces wrong/misaligned results (or a `ValueError` depending on pandas version) — `avg` is grouped (one row per category), not row-aligned to `df`. **Fix:** use `transform`:
```python
df["Category_Avg"] = df.groupby("Category")["Purchase_Amount"].transform("mean")
df["Above_Category_Avg"] = df["Purchase_Amount"] > df["Category_Avg"]
```
**This is one of the single most common real interview "spot the bug" questions** — confusing an aggregated groupby result (fewer rows) with a row-aligned `transform()` result (same rows as original).

---

## PART 17 — DEBUGGING QUESTIONS

| Error | Why it happens | How to fix |
|---|---|---|
| `KeyError: 'Purchase_Amount'` | Column name typo, extra whitespace, or wrong case | `print(df.columns.tolist())` to see exact names; `df.columns = df.columns.str.strip()` |
| `TypeError` | Operation on incompatible types, e.g., string + int | `df.dtypes` to check; `.astype()` to convert |
| `ValueError` | e.g., truth value of a Series is ambiguous (used `and`/`or` instead of `&`/`\|`), or shape mismatch | Use `&`/`\|` with parentheses per condition; check `.shape` alignment |
| `IndexError` | Accessing an index/position that doesn't exist (empty result, off-by-one) | Check `len(df)` / `.empty` before indexing; use `.iloc` bounds carefully |
| `AttributeError` | Calling a method that doesn't exist for that object's type (e.g., `.str` on a non-string/object column) | `df.dtypes`; ensure the column is the type you think it is before calling type-specific methods |
| `SettingWithCopyWarning` | Assigning into a DataFrame that may be a view/copy of another (chained indexing, e.g., `df[df.A>0]["B"]=1`) | Use `.loc[row_filter, "B"] = 1`, or `.copy()` explicitly when you intend to work on an independent copy |
| `FileNotFoundError` | Wrong path, relative path run from a different working directory | `import os; print(os.getcwd())`; use absolute paths or verify the working directory |
| `ModuleNotFoundError` | Package not installed in the active environment | `pip install <package>`; verify you're in the right virtual environment |

**Hidden spaces in column names — common gotcha:**
```python
df.columns = df.columns.str.strip()   # " Purchase_Amount " -> "Purchase_Amount"
```
**Diagnosis workflow (always the same shape):** `df.columns` → `df.info()` → `df.dtypes` → then decide the fix.

---

## PART 18 — PYTHON OPTIMIZATION

| Approach | Speed | When to use |
|---|---|---|
| Plain Python `for` loop | Slowest | Small data, complex per-row branching logic that's hard to vectorize |
| `.apply()` | Medium (still loops in Python under the hood) | Row-wise logic across multiple columns that's genuinely hard to vectorize |
| Vectorized (pandas/numpy operation directly) | Fastest | Almost always the first choice for numeric transformations |

```python
# SLOW: explicit loop
result = []
for x in df["Purchase_Amount"]:
    result.append(x * 0.9)

# MEDIUM: apply
df["Purchase_Amount"].apply(lambda x: x * 0.9)

# FAST: vectorized
df["Purchase_Amount"] * 0.9
```
**Why vectorized wins:** the operation runs as a single compiled (C-level) pass over the whole array instead of per-element Python bytecode execution — orders of magnitude faster on large data.

**Other optimization levers:** `df.query()` for readable, sometimes faster filtering on large frames; downcasting dtypes (`float64`→`float32`, `int64`→`int32`/`category` for low-cardinality strings) to cut memory; reading only needed columns (`usecols=`) and rows (`nrows=`, `chunksize=`) from source files; avoiding repeated `.copy()`s in tight loops; using `groupby` + vectorized aggregation instead of manual looping over unique values.

---

## PART 19 — LARGE DATASETS

**Interview scenario: "Your Pandas code works on 10,000 rows but crashes on 10 million rows."**
**Structured answer:**
1. **Diagnose:** is it a memory issue (`MemoryError`) or a speed issue (just slow)?
2. **Memory fixes:** read in `chunksize` batches and process/aggregate incrementally; downcast dtypes; convert repeated string columns to `category` dtype; drop unneeded columns before heavy transforms; switch file format to Parquet (columnar, compressed, much smaller and faster to read than CSV — and lets you read only needed columns).
3. **Speed fixes:** replace loops/`.apply()` with vectorized operations; push filtering/aggregation to the database (SQL) before pulling into Python; for truly large-scale, use Dask (pandas-like API, parallelized/out-of-core) or PySpark (distributed compute cluster).
4. **Validate:** confirm results on a sample match the original small-scale logic before trusting the full-scale run.

```python
total = 0
for chunk in pd.read_csv("big_file.csv", chunksize=500_000):
    total += chunk["Revenue"].sum()
```
**Why chunking works:** only one chunk is held in memory at a time, so total memory use stays bounded regardless of file size — at the cost of not being able to do operations that need the *whole* dataset at once (like a global sort) without extra work.

---

## PART 20 — OOP FOR DATA SCIENCE

```python
class RevenueAnalyzer:
    def __init__(self, df):          # constructor — runs when an object is created
        self.df = df                   # self = reference to this specific instance's data

    def total_revenue(self):
        return self.df["Revenue"].sum()

    def revenue_by(self, group_col):
        return self.df.groupby(group_col)["Revenue"].sum()


class SubscriptionAnalyzer(RevenueAnalyzer):     # inheritance — reuses RevenueAnalyzer's logic
    def subscriber_revenue(self):
        return self.df[self.df["Subscription_Status"]=="Yes"]["Revenue"].sum()


analyzer = SubscriptionAnalyzer(df)
analyzer.total_revenue()           # inherited method
analyzer.subscriber_revenue()       # subclass-specific method
```
**Why OOP matters for a data role (not just software engineers):** wrapping a reusable analysis pipeline (e.g., a `ModelEvaluator` or `DataCleaner` class) in a class keeps related data + methods together, avoids passing the same DataFrame through a dozen loose functions, and makes it easy to build variants via inheritance (e.g., different cleaning rules per data source) — this is the most common practical use in DS/analytics work, more than deep OOP theory.

---

## PART 21 — FILE HANDLING

| Format | Read | Write | When to use |
|---|---|---|---|
| CSV | `pd.read_csv()` | `.to_csv()` | Universal, human-readable, but slow/large for big data, no dtype preservation |
| Excel | `pd.read_excel()` | `.to_excel()` | Business stakeholders who work in Excel; supports multiple sheets |
| JSON | `pd.read_json()` | `.to_json()` | Nested/semi-structured data, common from APIs |
| Pickle | `pd.read_pickle()` | `.to_pickle()` | Fast Python-native serialization; NOT for long-term/cross-version storage or untrusted files (security risk) |
| Parquet | `pd.read_parquet()` | `.to_parquet()` | Large data, columnar storage, preserves dtypes, much faster/smaller than CSV |

---

## PART 22 — API & DATA EXTRACTION

```python
import requests
import pandas as pd

response = requests.get("https://api.example.com/sales", params={"month": "2026-08"})
response.raise_for_status()          # raises an exception on 4xx/5xx instead of silently continuing
data = response.json()
df = pd.DataFrame(data)
```
**Status codes to know:** `200` success, `201` created, `400` bad request (client error, e.g., malformed params), `401`/`403` auth issues, `404` not found, `429` rate limited, `500` server error.
**FOLLOW-UP:** *"How do you handle a paginated API?"* → loop while a `next_page` field exists in the response, appending each page's results, respecting any rate limits with delays/backoff.

---

## PART 23 — DATA SCIENCE LIBRARY MAP

| Task | Library | Common Functions | Purpose |
|---|---|---|---|
| Numeric arrays/math | NumPy | `array, mean, std, reshape` | Fast numeric computation |
| Tabular data manipulation | Pandas | `read_csv, groupby, merge, apply` | Data wrangling/analysis |
| Static plotting | Matplotlib | `plot, bar, hist, subplots` | Low-level, fully customizable charts |
| Statistical plotting | Seaborn | `boxplot, heatmap, pairplot` | Higher-level, statistically-aware charts |
| Scientific/statistical functions | SciPy | `stats.ttest_ind, stats.chi2_contingency` | Hypothesis tests, distributions |
| Statistical modeling | Statsmodels | `OLS, summary()` | Regression with detailed statistical output (p-values, CIs) |
| Machine learning | Scikit-learn | `fit, predict, train_test_split` | Modeling, preprocessing, evaluation |
| Gradient boosting | XGBoost | `XGBClassifier, XGBRegressor` | High-performance tree-based models |

**When each is used, briefly:** NumPy underlies almost everything numeric; Pandas is the daily driver for cleaning/aggregating; Matplotlib/Seaborn for communicating findings visually; SciPy/Statsmodels when a question needs formal statistical inference (not just description); Scikit-learn/XGBoost once the task moves from analysis to predictive modeling.

---

## PART 24 — 150+ INTERVIEW QUESTIONS BY LEVEL (rapid-reference)

*Each entry: Question — Ideal short answer.*

**LEVEL 1 — Beginner**
1. What is Python? — A high-level, interpreted, dynamically-typed general-purpose language, popular in data science for its readable syntax and rich ecosystem (pandas/numpy/sklearn).
2. List vs tuple? — Mutable/ordered vs immutable/ordered (Part 2).
3. What is a dictionary? — Key-value store, O(1) average lookup by key.
4. What is indentation used for in Python? — Defines code blocks (replaces `{}` braces used in many other languages).
5. Difference between `is` and `==`? — `is` checks identity (same object in memory); `==` checks value equality.
6. What is a module? — A `.py` file whose functions/variables can be imported elsewhere.
7. What is a package? — A directory of modules with an `__init__.py`, forming a namespace.
8. What's the difference between a function and a method? — A method is a function defined inside a class, called on an instance.
9. What does `len()` do? — Returns the number of items in a sequence/collection.
10. What is `None`? — Python's null/absence-of-value singleton.
11. What's a f-string? — `f"{var}"` — inline string formatting/interpolation.
12. What is a virtual environment and why use one? — An isolated Python install per project, avoiding dependency version conflicts across projects.
13. What is `pip`? — Python's package installer, used to install libraries from PyPI.
14. What's the difference between `range()` and a list? — `range()` is a lazy, memory-efficient sequence generator; a list stores all values in memory.
15. What is a docstring? — A string literal right after a `def`/`class` documenting what it does.
16. What is exception handling (`try/except`)? — Gracefully catching and responding to runtime errors instead of crashing.
17. What is `with` used for (context managers)? — Automatically handles setup/teardown (e.g., closing a file) even if an error occurs.
18. Difference between `append()` and `extend()` on a list? — `append` adds one element (even if it's a list, as a nested element); `extend` adds each element of an iterable individually.
19. What does `.sort()` vs `sorted()` do differently? — `.sort()` mutates the list in place, returns `None`; `sorted()` returns a new sorted list, leaves original unchanged.
20. What is a set used for? — Fast membership testing and removing duplicates; unordered, unique elements only.

**LEVEL 2 — Intermediate**
21. loc vs iloc — Part 6.
22. apply vs transform — Part 6.
23. merge vs concat — Part 6/13.
24. What's `groupby().agg()` vs `groupby().transform()`? — `agg` reduces to one row per group; `transform` returns a value per original row.
25. What's `.value_counts()` used for? — Frequency count of unique values in a Series, sorted descending by default.
26. Explain `pivot_table` vs `groupby`. — Part 7 (Q19).
27. What is method chaining in pandas and why use it? — Calling multiple operations in sequence (`df.dropna().groupby(...).mean()`) — improves readability, avoids intermediate variable clutter.
28. What does `inplace=True` do, and why is it often discouraged? — Mutates the object directly instead of returning a new one; discouraged because it can silently break chained/copy-based workflows and makes debugging harder (no explicit reassignment to trace).
29. What is broadcasting in NumPy? — Part 5.
30. Why is NumPy faster than pure Python for numeric work? — Part 5.
31. Explain vectorization vs looping. — Part 18.
32. What's a `MultiIndex` in pandas? — A hierarchical row/column index (e.g., after `groupby(["A","B"])`), letting you index by multiple levels.
33. What does `df.reset_index()` do and when do you need it? — Turns the index back into a column and creates a fresh default integer index — needed after operations (like filtering or groupby) leave a non-contiguous or non-default index.
34. What is `SettingWithCopyWarning` and how do you avoid it? — Part 17.
35. What's the difference between a Series and a DataFrame? — 1-D labeled array vs 2-D labeled table (a DataFrame is essentially a dict of Series sharing an index).
36. How do you check for and handle outliers? — Part 7 (Q25), Part 8.
37. What's the difference between `dropna()` and `fillna()` strategies, and when would you choose each? — Part 6/8.
38. Explain one-hot encoding vs label encoding, and when to use each. — One-hot avoids implying false ordinal relationships (good for nominal categories, but increases dimensionality); label encoding is compact but implies an order — fine for tree-based models, risky for linear/distance-based models.
39. What is multicollinearity and how do you detect it? — Predictor variables are highly correlated with each other, destabilizing coefficient estimates in linear models; detect via correlation matrix or VIF (Variance Inflation Factor).
40. What does `random_state` do and why set it? — Fixes the random seed for reproducibility — same "random" split/sample every run.

**LEVEL 3 — Advanced**
41. Explain data leakage with a concrete example. — Part 8; e.g., scaling before train/test split, or including a feature only known after the outcome occurs.
42. What is target leakage specifically (vs general data leakage)? — A feature that's a proxy for, or directly derived from, the target itself (e.g., using "amount refunded" to predict "is this a return" — refunds only happen *after* a return is confirmed).
43. How would you detect multicollinearity and what would you do about it? — VIF > 5-10 as a rule of thumb; drop or combine correlated features, or use regularization (Ridge/Lasso).
44. Explain the bias-variance tradeoff conceptually. — High-bias models underfit (too simple, miss real patterns); high-variance models overfit (too complex, fit noise); the goal is the sweet spot minimizing total error on unseen data.
45. What's the difference between correlation and causation, and how do you test for causal effect? — Correlation just means two variables move together; causation requires ruling out confounders/reverse causality — often via controlled experiments (A/B tests) or causal inference techniques, not observational correlation alone.
46. How do you choose between Pearson and Spearman correlation? — Pearson assumes a linear relationship and continuous, roughly normal data; Spearman is rank-based, robust to non-linear monotonic relationships and outliers/ordinal data.
47. What's the Central Limit Theorem and why does it matter practically? — Part 10.
48. Explain Type I vs Type II error with a business example. — Part 11 (e.g., falsely concluding a marketing campaign worked vs missing a campaign that actually did work).
49. What is p-hacking and why is it a problem? — Repeatedly testing/slicing data until a significant p-value appears by chance, inflating false-positive findings — mitigated by pre-registering hypotheses and correcting for multiple comparisons.
50. How would you design an A/B test for a new checkout flow? — define the metric (e.g., conversion rate), calculate required sample size for desired power, randomize users to control/treatment, run for a pre-defined duration avoiding peeking, then run the appropriate hypothesis test (often a two-proportion z-test or t-test) at the end.

**LEVEL 4 — Scenario-Based** — see Part 26 for the full structured-answer format; representative questions: handling 30% missing data, whether to remove outliers, investigating revenue-up-profit-down, correlation-vs-causation pushback, code crashing at scale, duplicate customer IDs.

**LEVEL 5 — Code Interpretation** — see Part 16 for the full worked examples and the classic `transform` vs group-mean-comparison bug.

**LEVEL 6 — Debugging** — see Part 17 for the full error-type table and diagnosis workflow.

**LEVEL 7 — Business Analytics** — see Part 14 for the full worked business-problem examples.

*(Questions 51–150+ are the applied continuation of the same shape across Parts 7, 14, 15, 27 — the pandas question bank, business analytics bank, dataset round, and coding-problem bank together total well past 150 distinct questions; they're organized there by theme rather than repeated again here, to keep this document navigable rather than repetitive.)*

---

## PART 25 — RAPID-FIRE ROUND (50 one-liners)

1. **Pandas?** — Python library for labeled, tabular data manipulation and analysis.
2. **NumPy?** — Python library for fast numeric array computation.
3. **loc vs iloc?** — Label-based vs position-based indexing.
4. **merge vs concat?** — Key-based join vs simple stacking.
5. **apply vs transform?** — Any-shape output vs same-length-as-input output.
6. **mean vs median?** — Average vs middle value; median is outlier-robust.
7. **Inner join?** — Keeps only rows with matching keys in both tables.
8. **VIF?** — Variance Inflation Factor — measures multicollinearity severity.
9. **p-value?** — Probability of the observed data (or more extreme) under H₀.
10. **Standard deviation?** — Typical distance of values from the mean.
11. **Correlation?** — Strength/direction of a linear (Pearson) or monotonic (Spearman) relationship, -1 to 1.
12. **groupby?** — Splits data into groups, applies an aggregation, combines results.
13. **Lambda?** — Anonymous single-expression function.
14. **List vs tuple?** — Mutable vs immutable ordered collection.
15. **Mutable vs immutable?** — Can vs cannot be changed in place after creation.
16. **Left join?** — All rows from the left table, matched rows from the right, else null.
17. **Outer join?** — All rows from both tables, unmatched filled with null.
18. **`NaN`?** — Not-a-Number — pandas/numpy's missing numeric value marker.
19. **`.copy()`?** — Creates an independent copy so mutations don't affect the original.
20. **Skewness?** — Asymmetry of a distribution's tail.
21. **IQR?** — Interquartile range (Q3-Q1); robust spread measure, used for outlier bounds.
22. **Z-score?** — Number of standard deviations from the mean.
23. **Normal distribution?** — Symmetric, bell-shaped distribution defined by mean and std.
24. **CLT?** — Sample means approach normality as sample size grows, regardless of population shape.
25. **Type I error?** — False positive — rejecting a true null hypothesis.
26. **Type II error?** — False negative — failing to reject a false null hypothesis.
27. **Confidence interval?** — A range likely (e.g., 95%) to contain the true population parameter.
28. **Chi-square test?** — Tests independence/association between two categorical variables.
29. **t-test?** — Tests whether means differ, for one or two groups.
30. **ANOVA?** — Tests whether means differ across 3+ groups.
31. **One-hot encoding?** — Converts a categorical column into binary indicator columns.
32. **Label encoding?** — Converts categories into integer codes.
33. **Feature engineering?** — Creating new predictive variables from raw data.
34. **Data leakage?** — Information from outside the legitimate training scope influencing the model.
35. **Overfitting?** — Model fits training noise, generalizes poorly to new data.
36. **Underfitting?** — Model is too simple to capture real patterns.
37. **Train-test split?** — Dividing data to fit on one part, evaluate on unseen data.
38. **Cross-validation?** — Repeated train/test splits to get a more robust performance estimate.
39. **Vectorization?** — Applying operations to whole arrays at once instead of looping.
40. **Broadcasting?** — NumPy auto-expanding smaller arrays to match a larger array's shape in operations.
41. **`.fillna()`?** — Fills missing values with a specified value/strategy.
42. **`.dropna()`?** — Removes rows/columns containing missing values.
43. **`.duplicated()`?** — Flags duplicate rows.
44. **`.astype()`?** — Converts a column's data type.
45. **Chunking?** — Reading/processing large files in smaller pieces to limit memory use.
46. **Parquet?** — A compressed, columnar file format, faster/smaller than CSV for large data.
47. **API?** — A defined interface for programs to exchange data/requests.
48. **JSON?** — A lightweight, human-readable, key-value/nested text data format.
49. **`GROUP BY` (SQL)?** — Groups rows sharing a column value for aggregation, same concept as pandas `groupby`.
50. **Window function (SQL)?** — Computes a value across a set of rows related to the current row without collapsing them, e.g., `RANK() OVER (...)`.

---

## PART 26 — SCENARIO-BASED INTERVIEW ROUND

**Structured answer shape for every scenario below: Clarify → Investigate → Analyze → Decide → Validate → Communicate.**

**"You have 50,000 rows and 30% missing values. What will you do?"**
1. *Clarify:* which columns, is missingness concentrated or spread out, is it MCAR/MAR/MNAR (does missingness correlate with other variables/the outcome)?
2. *Investigate:* `df.isnull().sum()` per column; visualize missingness pattern.
3. *Analyze:* if concentrated in a few low-importance columns, consider dropping those columns; if spread and important, imputation is likely necessary.
4. *Decide:* median/mode imputation for straightforward cases, or group-wise/model-based imputation for more accuracy; document the assumption made.
5. *Validate:* compare summary stats before/after imputation to confirm no major distortion introduced.
6. *Communicate:* clearly state what was imputed and how, so downstream consumers understand the resulting confidence level.

**"Your data has many outliers. Will you remove them?"**
Not automatically — first determine if they're genuine extreme values or data errors (Part 8); genuine outliers relevant to the business question (e.g., VIP customers in a revenue analysis) are often kept or analyzed separately, not deleted; errors are corrected/removed. State the decision and its rationale explicitly rather than silently dropping rows.

**"Revenue increased but profit decreased. How would you investigate?"** — see Part 14.

**"Your correlation is high. Does that mean causation?"** — No (Part 24, Q45); explain confounders and propose an experiment (A/B test) or causal inference method to establish causation if it matters for the decision.

**"Your Pandas code works on 10,000 rows but crashes on 10 million rows."** — see Part 19.

**"Your dataset contains duplicate customer IDs. What will you do?"**
1. *Clarify:* are duplicates exact row duplicates (likely a data pull error) or the same customer with multiple legitimate transactions (expected, not a bug)?
2. *Investigate:* `df[df.duplicated(subset='Customer_ID', keep=False)]` to inspect actual duplicate rows.
3. *Analyze:* check if other columns (timestamp, order ID) differ — if so, they're likely legitimate repeat purchases, not errors.
4. *Decide:* drop true exact duplicates; keep legitimate repeat-purchase rows, aggregating only if the analysis requires customer-level (not transaction-level) granularity.
5. *Validate:* confirm row count changes make sense against the expected data volume.
6. *Communicate:* note the distinction made and why, especially since it directly affects revenue totals.

---

## PART 27 — CODE-FIRST QUESTIONS (50 Coding Problems)

*Format: Problem → Code → one-line note on alternative/optimization.*

1. **Reverse a string** → `s[::-1]` — slicing with step -1; alt: `"".join(reversed(s))`.
2. **Find duplicates in a list** → `[x for x in set(lst) if lst.count(x)>1]`; optimized: `[x for x,c in Counter(lst).items() if c>1]` (avoids O(n²) `.count()` in a loop).
3. **Find second largest number** → `sorted(set(lst))[-2]`; optimized (single pass): track top-2 while iterating once, O(n) instead of O(n log n).
4. **Count frequency of elements** → `Counter(lst)` from `collections`.
5. **Find missing values in a DataFrame** → `df.isnull().sum()` (Part 7).
6. **Groupby aggregation** → `df.groupby("Category")["Revenue"].agg(["sum","mean"])`.
7. **Top N customers** → `df.groupby("Customer_ID")["Revenue"].sum().nlargest(N)`.
8. **Ranking** → `df["Revenue"].rank(ascending=False)` (Part 13).
9. **Rolling average** → `df["Revenue"].rolling(7).mean()` (Part 7, Q24).
10. **Cumulative sum** → `df["Revenue"].cumsum()`.
11. **Percentage contribution** → `df["Revenue"]/df["Revenue"].sum()*100`.
12. **Date difference** → `(df["End_Date"] - df["Start_Date"]).dt.days`.
13. **Customer retention** (simplified) →
```python
this_period = set(df[df.Period=="T"]["Customer_ID"])
last_period = set(df[df.Period=="T-1"]["Customer_ID"])
retention_rate = len(this_period & last_period) / len(last_period)
```
14. **Cohort analysis (skeleton)** →
```python
df["Cohort"] = df.groupby("Customer_ID")["Order_Date"].transform("min").dt.to_period("M")
df["Order_Period"] = df["Order_Date"].dt.to_period("M")
df["Cohort_Index"] = (df["Order_Period"] - df["Cohort"]).apply(lambda x: x.n)
cohort_table = df.groupby(["Cohort","Cohort_Index"])["Customer_ID"].nunique().unstack()
```
15. **Outlier detection (IQR)** → Part 7, Q25.
16. **Find the mode of a column** → `df["Category"].mode()[0]`.
17. **Check if a string is a palindrome** → `s == s[::-1]`.
18. **Find common elements between two lists** → `set(a) & set(b)`.
19. **Flatten a nested list** → `[x for sub in nested for x in sub]`.
20. **Remove duplicates while preserving order** → `list(dict.fromkeys(lst))` — dicts preserve insertion order (3.7+) and keys are unique, an elegant dedup trick.
21. **FizzBuzz** → `["FizzBuzz" if i%15==0 else "Fizz" if i%3==0 else "Buzz" if i%5==0 else str(i) for i in range(1,101)]`.
22. **Word frequency in text** → `Counter(text.lower().split())`.
23. **Find the longest word in a sentence** → `max(sentence.split(), key=len)`.
24. **Check for anagram** → `sorted(a) == sorted(b)`.
25. **Merge two dictionaries** → `{**d1, **d2}` (Python 3.5+) or `d1 | d2` (3.9+).
26. **Find rows where a value crosses a threshold repeatedly (streaks)** → `(df["Revenue"]>1000).astype(int).groupby((df["Revenue"]<=1000).cumsum()).cumsum()` — a classic advanced pandas "streak counting" trick using cumsum-of-breaks as a grouping key.
27. **Top N per group (not just overall top N)** → Part 7, Q39.
28. **Calculate customer lifetime value (simplified)** → `df.groupby("Customer_ID")["Revenue"].sum()`.
29. **Detect a sudden spike/drop vs previous period** → `df["Revenue"].pct_change().abs() > 0.5` — flags >50% period-over-period swings.
30. **Bucket continuous values into custom labeled bins** → `pd.cut()` (Part 7, Q20).
31. **Find the most common value per group** → `df.groupby("Category")["Product"].agg(lambda x: x.mode()[0])`.
32. **Calculate a running rank within groups** → `df.groupby("Category")["Revenue"].rank(ascending=False)`.
33. **Identify gaps in a sequence of dates** →
```python
full_range = pd.date_range(df["Date"].min(), df["Date"].max())
missing_dates = full_range.difference(df["Date"])
```
34. **Find pairs of correlated columns above a threshold** → Part 7, Q62.
35. **Deduplicate based on multiple columns, keeping most recent** → `df.sort_values("Order_Date").drop_duplicates(subset=["Customer_ID","Product"], keep="last")`.
36. **Compute month-over-month and year-over-year growth** → `.pct_change(1)` and `.pct_change(12)` on a monthly-resampled Series.
37. **Find customers who churned (no purchase in last N days)** →
```python
last_purchase = df.groupby("Customer_ID")["Order_Date"].max()
churned = last_purchase[last_purchase < (pd.Timestamp.today() - pd.Timedelta(days=90))]
```
38. **Compute a weighted score across multiple metrics** → `df["Score"] = 0.5*df["A"] + 0.3*df["B"] + 0.2*df["C"]` — after normalizing each metric to a comparable scale first (e.g., min-max) so the weights are meaningful.
39. **Implement a basic moving z-score anomaly flag** → `((df["Revenue"] - df["Revenue"].rolling(30).mean()) / df["Revenue"].rolling(30).std()).abs() > 3`.
40. **Group and get the row with the max value per group (not just the max value itself)** → `df.loc[df.groupby("Category")["Revenue"].idxmax()]`.
41. **Compute Gini/concentration of revenue across customers (simplified)** → sort customer revenue ascending, compute cumulative share vs cumulative customer share — the basis of a Lorenz curve / Gini coefficient, used to show revenue concentration risk.
42. **Detect consecutive missing values** → `df["col"].isnull().astype(int).groupby(df["col"].notnull().cumsum()).cumsum()`.
43. **Calculate percentile rank of each row within its group** → `df.groupby("Category")["Revenue"].rank(pct=True)`.
44. **Find the first purchase date per customer** → `df.groupby("Customer_ID")["Order_Date"].min()`.
45. **Compute average time between purchases per customer** → `df.sort_values("Order_Date").groupby("Customer_ID")["Order_Date"].diff().mean()`.
46. **Identify customers in the top X% by spend (Pareto/80-20 check)** → sort descending, cumulative sum, find where cumulative share crosses 80%.
47. **Simulate a simple A/B test significance check** → `stats.ttest_ind(group_a, group_b)` (Part 11).
48. **Build a simple funnel conversion rate table** → `df.groupby("Funnel_Stage")["User_ID"].nunique()` then compute stage-over-stage ratios.
49. **Detect schema drift between two DataFrame versions** → `set(df_new.columns) - set(df_old.columns)` (new cols) and vice versa (dropped cols); compare `.dtypes` for type changes.
50. **Write a reusable "quick EDA" function** → a function wrapping `.info()`, `.describe()`, `.isnull().sum()`, and a few standard plots, called at the start of every new dataset — demonstrates good engineering habits, not just one-off analysis.

---

## PART 28 — COMMON INTERVIEW TRICKS (Interviewer Traps)

- **Sum vs revenue:** "sum" is a math operation; "revenue" is a specific business concept (price × quantity, possibly net of returns/discounts) — conflating them can silently misstate the actual number a stakeholder cares about.
- **Revenue vs profit:** revenue is top-line sales; profit subtracts costs — a rising-revenue/falling-profit answer (Part 14) is a classic follow-up trap testing whether you jump to conclusions.
- **Mean vs weighted mean:** a simple mean of per-group averages ≠ the true overall average unless group sizes are equal — this is the "average of averages" trap (see next point).
- **Correlation vs causation:** covered in Part 24 (Q45) — always the first thing interviewers probe after you report a correlation.
- **Missing value vs zero:** a missing value means "unknown"; a zero means "known to be nothing" — treating a `NaN` as `0` (or vice versa) can silently change every downstream aggregate.
- **Duplicate row vs duplicate customer:** an exact duplicate row is likely a data error; a customer appearing multiple times with different order details is expected — conflating the two leads to either over-deletion or missed data quality bugs (Part 26).
- **"Average of averages" trap:** `df.groupby("store")["sales"].mean().mean()` is NOT the same as `df["sales"].mean()` unless every store has the same number of transactions — because the first weights every store equally regardless of size, while the second is transaction-weighted.
- **Percentage vs percentage points:** going from 10% to 15% is a 5 percentage-point increase, but a 50% *relative* increase — interviewers often ask you to state which one you mean, since they tell very different stories.
- **Population vs sample standard deviation:** `ddof=0` (population, NumPy default) vs `ddof=1` (sample, pandas default) — same data, different `std()` output depending on the library, a classic "gotcha" (Part 5 follow-up).
- **Inner join accidentally removing rows:** an inner join silently drops any row without a matching key on the other side — always check row counts before/after a join to catch unintended data loss.
- **Data leakage / target leakage:** covered fully in Part 8 and Part 24 (Q41-42).
- **Wrong aggregation level:** e.g., computing "average order value" by averaging *pre-aggregated* per-customer totals instead of averaging individual order amounts — silently changes the meaning of "average" depending on which level you aggregated at first; always be explicit about the grain (row = what?) before aggregating.

**How interviewers test these:** usually by asking you to explain a metric out loud after you compute it ("what does that number actually mean?") — the trap isn't in writing correct code, it's in whether you can correctly articulate what the code computed and whether that matches the business question actually asked.

---

## PART 29 — PROJECT-BASED QUESTIONS (STAR-format Answers)

**"What was the business problem?"**
*Situation:* [Company] needed to reduce customer churn in [segment]. *Task:* Identify key churn drivers and build an early-warning signal for the retention team. *Action:* Pulled transactional + support-ticket data, cleaned and joined it, ran EDA and hypothesis tests to identify significant behavioral differences between churned/retained customers, and summarized findings into a scored risk segment. *Result:* Delivered a ranked list of at-risk customers and 2-3 concrete drivers (e.g., declining purchase frequency, rising support ticket volume) that the retention team used to prioritize outreach.

**"How did you clean the data? / Why did you use median? / How did you validate results?"** — answer each using the exact reasoning already built in Parts 6, 8, and 10: median for skew-robustness, cross-checking summary statistics before/after cleaning, and comparing findings against a holdout period or a simple baseline to sanity-check conclusions before presenting them.

**"What challenges did you face?"** — good answers name a *specific, real* obstacle (e.g., "the customer ID wasn't consistent across two source systems, so I had to build a fuzzy-matching join and manually validate a sample") rather than a vague generality — specificity is what distinguishes a strong project answer from a rehearsed one.

**General STAR reminder:** every project question maps to Situation → Task → Action → Result — always end with a *quantified* result where possible ("reduced investigation time by X%", "flagged Y% of eventual churners a month in advance") since interviewers weight concrete impact heavily.

---

## PART 30 — FINAL MOCK INTERVIEW

**ROUND 1 — Python (20 Qs):** draw from Part 1 (Q1-6 + rapid topics) and Part 24 Level 1 (Q1-20). Answers: as given in those sections.

**ROUND 2 — Pandas (20 Qs):** Part 7, Q1-20, plus Part 6's `loc/iloc`, `apply/transform`, `merge/concat` comparisons. Answers: as given.

**ROUND 3 — Statistics (15 Qs):** Part 10 (mean/median/std/skew/CLT/CI) + Part 11 (t-test, chi-square, ANOVA, p-value, Type I/II). Answers: as given.

**ROUND 4 — SQL (15 Qs):** Part 13's GROUP BY/JOIN/window functions/CASE WHEN/CTE, each paired with its Pandas equivalent. Answers: as given.

**ROUND 5 — Data Analytics (15 Qs):** Part 14's business-problem set (segment revenue, retention, conversion, bad rate, revenue-up-profit-down investigation). Answers: as given.

**ROUND 6 — Coding (10 Qs):** pick 10 from Part 27's 50 coding problems (recommend: #1, 2, 3, 7, 10, 12, 13, 20, 30, 40 for a balanced Python + pandas mix). Answers: as given.

**ROUND 7 — Case Study (5 Qs):** pick 5 from Part 26's scenario bank (missing data, outliers, revenue/profit divergence, correlation/causation, duplicate IDs). Answers: use the Clarify→Investigate→Analyze→Decide→Validate→Communicate structure exactly as shown.

**ROUND 8 — Project (10 Qs):** use Part 29's STAR-format answer as the template, adapted to your actual project(s); prepare specific numbers/results in advance rather than improvising them in the room.

---

## PART 31 — FINAL CHEAT SHEET

### Python syntax
```python
[x for x in lst if cond]         # list comprehension
lambda x: x*2                     # anonymous function
try: ... except Exception as e: ...   # error handling
with open("f") as f: ...          # context manager
```

### Pandas
```python
df.head(); df.info(); df.describe(); df.shape
df.loc[rows, cols]; df.iloc[rows, cols]
df[df.col > x]; df[(df.a>x)&(df.b<y)]
df.isnull().sum(); df.fillna(val); df.dropna()
df.duplicated(); df.drop_duplicates()
df.groupby("g")["v"].agg(["mean","sum"])
df.groupby("g")["v"].transform("mean")
pd.merge(a,b,on="k",how="left"); pd.concat([a,b])
df.sort_values("col", ascending=False)
df["col"].value_counts(normalize=True)
pd.to_datetime(df["d"]); df["d"].dt.month
df["col"].rolling(7).mean(); .cumsum(); .pct_change()
```

### NumPy
```python
np.array(); .shape; .ndim; .dtype
arr.mean(); arr.std(ddof=0)   # population std by default (vs pandas ddof=1)
np.select(conditions, choices, default)
np.where(condition, a, b)
```

### Statistics
```python
Mean = sum(x)/n | Median = middle value | Std = sqrt(variance)
IQR = Q3 - Q1 | Z = (x-mean)/std
p < 0.05 -> reject H0 (conventional threshold)
```

### Visualization
```python
sns.boxplot / histplot / heatmap / countplot / scatterplot
plt.figure(figsize=(w,h)); plt.title(); plt.show()
```

### SQL <-> Pandas equivalents
| SQL | Pandas |
|---|---|
| `SELECT col FROM t` | `df["col"]` |
| `WHERE` | boolean filter `df[cond]` |
| `GROUP BY ... SUM()` | `df.groupby()[col].sum()` |
| `JOIN` | `pd.merge()` |
| `ORDER BY` | `df.sort_values()` |
| `RANK() OVER()` | `df["col"].rank()` |
| `CASE WHEN` | `np.select()` / `.apply()` |
| `LIMIT n` | `.head(n)` / `.nlargest(n, col)` |
| `DISTINCT` | `.unique()` / `.drop_duplicates()` |

### Common debugging commands
```python
df.columns.tolist(); df.dtypes; df.shape; df.info()
df.columns = df.columns.str.strip()
import os; os.getcwd()
```

### Most commonly asked questions (top 10 to have airtight)
1. loc vs iloc 2. apply vs transform 3. merge vs concat 4. mean vs median (and why choose one) 5. p-value meaning 6. correlation vs causation 7. data leakage 8. RANK vs ROW_NUMBER vs DENSE_RANK 9. groupby aggregation vs `.transform()` row alignment bug (Part 16) 10. how you'd scale pandas code to millions of rows.

### Common mistakes to avoid saying/doing in an interview
- Jumping straight to code before clarifying the business question.
- Reporting a number without stating what it means for the business.
- Confusing an aggregated groupby result with a row-aligned one.
- Claiming correlation implies causation.
- Deleting missing/outlier data without justifying the decision.
- Forgetting to mention how you'd validate/sanity-check a result.

### Interview phrases worth using naturally
- "Before I write code, let me confirm what grain we need this at — per order or per customer?"
- "I'd choose median here because the distribution looks right-skewed."
- "That's correlation, not necessarily causation — here's how I'd test that more rigorously."
- "Let me sanity-check this against a simple baseline before trusting it."
- "Here's the number, and here's what it means for the business."

### 1-Day Revision Sheet
Morning: Part 6-7 (pandas core + 75 Qs skim), Part 10-11 (stats + hypothesis testing). Afternoon: Part 13 (SQL<->Pandas), Part 14-15 (business + dataset round), Part 16-17 (code interpretation + debugging). Evening: Part 24-25 (rapid-fire + level-based Qs), Part 26 (scenario structure), Part 31 (this cheat sheet) — do a full run-through out loud.

### 7-Day Preparation Plan
- **Day 1:** Python fundamentals + data structures (Parts 1-4) — write and run every code example yourself.
- **Day 2:** NumPy + Pandas core (Parts 5-6) — rebuild the 75-question bank (Part 7) from memory.
- **Day 3:** Data cleaning + EDA + statistics (Parts 8-10) — practice explaining outputs in plain business language out loud.
- **Day 4:** Hypothesis testing + visualization + SQL (Parts 11-13) — drill SQL/Pandas equivalents until automatic.
- **Day 5:** Business analytics + real dataset round + code interpretation + debugging (Parts 14-17) — time yourself solving the dataset questions.
- **Day 6:** Optimization + large data + OOP + files + APIs + library map (Parts 18-23) — focus on the "why," not just syntax.
- **Day 7:** Full mock interview (Part 30) end-to-end, timed, followed by the 1-Day Revision Sheet as a final pass.

---

*End of reference. Use Parts 1-23 to learn the concepts deeply, Parts 24-27 as a rapid-fire practice bank, Parts 28-29 to sharpen how you communicate answers, and Part 30-31 to rehearse and revise.*
