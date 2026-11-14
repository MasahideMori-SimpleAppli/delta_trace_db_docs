# Aggregate query (Python)

`aggregate` calculates several aggregations over the items that match an optional condition and
returns them as **one dict** in `result[0]`. `hit_count` is the number of matched items.
It is read-only, can be used in transactions, and needs its own `EnumQueryType.aggregate` permission
on servers (a `search` permission does not allow it). Requires DeltaTraceDB 0.2.0 or later.

```
QueryBuilder.aggregate(target, aggregations, query_node=None, cause=None)   # RawQueryBuilder is the same
```

## Aggregation

`Aggregation(type_, field=None, v_type=EnumValueType.floatStrict_, alias=None, scale=None, rounding=None)`

| Argument | Meaning |
|---|---|
| `type_` | `EnumAggregateType.count_`, `sum_`, `min_`, `max_` or `mean_`. |
| `field` | Target field (dot notation allowed). Required except for `count_`. |
| `v_type` | Calculation type: `floatStrict_` (default), `decimal_` or `rational_`. Other types make the query fail. Ignored by `count_`. |
| `alias` | Key in the result dict. Default: type name + field, e.g. `"sum_price"`; `"count_"` for `count_` without a field. Duplicate keys make the query fail. |
| `scale`, `rounding` | Only for `mean_` with `decimal_`, and required there: digits after the decimal point and `EnumRoundingMode` (`halfEven`, `halfUp`, `down`, `floor`, `ceiling`). |

Rules:

- `count_` without a field counts matched items; with a field it counts items whose value is not `None`.
  It never converts values, so it works for any type.
- `None` / missing values are skipped by every aggregation.
- A value that cannot be converted makes the whole query fail (`is_success` False, reason in `error_message`),
  so a wrong total is never returned silently.
- With no values: `count_` is 0, `sum_` is 0 (`0.0` or `"0"`), and `min_` / `max_` / `mean_` are `None`.

| v_type | Accepts | Result |
|---|---|---|
| `floatStrict_` | int and float (not bool), and plain number strings such as `"1.5"`, `"1e3"`, `".5"` (not `"inf"`, `"NaN"`, `"1_000"`, whitespace) | `float` |
| `decimal_` | `str` and `PortableDecimal` (not numbers) | `str`, e.g. `"43.80"` (sum keeps the largest scale; min / max keep the original string form) |
| `rational_` | `str` (including `"1/3"`), `PortableDecimal`, `PortableRational` (not numbers) | `str`, e.g. `"73/5"` (mean is exact) |

`floatStrict_` adds values left to right in the stored order, so the Dart version of DeltaTraceDB
returns the same float for the same data. This differs from the built-in `sum()`, which uses
compensated summation since Python 3.12.

## Example

```python
from delta_trace_db import *

db = DeltaTraceDatabase()
db.execute_query(RawQueryBuilder.add(target="orders", raw_add_data=[
    {"status": "paid", "price": "10.50", "qty": 2},
    {"status": "paid", "price": "3.3", "qty": 1.5},
    {"status": "paid", "price": "30", "qty": 4},
    {"status": "cancel", "price": "100", "qty": 10},
]).build())
r = db.execute_query(RawQueryBuilder.aggregate(
    target="orders",
    query_node=FieldEquals("status", "paid"),
    aggregations=[
        Aggregation(EnumAggregateType.count_, alias="n"),
        Aggregation(EnumAggregateType.sum_, field="price", v_type=EnumValueType.decimal_, alias="total"),
        Aggregation(EnumAggregateType.mean_, field="price", v_type=EnumValueType.decimal_, scale=2,
                    rounding=EnumRoundingMode.halfEven, alias="avg"),
        Aggregation(EnumAggregateType.max_, field="price", v_type=EnumValueType.decimal_),
        Aggregation(EnumAggregateType.sum_, field="qty"),
    ],
).build())
print(r.is_success)  # True
print(r.result[0])  # {'n': 3, 'total': '43.80', 'avg': '14.60', 'max_price': '30', 'sum_qty': 7.5}
```

Failure example (numbers are not converted by `decimal_`):

```python
bad = db.execute_query(RawQueryBuilder.aggregate(target="orders", aggregations=[
    Aggregation(EnumAggregateType.sum_, field="qty", v_type=EnumValueType.decimal_)]).build())
print(bad.is_success)  # False
print(bad.error_message)  # Cannot convert the value of field 'qty' to decimal_: 2
```
